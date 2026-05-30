# Transformation: Route Extraction with City and Country Resolution

Complexity level: advanced. This transformation asks the LLM to parse a free-text route string, resolve IATA codes to cities, and emit a structured multi-segment output. It combines parsing, world knowledge, and strict format compliance — three things small local models often struggle to do simultaneously.

## What we want to do

The `Route` column in the airline reviews dataset is unstructured. Across rows it appears as:

- IATA trigrams: `CDG to JFK`, `LHR-SIN via DXB`
- City names: `London to Dubai to Sydney`
- Mixed separators: `to`, `-`, `via`, `→`, `/`, commas
- Direct flights or one-to-many transits
- Occasionally empty, unreadable, or unrelated text

We want a single new column with a canonical format that downstream tools (GIS, Gephi, BI) can consume without further parsing:

```
departure_city, country | transit_city, country | arrival_city, country
```

Direct flights collapse to two segments. Multi-leg routes expand to N segments. Unresolved tokens become `UNKNOWN`. Non-route inputs become `N/A`.

## The prompt

Configure the AI provider in OpenRefine, then in **Add column by AI** paste the following as the instruction:

```
You are an aviation routing expert. From the input route string, identify the
departure, transit (if any), and arrival airports, then output them in this
exact format:

  departure_city, country | transit_city, country | arrival_city, country

Rules:
1. Inputs may be city names, airport names, or IATA 3-letter codes (e.g. CDG,
   JFK, DXB). Resolve every code to its city and country.
2. If the route has no transit (direct flight), output only:
     departure_city, country | arrival_city, country
3. If the route has multiple transits, list each transit between pipes in order.
4. Use English city names and full country names (e.g. "Paris, France" not
   "Paris, FR").
5. Separators in the input vary (to, -, via, arrow, slash, comma). Parse them
   all.
6. If a token cannot be resolved with confidence, output "UNKNOWN" in its place.
   Do not guess.
7. If the entire input is empty, unreadable, or not a flight route, output
   exactly: N/A
8. Output only the formatted string. No explanation, no quotes, no leading or
   trailing whitespace.

Examples:
- Input: "CDG to JFK"            -> Paris, France | New York, United States
- Input: "London to Dubai to Sydney" -> London, United Kingdom | Dubai, United Arab Emirates | Sydney, Australia
- Input: "LHR-SIN via DXB"       -> London, United Kingdom | Dubai, United Arab Emirates | Singapore, Singapore
- Input: "Manchester to Faro"    -> Manchester, United Kingdom | Faro, Portugal
```

Set the response format to **Text**.

## Limits of the result

A 3B parameter model running locally has real ceilings on this task. Expect the following failure modes and budget time to verify them on your output.

**Hallucinated IATA codes.** Ministral 3B knows the major hubs (CDG, JFK, LHR, DXB, SIN, HKG) accurately. It will invent plausible-but-wrong cities for regional codes (think domestic Brazilian or Indonesian airports). Rule 6 forces an `UNKNOWN` rather than a confident invention, but the model still guesses occasionally. Spot-check at least 10 percent of resolved trigrams against a reference IATA table.

**Country naming drift.** Despite rule 4, you will see "USA", "U.S.", "United States of America", and "United States" mixed in the same column. Plan to normalize country names downstream with a `value.replace()` chain or a Reconcile step against Wikidata.

**Multi-leg routes beyond two transits.** Performance degrades sharply once the input lists three or more transits. The model tends to drop the middle segments or merge them. If your data contains heavy multi-stop itineraries, run a second pass on the malformed rows with a larger model (`mistral:7b` or `qwen2.5:7b`).

**Ambiguous city names.** "Birmingham" resolves to the UK by default, even when the review context makes the US city obvious. The transformation has no access to surrounding columns, so it cannot disambiguate. Accept this or write a follow-up prompt that takes the full review text as context.

**Output format violations.** Even with rule 8, roughly 1 to 3 percent of outputs will include a leading "Route:" prefix or trailing punctuation. Add a GREL guard column to flag malformed rows:

```grel
value.match(/^([A-Za-z .'-]+, [A-Za-z .'-]+)(\s\|\s[A-Za-z .'-]+, [A-Za-z .'-]+){1,4}$|^N\/A$/) != null ? "OK" : "MALFORMED"
```

Facet on `MALFORMED`, re-run those rows individually, or fix by hand.

## Model settings and why

The provider configuration in OpenRefine should look like this for this specific transformation:

| Field | Value |
|---|---|
| Model | `ministral-3:3b` |
| Temperature | `0.1` |
| Top-P | `0.9` |
| Seed | `42` |
| Max Tokens | `256` |
| Wait Time | `0` |

**Temperature 0.1.** This is an extraction task with one correct answer per input. Low temperature concentrates the probability mass on the most likely next token, which suppresses creative paraphrasing of city names and minimizes format drift. The default 0.7 would cause the model to occasionally rewrite "United Kingdom" as "the UK" or insert filler words, both of which break downstream parsing.

**Top-P 0.9.** Slightly tighter than the conversational default of 0.95. Combined with low temperature it cuts off the long tail of unlikely tokens (typos, foreign-language city names, alternate spellings) without making the output fully deterministic at the token level. Going lower (0.7 or below) starts forcing the model into repetitive outputs when two cities have similar embeddings.

**Seed 42.** Fixes the sampling RNG so re-running the column produces identical results. Essential when you iterate on the prompt: without a seed you cannot tell whether a change in output came from your edit or from random variation. Pick any integer and stick with it during prompt tuning, then remove the seed only if you want diversity in production.

**Max Tokens 256.** Each output is at most a few hundred characters. Capping at 256 tokens prevents the model from running away into an explanation paragraph if it ignores rule 8, and shortens latency on rows where the model would otherwise ramble before stopping.

**Wait Time 0.** Local inference, no rate limit. Only raise this if you observe Ollama queueing requests under load (visible in `ollama ps` and as growing latency in the OpenRefine progress bar).

**Model choice.** Ministral 3B is the floor of what works here. It handles the major IATA codes and the format constraints, but accuracy on regional airports is mediocre. If accuracy matters more than local-only constraints, swap to `mistral:7b` or `qwen2.5:7b` — the prompt and settings stay identical.
