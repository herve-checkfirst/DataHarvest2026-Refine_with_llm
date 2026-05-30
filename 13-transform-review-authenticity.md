# Transformation: Review Authenticity Signal

Complexity level: advanced. This transformation flags reviews that show patterns commonly associated with non-genuine content: generic praise without specifics, marketing-flavored vocabulary, anomalous length-to-rating ratios, suspicious uniformity. It is the most journalism-oriented worksheet in the suite.

This fiche carries a warning: the output is a **signal**, not a verdict. The LLM does not know whether any individual review is authentic. It can only highlight surface patterns. Treat the column as a triage tool that points where to read, never as proof.

## What we want to do

Online airline reviews are a known target for manipulation — both positive (paid reviews boosting reputation) and negative (competitor smearing, organized customer revenge campaigns). The Kaggle dataset includes a `Verified` column flagging whether the platform confirmed the reviewer flew the airline, but verification is binary and limited: many genuine reviews are unverified, and verification does not prevent paid reviews from verified users.

We extract a new column `Authenticity signal` with four values capturing how generic or specific each review feels:

- `specific-detailed` — the review contains concrete details (flight numbers, dates, locations, named staff, sums of money, durations, named aircraft features) that would be hard to invent
- `specific-emotional` — strong emotional content tied to a specific event, even without verifiable details
- `generic-positive` — uniformly positive, marketing-flavored, no concrete detail (typical pattern of paid favorable reviews)
- `generic-negative` — uniformly negative, generic complaints, no concrete detail (typical pattern of competitor smearing or copy-paste outrage)
- `N/A` — empty or unreadable

This column then feeds two analyses:

1. **Cross-check against `Verified`.** Reviews labeled `generic-positive` AND `Verified = no` are the prime candidates for inauthentic positive content. Reviews labeled `generic-negative` AND `Verified = no` are the prime candidates for organized negative campaigns.
2. **Per-airline ratio.** Airlines whose unverified reviews skew heavily toward `generic-positive` deserve closer reading. This is a starting point for investigation, not a conclusion.

## The prompt

In **Add column by AI**, name the new column `Authenticity signal` and paste this instruction:

```
You are a review authenticity classifier. Read the airline review text and
classify it based on the level of concrete detail and the emotional tone.

Output exactly one of these values, nothing else:
- specific-detailed   (review contains verifiable concrete details: flight
                       numbers, specific dates, route names, named staff,
                       sums of money, durations, specific aircraft features,
                       named airports, specific menu items)
- specific-emotional  (review describes a specific event with strong feeling
                       but no easily verifiable concrete details)
- generic-positive    (review is uniformly positive, uses marketing-style
                       language like "excellent service", "highly recommend",
                       "amazing crew" without any concrete detail or specific
                       event)
- generic-negative    (review is uniformly negative, uses generic complaint
                       language like "worst airline ever", "terrible service",
                       "would never recommend" without any concrete detail or
                       specific event)
- N/A                 (empty or unreadable input)

Rules:
1. Concrete details that count: a specific flight number (e.g. "S4301"), a
   specific airport code (e.g. "CDG"), a duration ("6 hour delay"), a sum
   ("EUR 200 voucher"), a date, a named individual, a specific aircraft type
   ("A330-300"), a named menu item, a specific seat number.
2. Emotional intensity alone does not make a review specific. "Worst flight
   of my life" without details -> generic-negative. "Worst flight of my
   life, the 6-hour delay in Frankfurt cost me my connection" -> specific.
3. Length alone does not make a review specific. A long review of generic
   praise stays generic-positive. A short review with a flight number and
   a duration is specific-detailed.
4. Reviews with mixed positive and negative tone — if they contain concrete
   details, classify as specific-detailed or specific-emotional based on
   detail level; if not, pick whichever generic tone dominates.
5. Do not judge whether the review is true or false. Only judge the surface
   pattern: detailed vs generic, positive vs negative tone.
6. Output only the single label. No punctuation, no quotes, no explanation.

Examples:
- Review: "Flight S4301 from Lisbon to Toronto on Feb 10th, entertainment
  system did not work, baggage limit only 8 kg." -> specific-detailed
- Review: "Six hour delay on outbound, two hour delay on inbound, paid extra
  for seats with extra space but pitch was small." -> specific-detailed
- Review: "Excellent service, amazing crew, comfortable flight, highly
  recommended for everyone, would fly again." -> generic-positive
- Review: "Worst airline ever. Terrible service. Avoid at all costs."
  -> generic-negative
- Review: "I cried during this flight because the crew was so rude to me, I
  felt humiliated and powerless." -> specific-emotional
- Review: "" -> N/A
```

Set the response format to **Text**.

## Limits of the result

This is the highest-risk worksheet in the suite. Read these limits before publishing anything based on the column.

**The label is a signal, not a verdict.** A `generic-positive` review is not necessarily fake. Many genuine satisfied customers write short uniformly positive blurbs because that is how satisfaction sounds. Conversely, a `specific-detailed` review can be a sophisticated fake. The label tells you where to read carefully, not what to conclude.

**Marketing vocabulary is not exclusive to fakes.** Real customers absorb marketing language and reuse it. The model cannot distinguish "amazing crew" written by a paid reviewer from "amazing crew" written by a sincerely impressed grandmother. Cross-check against `Verified` and look at the reviewer's history (if your dataset has user IDs) before drawing conclusions.

**Cultural and linguistic bias.** The model was trained predominantly on English text and may misclassify reviews translated from other languages, where direct translation produces patterns that look generic in English but were specific in the source. Spot-check non-English reviewer cohorts manually.

**Verified column interaction.** The cleanest signal is `generic-positive` AND `Verified = no` AND `Recommended = yes` AND `Overall_Rating = 10`. Use this combination as a facet and read 20 to 50 examples by hand before claiming a pattern. Do the same for `generic-negative` AND `Verified = no` AND `Recommended = no` AND `Overall_Rating = 1`.

**Format violations.** Around 2 percent of outputs come back with extra punctuation, casing variants, or shortened forms ("specific", "generic"). Add a GREL guard:

```grel
forEach(
  ["specific-detailed","specific-emotional","generic-positive","generic-negative","N/A"],
  label, if(value.trim().toLowercase() == label.toLowercase(), label, null)
).filter(v, v != null)[0]
```

If empty, mark `MALFORMED` and re-run.

**Aggregate carefully.** "X percent of unverified reviews of airline Y are generic-positive" is a defensible statement. "Airline Y is buying fake reviews" is not. Stop at the descriptive layer unless you have independent corroboration.

**Legal sensitivity.** Publishing per-airline authenticity statistics may trigger reputation claims. Make sure your editorial process is comfortable with the framing before going to press.

**Runtime.** Long reviews mean longer inference time per row. Budget around 1.5 seconds per row, or about 10 hours for the full 23,000-row dataset. Sample first.

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

**Temperature 0.1.** Classification into a five-value vocabulary on a fuzzy boundary. The model needs minimal flexibility to weigh competing signals (concrete details vs marketing language can coexist in the same review). Pure 0.0 locks too early on the first signal it sees.

**Top-P 0.9.** Standard tight value. Suppresses tempting near-labels ("mixed", "neutral", "borderline") that the model would otherwise invent on ambiguous reviews, which would violate the closed vocabulary.

**Seed 42.** Locks reproducibility. **Critical here for a different reason than the other worksheets:** because the output drives a potentially consequential narrative ("X airline's unverified reviews skew toward generic-positive"), the run must be exactly reproducible by an editor, a fact-checker, or a regulator. A seeded run is auditable. An unseeded run is not.

**Max Tokens 12.** Output is at most two hyphenated words. 12 tokens stops the model from appending the justification it is tempted to attach on ambiguous reviews.

**Wait Time 0.** Local inference.

**Model choice.** Ministral 3B handles the obvious cases (short generic reviews, long detailed reviews) reliably. The borderline cases — short reviews with one specific detail, long reviews that mix generic praise with one concrete grievance — are where it slips. For investigation-grade work, swap to `mistral:7b` or `qwen2.5:7b`. Both reduce borderline misclassification by roughly 30 percent. For maximum reliability on a publication, run the same prompt with two different models and only trust rows where both agree.

**One non-negotiable.** Never present the `Authenticity signal` column to the public without exposing the prompt, the model, the settings, the limits section above, and ideally the per-row reasoning behind each label. Opacity in this kind of analysis is what makes it dangerous.
