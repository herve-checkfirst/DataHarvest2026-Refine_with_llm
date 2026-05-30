# Transformation: Review Language Detection and English Summary

Complexity level: intermediate. This transformation produces two columns from the `Review` text: the language the review was written in (ISO 639-1 code), and a one-sentence English summary. Together they make the entire corpus accessible to an English-speaking analyst regardless of the original language.

## What we want to do

The Kaggle dataset is predominantly English, but a non-trivial minority of reviews are in German, Spanish, French, Italian, Portuguese, Dutch, and occasionally Chinese, Japanese, or Russian. Reviews in non-English languages are invisible to keyword searches, faceting on English terms, and most downstream prompts in this repository. They effectively drop out of every analysis run on the English subset, biasing the findings toward English-speaking customer cohorts.

We extract two new columns in two sequential passes:

- `Review language` — ISO 639-1 code (`en`, `de`, `es`, `fr`, `it`, `pt`, `nl`, `zh`, `ja`, `ru`, `tr`, `ar`, `pl`, ...) or `unknown` if detection fails, or `N/A` if empty
- `Review summary EN` — one English sentence (max 30 words) capturing the main point of the review, regardless of source language

This pair lets you:

- Facet the corpus by language and study cohort-specific patterns (do German-speaking reviewers complain about different things than English-speaking ones?)
- Run downstream prompts on the summary column instead of the original `Review`, normalizing the input language to English
- Build a multilingual word cloud or theme analysis without losing any review

## The prompts

Two separate prompts, run in sequence. Pass 1 produces a language code; pass 2 produces an English summary.

### Pass 1: `Review language`

In **Add column by AI**, name the new column `Review language` and paste:

```
You are a language detector. Identify the primary language of the input text
and output its ISO 639-1 two-letter code in lowercase.

Output exactly one of:
- en   (English)
- de   (German)
- es   (Spanish)
- fr   (French)
- it   (Italian)
- pt   (Portuguese)
- nl   (Dutch)
- pl   (Polish)
- tr   (Turkish)
- ru   (Russian)
- ar   (Arabic)
- zh   (Chinese)
- ja   (Japanese)
- ko   (Korean)
- other  (a real language not listed above)
- unknown  (cannot determine, mixed languages with no dominant one)
- N/A  (empty or unreadable input)

Rules:
1. If the text mixes languages, output the dominant one (more than 60 percent
   of the words).
2. Code-switching with English (typical in reviews of Asian carriers) -> use
   the non-English language if it carries the substantive content, English
   only if the non-English fragments are loanwords.
3. Place names and airline names alone do not constitute a language signal.
4. Output only the two-letter code or the literal "other" / "unknown" /
   "N/A". No punctuation, no quotes, no explanation.
```

Set the response format to **Text**.

### Pass 2: `Review summary EN`

In **Add column by AI**, name the new column `Review summary EN` and paste:

```
You are a multilingual review summarizer. Read the input airline review,
which may be in any language, and produce a single English sentence (maximum
30 words) capturing the main point.

Rules:
1. Always output English, regardless of the source language.
2. Focus on the substantive content: what happened, what the reviewer felt,
   what they would do differently.
3. Preserve concrete details (flight numbers, durations, locations, sums)
   when present.
4. Do not editorialize. Summarize what the reviewer says, not what you think
   about it.
5. Maximum 30 words. One sentence, ending with a period.
6. If the review is empty or unreadable, output exactly: N/A
7. Output only the summary sentence. No quotes, no labels, no source-language
   echo, no explanation.

Examples:
- Review (German): "Ein katastrophaler Flug von München nach Madrid. Sechs
  Stunden Verspätung ohne Erklärung, das Personal war unfreundlich."
  -> Catastrophic Munich to Madrid flight with a six-hour unexplained delay
  and unfriendly staff.
- Review (English): "Great service from Singapore to Tokyo, comfortable seat,
  good food, would fly again."
  -> Comfortable and well-served Singapore to Tokyo flight that the reviewer
  would repeat.
- Review: "" -> N/A
```

Set the response format to **Text**.

## Limits of the result

Language detection is a well-solved problem at this scale; English summarization across languages is where the risks concentrate.

**Hallucination in summaries.** The summary prompt is the only one in the repository that asks the model to generate prose rather than choose a label. Generated prose can drift from the source. Mitigations:

- Keep the 30-word cap strict; long summaries hallucinate more than short ones.
- Always preserve the source `Review` column alongside the summary. Never present the summary as a quote.
- Spot-check 30 random summaries per non-English language against a translator (Google Translate, DeepL) or a native speaker.

**Loss of nuance.** Strong emotions, sarcasm, cultural references, and idiomatic complaints often flatten in summary. A summary that reads as neutral may correspond to a passionately negative source. If sentiment matters for your analysis, run `05-transform-aspect-sentiment.md` on the original `Review` column, not on the summary.

**Tonal smoothing.** The model tends to summarize negative reviews with polite English ("the reviewer was dissatisfied with the service"), which loses the original intensity. Downstream sentiment analysis on the summary will systematically under-rate negative cases.

**Language detection edge cases.** Very short reviews (under 50 characters) are detected unreliably. Reviews in transliterated languages (e.g. Arabic written in Latin characters) often get labeled `other` or guessed wrong. Romance languages can be confused on short inputs.

**Format violations.** Around 1 percent of language codes come back as `English` or `EN` instead of `en`. Add a GREL guard:

```grel
value.trim().toLowercase().length == 2 ? value.trim().toLowercase() :
value.trim().toLowercase() == "english" ? "en" :
value.trim().toLowercase() == "spanish" ? "es" :
value.trim().toLowercase() == "german" ? "de" :
value.trim().toLowercase() == "french" ? "fr" :
value.trim().toLowercase().contains("unknown") ? "unknown" :
value.trim().toLowercase() == "n/a" ? "N/A" :
value.trim().toLowercase().contains("other") ? "other" : "MALFORMED"
```

For summaries, validate that word count is under 30 and that the output is in English (a simple heuristic: it should contain at least one common English function word like "the", "and", "to"). Flag and re-run failures.

**Runtime.** Two passes. Summary generation is slower than classification: budget around 2 seconds per row. On the full 23,000-row dataset that is around 12 hours per pass. The language detection pass is fast (around 1 second per row, six hours total). If you only need the language code without summaries, skip pass 2.

## Model settings and why

Different settings for the two passes.

### Pass 1 (language detection)

| Field | Value |
|---|---|
| Model | `ministral-3:3b` |
| Temperature | `0.0` |
| Top-P | `0.9` |
| Seed | `42` |
| Max Tokens | `4` |
| Wait Time | `0` |

**Temperature 0.0.** Language detection is deterministic by design; there is no creative judgment. Pure zero locks the model onto its highest-probability detection.

**Max Tokens 4.** Output is a 2-letter code or a 7-letter word at most ("unknown"). 4 tokens covers both with margin.

### Pass 2 (English summary)

| Field | Value |
|---|---|
| Model | `ministral-3:3b` |
| Temperature | `0.3` |
| Top-P | `0.9` |
| Seed | `42` |
| Max Tokens | `60` |
| Wait Time | `0` |

**Temperature 0.3.** Generation needs more flexibility than classification — pure 0.0 produces repetitive, mechanical summaries that read poorly and miss context. 0.3 is the sweet spot for summarization at this model size: fluent enough to read naturally, conservative enough to stick to the source.

**Top-P 0.9.** Standard value. Tighter (0.7) makes summaries cliché; looser (0.95) lets the model drift.

**Seed 42.** Locks reproducibility. Especially important for summaries — without a seed, the same source review produces different English wordings across runs, which makes auditing impossible.

**Max Tokens 60.** 30 English words is roughly 40 to 50 tokens. 60 gives headroom and stops runaway generation.

**Wait Time 0.** Local inference.

**Model choice.** Ministral 3B detects major European languages well but struggles with Asian languages and code-switched text. Summary quality is acceptable for journalism triage but lacks the polish needed for publication-as-quote. For publication-grade summaries, swap to `mistral:7b` or `qwen2.5:7b`. Qwen2.5 in particular has stronger multilingual coverage and produces noticeably better Chinese, Japanese, and Korean summaries.

**Never publish a summary as a quote.** The summary is a navigation aid for the analyst, not text the reviewer wrote. If you want to quote a non-English reviewer, get a professional translation of the original `Review` value.
