# Transformation: Aircraft Family Reconciliation

Complexity level: advanced. This transformation is the showcase for the "Smart Disambiguation" angle of the talk. It reconciles a free-text `Aircraft` column with dozens of inconsistent values into a canonical family code, suitable for aggregation and visualization.

## What we want to do

The `Aircraft` column in the airline reviews dataset is a mess of inconsistent notation. The same sample of 100 reviews contains all of these:

```
Boeing 737-800
Boeing 737
Boeing 737-900
Boeing 787-9
Boeing 787-8
Boeing 787-8 and Boeing 787-9
A319
A320
A320-200
A321
A330
A330-300
A330-300 / A350-900
A350
A350-900
A380
A320 / Boeing 767-300
Boeing 787 / Boeing 737
Embraer 195
Fokker 100
Bombardier CRJ-900
```

These look like data, but they cannot be aggregated. A "count of flights by aircraft" query returns dozens of singleton buckets instead of a useful distribution. The reviewer who wrote "A330" and the one who wrote "A330-300" flew the same plane family — but the dataset treats them as different aircraft.

We want a new column `Aircraft family` that maps every input to a canonical family code, drawn from a fixed vocabulary covering the major commercial families. Multi-aircraft entries (the reviewer flew one type on the outbound and another on the return) produce a pipe-separated list of canonical codes, ordered as they appear in the input.

Vocabulary (family codes):

- `Airbus A220`, `Airbus A320`, `Airbus A330`, `Airbus A340`, `Airbus A350`, `Airbus A380`
- `Boeing 717`, `Boeing 737`, `Boeing 747`, `Boeing 757`, `Boeing 767`, `Boeing 777`, `Boeing 787`
- `Embraer E-Jet`, `Bombardier CRJ`, `Bombardier Dash 8`, `ATR 42/72`, `Fokker`
- `Other` — fits an aircraft type not in the vocabulary
- `N/A` — input empty, unreadable, or not an aircraft

This column lets you build the visualization the raw data refused: distribution of reviews by aircraft family, average rating per family, complaint patterns per family.

## The prompt

In **Add column by AI**, name the new column `Aircraft family` and paste this instruction:

```
You are an aviation reconciliation expert. Read the input aircraft string and
map it to one or more canonical aircraft family codes from the fixed vocabulary
below. Do not invent new codes.

Vocabulary (output exactly these strings, character for character):
- Airbus A220
- Airbus A320       (covers A318, A319, A320, A321 and their variants like A320neo)
- Airbus A330       (covers A330-200, A330-300, A330neo)
- Airbus A340
- Airbus A350       (covers A350-900, A350-1000)
- Airbus A380
- Boeing 717
- Boeing 737        (covers 737-300, -700, -800, -900, MAX, etc.)
- Boeing 747
- Boeing 757
- Boeing 767
- Boeing 777        (covers 777-200, -300, 777X)
- Boeing 787        (covers 787-8, 787-9, 787-10 Dreamliner)
- Embraer E-Jet     (covers E170, E175, E190, E195, E2 variants)
- Bombardier CRJ    (covers CRJ-100, -200, -700, -900, -1000)
- Bombardier Dash 8 (covers Q200, Q300, Q400)
- ATR 42/72
- Fokker            (covers Fokker 50, 70, 100)
- Other
- N/A

Rules:
1. Strip variant suffixes and map to the family. "A330-300" -> "Airbus A330".
   "Boeing 787-9" -> "Boeing 787". "CRJ-900" -> "Bombardier CRJ".
2. If the input lists multiple aircraft (separated by "/", "and", or commas),
   output each family code in order, separated by " | ". Deduplicate within
   the same input: "A330-300 / A350-900" -> "Airbus A330 | Airbus A350";
   "Boeing 737-800 / Boeing 737-900" -> "Boeing 737".
3. "Dreamliner" -> "Boeing 787". "Jumbo" or "Queen of the Skies" -> "Boeing 747".
4. If the aircraft is real but not in the vocabulary (Antonov, Ilyushin,
   Tupolev, McDonnell Douglas, Sukhoi Superjet, etc.), output "Other".
5. If the input is empty, says "Unknown", "?", or is clearly not an aircraft,
   output exactly: N/A
6. Output only the canonical code or pipe-separated list. No punctuation, no
   quotes, no explanation.

Examples:
- Input: "Boeing 737-800"                 -> Boeing 737
- Input: "A330-300"                       -> Airbus A330
- Input: "A330-300 / A350-900"            -> Airbus A330 | Airbus A350
- Input: "Boeing 787-8 and Boeing 787-9"  -> Boeing 787
- Input: "Embraer 195"                    -> Embraer E-Jet
- Input: "Bombardier CRJ-900"             -> Bombardier CRJ
- Input: "Fokker 100"                     -> Fokker
- Input: "A320 / Boeing 767-300"          -> Airbus A320 | Boeing 767
- Input: "Antonov An-148"                 -> Other
- Input: ""                               -> N/A
```

Set the response format to **Text**.

## Limits of the result

Fuzzy matching with a closed vocabulary is well within reach of a 3B model when the vocabulary covers most of the input distribution. Expect a 3 to 5 percent error rate concentrated on the cases below.

**Variant boundary calls.** The A320 family includes the A318, A319, A320, and A321. Some analysts argue the A321XLR is distinct enough to deserve its own bucket. The model follows whatever you put in the vocabulary — if you need finer granularity (e.g. separate A220-100 from A220-300), expand the vocabulary explicitly and re-run.

**Regional jets confusion.** Embraer E-Jet, Bombardier CRJ, and Bombardier Dash 8 occupy similar market segments and reviewers often misname them ("CRJ" when they actually flew an Embraer 175). The model trusts the input. Cross-check by faceting `Aircraft family` against `Airline Name`: airlines that do not operate a given family but show up with it are likely mislabeled at the source — not by the model.

**Cargo / military / private. ** Anything outside passenger commercial aviation will be labeled `Other`. If your dataset contains those, add categories to the vocabulary or accept the lossy bucket.

**Multi-aircraft ordering.** Rule 2 preserves input order. But the input order is reviewer-chosen and unreliable (some put the outbound first, others the most recent leg first). If you need outbound-vs-return, you need a different prompt that uses the `Route` and `Date Flown` columns as context — not feasible row-by-row in the current AI Extension.

**Format violations.** Around 1 to 2 percent of outputs come back with extra punctuation, lowercase variants, or extra spaces inside the pipe separator. Add a GREL guard to canonicalize:

```grel
value.split("|").map(v,
  filter(
    ["Airbus A220","Airbus A320","Airbus A330","Airbus A340","Airbus A350","Airbus A380",
     "Boeing 717","Boeing 737","Boeing 747","Boeing 757","Boeing 767","Boeing 777","Boeing 787",
     "Embraer E-Jet","Bombardier CRJ","Bombardier Dash 8","ATR 42/72","Fokker","Other","N/A"],
    canon, v.trim().toLowercase() == canon.toLowercase()
  )[0]
).filter(v, v != null).uniques().join(" | ")
```

Empty result after the guard means the row needs manual inspection.

**Cross-check against airline fleet.** For published work, validate the reconciled column against a reference fleet list (Wikipedia, ch-aviation, the airline's own site). The model knows mainstream fleets well but invents plausible-sounding mismatches on smaller carriers. Sample 30 rows per major airline before claiming the reconciled column is publication-ready.

**Why this matters for the pitch.** This transformation is the canonical "Smart Disambiguation" case: same real-world entity expressed in many surface forms, no GREL pattern strong enough to reconcile them, no clustering algorithm in OpenRefine (KNN, fingerprint) able to merge "A330" with "A330-300" and "Boeing 787-8 and Boeing 787-9" with "Boeing 787". The LLM brings the world knowledge that closes that gap — and the closed vocabulary keeps the output deterministic enough to defend.

**Runtime.** One pass at roughly one second per row on the full 23,000-row dataset is about six hours of inference on a laptop. The `Aircraft` column has many empty values (around 60 percent in the sample), so the effective runtime is lower if you facet to non-empty before running.

## Model settings and why

The provider configuration in OpenRefine for this transformation:

| Field | Value |
|---|---|
| Model | `ministral-3:3b` |
| Temperature | `0.1` |
| Top-P | `0.9` |
| Seed | `42` |
| Max Tokens | `32` |
| Wait Time | `0` |

**Temperature 0.1.** Reconciliation has a single correct answer per input. The model needs minimal flexibility — just enough to weigh "A320 / Boeing 767-300" as two distinct families rather than locking onto the first one. Pure 0.0 fails on multi-aircraft inputs in roughly 1 percent of cases by dropping the second family.

**Top-P 0.9.** Standard tight value. Critical here because the model is constantly tempted to output near-variants of the vocabulary ("Airbus 330" without the A, "Boeing 737NG" with a generation suffix). Tight top-p suppresses those low-probability deviations.

**Seed 42.** Locks reproducibility. Essential on a reconciliation task because the column you produce becomes a join key for downstream aggregation — if the seed drifted, the same input row could change family between two runs and corrupt your aggregates without warning.

**Max Tokens 32.** Larger than the binary tasks because multi-aircraft outputs can stretch to four families joined by pipes (rare but possible: `Airbus A320 | Boeing 737 | Boeing 787 | Other` is 47 characters). 32 tokens gives headroom while still cutting off the model if it tries to add an explanation.

**Wait Time 0.** Local inference.

**Model choice.** Ministral 3B handles the major Airbus and Boeing families reliably but is less confident on regional jets (CRJ, Dash 8, ATR variants) and uncommon types (Antonov, Tupolev). For a publication-grade reconciliation, swap to `mistral:7b` or `qwen2.5:7b` — both have stronger aviation knowledge in training and reduce the `Other` and misclassification rates by roughly half.
