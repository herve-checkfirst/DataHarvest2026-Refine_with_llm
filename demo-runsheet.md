# Demo Runsheet — Beyond Data Cleaning: Enhancing OpenRefine with LLM

Dataharvest 2026 — live demo segment. Target: 15 minutes, five transformations, building from trivial to investigation-grade.

## Narrative arc

The five fiches are not a random sample. They tell one story:

1. **00 Review Date** — be honest: for this, GREL beats the LLM. Earns the room's trust.
2. **02 Factual Incident** — the journalist's core move: separate verifiable events from opinion.
3. **06 Primary Complaint** — closed vocabulary turns prose into a countable distribution.
4. **07 Aircraft Reconciliation** — the wow moment: unaggregatable chaos becomes clean families.
5. **10 Incident Extraction** — the payoff: reuse the 02 filter, then chain three extractions into investigation data.

The thread to say out loud: 02 *builds* the filter, 10 *consumes* it. That is the composition pattern — filter first, extract second — and it is the real methodological takeaway of the talk.

## Before you start (pre-flight checklist)

- [ ] OpenRefine open on `Airline_review_sample100.csv` (100 rows, not the full 23k — live speed matters).
- [ ] Ollama running, `ministral-3:3b` pulled, AI Extension configured and tested on one row.
- [ ] Provider defaults set: Temperature per fiche, Top-P 0.9, Seed 42, Wait Time 0.
- [ ] Project cloned/duplicated so you can re-run from a clean state if a pass goes wrong.
- [ ] **Safety net:** a second project copy with all five columns PRE-COMPUTED, in case live inference stalls. Switch to it rather than waiting on a frozen progress bar.
- [ ] Prompts copied into a scratch file for fast paste (don't type them live).
- [ ] Browser zoom up so the back row can read the grid.

A note on timing: each LLM pass on 100 rows takes ~1–2 minutes of real inference. Talk through the prompt design *while it runs* — never watch the bar in silence. If you need to fill, narrate the model settings table (temperature/seed/max-tokens reasoning).

---

## 1. Review Date — the honest baseline (1.5 min)

**Column:** `Review Date` -> `Review Date ISO`
**Settings:** Temperature 0.0, Max Tokens 16.

**Say:** "Before anything clever — here is a task an LLM can do, but should not. Watch."

**Do:**
- Show `Review Date`: `26th September 2022`, `9th September 2018`. Point out it does not sort.
- Run the LLM prompt on the sample. While it runs, explain: ordinal suffix, month name to number, zero-padding.
- Show the result column sorts correctly.

**The punchline (do not skip):** flip to the GREL one-liner and run it instantly:
```grel
value.replace(/(\d+)(st|nd|rd|th)/, "$1").toDate("d MMMM yyyy").toString("yyyy-MM-dd")
```
"Same result, zero hallucination, native speed, free. Use the LLM where GREL *can't* reach — that's the rest of this demo."

**Pre-select to show:** rows 1–4 (clean ordinal dates).
**Watch out:** ~0.5% of LLM outputs miss the leading zero (`2022-9-26`). If one shows, that's a teaching moment for the guard column, not an embarrassment.

---

## 2. Factual Incident — separating events from opinion (2.5 min)

**Column:** `Review` -> `Has factual incident` (`yes` / `no` / `N/A`)
**Settings:** Temperature 0.1, Max Tokens 8.

**Say:** "A journalist doesn't care how someone *felt*. They care what *happened*. This pass splits the corpus."

**Do:**
- Read one pure-opinion review aloud and one event review aloud (see pre-selects).
- Run the prompt. While it runs, explain the rule that makes it work: a concrete event needs a duration, a location, a cause, or a quantified consequence — generic "the flight was delayed" is a `no`.
- When done, **add a facet** on `Has factual incident`. Show the split (~40% `yes`).
- "This `yes` subset is now the population I investigate. Hold that thought — we come back to it in the last demo."

**Pre-select to show:**
- A `no`: "Worst seats ever, terrible food, would not recommend."
- A `yes`: "Flight diverted to Lisbon after a passenger had a medical emergency..."

**Watch out:** short "they lost my bags" reviews get over-classified as `yes`. If asked about accuracy, be candid: 5–8% error, this is a triage tool not a measurement. That candor is part of the talk's credibility.

---

## 3. Primary Complaint — prose into a countable distribution (3 min)

**Column:** `Review` -> `Primary complaint` (11-label closed vocabulary)
**Settings:** Temperature 0.1, Max Tokens 8.

**Say:** "Free text can't be counted. A *closed vocabulary* is the trick that makes it countable."

**Do:**
- Show the 11 labels. Stress the design point: without a fixed list the model invents "Late" / "Delay" / "Bad info" and nothing aggregates.
- Run on the sample. While running, explain rule 3 (pick the dominant complaint by length/intensity/position) — this is how multi-complaint reviews collapse to one label.
- **Facet** on `Primary complaint`. Show the distribution bars.
- If time: **text-facet `Airline Name` + facet `Primary complaint`** to show per-airline complaint patterns. This is the "what every reporter wants" view.

**Pre-select to show:** a multi-complaint review (delay + rude staff + baggage) — show it resolves to the dominant one.

**Watch out:** model under-uses "Other" and over-fits positive reviews into a complaint instead of "None". Mention the `Recommended` cross-check column as the sanity test (don't build it live unless asked).

---

## 4. Aircraft Reconciliation — the wow moment (3.5 min)

**Column:** `Aircraft` -> `Aircraft family` (canonical family codes, pipe-separated for multi)
**Settings:** Temperature 0.1, Max Tokens 32.

**Say:** "This is the one no GREL pattern and no OpenRefine clustering can solve."

**Do:**
- Show the raw `Aircraft` chaos. Read a few: `A330`, `A330-300`, `A330-300 / A350-900`, `Boeing 787-8 and Boeing 787-9`.
- "Try OpenRefine's own clustering on this — fingerprint and KNN both fail, because 'A330' and 'A330-300' aren't textually close enough and the model needs to *know* they're the same family." (Optionally open the Cluster dialog and show it doesn't merge them.)
- Run the LLM prompt. While running, explain: strip variant suffix -> family; multi-aircraft inputs become `Airbus A330 | Airbus A350`.
- Show the clean `Aircraft family` column. **Facet** it: dozens of singleton buckets become a usable distribution.

**The line to land:** "Same real-world entity, many surface forms, no deterministic rule strong enough — that's *smart disambiguation*. The LLM brings the world knowledge; the closed vocabulary keeps the output defensible."

**Pre-select to show:** `A330-300 / A350-900` -> `Airbus A330 | Airbus A350` (the multi-aircraft split is the most impressive single row).

**Watch out:** regional jets (CRJ / Embraer / Dash 8) are where the 3B model slips. If pushed on rigor, mention the `mistral:7b` upgrade path halves the error rate, same prompt.

---

## 5. Incident Extraction — the composition payoff (3.5 min)

**Columns (3 sequential passes):** `Incident type`, `Incident location`, `Incident severity`
**Settings:** Temperature 0.1, Max Tokens 16.
**Precondition:** run on the `Has factual incident = yes` facet from demo 2.

**Say:** "Now we cash in the filter from earlier. We don't extract on all 100 rows — only on the events."

**Do:**
- Re-activate the `Has factual incident = yes` facet from demo 2. "We're down to the rows where something happened."
- Run **Pass 1 — `Incident type`** (vocabulary: Diversion, Emergency landing, Medical emergency, ...). Show output.
- Run **Pass 2 — `Incident location`** (`City, Country` / `IATA` / `In flight`).
- Run **Pass 3 — `Incident severity`** (low / medium / high).
- Show the three columns together on one incident row — type + place + severity, structured from a paragraph of prose.

**The line to land:** "One prompt can't do this. Two passes can: filter, then extract. That composition pattern is the thing to take home — reduce the search space first, then pull structure from what's left. It works on *your* data, not just airline reviews."

**If time (10–15 sec):** mention the downstream payoff — map incidents by location, severity distribution per airline, timeline if joined with the ISO date from demo 1. Don't build the viz live.

**Pre-select to show:** the Lisbon medical-emergency diversion from demo 2 — it carries through all three passes cleanly (type=Diversion or Medical emergency, location=Lisbon Portugal, severity=medium/high).

**Watch out:** type/severity can be mutually inconsistent (independent passes). Mention the GREL sanity column that flags `Lost baggage + high` etc. If a pass is slow, this is the moment to fall back to the pre-computed project copy.

---

## Time budget

| # | Fiche | Minutes | Cumulative |
|---|---|---|---|
| 1 | Review Date (baseline) | 1.5 | 1.5 |
| 2 | Factual Incident (filter) | 2.5 | 4.0 |
| 3 | Primary Complaint (vocabulary) | 3.0 | 7.0 |
| 4 | Aircraft Reconciliation (wow) | 3.5 | 10.5 |
| 5 | Incident Extraction (composition) | 3.5 | 14.0 |
| — | Buffer / questions | 1.0 | 15.0 |

## What to cut if you're running long

- Drop the per-airline facet in demo 3 (keep just the overall distribution).
- In demo 5, run only Pass 1 + Pass 3 (type + severity), skip location — the composition point still lands.
- Never cut demo 1's GREL punchline or demo 4 — those are the two moments the room remembers.

## Recurring talking points (weave throughout)

- **Reproducibility:** Seed 42 + low temperature + pinned model tag = re-runnable, defensible output. OpenRefine records every step as replayable JSON. "If you can't reproduce a transformation, you can't defend it in print."
- **Max Tokens as a guardrail:** capping output (8–32 tokens) stops the model appending "yes — because of the six-hour delay", which would break every downstream facet.
- **The guard column:** every fiche ships a GREL validator that catches malformed output and flags rows for re-run. The LLM is never a black box.
- **Honesty about error rates:** 1% (date) -> 5–8% (factual) -> 5–10% per column (incident). Always read the rows before publishing. The 7B upgrade path exists when rigor demands it.

## Sensitive fiches — slides only, not live

`12 discrimination-signal` and `13 review-authenticity` are deliberately **excluded from the live demo**. They need editorial/legal framing that costs time and shifts the tone. Use them on a closing slide instead: "here's what an LLM can *flag* — and why it never reaches print without a human and a lawyer." That shows ethical maturity without risking a live misfire.
