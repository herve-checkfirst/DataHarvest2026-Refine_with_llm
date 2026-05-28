# Transformation: Flight Type Classification (Direct vs Connecting)

Complexity level: basic. This is a binary classification task — the model only has to decide whether the route mentions a stop or not. It is a good first transformation to demonstrate the AI Extension because the output space is tiny, errors are easy to spot, and a small local model handles it reliably.

## What we want to do

The `Route` column contains free-text itineraries in mixed formats: IATA trigrams, full city names, varied separators. We want a new column `Flight Type` with exactly one of two values:

- `Direct` — a single leg from origin to destination, no intermediate stop
- `Connecting` — at least one transit between origin and destination

Empty or unreadable inputs return `N/A` so they can be facet-filtered out.

This column then powers a quick facet in OpenRefine to compare ratings on direct versus connecting flights, or to filter the dataset before more expensive downstream transformations.

## The prompt

In **Add column by AI**, name the new column `Flight Type` and paste this instruction:

```
You are an aviation routing classifier. Read the input route string and decide
whether it describes a direct flight or a connecting flight.

Output exactly one of these three values, nothing else:
- Direct       (no intermediate stop between origin and destination)
- Connecting   (one or more transits, layovers, or stops between origin and
                destination)
- N/A          (input is empty, unreadable, or not a flight route)

Rules:
1. Inputs may use city names, airport names, or IATA 3-letter codes.
2. Separators vary: "to", "-", "via", arrow, slash, comma. Treat them all as
   leg boundaries.
3. The keyword "via" always implies a connecting flight.
4. Two locations only -> Direct. Three or more locations -> Connecting.
5. Output only the single word "Direct", "Connecting", or "N/A". No
   punctuation, no quotes, no explanation.

Examples:
- Input: "CDG to JFK"                    -> Direct
- Input: "LHR-SIN via DXB"               -> Connecting
- Input: "London to Dubai to Sydney"     -> Connecting
- Input: "Manchester to Faro"            -> Direct
- Input: "BKK / HKG / LAX"               -> Connecting
- Input: ""                              -> N/A
```

Set the response format to **Text**.

## Limits of the result

The error surface is narrow but not zero. Watch for these cases.

**Round trips written as a single string.** Inputs like `LHR to JFK to LHR` are technically two direct flights stitched together, not a connecting itinerary. The model will label them `Connecting` because it counts three tokens. If your dataset has many round trips, add a rule to the prompt that says "if the first and last location are identical, treat it as Direct".

**Ambiguous separators.** Some reviewers write `London, Dubai` to mean a single direct flight to a city listed with its country, not a connecting flight via Dubai. The model has no way to tell. Expect a small false-positive rate for `Connecting` on comma-separated inputs.

**Format violations.** Even on a binary task, around 1 percent of outputs will arrive with a trailing period, a leading "Answer: ", or a lowercase variant. Add a GREL guard to canonicalize:

```grel
value.trim().toLowercase().contains("connect") ? "Connecting" :
value.trim().toLowercase().contains("direct") ? "Direct" :
value.trim().toLowercase().contains("n/a") ? "N/A" : "MALFORMED"
```

Facet on `MALFORMED` and fix the few outliers by hand.

**Non-route text.** Reviewers sometimes write things like `weekend getaway` or `business trip` in the Route field. The model should return `N/A` for these, but occasionally returns `Direct` by default. The guard column above will not catch this — only manual review will.

## Model settings and why

The provider configuration in OpenRefine for this transformation:

| Field | Value |
|---|---|
| Model | `ministral-3:3b` |
| Temperature | `0.0` |
| Top-P | `0.9` |
| Seed | `42` |
| Max Tokens | `8` |
| Wait Time | `0` |

**Temperature 0.0.** Binary classification has one correct answer per input. Setting temperature to zero forces the model to always pick the most likely next token, which on this task means the same word every time for the same input. There is no creative judgment to preserve.

**Top-P 0.9.** Mostly redundant when temperature is 0, but kept at the standard value as a safety net in case the inference backend interprets temperature 0 with a small floor. No reason to constrain further.

**Seed 42.** With temperature 0 the seed is theoretically unnecessary, but keeping it makes results bit-for-bit reproducible across runs and across machines, which matters when you compare prompt variants during a presentation or a workshop.

**Max Tokens 8.** The output is always one short word: `Direct`, `Connecting`, or `N/A`. Capping at 8 tokens cuts off any attempt by the model to add an explanation, halves latency on misbehaving rows, and gives a clear signal in logs when a row would have rambled. Going lower (4) risks truncating `Connecting`.

**Wait Time 0.** Local inference. Raise only if Ollama starts queueing under load.

**Model choice.** Ministral 3B is overkill in capability but right in latency. For 23 000 rows this is the cheapest model that gives consistent results. A heavier model (`mistral:7b`) brings no accuracy gain on a task this simple and roughly triples the runtime.
