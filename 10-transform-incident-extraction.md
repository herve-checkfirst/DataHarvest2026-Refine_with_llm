# Transformation: Structured Incident Extraction

Complexity level: advanced. This transformation extracts three structured fields per row — incident type, location, and severity — from reviews that describe a concrete event. It is the natural continuation of `02-transform-factual-incident.md` and demonstrates the **composition pattern** that any reader of this repository should be able to reproduce on their own data: filter with one LLM pass, then extract structure with a second pass on the smaller, cleaner subset.

## What we want to do

Worksheet `03` produces a `Has factual incident` column. Faceting that column to `yes` reduces the dataset to the rows where something concrete happened — typically 30 to 45 percent of the original corpus. On this filtered subset, the next analytical step is to pull out *what* happened, *where*, and *how bad it was*, in a form that can be mapped, charted, and counted.

This worksheet produces three new columns in three sequential passes (the AI Extension cannot emit multiple columns at once):

- `Incident type` — one canonical label from a fixed vocabulary
- `Incident location` — the airport or city where the incident occurred, in the form `City, Country` or `IATA`, or `In flight` for in-air events
- `Incident severity` — `low` / `medium` / `high` / `unknown`

Together, these three columns power the journalism payoff:

- Map of incidents by location and type
- Distribution of severity per airline
- Timeline of incidents per airline if combined with the ISO date column from worksheet `00`

Running these prompts on the unfiltered dataset wastes inference and produces noisy output (the model invents incidents when there were none). Always run worksheet `03` first, facet to `yes`, and run these three only on the filtered rows.

## The prompts

Three separate prompts, run in sequence on the filtered subset. Each one produces one column.

### Pass 1: `Incident type`

In **Add column by AI**, name the new column `Incident type` and paste:

```
You are an aviation incident classifier. The input is an airline review that
describes at least one concrete factual event. Identify the single most
serious incident described and output one label from the fixed vocabulary
below.

Vocabulary (output exactly these strings):
- Delay
- Cancellation
- Diversion
- Emergency landing
- Medical emergency
- Equipment failure
- Cabin depressurization
- Fire or smoke
- Evacuation
- Runway incident
- Lost baggage
- Damaged baggage
- Denied boarding
- Missed connection
- Other
- None

Rules:
1. If multiple incidents are described, pick the most severe. Severity ranking
   high to low: Evacuation, Fire or smoke, Emergency landing, Cabin
   depressurization, Diversion, Medical emergency, Runway incident, Equipment
   failure, Denied boarding, Missed connection, Cancellation, Delay, Damaged
   baggage, Lost baggage.
2. "Diversion" means the flight landed somewhere other than its planned
   destination. "Emergency landing" means an unplanned landing for safety
   reasons, usually at the planned destination or the nearest airport.
3. "Equipment failure" covers in-flight system failures (engine, hydraulics,
   pressurization, entertainment, lights) when called out specifically.
4. "Other" only when the incident is real but does not match the vocabulary.
5. "None" only if no incident is actually described after rereading. This
   should be rare on this filtered subset; flag it for manual review.
6. Output only the single label. No punctuation, no quotes, no explanation.
```

### Pass 2: `Incident location`

In **Add column by AI**, name the new column `Incident location` and paste:

```
You are an aviation incident locator. The input is an airline review that
describes a concrete event. Identify where the incident took place and output
in one of these forms:

- "City, Country"      (when a named city or airport is given)
- "IATA"               (when only a 3-letter airport code is given, e.g. CDG)
- "In flight"          (when the event occurred during the flight, with no
                        specific named location)
- "Unknown"            (when the review describes an event but gives no
                        location detail)
- "N/A"                (input is empty or describes no event)

Rules:
1. Use the location where the *incident* occurred, not the origin or
   destination of the trip. If the flight was diverted to Lisbon, the
   location is "Lisbon, Portugal" — not the planned destination.
2. Resolve IATA codes only if you are confident. Otherwise output the IATA
   code verbatim and let downstream processing resolve it.
3. For events that span multiple locations (delayed at departure, then
   diverted), output the location of the most severe event.
4. Use English city names. Use full country names ("United Kingdom" not
   "UK").
5. Output only the location string. No punctuation, no quotes, no
   explanation.
```

### Pass 3: `Incident severity`

In **Add column by AI**, name the new column `Incident severity` and paste:

```
You are an aviation incident severity rater. The input is an airline review
that describes a concrete event. Rate the severity using this scale:

- low      (inconvenience without safety implications: short delay, lost
            baggage, denied boarding, IFE failure, damaged baggage,
            minor equipment failure)
- medium   (significant disruption or risk: long delay over 4 hours,
            cancellation, missed connection with overnight stay, medical
            emergency handled in flight, diversion to nearby airport)
- high     (safety-critical event: emergency landing, evacuation, fire or
            smoke, cabin depressurization, runway incident, injury,
            hospitalization)
- unknown  (the review describes an event but the severity cannot be
            determined from the text)

Rules:
1. Consider both the nature of the event and its consequences as described.
   A 2-hour delay with no further impact is low; a 2-hour delay that caused
   a missed connection and an overnight stay is medium.
2. Subjective intensity ("worst experience of my life") does not raise
   severity. Only objective consequences do.
3. If multiple incidents are described, rate based on the most severe one.
4. Output only the single label. No punctuation, no quotes, no explanation.
```

Set all three response formats to **Text**. Run them in sequence on the filtered subset.

## Limits of the result

This worksheet is the most ambitious in the suite because it combines three classifications that have to be internally consistent. Expect a 5 to 10 percent error rate per column, and roughly 15 percent of rows will have at least one of the three columns wrong.

**Type-severity inconsistency.** The model can label `Incident type = Lost baggage` and `Incident severity = high` on the same row. The two prompts are run independently and the model has no way to enforce consistency between them. After running all three columns, add a GREL sanity column:

```grel
if(cells["Incident type"].value == "Evacuation" && cells["Incident severity"].value == "low", "inconsistent",
  if(cells["Incident type"].value == "Lost baggage" && cells["Incident severity"].value == "high", "inconsistent", "ok"))
```

Extend the rule list to cover the obvious mismatches. Facet on `inconsistent` and re-read by hand.

**Location confusion between trip endpoints and incident location.** Rule 1 of the location prompt tries to push the model toward the incident location specifically, but it occasionally returns the departure or destination instead. Cross-check: facet `Incident type = Diversion` and `Incident location = trip origin or destination` — these are likely wrong.

**Hallucinated specificity.** For incidents described vaguely, the model sometimes invents a plausible location ("Frankfurt, Germany") that is not in the text. Mitigation: when in doubt, output `Unknown` is preferred — rule 4 of the location prompt was designed to encourage this. Spot-check by reading 30 random `Incident location` values against the original review.

**Severity scale subjectivity.** The low/medium/high boundary is editorial. Different newsrooms will draw it differently. Treat the scale as a starting point and document any tweaks you make to the prompt's definitions.

**The composition pattern itself.** This worksheet is also a teaching example: it shows how to chain LLM transformations to do what no single prompt can. Worksheet `03` is the filter; this one is the extractor. The same pattern applies broadly — first reduce the search space, then extract structure. Trying to extract structure from the full unfiltered dataset wastes time and produces unreliable output because the model has to handle the "no event" case in every row.

**Runtime.** Three passes on the filtered subset. If the filter cuts to 40 percent of 23,000 rows = 9,200 rows, expect three times two hours = six hours total on a CPU laptop. The filter step itself was already six hours, so plan for an overnight run total.

## Model settings and why

The provider configuration in OpenRefine for all three prompts:

| Field | Value |
|---|---|
| Model | `ministral-3:3b` |
| Temperature | `0.1` |
| Top-P | `0.9` |
| Seed | `42` |
| Max Tokens | `16` |
| Wait Time | `0` |

**Temperature 0.1.** Three closed-vocabulary classifications. Pure 0.0 produces brittle decisions on borderline events; 0.1 gives the right amount of flexibility without drift. Same value across the three passes keeps the behavior consistent and the failure modes comparable.

**Top-P 0.9.** Standard tight value, suppressing near-vocabulary variants ("delay (long)", "Frankfurt airport", "moderate") that the model would otherwise produce.

**Seed 42.** Locks reproducibility across all three passes. Critical because the three columns will be cross-joined in downstream analysis — if the seed drifted between runs, the consistency check above would falsely flag good rows as inconsistent.

**Max Tokens 16.** Longer than other classification tasks because `Incident location` can stretch ("Frankfurt, Germany" plus delimiter). 16 tokens fits the longest legitimate output while truncating any attempt at explanation.

**Wait Time 0.** Local inference.

**Model choice.** Ministral 3B handles type and severity reliably. Location resolution is where it slips — it knows major hubs but invents plausible-sounding cities for events described vaguely. For publication-grade incident reporting, swap to `mistral:7b` or `qwen2.5:7b`. Both improve location accuracy by roughly 15 percentage points. For maximum reliability, run location resolution twice with different models and only trust agreement.
