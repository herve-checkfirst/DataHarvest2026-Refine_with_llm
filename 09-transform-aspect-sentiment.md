# Transformation: Aspect-Based Sentiment Extraction

Complexity level: advanced. This transformation breaks a free-text review into four parallel sentiment signals, one per service aspect. It is one of the strongest demonstrations of the AI Extension because no combination of GREL, facets, or clustering in OpenRefine can produce this output — it requires reading comprehension over hundreds of characters of unstructured prose.

## What we want to do

The `Review` column contains 150 to 2500 characters of narrative. The existing structured columns (`Seat Comfort`, `Cabin Staff Service`, `Food & Beverages`, `Ground Service`, `Inflight Entertainment`) capture the reviewer's numeric scores, but those scores are often inconsistent with the prose — a customer who rates seat comfort 8/10 still complains about the legroom in their review, and vice versa.

We extract four new columns, each with a ternary value:

- `Seat sentiment` — positive / negative / not mentioned
- `Crew sentiment` — positive / negative / not mentioned
- `Food sentiment` — positive / negative / not mentioned
- `Punctuality sentiment` — positive / negative / not mentioned

The result is a sentiment matrix that powers immediate visualizations: stacked bars per airline, heatmaps of aspect mentions, and crosstabs against the numeric ratings to flag rows where the prose contradicts the score.

Because the AI Extension produces one column at a time, you run the same prompt four times, changing only the target aspect in step 5 of the dialog and the output column name.

## The prompt

In **Add column by AI**, name the new column (one of `Seat sentiment`, `Crew sentiment`, `Food sentiment`, `Punctuality sentiment`) and paste the prompt below, replacing `<ASPECT>` with the matching aspect description from the table.

| Column name | `<ASPECT>` value |
|---|---|
| `Seat sentiment` | seat comfort, legroom, pitch, recline, seat width, or cabin space |
| `Crew sentiment` | flight attendants, cabin crew, pilots, ground staff attitude or helpfulness |
| `Food sentiment` | meals, snacks, drinks, beverage service, food quality or quantity |
| `Punctuality sentiment` | on-time performance, delays, cancellations, missed connections |

Prompt:

```
You are an aspect-based sentiment classifier for airline reviews. Read the
review text and decide how the reviewer feels about the following aspect:

  <ASPECT>

Output exactly one of these three values, nothing else:
- positive       (reviewer expresses satisfaction or praise about this aspect)
- negative       (reviewer expresses dissatisfaction or complaint about this
                  aspect)
- not mentioned  (this aspect is absent from the review, or mentioned without
                  any sentiment)

Rules:
1. Base your judgment only on the review text. Ignore any numeric ratings or
   metadata.
2. A factual statement with no evaluative language counts as "not mentioned"
   (example: "the meal was chicken or pasta" -> not mentioned for food
   sentiment).
3. Mixed sentiment about the same aspect (some positive, some negative) -> use
   the dominant tone. If truly balanced, choose negative.
4. If the aspect is implied but not explicit (example: "I slept well" for seat
   sentiment), still classify the sentiment.
5. Output only the single phrase "positive", "negative", or "not mentioned".
   No punctuation, no quotes, no explanation, no leading or trailing space.

Examples for the aspect "seat comfort, legroom, pitch, recline, seat width, or
cabin space":
- Review: "The legroom was generous and I slept the whole flight." -> positive
- Review: "Seat pitch was painfully small, I could not stretch." -> negative
- Review: "Boarding was chaotic but the crew handled it well." -> not mentioned
- Review: "Comfortable seat but the food was terrible." -> positive
```

Set the response format to **Text**. Run the prompt four times, once per aspect, swapping `<ASPECT>` and the output column name each time.

## Limits of the result

This is the hardest task in the suite for a 3B model. Expect a 5 to 10 percent error rate and budget time for spot-checking and a GREL guard.

**Aspect leakage.** The model occasionally labels `Crew sentiment` as negative because the review complains about food — it conflates "bad experience overall" with "bad crew". Worst on long reviews (over 1500 characters) where many aspects coexist. Mitigation: re-run failing rows with `mistral:7b`, which holds the aspect distinction more reliably.

**Polite negatives missed.** British understatement ("the seat was *adequate*") is often classified positive when it is a soft negative. The model takes lexical positivity at face value. No good fix at this model size; flag manually if the analysis depends on it.

**Overuse of "negative".** Rule 3 biases toward negative on mixed cases, which is intentional for complaint analysis but skews the global distribution. If you want neutral balance, change rule 3 to "if truly balanced, choose positive" and re-run.

**Inconsistent label spelling.** Despite rule 5, around 2 percent of outputs come back as `Positive`, `Not Mentioned`, `negative.`, or `none mentioned`. Canonicalize with a GREL guard on each column:

```grel
value.trim().toLowercase().contains("not") ? "not mentioned" :
value.trim().toLowercase().contains("posit") ? "positive" :
value.trim().toLowercase().contains("negat") ? "negative" : "MALFORMED"
```

Facet on `MALFORMED` and re-run those rows.

**Cross-check against numeric ratings.** Once all four columns exist, create a derived column `Seat consistency` with:

```grel
if(cells["Seat Comfort"].value == "", "n/a",
  if(toNumber(cells["Seat Comfort"].value) >= 4 && cells["Seat sentiment"].value == "negative", "score-prose mismatch",
    if(toNumber(cells["Seat Comfort"].value) <= 2 && cells["Seat sentiment"].value == "positive", "score-prose mismatch", "consistent")))
```

The mismatched rows are often the most analytically interesting — reviewers who rated kindly but vented in prose, or vice versa.

**Cost on the full 23,000-row dataset.** Four passes at roughly one second per row each is around 25 hours of inference on a typical laptop. Test on the 100-row sample first; only commit to the full dataset once the prompt is stable. Run overnight, not during a workshop.

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

**Temperature 0.1.** Sentiment classification has a single defensible answer per (review, aspect) pair, but the model needs a sliver of flexibility to weigh competing signals in the prose. Pure 0.0 makes the model lock onto the first sentiment-bearing word it sees, which on long reviews is often misleading. 0.1 lets it consider the dominant tone across the whole text while still suppressing creative output.

**Top-P 0.9.** Standard tight value. Cuts off low-probability label variants ("mixed", "unclear", "ambivalent") that the model is tempted to invent on borderline cases — those would violate rule 5 and produce MALFORMED rows.

**Seed 42.** Locks reproducibility. Critical here because you run the prompt four times across four columns and need to compare results meaningfully. A drifting seed makes it impossible to tell whether a label change between aspects is a real signal or sampling noise.

**Max Tokens 8.** Output is at most three words ("not mentioned"). Capping at 8 stops the model from appending a justification, which it is strongly tempted to do on long inputs. Going lower truncates "not mentioned".

**Wait Time 0.** Local inference.

**Model choice.** Ministral 3B is the floor of viability for aspect-based sentiment. It works for the major signals (delays, rude staff, terrible food) but misses subtle cases. For research-grade output, swap to `mistral:7b` or `qwen2.5:7b` — the prompt and settings stay identical, accuracy improves by roughly 10 percentage points, and runtime triples.
