# Transformation: Physical Harm Reported

Complexity level: advanced. This transformation flags reviews that describe physical illness, injury, or medical events affecting the reviewer or other passengers. It is the most directly safety-relevant worksheet in the suite and pairs well with the incident extraction in `09`.

**Read this before running.** Just as with the discrimination signal in worksheet `10`, the LLM detects the surface pattern of a harm report, not whether harm actually occurred. The output column is a triage tool that surfaces accounts for human review — not a measurement of safety performance.

## What we want to do

Many airline reviews mention physical effects: dehydration on long flights, asthma exacerbated by poor air quality, falls during turbulence, food poisoning, panic attacks, evacuations, injuries from falling luggage. These accounts are scattered across the `Review` text and invisible to any structured column. Aggregated and faceted by airline and route, they reveal patterns that ratings and recommendations cannot — patterns that may indicate cabin air quality issues, inadequate crew training for medical emergencies, or operational practices with safety implications.

We extract a new column `Physical harm reported` with these values:

- `none` — no physical effect mentioned
- `discomfort` — non-medical discomfort (cramped seat, cold cabin, mild dehydration, ear pressure, fatigue)
- `minor` — minor illness or injury without medical intervention (nausea, vomiting, headache, bruise, minor cut, food poisoning resolved at home)
- `serious` — significant medical event requiring intervention (medical emergency on board, hospitalization, fall with injury, evacuation, allergic reaction, anaphylaxis, broken bone, surgery, ambulance)
- `to-other` — the harm described affected another passenger, not the reviewer (witness account)
- `N/A` — empty or unreadable

The `to-other` separation is important because witness accounts are valuable for incident research but represent a different evidentiary tier than first-person reports.

## The prompt

In **Add column by AI**, name the new column `Physical harm reported` and paste this instruction:

```
You are a content classifier for airline reviews. Read the review text and
identify whether the reviewer reports physical harm — to themselves or to
another passenger they describe.

Output exactly one of these values, nothing else:
- none         (no physical harm or illness mentioned)
- discomfort   (non-medical discomfort: cramped seat, cold cabin, mild
                dehydration, ear pressure, fatigue, hunger)
- minor        (minor illness or injury without medical intervention:
                nausea, vomiting, headache, bruise, minor cut, food
                poisoning treated at home, mild allergic reaction)
- serious      (significant medical event requiring intervention: medical
                emergency on board, hospitalization, fall with injury,
                ambulance, evacuation, anaphylaxis, broken bone, surgery,
                cardiac event, severe allergic reaction)
- to-other     (the physical harm described affected another passenger,
                not the reviewer themselves. Choose this even if the
                witnessed harm is serious.)
- N/A          (empty or unreadable input)

Rules:
1. Physical complaints about the seat or cabin environment alone (e.g.
   "the seat hurt my back") -> discomfort, not minor.
2. Medical events handled by the crew or paramedics on arrival -> serious.
3. Vague references like "I felt unwell" without further detail -> minor.
4. If the reviewer reports harm to themselves AND witnesses harm to others,
   classify based on the reviewer's own harm (their first-person account
   takes priority over the witness component).
5. Psychological distress ("I cried", "panic attack") counts as minor if the
   reviewer describes it explicitly with physical symptoms (shaking, racing
   heart). Pure emotional reaction without physical symptom -> none.
6. Do not assess whether the harm was the airline's fault. Only classify
   what the reviewer describes happening to a body.
7. Output only the single label. No punctuation, no quotes, no explanation.

Examples:
- Review: "Cramped seat, painful flight, terrible food." -> discomfort
- Review: "I had food poisoning from the chicken meal, spent two days
  recovering at home." -> minor
- Review: "A passenger collapsed during the flight and was met by
  paramedics on arrival." -> to-other
- Review: "I fell during heavy turbulence and broke my wrist, was taken to
  hospital after landing." -> serious
- Review: "The cabin air was so dry I had a nosebleed and headache."
  -> minor
- Review: "Worst service, would not recommend." -> none
- Review: "" -> N/A
```

Set the response format to **Text**.

## Limits of the result

The harm-detection task is editorially sensitive in a different way than discrimination detection: errors here can both miss real safety patterns and over-state them. Read all of this section.

**The label is a report, not a finding.** A `serious` label means the reviewer wrote about a serious medical event. It does not mean the event occurred, that it was the airline's fault, or that the airline failed to respond appropriately. The column points the analyst at accounts worth reading; it does not draw conclusions.

**Discomfort versus minor harm boundary.** A passenger who writes "my back hurt the whole flight" might be reporting a chronic pre-existing condition aggravated by the seat, or a transient discomfort, or a serious injury developing over hours. The model collapses this to `discomfort` by default, which is editorially safer but loses signal. If you are investigating cabin ergonomics or seat design, read the `discomfort` rows too.

**Witness account weighting.** The `to-other` category is intentionally separate. Witness accounts are valuable but should be weighted differently than first-person reports in any quantitative analysis. The same medical event can be described in three reviews — the patient's and two witnesses' — and an unweighted count triple-counts it.

**False positives from food and beverage complaints.** Reviewers often describe food as "making me sick" rhetorically. The model can take this literally and label `minor`. Spot-check: any `minor` row tied to food complaints needs a manual read before counting.

**Under-detection of cabin air quality cases.** Reviewers may describe headaches, dizziness, nausea without connecting them to cabin air quality. The model labels them `minor` correctly but the cabin-air-quality story is lost. If you are investigating cabin air, run a follow-up prompt on the `minor` subset asking whether cabin air quality is implicated.

**Cross-check with `Verified`.** A serious harm report from an unverified reviewer is not necessarily false, but it is a different evidentiary tier than a verified report. Surface this combination as a separate cohort for editorial review.

**Format violations.** Around 1 to 2 percent of outputs come back with extra characters or capitalization. Add a GREL guard:

```grel
forEach(
  ["none","discomfort","minor","serious","to-other","N/A"],
  label, if(value.trim().toLowercase() == label.toLowercase(), label, null)
).filter(v, v != null)[0]
```

**Composition with worksheet 09.** Rows labeled `serious` should be cross-checked against `Incident type` from worksheet `09`. Mismatches (a `serious` harm but `Incident type = Delay`) often mean the model missed the actual incident type — read those rows.

**Legal and editorial process.** Before publishing any finding from this column, have your editorial process review every `serious` row by hand, attempt to corroborate through external records (NTSB, AAIB, EASA, regulatory filings, news archives), and contact the airlines named in those accounts. The dataset itself is not evidence; it is a discovery tool.

**Runtime.** Long reviews mean longer inference. Budget around 1.5 seconds per row, around 10 hours for the full 23,000-row dataset on a CPU laptop. Sample first.

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

**Temperature 0.1.** Six-value classification on a graded scale (discomfort → minor → serious). The model needs minimal flexibility to weigh severity correctly. Pure 0.0 makes it latch onto the first physical-symptom word it sees regardless of the broader description; 0.1 lets it consider the full context (was there medical intervention, was the symptom resolved, was it pre-existing).

**Top-P 0.9.** Standard tight value. Suppresses tempting near-vocabulary outputs ("moderate", "severe", "trauma", "unwell") that would violate the closed vocabulary.

**Seed 42.** Locks reproducibility. **Mandatory** for safety-relevant classifications because findings derived from this column need to be reproducible by an editor, fact-checker, regulator, or counsel. An unseeded run is not auditable.

**Max Tokens 8.** Output is at most one hyphenated word. 8 tokens fits "to-other" with margin and truncates explanation attempts.

**Wait Time 0.** Local inference.

**Model choice.** Ministral 3B handles `none`, `discomfort`, and obvious `serious` cases reliably. The `minor` versus `serious` boundary and the `to-other` distinction are where it slips. For investigation-grade work on safety patterns, this is another worksheet where model size matters. Swap to `mistral:7b` or `qwen2.5:7b`, and **run the same prompt with two different models, only trusting `serious` labels where both agree**. Disagreement rows are interesting and should be read individually.

**Pairing with worksheet 09.** Run `09-transform-incident-extraction.md` first on the factual-incident subset, then run this prompt on the same subset. The two columns cross-checked give a much richer picture of safety-relevant accounts than either alone — and the cross-check itself catches model errors in both columns.
