# AI Flight Parser — Agent Design

## Overview

The AI flight parser extracts structured flight data from free-text user input (e.g. "Yesterday JFK to LHR with Delta"). It runs a 3-step pipeline: **Reason → Judge → Extract**.

**Implementation:** `src/composables/useAIFlightParser.js`
**Provider:** [Pollinations.ai](https://pollinations.ai) (OpenAI-compatible proxy to Gemini)
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

**Inputs:** user query, today's date (injected at runtime as `6 March 2026`)

**Logic:**
- If route is not mentioned at all → abort, return nulls
- Airline: preserve if stated, otherwise infer from regional knowledge
- Aircraft: infer most common type for this airline on this route
- Date: resolve relative dates (e.g. "yesterday") using injected today's date

**Output format:** Prose reasoning followed by a structured `OUTPUT` block with short values (IATA codes, standard aircraft names).

---

## Step 2 — Judge Prompt

**Goal:** Verify the reasoning's claims using web search. Assess three dimensions:

| Check | Method | Verdict |
|-------|--------|---------|
| FIDELITY | No search — did reasoning preserve explicit user data? | PASS/FAIL |
| ROUTE | Search `[airline] routes [dep] [arr]` | PASS/FAIL |
| FLEET | Search `[airline] fleet` | PASS/FAIL |

**Output format:**
```
FIDELITY: PASS — [reason]
ROUTE: PASS — [what search found]
FLEET: FAIL — [what search found]
SOURCES: flightconnections.com, skyscanner.com
```

---

## Step 3 — Extraction Prompt

**Goal:** Produce validated structured JSON using the reasoning and judge verdicts.

**Extraction rules:**
1. Use reasoning as primary source for all fields
2. `FLEET: FAIL` → set `aircraftName` to null
3. `ROUTE: FAIL` → set `airlineCode`, `departureAirportCode`, `arrivalAirportCode` to null
4. `FIDELITY: FAIL` → override with values explicitly stated in the user query
5. Airport codes: 3-letter IATA only (JFK not NYC)
6. Airline codes: 2-letter IATA only (never ICAO like BAW, EZY)
7. Dates: DD-MM-YYYY
8. Aircraft: normalize to canonical names (see normalization table in source)

**Output schema (Zod):**
```js
{
  airlineCode:           string | null,  // 2-letter IATA
  departureAirportCode:  string | null,  // 3-letter IATA
  arrivalAirportCode:    string | null,  // 3-letter IATA
  flightDate:            string | null,  // DD-MM-YYYY
  aircraftName:          string | null   // must match aircraft-types.json exactly
}
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
