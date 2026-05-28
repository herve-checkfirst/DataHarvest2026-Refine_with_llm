# Beyond Data Cleaning: Enhancing OpenRefine with LLM

Companion repository for the Dataharvest 2026 session **Beyond data cleaning: Enhancing OpenRefine with LLM**, presented by Hervé Letoqueux (Checkfirst).

- **When:** Saturday, May 30, 2026, 16:15 – 16:45 CEST
- **Where:** Room 3.05
- **Session page:** [dataharvest26.sched.com/event/2LYtF](https://dataharvest26.sched.com/event/2LYtF/beyond-data-cleaning-enhancing-openrefine-with-llm)

## What this repository contains

This repo bundles everything needed to reproduce the live demo and to extend it after the session:

- An installation tutorial for running OpenRefine with a local LLM via Ollama, on macOS, Linux, and Windows
- Six transformation worksheets graded from basic to advanced, each one a self-contained recipe for a real data-cleaning task
- The sample dataset used throughout the demo and a 100-row excerpt for fast iteration
- A `slides/` directory containing the conference presentation in PDF

## Files

| File | Purpose |
|---|---|
| [openrefine-llm-tutorial.md](openrefine-llm-tutorial.md) | Step-by-step setup for OpenRefine + Ollama + Ministral 3B, including hardware requirements for Apple Silicon and Linux/Windows machines |
| [00-transform-review-date.md](00-transform-review-date.md) | Basic — Date normalization to ISO 8601. Baseline that shows when GREL beats an LLM |
| [01-transform-flight-type.md](01-transform-flight-type.md) | Basic — Binary classification: direct versus connecting flights |
| [02-transform-route-extraction.md](02-transform-route-extraction.md) | Advanced — Parse free-text routes, resolve IATA codes, emit structured multi-segment output |
| [03-transform-factual-incident.md](03-transform-factual-incident.md) | Intermediate — Distinguish concrete events from subjective opinion. Useful as a pre-filter for downstream extractions |
| [04-transform-primary-complaint.md](04-transform-primary-complaint.md) | Intermediate — Classify the dominant complaint into a fixed eleven-label vocabulary |
| [05-transform-aspect-sentiment.md](05-transform-aspect-sentiment.md) | Advanced — Four parallel sentiment columns (seat, crew, food, punctuality) per review |
| [Airline_review.csv](Airline_review.csv) | Full dataset (23,171 reviews, 20 MB) |
| [Airline_review_sample100.csv](Airline_review_sample100.csv) | 100 random reviews for fast iteration during prompt tuning |
| [slides/](slides/) | Conference presentation in PDF |

## The dataset

The reviews come from the public Kaggle dataset **Airline Reviews** by Juhi Bhojani: [kaggle.com/datasets/juhibhojani/airline-reviews](https://www.kaggle.com/datasets/juhibhojani/airline-reviews). It contains roughly 23,000 user-submitted reviews of airlines worldwide, scraped from a review aggregator, with mixed structured (ratings, recommendation, route) and unstructured (free-text review) columns. It is well-suited to demonstrating LLM-assisted data wrangling because it combines:

- A column with multiple inconsistent formats (`Route`)
- A column with rich narrative text (`Review`)
- A date column in a non-sortable English ordinal format (`Review Date`)
- Numeric ratings that often contradict the prose, which makes consistency checks possible

The 100-row sample was generated with `random.seed(42)` for reproducibility — the same seed produces the same rows on any machine.

## Why OpenRefine — and why reproducibility matters

OpenRefine is not just a spreadsheet with extra features. Every action you perform on a column is recorded in a JSON operation history that you can export, share, and replay on a different dataset. That property is what makes OpenRefine a serious tool for investigative journalism, regulatory analysis, and any work where a third party may need to audit how raw data became published findings.

Adding an LLM to the workflow puts that reproducibility at risk. Language models are non-deterministic by default: the same prompt on the same row can produce different outputs across runs, across machines, and across model versions. If you cannot reproduce your column transformation, you cannot defend it.

This repository takes reproducibility seriously. Every transformation worksheet specifies:

- The **exact model and tag** (`ministral-3:3b`) so the version is pinned
- The **temperature, top-p, seed, and max tokens** values, with an explanation of why each was chosen
- A **GREL guard column** that validates the output format and flags rows that need re-running
- Explicit **limits and known failure modes** so a reader knows what they should manually verify

The seed value (`42` throughout) locks the sampling RNG so re-running a column produces identical output. The combination of low temperature and a fixed seed brings LLM transformations close to deterministic in practice, which is what makes them defensible in a published investigation.

If you adapt these prompts to your own data, keep the same discipline: pin the model, fix the seed, document the settings, and write a guard column. The point of using OpenRefine over a Python script is the operation history that anyone can replay — do not throw it away by treating the LLM as a black box.

## Quick start

1. Install OpenRefine 3.8.7 or later from [openrefine.org/download](https://openrefine.org/download).
2. Follow [openrefine-llm-tutorial.md](openrefine-llm-tutorial.md) to install Ollama, pull the Ministral 3B model, and configure the AI Extension.
3. Open `Airline_review_sample100.csv` in OpenRefine.
4. Work through the transformation worksheets in order (`00` to `05`), running each prompt on the sample and comparing the output against the documented expectations.
5. Once a prompt produces stable output on the sample, optionally run it on the full `Airline_review.csv` — expect runtime in the range of several hours per pass on a CPU-only machine.

## License and credits

- Dataset: [Airline Reviews](https://www.kaggle.com/datasets/juhibhojani/airline-reviews) by Juhi Bhojani on Kaggle, used under its original license.
- AI Extension for OpenRefine: [sunilnatraj/llm-extension](https://github.com/sunilnatraj/llm-extension).
- Worksheets and tutorial: authored for the Dataharvest 2026 session by Hervé Letoqueux, Checkfirst.
