# AI Flight Parser — Agent Design

## Overview

The AI flight parser extracts structured flight data from free-text user input (e.g. "Yesterday JFK to LHR with Delta"). It runs a 3-step pipeline: **Reason → Judge → Extract**.

**Implementation:** `src/composables/useAIFlightParser.js`
**Provider:** [![Built with Pollinations](https://img.shields.io/badge/Built%20with-Pollinations-8a2be2?style=for-the-badge&logo=data:image/svg+xml,%3Csvg%20xmlns%3D%22http://www.w3.org/2000/svg%22%20viewBox%3D%220%200%20124%20124%22%3E%3Ccircle%20cx%3D%2262%22%20cy%3D%2262%22%20r%3D%2262%22%20fill%3D%22%23ffffff%22/%3E%3C/svg%3E&logoColor=white&labelColor=6a0dad)](https://pollinations.ai) — OpenAI-compatible proxy to Gemini
**Framework:** LangChain (`@langchain/openai`, `@langchain/core`)

---

## Pipeline

```
User query
    │
    ▼
┌─────────────┐
│  STEP 1     │  gemini-fast · parametric knowledge only · no search
│  Reason     │  → prose reasoning + OUTPUT block with field values
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  STEP 2     │  gemini-search · Google Search grounding
│  Judge      │  → FIDELITY / ROUTE / FLEET verdicts + sources
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  STEP 3     │  gemini-fast · structured output (Zod schema)
│  Extract    │  → JSON: airlineCode, airports, date, aircraft
└──────┬──────┘
       │
       ▼
  sanitizeFlightData()
  · airport code validation (must exist in airports DB)
  · ICAO → IATA normalization (icao-iata-fallback.json)
  · airline code validation (must exist in airlines DB)
  · aircraft name validation (must match aircraft-types.json exactly)
  · date format validation (DD-MM-YYYY)
```

Budget is consumed after Step 1 succeeds. If Step 2 or 3 fail, the request was still charged.

---

## Models

| Step | Model | Why |
|------|-------|-----|
| Reason | `gemini-fast` | Fast parametric reasoning, no search needed |
| Judge | `gemini-search` | Google Search grounding to verify route/fleet claims |
| Extract | `gemini-fast` | Structured output via LangChain `.withStructuredOutput()` |

**Note on grounding metadata:** `gemini-search` performs real web searches, but LangChain's `ChatOpenAI` wrapper strips non-standard response fields (`groundingMetadata`) since it targets the OpenAI spec only. The search happens; the source URLs are not accessible programmatically through LangChain.

---

## Step 1 — Reasoning Prompt

**Goal:** Reconstruct flight fields from the user's memory using parametric knowledge. No web search.

**Inputs:** user query + today's date (injected at call time via `new Date()`)

```
You are reconstructing a flight record from a user's memory. Use your training knowledge about airlines, their fleets, and typical routes. Do NOT search — reason from what you know.

TODAY'S DATE: {today}

USER QUERY: {userQuery}

Analyse the user query. It may or may not contain explicit information about the flight details.

What are we looking for:
- Departure airport (IATA code)
- Arrival airport (IATA code)
- Airline (IATA code)
- Aircraft type (standard name, e.g. "Boeing 737MAX", "Airbus A320", "Embraer E190")

Perform the next steps:

1. Identify data that cannot be directly extracted from the query.
2. For missing fields use the following logic:

ROUTE: If route is missing: abort. We cannot infer a route that isn't mentioned at all. Output null for all fields.
DATE: What date was provided?
AIRLINE: Was an airline stated in the query? If yes, preserve it exactly. If no, can you infer one from knowledge of which airlines operate in that region? What is the most likely candidate?
AIRCRAFT: What aircraft does this airline typically operate on this route? Name the most likely type. If multiple types are possible, name the most common one.

Think through your reasoning in prose, then end with OUTPUT containing short values only — no sentences, no explanations.

OUTPUT:
  "departureAirportCode": "3-letter IATA code or null",
  "arrivalAirportCode": "3-letter IATA code or null",
  "airlineCode": "2-letter IATA code or null",
  "aircraftName": "standard name e.g. Boeing 777, Airbus A350, or null"
```

---

## Step 2 — Judge Prompt

**Goal:** Verify the reasoning's claims using web search. Assess three dimensions.

**Inputs:** original user query + Step 1 reasoning output

```
You are a flight research assistant. Extract key claims from the reasoning below, then look them up to verify.

USER QUERY:
{userQuery}

AI REASONING:
{reasoningOutput}

STEP 1 — Extract the claims:
From the reasoning above, identify: airline (IATA code), departure airport, arrival airport, aircraft type.

STEP 2 — Look up each fact using web search:
- Search "[airline name] routes [departure] [arrival]" to confirm the airline operates this route
- Search "[airline name] fleet" to confirm this aircraft type is in the airline's fleet

STEP 3 — Check parse fidelity (no search needed):
Did the reasoning preserve any data explicitly stated in the user query (airports, airline, dates)?

Output exactly this format:
FIDELITY: [PASS/FAIL] — [reason]
ROUTE: [PASS/FAIL] — [what search found]
FLEET: [PASS/FAIL] — [what search found]
SOURCES: [comma-separated list of domains consulted, e.g. flightconnections.com, skyscanner.com — or "none" if no search was needed]
```

---

## Step 3 — Extraction Prompt

**Goal:** Produce validated structured JSON using the reasoning and judge verdicts.

**Inputs:** original user query + Step 1 reasoning + Step 2 judge output

```
Extract structured flight data from the reasoning and judge verdicts below.

USER QUERY:
{query}

REASONING:
{reasoning}

JUDGE VERDICTS:
{judgeOutput}

EXTRACTION RULES:
1. Use reasoning as primary source for all fields
2. If FLEET: FAIL → set aircraftName to null
3. If ROUTE: FAIL → set airlineCode, departureAirportCode, arrivalAirportCode to null
4. If FIDELITY: FAIL → override with values explicitly stated in the user query
5. Airport codes: 3-letter IATA only, never city codes (JFK not NYC)
6. Airline codes: 2-letter IATA only (AA, DL, BA — never ICAO like AAL, DAL, BAW)
7. Dates: DD-MM-YYYY format
8. Aircraft: normalize to standard names using mapping below — strip variant suffixes

AIRCRAFT NORMALIZATION:
Boeing 737:  B738/B739/B73J/B737/Boeing 737-800/Boeing 737-900 → Boeing 737NG
             B38M/B39M/Boeing 737 MAX 8/Boeing 737 MAX 9 → Boeing 737MAX
Boeing:      B712 → Boeing 717
             B752/B753/Boeing 757-200 → Boeing 757
             B762/B763/B764/Boeing 767-200/Boeing 767-300 → Boeing 767
             B77W/B77L/B772/B773/Boeing 777-200/Boeing 777-300ER → Boeing 777
             B78X/B788/B789/Boeing 787-8/Boeing 787-9/Boeing 787-10 → Boeing 787
             B744/B748/Boeing 747-400/Boeing 747-8 → Boeing 747
Airbus A320: A320/A20N/Airbus A320-200 → Airbus A320
             A321/A21N/Airbus A321-200/Airbus A321neo → Airbus A321
             A319/A19N/Airbus A319-100 → Airbus A319
             BCS1/BCS3/Airbus A220-100/Airbus A220-300 → Airbus A220
Airbus Wide: A333/A332/A339/A338/Airbus A330-200/Airbus A330-300/Airbus A330-900 → Airbus A330
             A342/A343/A345/A346/Airbus A340-300/Airbus A340-600 → Airbus A340
             A359/A35K/Airbus A350-900/Airbus A350-900ULR/Airbus A350-1000 → Airbus A350
             A388/Airbus A380-800 → Airbus A380
Regional:    E75L/E75S/Embraer E175/Embraer 175 → Embraer E175-E2
             E170/Embraer E170 → Embraer E170
             E190/E290/Embraer E190/Embraer 190 → Embraer E190
             E195/E295/Embraer E195/Embraer 195 → Embraer E195-E2
             CRJ9/CRJ7/CRJ2 → Bombardier CRJ
             DH8D/Dash 8/Q400/Bombardier Q400 → DHC Dash 8
             AT76/AT72/ATR 72/ATR 42 → ATR 42/72

OUTPUT: JSON with exactly these fields (use null for missing/rejected):
{
  "airlineCode": "2-letter IATA or null",
  "departureAirportCode": "3-letter IATA or null",
  "arrivalAirportCode": "3-letter IATA or null",
  "flightDate": "DD-MM-YYYY or null",
  "aircraftName": "normalized full name or null"
}
```

**Output schema (Zod):**
```js
z.object({
  airlineCode:           z.string().nullable(),  // 2-letter IATA
  departureAirportCode:  z.string().nullable(),  // 3-letter IATA
  arrivalAirportCode:    z.string().nullable(),  // 3-letter IATA
  flightDate:            z.string().nullable(),  // DD-MM-YYYY
  aircraftName:          z.string().nullable()   // must match aircraft-types.json exactly
})
```

**Intentionally excluded:** flight number (accuracy too low), flight duration (calculated in-app from schedule data).

---

## Post-processing

After extraction, `sanitizeFlightData()` validates each field against the app's reference data:
- Airports → `useAirportData().findAirportByCode()`
- Airlines → `useAirportData().airlines` — ICAO codes are first normalised to IATA via `public/icao-iata-fallback.json`
- Aircraft → `useAirportData().aircraftTypes` — name must be an exact match
- Date → regex `/^\d{2}-\d{2}-\d{4}$/`

Any field that fails validation is set to null rather than propagating bad data.

---

## App Integration

**Form auto-fill** (`src/pages/AddFlightPage.vue`): when AI data is applied to the flight form, missing flight number and aircraft are automatically marked as "unknown" rather than left blank.

**Raw AI Output panel** (`src/components/AIFlightParserModal.vue`): shows all three agent responses (reasoning, judge verdicts, extracted JSON) in a collapsible section.
