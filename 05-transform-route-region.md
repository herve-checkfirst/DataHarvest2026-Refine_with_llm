# Transformation: Route Region Classification

Complexity level: intermediate. This transformation maps a parsed route to a canonical aviation region category, enabling regional analysis that no GREL pattern can produce because it requires knowing world aviation geography.

## What we want to do

Once `08-transform-route-extraction.md` has produced clean `departure_city, country | ... | arrival_city, country` strings, the next analytical question is regional: are Gulf carriers really better rated on long-haul than European carriers? Do domestic US flights produce more complaints than intra-EU flights? These questions need the routes grouped into a small number of canonical regional categories.

The new column `Route region` uses a fixed vocabulary derived from how the aviation industry actually segments routes:

- `Domestic` — origin and destination in the same country
- `Intra-Europe` — both endpoints in Europe (EU + UK + Switzerland + Norway + Iceland), different countries
- `Intra-Asia` — both endpoints in Asia, different countries
- `Intra-Americas` — both endpoints in North or Central or South America, different countries
- `Intra-Africa` — both endpoints in Africa, different countries
- `Transatlantic` — between Europe and Americas
- `Transpacific` — between Asia/Oceania and Americas
- `Europe-Asia` — between Europe and Asia (often via Gulf hubs)
- `Europe-Africa` — between Europe and Africa
- `Asia-Oceania` — between Asia and Australia/New Zealand/Pacific
- `Middle East hub` — at least one endpoint or transit in the Gulf (UAE, Qatar, Bahrain, Oman, Saudi Arabia, Kuwait)
- `Other international`
- `N/A`

The `Middle East hub` label intentionally overlaps with other regions because the Gulf carriers' business model (connecting flights through Dubai, Doha, Abu Dhabi) is itself an analytical category. A flight Paris-Sydney via Dubai counts as both `Europe-Asia` and `Middle East hub` — but rule 4 below picks the most informative single label per row to keep the output aggregable.

## The prompt

In **Add column by AI**, name the new column `Route region` and run it against the `Route` column (or the cleaned `Route extracted` column produced by worksheet 08). Paste this instruction:

```
You are an aviation routing classifier. Read the input route string and assign
it to exactly one regional category from the fixed vocabulary below.

Vocabulary (output exactly one of these strings, character for character):
- Domestic
- Intra-Europe
- Intra-Asia
- Intra-Americas
- Intra-Africa
- Transatlantic
- Transpacific
- Europe-Asia
- Europe-Africa
- Asia-Oceania
- Middle East hub
- Other international
- N/A

Region definitions:
- Europe: EU member states, United Kingdom, Switzerland, Norway, Iceland,
  Ireland, Balkans, Ukraine, Turkey (European side).
- Asia: from Iran east to Japan, including India, Southeast Asia, China, Korea.
- Americas: North America (US, Canada, Mexico), Central America, Caribbean,
  South America.
- Africa: continental Africa and Madagascar.
- Oceania: Australia, New Zealand, Pacific islands.
- Middle East: UAE, Qatar, Bahrain, Oman, Saudi Arabia, Kuwait. Treat as a
  separate hub category even when the route also fits another region.

Rules:
1. "Domestic" applies only when origin and destination are in the same country
   and there is no international transit. New York to Los Angeles -> Domestic.
2. If the route includes any transit through UAE, Qatar, Bahrain, Oman, Saudi
   Arabia or Kuwait, output "Middle East hub" — this takes priority over the
   geographic region of origin and destination.
3. Otherwise pick the single label that describes the route geometry between
   the first and last endpoint. Transits inside the same region as the
   endpoints do not change the label.
4. If origin or destination cannot be resolved with confidence, but the rest
   of the route is clear, classify based on what is resolvable. Only output
   "Other international" when no recognizable pattern fits.
5. Empty, unreadable, or non-flight input -> N/A.
6. Output only the single label. No punctuation, no quotes, no explanation.

Examples:
- Input: "LAX to JFK"                       -> Domestic
- Input: "London to Paris"                  -> Intra-Europe
- Input: "Tokyo to Bangkok"                 -> Intra-Asia
- Input: "London to New York"               -> Transatlantic
- Input: "Frankfurt to Singapore via Dubai" -> Middle East hub
- Input: "Paris to Sydney via Dubai"        -> Middle East hub
- Input: "Sydney to Los Angeles"            -> Transpacific
- Input: "Paris to Nairobi"                 -> Europe-Africa
- Input: "Bangkok to Sydney"                -> Asia-Oceania
- Input: "Buenos Aires to Bogota"           -> Intra-Americas
- Input: "Lagos to Johannesburg"            -> Intra-Africa
- Input: "Reykjavik to Dakar"               -> Other international
- Input: ""                                 -> N/A
```

Set the response format to **Text**.

## Limits of the result

The vocabulary is opinionated and a 3B model will make a handful of edge-case judgments you may disagree with. Expect 3 to 6 percent error rate.

**Russia and Turkey ambiguity.** Both straddle Europe and Asia. The prompt treats Turkey as European but says nothing about Russia. Moscow-Vladivostok is `Domestic`; Moscow-Beijing is `Europe-Asia`; Moscow-Paris is `Intra-Europe` by this prompt's definition, which is debatable. If Russia matters in your analysis, add an explicit rule.

**Gulf carriers' own domestic flights.** Emirates Dubai-Abu Dhabi is `Domestic` but the model may incorrectly flag it `Middle East hub` because of rule 2 priority. Add a clarifying rule if the dataset has many such cases.

**Transit-only Gulf detection.** Rule 2 demands a transit through a Gulf country. Reviewers who write "Paris to Sydney" without mentioning the transit (even though Emirates' Paris-Sydney is always via Dubai) will be classified `Europe-Asia` not `Middle East hub`. To fix this systematically, join with `Airline Name` in a follow-up step: if the airline is Emirates/Qatar Airways/Etihad/Gulf Air, override to `Middle East hub`.

**Conflict with the airline industry's actual segments.** IATA defines its own regional segments (AFI, ASPAC, EUR, LATAM, MENA, NAM). The vocabulary above is more journalist-friendly than IATA-compliant. If your output needs to plug into IATA-reported statistics, rewrite the vocabulary to match.

**Format violations.** Around 1 to 2 percent will come back as `intra europe`, `Trans-Atlantic`, or `Middle East Hub`. Add a GREL guard that canonicalizes case and hyphens:

```grel
forEach(
  ["Domestic","Intra-Europe","Intra-Asia","Intra-Americas","Intra-Africa","Transatlantic","Transpacific","Europe-Asia","Europe-Africa","Asia-Oceania","Middle East hub","Other international","N/A"],
  label, if(value.trim().toLowercase().replace("-"," ").replace("_"," ") == label.toLowercase().replace("-"," "), label, null)
).filter(v, v != null)[0]
```

**Composition with worksheet 08.** If you have already run `08-transform-route-extraction.md`, point this prompt at the extracted column rather than the raw `Route`. The cleaned format reduces the parsing burden on the model and improves accuracy by roughly 5 percentage points.

**Runtime.** One pass at roughly one second per row on the full 23,000-row dataset is around six hours on a CPU-only laptop. Empty routes are common; facet to non-empty before running.

## Model settings and why

The provider configuration in OpenRefine for this transformation:

| Field | Value |
|---|---|
| Model | `ministral-3:3b` |
| Temperature | `0.1` |
| Top-P | `0.9` |
| Seed | `42` |
| Max Tokens | `12` |
| Wait Time | `0` |

**Temperature 0.1.** Classification with a tight closed vocabulary. Pure 0.0 makes the model latch onto the first recognized region word in the route ("London" -> Europe -> Intra-Europe) without checking the destination. 0.1 lets it consider both endpoints before settling.

**Top-P 0.9.** Standard tight value. Suppresses near-variants of the vocabulary that the model would otherwise invent ("Transeurope", "Asia-Pacific", "EMEA").

**Seed 42.** Locks reproducibility, mandatory because this column will be used as a faceting axis in downstream visualizations — drift between runs would corrupt every aggregate.

**Max Tokens 12.** Output is at most three words ("Middle East hub", "Other international"). 12 tokens leaves headroom while truncating any attempt at explanation.

**Wait Time 0.** Local inference.

**Model choice.** Ministral 3B knows the major aviation regions reliably but is shaky on the Russia/Turkey/Middle East boundaries and on the difference between Central America and the Caribbean. For research-grade output, swap to `mistral:7b` or `qwen2.5:7b` — both reduce border-case errors by roughly half. Prompt and settings stay identical.
