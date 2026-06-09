# Transformation: Named Entity Extraction

Complexity level: advanced. This transformation reads the free-text `Review` column and extracts **named entities** from it — airlines, airports, cities, aircraft types, and dates — as structured JSON. Unlike the closed-vocabulary classification tasks elsewhere in this suite, the number of entities per row is variable, which forces a JSON output rather than a single label.

## What we want to do

The `Review` column holds 150 to 2500 characters of free narrative, in several languages. Usable entities sit buried in the prose: "*Flew Lufthansa from Frankfurt to JFK on an A380 last December, crew were great*". The goal is to produce an `Entities` column in JSON that groups these entities by type, so you can subsequently:

- count the most frequently mentioned airports, independent of the often-incomplete `Route` column,
- cross-check the aircraft types named in the text against the `Aircraft` column,
- feed a graph (Gephi) linking airlines, cities, and aircraft.

The central challenge is twofold. First, **the number of entities varies** from row to row: zero for a purely emotional review, a dozen for a detailed account. That is why we emit a JSON object with fixed keys but variable-length arrays, rather than a label. Second, **the line between extraction and invention is thin**: a 3B model is tempted to "complete" a plausible city or to normalize an IATA code it thinks it recognizes. The prompt rules are written to hold it back.

Target output schema:

```json
{
  "airlines": ["Lufthansa"],
  "airports": ["Frankfurt", "JFK"],
  "cities": ["Frankfurt", "New York"],
  "aircraft": ["A380"],
  "dates": ["December"]
}
```

## The prompt

Configure the AI provider in OpenRefine, then in **Add column by AI** name the new column `Entities` and paste the following instruction:

```
You are a named-entity extractor for airline reviews. Read the review text and
extract every named entity you find, grouped by type, then output a single JSON
object. Extract only entities that appear explicitly in the text. Do not infer,
normalize, or invent anything.

Output a JSON object with exactly these five keys, in this order:
{
  "airlines": [],
  "airports": [],
  "cities": [],
  "aircraft": [],
  "dates": []
}

Definitions:
- "airlines": names of carriers (e.g. "Lufthansa", "Air France", "Ryanair").
- "airports": airport names or IATA/ICAO codes that appear as written
  (e.g. "Heathrow", "JFK", "CDG"). Keep them exactly as in the text.
- "cities": city or country names used as origin, destination, or transit
  (e.g. "Frankfurt", "New York", "Dubai").
- "aircraft": aircraft types or models mentioned (e.g. "A380", "Boeing 777",
  "737 MAX").
- "dates": any date or time expression naming a month, season, or year
  (e.g. "December", "summer 2023", "last March"). Copy the surface form.

Rules:
1. Extract surface forms exactly as they appear. Do not translate, do not
   expand IATA codes into city names, do not add a country to a city.
2. A token can legitimately appear in two lists. "Frankfurt" is both an airport
   reference and a city if the text uses it both ways; otherwise place it where
   the text uses it.
3. Deduplicate within each list. If "Lufthansa" appears three times, list it
   once.
4. If a type has no entity, output an empty array for that key. Never omit a
   key.
5. Do not extract generic words ("the airline", "the plane", "the airport")
   that are not proper names.
6. Output only the JSON object. No markdown fences, no comments, no text before
   or after.

Examples:
- Review: "Flew Lufthansa from Frankfurt to JFK on an A380 last December, crew
  were great."
  -> {"airlines":["Lufthansa"],"airports":["Frankfurt","JFK"],"cities":["Frankfurt"],"aircraft":["A380"],"dates":["last December"]}
- Review: "Worst experience of my life, never again."
  -> {"airlines":[],"airports":[],"cities":[],"aircraft":[],"dates":[]}
- Review: "BA flight LHR-DXB on a 777, then Emirates onward to Sydney in
  summer 2023."
  -> {"airlines":["BA","Emirates"],"airports":["LHR","DXB"],"cities":["Sydney"],"aircraft":["777"],"dates":["summer 2023"]}
```

Set the response format to **JSON Object** (not **Text**). The **JSON Object** format tells the extension to expect a JSON object, which reduces outputs wrapped in stray text or markdown fences.

## Limits of the result

Open-ended entity extraction is markedly harder for a 3B model than closed-vocabulary classification. Plan to verify the output before relying on it.

**Invented entities (hallucinations).** This is the primary risk. The model sometimes adds a plausible city absent from the text, or "corrects" an IATA code into a major city it thinks it recognizes. Rule 1 limits this behavior but does not eliminate it. Read 30 random rows, comparing each entity against the source text; if more than 3 contain a ghost entity, move to a larger model.

**Airport/city noise.** The boundary between `airports` and `cities` is fuzzy in the text as well as in the model's head. "Frankfurt" lands sometimes in both lists, sometimes in only one depending on its mood. Do not treat these two lists as watertight: for geographic analysis, merge them and then deduplicate downstream.

**Malformed JSON output.** Even with the **JSON Object** format, roughly 2 to 5 percent of rows come back with a trailing comma, a missing key, or an unescaped quote pulled from the review text. Add a GREL guard column that attempts the parse:

```grel
value.parseJson().airlines.length() >= 0 ? "OK" : "MALFORMED"
```

If `parseJson()` fails, GREL returns an error you can turn into `MALFORMED`. Facet on `MALFORMED` and re-run those rows individually or fix them by hand.

**Exploding into usable columns.** The raw JSON is not directly analyzable. Once the output is validated, explode each type into its own column with GREL. For example, for airports:

```grel
value.parseJson().airports.join("; ")
```

Repeat for `airlines`, `cities`, `aircraft`, `dates`. You get five multi-valued columns, ready for **Split multi-valued cells** if you want one row per entity.

**Cross-checking against the structured columns.** Compare `aircraft` extracted from the text with the dataset's `Aircraft` column, and `airports`/`cities` with the `Route` column. Divergences are either model errors or interesting cases where the narrative contradicts the metadata (flight booked on a 777, flown on an A330 according to the passenger). A consistency column:

```grel
if(cells["Entities"].value.parseJson().aircraft.length() == 0, "no-mention",
  if(cells["Entities"].value.parseJson().aircraft.join(" ").toLowercase().contains(cells["Aircraft"].value.toLowercase()), "match", "divergent"))
```

**Multiple languages.** The dataset contains reviews in several languages. Ministral 3B extracts Latin-script proper names correctly regardless of the surrounding language, but loses recall on non-Latin scripts (Cyrillic, CJK). If your corpus is heavily multilingual, run worksheet `03` (language detection) first and consider a larger model for non-English rows.

**Runtime.** JSON output is longer than a label, so slower to generate. Count on 1.5 to 2 seconds per row, roughly eight to ten hours on the full 23,000 rows on CPU. Iterate on the 100-row sample first, then run overnight.

## Model settings and why

The provider configuration in OpenRefine for this transformation:

| Field | Value |
|---|---|
| Model | `ministral-3:3b` |
| Temperature | `0.1` |
| Top-P | `0.9` |
| Seed | `42` |
| Max Tokens | `256` |
| Wait Time | `0` |

**Temperature 0.1.** Extraction has one correct answer per row: the entities present in the text, and no others. Low temperature concentrates the probability mass on faithful copying of the source text and discourages the "creativity" that produces invented entities. The default 0.7 would sharply raise the hallucination rate.

**Top-P 0.9.** Slightly tighter than the conversational default. Combined with low temperature, it cuts off the long tail of unlikely tokens — alternate spellings, translations, neighboring IATA codes — without making the output fully deterministic at the token level.

**Seed 42.** Fixes the sampling RNG so re-running the column produces identical JSON. Essential while tuning the prompt: without a seed you cannot tell whether an output change came from your edit or from random variation. It is also what makes the transformation defensible in a published investigation.

**Max Tokens 256.** JSON output can stretch on an entity-rich review (multiple stopovers, multiple aircraft). 256 tokens comfortably covers the densest case while truncating any attempt at prose explanation if the model ignores rule 6. If you see truncated JSON (missing closing brace) on the longest reviews, raise it to 384.

**Wait Time 0.** Local inference, no rate limit.

**Model choice.** Ministral 3B is the functional floor here: it extracts well-known airlines and aircraft correctly, but hallucinates more on regional cities and airport codes than it does on a closed classification task. Entity extraction is precisely the kind of task where scaling up pays the most: `mistral:7b` or `qwen2.5:7b` noticeably reduce hallucinations and improve recall on non-English entities, at identical prompt and settings. For publication-grade use, run the extraction twice with two models and keep only the entities both agree on.
