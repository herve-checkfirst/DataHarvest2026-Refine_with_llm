# Transformation: Discrimination Signal Detection

Complexity level: advanced. This transformation flags reviews that describe what the reviewer perceives as discriminatory or inequitable treatment. It is the most editorially sensitive worksheet in the suite and requires explicit guardrails.

**Read this before running the prompt.** The LLM does not adjudicate whether discrimination actually occurred. It only detects whether the reviewer's account contains the linguistic and narrative signals associated with such complaints. Treat the output column as a triage tool that surfaces accounts for human review, never as evidence that discrimination did or did not happen.

## What we want to do

Reviewers sometimes describe being treated differently based on what they perceive to be their identity: refused boarding while others were allowed, targeted for additional security checks, ignored by staff, given worse service than nearby passengers. These accounts are present in the dataset but invisible to any structured column.

We extract a new column `Discrimination signal` with four values:

- `yes-explicit` — the reviewer explicitly attributes differential treatment to a protected characteristic (race, ethnicity, religion, gender, disability, age, language, nationality, sexual orientation)
- `yes-implicit` — the reviewer describes differential treatment compared to other passengers without naming a protected characteristic, but the framing suggests they perceive the difference as unjust
- `no` — the review describes complaints with no comparative or identity-based framing
- `N/A` — empty or unreadable

The downstream use is **discovery**, not measurement. Facet `yes-explicit` and `yes-implicit`, read every flagged review, and decide editorially which ones warrant further investigation. Do not publish counts like "X percent of airline Y's reviews report discrimination" — that statistic conflates the reviewer's perception with verified fact, and the LLM's classification accuracy is well below the threshold required for that kind of claim.

## The prompt

In **Add column by AI**, name the new column `Discrimination signal` and paste this instruction:

```
You are a content classifier for airline reviews. Read the review text and
identify whether the reviewer describes differential treatment that they
perceive as unjust.

Output exactly one of these values, nothing else:
- yes-explicit   (reviewer explicitly attributes differential treatment to
                  a protected characteristic: race, ethnicity, skin color,
                  religion, gender, sexual orientation, disability, age,
                  language, accent, nationality, citizenship status)
- yes-implicit   (reviewer describes being treated differently from other
                  passengers, with framing that suggests they perceive the
                  difference as unjust, but without naming a protected
                  characteristic explicitly)
- no             (review contains complaints with no comparative or
                  identity-based framing)
- N/A            (empty or unreadable input)

Rules:
1. The reviewer's perception is what you classify. Do not assess whether
   discrimination actually occurred.
2. "Explicit" requires the reviewer to name a protected characteristic or
   identity category in connection with the treatment they describe.
   Examples: "as a Muslim woman wearing a hijab I was singled out", "they
   refused to seat my wheelchair near the front", "the staff laughed at my
   accent".
3. "Implicit" applies when the reviewer compares their treatment to other
   passengers and frames the difference as unjust, without naming the
   characteristic. Example: "other passengers in similar seats received
   their meal first while we were left waiting" with a tone of grievance.
4. Generic complaints about service quality without comparison or identity
   framing do not qualify. "The crew was rude to me" alone -> no. "The crew
   was rude to me but friendly to the passenger next to me" -> yes-implicit.
5. Do not consider the credibility of the account. Only the surface pattern
   matters for this label.
6. Output only the single label. No punctuation, no quotes, no explanation.

Examples:
- Review: "As a Black passenger I was the only one asked to show my passport
  three times during boarding." -> yes-explicit
- Review: "The crew greeted everyone in business class except me, and I had
  to ask twice for water while others were served immediately."
  -> yes-implicit
- Review: "Six hour delay, lost baggage, rude crew, would not fly again."
  -> no
- Review: "" -> N/A
```

Set the response format to **Text**.

## Limits of the result

This worksheet has the highest accuracy ceiling problem of any in the suite. Read all of this section before relying on the output.

**The model recognizes patterns, not truth.** A `yes-explicit` label tells you the reviewer wrote in a pattern consistent with reporting discrimination. It tells you nothing about whether discrimination occurred, whether the reviewer is acting in good faith, or whether the account is corroborated. Use only as a pointer to read by hand.

**Cultural framing bias.** Reviewers from different cultures describe perceived unfairness differently. North American English tends to be direct about identity; British English uses understatement; some Asian-language patterns translated to English read as neutral when they were specific in the source. The model was trained predominantly on direct North American English and will over-flag that pattern, under-flag others.

**False positives from competitive framing.** Reviewers sometimes write "they treated business class better than us" to complain about cabin class differences, not discrimination. Rule 3 tries to push toward grievance tone, but borderline cases will be misclassified. Read every flagged row before publishing.

**False negatives from understatement.** Reviewers who describe a serious incident in flat language ("the agent did not seem comfortable with my partner and me") may not be flagged because the model misses the implication. Counterbalance by spot-reading random `no` rows from airlines where you have other reasons to look.

**Verified column interaction.** Reviews labeled `yes-explicit` AND `Verified = no` warrant the most caution: they have the highest signal-to-noise ratio for fabricated accounts. This does not mean unverified accounts are false — it means you should corroborate before treating them as evidence.

**Per-airline aggregation is dangerous.** Do not publish "airline X has Y percent of reviews labeled as discrimination signals" as a finding. The accuracy of the column is not high enough to support that claim. It is high enough to point you at accounts that may be worth investigating, which is a different and much more defensible use.

**Format violations.** Around 2 to 3 percent of outputs come back with extra characters or near-variants ("yes", "implicit", "yes - implicit"). Add a GREL guard:

```grel
value.trim().toLowercase().replace(" - ","-").contains("yes-explicit") ? "yes-explicit" :
value.trim().toLowercase().replace(" - ","-").contains("yes-implicit") ? "yes-implicit" :
value.trim().toLowercase() == "no" ? "no" :
value.trim().toLowercase() == "n/a" ? "N/A" : "MALFORMED"
```

**Legal and editorial process.** Before publishing any finding derived from this column, have your editorial process review the flagged accounts, contact the airlines named in those accounts for response, and document the methodology including the prompt, model, settings, and limits. Opacity in this kind of analysis is what makes it dangerous.

**Runtime.** Long reviews mean longer inference. Budget around 1.5 seconds per row, around 10 hours for the full 23,000-row dataset on a CPU laptop. Sample first.

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

**Temperature 0.1.** Four-value classification on a deeply fuzzy boundary. Pure 0.0 makes the model latch onto the first identity-related keyword it sees ("Muslim", "wheelchair") regardless of whether the review actually frames a complaint around it. 0.1 lets it weigh the broader context.

**Top-P 0.9.** Standard tight value. Suppresses tempting near-vocabulary outputs ("possible", "ambiguous", "unclear") that would violate the closed vocabulary on borderline reviews.

**Seed 42.** Locks reproducibility. **Mandatory here** because the column potentially drives a high-stakes editorial process. Any finding from this column must be reproducible bit-for-bit by an editor, a fact-checker, a lawyer, or a regulator. An unseeded run is not auditable, and an unauditable analysis of discrimination claims is not publishable.

**Max Tokens 12.** Output is at most two hyphenated words. 12 tokens stops the model from appending the reasoning it is strongly tempted to write on flagged rows.

**Wait Time 0.** Local inference.

**Model choice.** Ministral 3B handles explicit cases reliably (when an identity is named directly) but is shaky on implicit framing. For investigation-grade work, this is one of the worksheets where model size matters most. Swap to `mistral:7b` or `qwen2.5:7b`, and ideally **run the same prompt with two different models, only trusting agreement on `yes-explicit` and `yes-implicit` labels**. The disagreement set itself is interesting and worth reading. For maximum reliability, add a third independent reading by a human.
