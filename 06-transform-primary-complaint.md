# Transformation: Primary Complaint Classification

Complexity level: intermediate. This transformation reads a free-text review and emits a single label from a closed vocabulary describing the main source of dissatisfaction. The output feeds directly into treemaps, Sankey diagrams, and per-airline complaint distributions.

## What we want to do

The `Review` column contains 150 to 2500 characters of narrative. Many reviews list multiple grievances, but there is usually one dominant complaint that drove the reviewer to write — a six-hour delay, lost baggage, rude staff. We want a single new column `Primary complaint` that captures this top-level cause using a fixed vocabulary.

The closed vocabulary is the central design choice. Without it, the model invents synonyms ("Communication failure" on one row, "Poor communication" on the next, "Bad info" on a third) and the resulting column is impossible to aggregate. By pinning the labels in advance, every row falls into one of eleven buckets that you can count, group, and visualize without further normalization.

Vocabulary:

- `Delay` — flight took off or landed significantly late
- `Cancellation` — flight cancelled outright
- `Lost baggage` — checked luggage lost, damaged, or delayed
- `Rude staff` — interaction with crew or ground staff was hostile or dismissive
- `Seat comfort` — seat itself, legroom, recline, pitch
- `Food quality` — meals, snacks, beverages, or the lack thereof
- `Boarding chaos` — disorganized check-in, gate, or boarding process
- `IFE broken` — inflight entertainment system not working or unavailable
- `Hidden fees` — surprise charges, extras pushed at the airport, baggage fee disputes
- `Other` — review is negative but the complaint does not fit any of the above
- `None` — review is positive overall, no significant complaint

## The prompt

In **Add column by AI**, name the new column `Primary complaint` and paste this instruction:

```
You are a complaint classifier for airline reviews. Read the review text and
identify the single dominant complaint, then output one label from the fixed
vocabulary below. Do not invent new labels.

Vocabulary (output exactly one of these strings, character for character):
- Delay
- Cancellation
- Lost baggage
- Rude staff
- Seat comfort
- Food quality
- Boarding chaos
- IFE broken
- Hidden fees
- Other
- None

Rules:
1. Use "None" only when the review expresses no significant complaint or is
   overall positive.
2. Use "Other" only when the review is clearly negative but the complaint
   does not match any specific category in the vocabulary.
3. If the review lists multiple complaints, pick the one given the most
   weight (length of discussion, intensity of language, position at start of
   review).
4. "Delay" covers any significant lateness regardless of cause. Use
   "Cancellation" only when the flight was cancelled outright.
5. "Lost baggage" covers lost, delayed, and damaged checked luggage.
6. "IFE" stands for inflight entertainment. Use this label when screens,
   wifi, or seatback systems are broken or missing.
7. Output only the single label string. No punctuation, no quotes, no
   explanation, no leading or trailing space.

Examples:
- Review: "Six hour delay on outbound, two hour delay on inbound, poorly
  communicated." -> Delay
- Review: "Our flight was cancelled and we had to rebook ourselves."
  -> Cancellation
- Review: "Bags never arrived in Toronto, took four days to get them back."
  -> Lost baggage
- Review: "The crew were dismissive when I asked for water and rolled their
  eyes at my request." -> Rude staff
- Review: "Excellent service, comfortable seat, on time arrival, would fly
  again." -> None
- Review: "The terminal smelled bad and the queue was outside in the rain."
  -> Other
```

Set the response format to **Text**.

## Limits of the result

A closed vocabulary makes this task tractable for a 3B model, but expect a few systematic problems.

**Vocabulary drift.** Despite rule 7, around 1 to 2 percent of outputs come back with extra characters: a trailing period, a leading "Label: ", or a casing variant ("delay" instead of "Delay"). Add a GREL guard column to canonicalize:

```grel
forEach(
  ["Delay","Cancellation","Lost baggage","Rude staff","Seat comfort","Food quality","Boarding chaos","IFE broken","Hidden fees","Other","None"],
  label, if(value.trim().toLowercase().contains(label.toLowercase()), label, null)
).filter(v, v != null)[0]
```

If the result is empty, mark the row as `MALFORMED` for manual review.

**Multi-complaint reviews forced into one bucket.** Rule 3 collapses rich reviews into a single label, which loses information. The reviewer who experienced a delay, then rude staff, then lost baggage gets reduced to "Delay" because that is what they wrote about first. If you need multi-label output, run the prompt once per category as a binary "is this complaint present yes/no" — but that multiplies inference cost by eleven.

**"Other" as a catchall.** The model under-uses "Other" and over-fits to existing labels, which means review categories you did not anticipate (poor air quality, broken aircon, lost pet) get misclassified into the closest available label. Sample 50 random "Delay" and "Seat comfort" rows after the run; if more than 5 of them are really something else, your vocabulary needs a new category.

**Positive reviews mislabeled as "Other".** The model sometimes treats neutral or positive reviews as needing a complaint label rather than "None". If your dataset is roughly half positive, half negative, the count of "None" should be in the same ballpark as the count of `Recommended = yes`. Big divergence means the prompt is biasing the model toward complaints.

**Cross-check against `Recommended`.** Create a sanity column:

```grel
if(cells["Primary complaint"].value == "None" && cells["Recommended"].value == "no", "mismatch",
  if(cells["Primary complaint"].value != "None" && cells["Recommended"].value == "yes", "mismatch", "consistent"))
```

Facet on `mismatch` to find the disagreements. These are either model errors or genuinely interesting cases (reviewers who recommend the airline despite a complaint, or warn against it despite no specific gripe).

**Runtime.** One pass at roughly one second per row on the full 23,000-row dataset is about six hours of inference on a laptop. Sample first, then commit.

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

**Temperature 0.1.** Classification into a fixed vocabulary needs near-deterministic output, but not pure 0.0. A sliver of flexibility helps the model weigh competing complaints in a long review and pick the dominant one rather than locking onto the first complaint keyword it sees. Going higher (0.3 and above) starts producing labels that drift slightly from the vocabulary, which breaks aggregation.

**Top-P 0.9.** Standard tight value. Important here because it suppresses synonyms — the model is constantly tempted to output "Late" instead of "Delay" or "Rude crew" instead of "Rude staff". Tight top-p makes those low-probability variants unlikely.

**Seed 42.** Locks reproducibility so prompt iterations are comparable. Especially useful when you tune the vocabulary list and need to know whether label shifts came from your edit or from sampling.

**Max Tokens 8.** Output is at most three words ("Boarding chaos", "Lost baggage", "Hidden fees"). Capping at 8 prevents the model from appending a justification ("Delay - the flight was six hours late") which would always violate rule 7 and require post-processing.

**Wait Time 0.** Local inference.

**Model choice.** Ministral 3B is well-suited to this task because the closed vocabulary does most of the work — the model only has to match a short text to one of eleven well-separated buckets, which is within reach of a small model. For a heavier dataset or noisier reviews, `mistral:7b` improves the rate of correct "Other" and "None" classifications without changing the prompt or settings.
