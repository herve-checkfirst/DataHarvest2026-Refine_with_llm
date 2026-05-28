# Transformation: Factual Incident Detection

Complexity level: intermediate. This transformation distinguishes reviews that describe a concrete, verifiable event from reviews that only express subjective opinion. The output is a binary signal that lets you filter the dataset down to incidents worth investigating — useful for journalism, complaint triage, and regulatory monitoring.

## What we want to do

The `Review` column blends two very different kinds of content. Some reviews are pure feeling:

> "The seat was uncomfortable, the food was bland, and I would not fly again."

Others describe verifiable events:

> "Our flight was diverted to Lisbon due to a medical emergency. We sat on the tarmac for three hours before being bussed to a hotel."

Both are legitimate, but they answer different questions. Subjective reviews tell you what customers feel about an airline. Factual reviews tell you what actually happened. A data journalist building a story about flight diversions, a regulator looking for safety incidents, or an analyst tracking lost-baggage events all need to filter to the factual subset.

We extract a single new column `Has factual incident` with three values:

- `yes` — the review describes at least one concrete, dated, or locatable event (delay with duration, diversion, medical emergency, equipment failure, baggage loss with details, named individual incidents)
- `no` — the review is purely opinion, ratings, or generic complaints with no specific event
- `N/A` — the review is empty, unreadable, or not about a flight

This column then powers a facet that cuts the dataset roughly in half and exposes the factual subset for follow-up extraction (incident type, location, date, severity).

## The prompt

In **Add column by AI**, name the new column `Has factual incident` and paste this instruction:

```
You are an incident classifier for airline reviews. Read the review text and
decide whether it describes at least one concrete, verifiable event — as
opposed to pure subjective opinion or generic complaints.

Output exactly one of these three values, nothing else:
- yes   (review describes a specific event: delay with duration, diversion,
         cancellation with cause, medical emergency, equipment failure, named
         baggage loss, fire, evacuation, runway incident, or any other
         concrete factual happening)
- no    (review only expresses opinions, feelings, generic praise, or generic
         complaints with no specific event)
- N/A   (review is empty, unreadable, or not about a flight)

Rules:
1. A concrete event has at least one of: a duration ("six hour delay"), a
   location ("diverted to Lisbon"), a cause ("medical emergency on board"),
   a quantified consequence ("missed connection, rebooked next day"), or a
   specific object ("the entertainment system failed").
2. Generic statements like "the flight was delayed" with no duration or cause
   count as "no". The reviewer must give specifics.
3. Subjective evaluations of objectively true facts still count as "yes" if
   the fact is specific (example: "the rude flight attendant named Sarah
   refused to bring water" -> yes, named individual and refused action).
4. Lost baggage counts as "yes" only if there are concrete details (duration
   of loss, contents, recovery process). "They lost my bags" alone is "no".
5. If the review contains both subjective opinion and a concrete event,
   output "yes". One event is enough.
6. Output only the single value "yes", "no", or "N/A". No punctuation, no
   quotes, no explanation.

Examples:
- Review: "Worst seats ever, terrible food, would not recommend." -> no
- Review: "Six hour delay on outbound, two hour delay on inbound, poorly
  communicated." -> yes
- Review: "Flight diverted to Lisbon after a passenger had a medical
  emergency. We landed safely and resumed two hours later." -> yes
- Review: "The crew were unfriendly and the cabin felt cramped." -> no
- Review: "Entertainment system did not work on outbound Jan 7th and still
  was broken on Feb 10th return flight." -> yes
- Review: "" -> N/A
```

Set the response format to **Text**.

## Limits of the result

The objective-versus-subjective boundary is genuinely fuzzy and a 3B model will draw it inconsistently. Expect a 5 to 8 percent error rate.

**Vague complaints over-classified as factual.** Reviewers who write "they lost my bags and it was a nightmare" often get labeled `yes` even though they gave no specifics. Rule 4 tries to prevent this but the model sometimes counts the event itself (loss) as enough. Mitigation: re-read all `yes` rows that are shorter than 200 characters — most false positives live there.

**Specific events under-classified as subjective.** Long emotional reviews that mention a real incident in passing ("after the diversion in Lisbon, the crew were rude to me") sometimes get labeled `no` because the model fixates on the emotional content. Spot-check long `no` rows for buried incidents if your use case is incident hunting.

**Named individuals.** Rule 3 says a named flight attendant counts as factual, but this is debatable — the reviewer's account is unverifiable. If you are doing journalism, treat "yes with named individual" as a separate, weaker tier. Use a follow-up GREL to flag those rows:

```grel
cells["Has factual incident"].value == "yes" && value.find(/\b(named|attendant|pilot|captain|crew member) [A-Z][a-z]+/) != null ? "named-individual" : ""
```

**Format violations.** Around 2 percent of outputs come back as `Yes`, `yes.`, `Yes - delay mentioned`, or similar. Add a GREL guard:

```grel
value.trim().toLowercase().startsWith("yes") ? "yes" :
value.trim().toLowercase().startsWith("no") ? "no" :
value.trim().toLowercase().contains("n/a") ? "N/A" : "MALFORMED"
```

**Use as a filter, not a measurement.** This column is a triage tool. It tells you where to look for incidents; it does not measure how many incidents an airline had. Always read the `yes` rows in full before drawing conclusions.

**Chain with downstream extractions.** Once you have the `yes` subset, run a second LLM pass on just those rows to extract structured incident data (type, location, duration). This is much cheaper than running the extraction on the full dataset and produces cleaner output because the prompt does not have to handle the "no event" case.

**Runtime.** One pass at roughly one second per row on the full 23,000-row dataset is about six hours of inference on a laptop. Test on the 100-row sample first.

## Model settings and why

The provider configuration in OpenRefine for this transformation:

| Field | Value |
|---|---|
| Model | `ministral-3:3b` |
| Temperature | `0.1` |
| Top-P | `0.9` |
| Seed | `42` |
| Max Tokens | `8` |
| Wait Time | `0` |

**Temperature 0.1.** Binary classification with a fuzzy boundary. The model needs a small amount of flexibility to weigh the specificity of a description, but no creative output. Pure 0.0 makes the model lock onto whichever side of the boundary the first specific noun pushes it toward, which is fragile on borderline cases. 0.1 lets it consider the full review before deciding.

**Top-P 0.9.** Standard tight value. Suppresses tempting label variants ("partially", "unclear", "maybe") that the model is drawn to on ambiguous reviews — those would violate rule 6.

**Seed 42.** Locks reproducibility. Important here because the labels you get drive the downstream filter; if the seed drifted between runs, the `yes` subset would change row-to-row and any follow-up extraction would target a moving population.

**Max Tokens 8.** Output is at most three characters ("yes", "no", "N/A"). Capping at 8 stops the model from appending the reasoning it is strongly tempted to attach ("yes - they describe a six hour delay"), which would always violate rule 6.

**Wait Time 0.** Local inference.

**Model choice.** Ministral 3B handles the binary decision reliably for clear-cut cases. The fuzzy middle (vague references to events, emotional descriptions of real incidents) is where it slips. For a research-grade or journalism-grade filter, swap to `mistral:7b` — the prompt and settings stay identical, error rate drops by roughly half, runtime triples.
