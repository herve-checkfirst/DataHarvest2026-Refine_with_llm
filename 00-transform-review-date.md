# Transformation: Review Date Normalization to ISO 8601

Complexity level: basic. This transformation converts a single date format into a sortable, machine-readable form.

**Important: do not actually use the LLM for this in production.** Plain GREL handles date reformatting deterministically, instantly, and for free. This fiche is included as a teaching baseline — it shows the AI Extension workflow end-to-end on a task so simple that you can verify every output by eye, before moving to harder transformations where GREL alone is not enough. The native GREL solution is given at the end of the "Limits" section.

## What we want to do

The `Review Date` column uses a human-readable English format with an ordinal suffix:

```
26th September 2022
9th September 2018
2nd January 2022
3rd March 2022
```

This format does not sort lexicographically, does not parse natively in most BI or charting tools, and breaks when you try to bin by month or year. We want a new column `Review Date ISO` with the ISO 8601 calendar date format:

```
2022-09-26
2018-09-09
2022-01-02
2022-03-03
```

ISO 8601 sorts correctly as a string, parses in Excel, Tableau, Datawrapper, Observable, Pandas, and SQL without configuration, and feeds time-series visualizations directly.

Empty or unrecognizable inputs return `N/A`.

> Note: a GREL expression can do this with a one-liner. We use the LLM here to show the workflow end-to-end on a trivial case before moving to harder transformations where GREL is not enough.

## The prompt

In **Add column by AI**, name the new column `Review Date ISO` and paste this instruction:

```
You are a date format converter. Convert the input date to ISO 8601 format
(YYYY-MM-DD).

Input format: an English date with an ordinal day suffix and a full month name,
for example "26th September 2022", "2nd January 2022", "3rd March 2022".

Output format: YYYY-MM-DD with zero-padded month and day, for example
"2022-09-26", "2022-01-02", "2022-03-03".

Rules:
1. Strip the ordinal suffix (st, nd, rd, th) from the day.
2. Convert the English month name to its two-digit number (January = 01,
   February = 02, ..., December = 12).
3. Zero-pad the day to two digits (2 -> 02, 9 -> 09).
4. Keep the year as a four-digit number.
5. If the input is empty, unreadable, or not a date, output exactly: N/A
6. Output only the formatted date string. No quotes, no explanation, no extra
   whitespace.

Examples:
- Input: "26th September 2022"  -> 2022-09-26
- Input: "9th September 2018"   -> 2018-09-09
- Input: "2nd January 2022"     -> 2022-01-02
- Input: "3rd March 2022"       -> 2022-03-03
- Input: "1st December 2010"    -> 2010-12-01
- Input: ""                     -> N/A
```

Set the response format to **Text**.

## Limits of the result

The task is mechanical and the model handles it with very high accuracy, but a few systematic errors still slip through.

**Month-name confusion on rare months.** Ministral 3B occasionally maps "May" to `05` correctly but writes "March" as `03` then flips to `04` on the next row. Re-running the row gives the right answer. The error rate is low (well under 1 percent) but non-zero.

**Year out of range.** Reviews from before 2000 or anything where the year is partially obscured can produce hallucinated years (the model "fixes" what it thinks is a typo). The Review Date column in this dataset spans roughly 2009 to 2023 cleanly, so this is mostly a theoretical risk here.

**Missing leading zero.** Despite rule 3, around 0.5 percent of outputs come back as `2022-9-26` instead of `2022-09-26`. This breaks lexicographic sorting. Add a GREL guard column to validate and re-format:

```grel
value.match(/^\d{4}-\d{2}-\d{2}$/) != null ? value :
value.match(/^\d{4}-\d{1,2}-\d{1,2}$/) != null ?
  value.split("-")[0] + "-" + value.split("-")[1].padLeftWith("0", 2) + "-" + value.split("-")[2].padLeftWith("0", 2) :
"MALFORMED"
```

**Trailing prose.** Occasionally the model appends "(September 26, 2022)" or similar. Truncate to the first 10 characters and re-validate.

**The honest GREL alternative.** For this exact dataset, the following GREL does the same job in one pass, with zero hallucinations and at native speed:

```grel
value.replace(/(\d+)(st|nd|rd|th)/, "$1").toDate("d MMMM yyyy").toString("yyyy-MM-dd")
```

Use the LLM version to illustrate the workflow; ship the GREL version in production.

## Model settings and why

The provider configuration in OpenRefine for this transformation:

| Field | Value |
|---|---|
| Model | `ministral-3:3b` |
| Temperature | `0.0` |
| Top-P | `0.9` |
| Seed | `42` |
| Max Tokens | `16` |
| Wait Time | `0` |

**Temperature 0.0.** Date conversion has exactly one correct answer per input. Any non-zero temperature introduces a chance of swapping month numbers or rewriting the year. Zero collapses the model to its most likely token at each step, which on this task is always the correct digit.

**Top-P 0.9.** Standard value, kept as a safety net under temperature 0. No reason to tune it further on a deterministic task.

**Seed 42.** Locks the sampling RNG so re-runs produce identical output. Useful when you A/B-test prompt wording: any difference you see is from the prompt, not from randomness.

**Max Tokens 16.** The output is always 10 characters (`YYYY-MM-DD`) or 3 (`N/A`). Capping at 16 tokens stops the model from appending parenthetical explanations or rewriting the input back, and gives a fast fail path when the model starts to ramble.

**Wait Time 0.** Local inference, no rate limit.

**Model choice.** Ministral 3B is the right tier. The task does not need world knowledge or reasoning, only pattern matching on a small vocabulary (12 month names, 4 ordinal suffixes). A larger model brings no measurable accuracy gain and triples the runtime on a column with 23 000 rows.
