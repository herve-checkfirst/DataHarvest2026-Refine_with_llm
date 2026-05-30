# Transformation: Compensation and Remediation Outcome

Complexity level: intermediate. This transformation extracts what the reviewer reports about compensation, refunds, vouchers, upgrades, and other remediation offered (or denied) by the airline after a problem. The output reveals customer-service patterns that no structured column in the dataset captures.

## What we want to do

The dataset records what airlines did wrong (through the reviews) and how customers rated the experience numerically, but it has no column for what the airline did to make things right afterwards. That information is buried in the review text and is exactly what investigative reporting on airline customer service needs: the gap between rated service and remediation behavior.

We extract a new column `Compensation outcome` with these values:

- `not mentioned` — review does not discuss compensation, refund, or remediation
- `requested-denied` — reviewer asked for compensation and the airline refused
- `requested-pending` — reviewer asked and is still waiting at time of writing
- `received-voucher` — reviewer received a credit, voucher, or future-flight discount
- `received-cash` — reviewer received a cash or original-payment refund
- `received-upgrade` — reviewer received a class upgrade or seat upgrade
- `received-other` — reviewer received tangible remediation that does not fit the categories above (hotel, meal voucher, lounge access, points)
- `proactive` — airline offered remediation without the reviewer asking
- `N/A` — empty or unreadable

This column then powers analyses like:

- Percentage of reviews mentioning compensation per airline
- Ratio of `requested-denied` to `received-*` per airline
- Whether airlines that compensate generously offset other complaints (cross with `Recommended`)

## The prompt

In **Add column by AI**, name the new column `Compensation outcome` and paste this instruction:

```
You are a customer service analyst for airline reviews. Read the review text
and identify how the airline handled any compensation or remediation related
to the reviewer's complaint.

Output exactly one of these values, nothing else:
- not mentioned     (review does not discuss any compensation, refund,
                     voucher, or remediation)
- requested-denied  (reviewer asked the airline for compensation and was
                     refused, ignored, or stonewalled)
- requested-pending (reviewer asked and is still waiting at time of writing,
                     or the case is unresolved)
- received-voucher  (reviewer received a credit, voucher, e-voucher, or
                     future-flight discount)
- received-cash     (reviewer received a cash refund or money back on their
                     original payment method)
- received-upgrade  (reviewer received a class upgrade or seat upgrade as
                     compensation)
- received-other    (reviewer received tangible remediation not matching the
                     categories above: hotel stay, meal voucher, lounge
                     access, frequent flyer points, ground transport)
- proactive         (airline offered remediation before the reviewer asked,
                     or without any complaint having been raised)
- N/A               (empty or unreadable input)

Rules:
1. The reviewer must mention the airline's remediation explicitly. Vague
   statements like "they did nothing" without context -> not mentioned.
2. Promised but not yet delivered remediation -> requested-pending.
3. If multiple forms of remediation are mentioned, pick the most substantive:
   cash > voucher > upgrade > other.
4. "I will never fly with them again" alone is not a compensation outcome.
   Only classify when the reviewer describes what the airline did or did
   not do in response to a problem.
5. EU Regulation 261 references (EC261, compensation claims) count as
   compensation discussions. Use requested-denied or received-cash based
   on the outcome described.
6. Output only the single label. No punctuation, no quotes, no explanation.

Examples:
- Review: "After the six hour delay we filed a claim under EC261 and were
  refused on the grounds of weather, which was untrue." -> requested-denied
- Review: "They offered me a EUR 200 voucher for future travel as
  compensation for the cancellation." -> received-voucher
- Review: "Got a full cash refund within two weeks, no questions asked."
  -> received-cash
- Review: "They put us up in a Marriott overnight when we missed the
  connection." -> received-other
- Review: "Without us asking, the captain came back and offered us business
  class seats for the inconvenience." -> proactive
- Review: "Worst flight ever, terrible food and rude crew." -> not mentioned
- Review: "" -> N/A
```

Set the response format to **Text**.

## Limits of the result

This task sits in a sweet spot for a 3B model: the categories are well-defined, the source mentions are usually explicit, and the output is a single label. Expect a 3 to 5 percent error rate.

**Promises versus deliveries.** Reviewers often write at the height of frustration before resolution. "They said they would refund me" — was the refund actually received? The model defaults to `requested-pending`, which is usually right but sometimes the reviewer meant the refund arrived. Read flagged rows before drawing conclusions.

**Class upgrade ambiguity.** Some reviewers describe being upgraded as a customer-loyalty gesture unrelated to any complaint. Rule 4 tries to require a complaint-response link, but the model occasionally labels these as `received-upgrade`. Cross-check by reading flagged rows where no complaint is otherwise visible — those are likely false positives.

**EU regulation jargon.** Rule 5 mentions EC261 explicitly because that is the most common compensation framework in the dataset. Other jurisdictions (US DOT, Canadian APPR, Brazilian ANAC rules) appear less often and the model handles them less reliably. Spot-check rows from non-EU routes.

**Aggregation interpretation.** The percentage of reviews mentioning compensation in any form is a function of both the airline's behavior and the reviewer cohort's willingness to write about it. A higher `received-voucher` rate at airline X does not necessarily mean airline X compensates more — it could mean its complainers are more vocal about resolution. Treat numbers comparatively (within similar route categories and rating distributions) rather than as absolutes.

**Format violations.** Around 1 to 2 percent of outputs come back with extra characters or near-variants. Add a GREL guard:

```grel
forEach(
  ["not mentioned","requested-denied","requested-pending","received-voucher","received-cash","received-upgrade","received-other","proactive","N/A"],
  label, if(value.trim().toLowercase().replace(" ","-").replace("_","-") == label.toLowercase().replace(" ","-"), label, null)
).filter(v, v != null)[0]
```

**Cross-check against `Recommended`.** A useful sanity check:

```grel
if(cells["Compensation outcome"].value.startsWith("received") && cells["Recommended"].value == "no", "compensated-but-unrecommended",
  if(cells["Compensation outcome"].value == "requested-denied" && cells["Recommended"].value == "yes", "denied-but-recommended", "consistent"))
```

The `compensated-but-unrecommended` cohort is the most interesting analytically: customers who got compensated but still warn others off. That gap reveals where compensation does not fix reputation.

**Runtime.** One pass at around 1 second per row, six hours for the full 23,000-row dataset on a CPU laptop. Empty values and pure-praise reviews (which always return `not mentioned`) make the effective rate faster — facet to non-empty and non-recommended before running if you want to spend inference where it matters.

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

**Temperature 0.1.** Classification into a nine-value vocabulary. Pure 0.0 makes the model lock onto the first compensation-related keyword without checking the outcome (it would label `requested-denied` as `received-cash` if cash is mentioned anywhere). 0.1 gives the right amount of contextual weighing.

**Top-P 0.9.** Standard tight value. Suppresses tempting variants ("partial refund", "compensation pending", "voucher denied") that would violate the closed vocabulary.

**Seed 42.** Locks reproducibility. This column will be used as an axis in per-airline aggregates and editorial comparisons — drift between runs would corrupt those numbers without warning.

**Max Tokens 12.** Output is at most two hyphenated words. 12 tokens fits the longest label ("requested-pending") with margin and truncates any attempt at justification.

**Wait Time 0.** Local inference.

**Model choice.** Ministral 3B handles this task well because the vocabulary categories are concrete and usually marked by specific keywords in the text (refund, voucher, upgraded, denied). For investigation-grade work, `mistral:7b` improves accuracy on the implicit cases (where compensation is implied rather than stated) by roughly 10 percentage points. Prompt and settings stay identical.
