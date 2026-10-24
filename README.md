# exdom
# EXDOM: explainable domain-model extraction from user stories

Prototype, frozen configurations, cached LLM outputs, trace books, rating sheets and results for the paper

> **EXDOM: An Explainable Framework for Automatic Domain Model Extraction from User Stories** .
> Authors: **Shohre Ajoudanian*, Sondos Bahadori and
Maryam Nooraei Abadeh**.

EXDOM extracts a UML class model (classes, attributes, associations) from English user stories. It combines

1. a **rule path** (spaCy dependency patterns R0–R3),
2. a **guided LLM path** (an open Qwen2.5 model that must quote an exact substring of the story as evidence; deterministic filters discard unverifiable suggestions),
3. a **scoring layer** (evidence signals, cross-story support, accept/gray/low zones),
4. a **constraint engine** (checks C1–C6 with every adjustment logged), and
5. **template explanations** with counterfactual scores and a **trace book** (one JSON record per decision, with evidence spans, signals, constraint log and configuration version).

The paper reports a reproducible evaluation on a public benchmark. Everything needed to recompute the reported numbers is in this repository; the LLM outputs are cached, so no new generation is required.

## What the evaluation shows (short version)

Benchmark: Bragilovski et al., nine user-story projects with gold classes and associations (487 stories). Fixed project-level split (seed 13): DEV = fishchips, sports, school, ticket, recycling; TEST = planningpoker, camperplus, grocery, supermarket. TEST was run once per model with the frozen configuration.

| Model | TEST class F1 (relaxed), EXDOM | LLM + scoring | Rules + scoring | Raw LLM |
|---|---|---|---|---|
| Qwen2.5-3B-Instruct (fp16) | 0.519 | 0.473 | 0.426 | 0.473 |
| Qwen2.5-7B-Instruct (4-bit) | 0.556 | 0.516 | 0.426 | 0.485 |

The advantage of combining both paths is small and was **not** statistically confirmed under our pre-specified criterion (leave-one-project-out over nine projects, Holm-corrected sign test). Association recovery is weak, and the constraint engine did not improve measured quality. See the paper (Sections 4 and 5) for all tables, negative results and limitations.

## Repository layout

Suggested layout (adapt the names to what you upload):

```
notebooks/
  exdom_pipeline_v2.ipynb          # Qwen2.5-3B pipeline (Google Colab, T4)
  exdom_pipeline_qwen7b.ipynb      # Qwen2.5-7B pipeline (4-bit)
qwen2.5-3b/
  frozen_config.json               # configuration frozen on DEV before TEST
  environment.json                 # software versions and benchmark archive md5
  llm_cache.jsonl                  # raw LLM outputs (keyed by model, prompt hash, temperature, run, text)
  TEST_RESULTS.json, LOPO9_RESULTS.json, explanation_stats_TEST.json
  tracebook_<project>.jsonl        # trace book of the four TEST projects
  explanations_*, matching_*       # rating sheets, filled sheets, keys, summaries
qwen2.5-7b/
  (same files for the 7B run)
README.md
LICENSE
```

## How to reproduce

The notebooks are numbered cells meant to be run in order in Google Colab. Cell numbers below follow the notebook headers (`# [n] ...`).

1. **Setup (cells 1–4).** Install packages (`spacy`, `transformers`, `accelerate`, `networkx`, `pandas`, `openpyxl`, `pyyaml`; the 7B notebook also needs `bitsandbytes`), download the spaCy model, and mount Google Drive. Set `OUT` (cell 17) and `CACHE_PATH` (cell 7) to the folder that holds the cache and results.
2. **Pipeline code (cells 5–10).** Rule path, LLM path with cache and filters, merge and scoring, constraint engine, explanations and trace book. The two notebooks have identical code here except for model loading (cell 6), the cache path, and a tolerant JSON parser in cell 7 of the 7B notebook.
3. **Benchmark (cells 11–13).** Cell 11 downloads the benchmark archive. **The benchmark is not redistributed here.** Get it from Zenodo ([doi:10.5281/zenodo.15223349](https://doi.org/10.5281/zenodo.15223349), CC BY 4.0). The archive we used has MD5 `f628b349b6f9cb9f6f196fce5d35a899`.
4. **DEV pools and tuning (cells 14–18).** Candidate pools are built from the cache. Cell 18 re-runs the 252-configuration grid on DEV and writes `frozen_config.json`. The 3B notebook asserts that the DEV numbers reproduce the earlier analysis (it stops with "pipeline drift" otherwise).
5. **TEST (cell 19).** One-shot evaluation. Set `CONFIRM_RUN_TEST = True` only after checking the frozen configuration. Writes `TEST_RESULTS.json`.
6. **Process statistics and trace books (cell 20)** and **leave-one-project-out over nine projects (cells 22–23, post hoc).**
7. **Human evaluation (cells 24–27).** Generates the rating sheets, and analyzes the filled sheets (quadratic-weighted kappa, name-matching agreement).

**Cache and GPU.** All LLM outputs used in the paper are in `llm_cache.jsonl`. A cache hit returns the stored raw output, so reproducing the reported numbers does not need new generation. The notebooks still load the model in cell 6; we have not tested a run that skips this cell. Without the cache, outputs depend on the GPU, library versions and sampling and will differ slightly (decoding: temperature 0.1, `top_p` 0.8, `top_k` 20, repetition penalty 1.05, seed 1234 + run index, two runs per story).

## Frozen configuration (both models)

| Parameter | Value |
|---|---|
| Rule weight / LLM weight | 0.15 / 0.15 |
| Support weight (fixed) / cap K | 0.20 / 12 |
| Single-path penalty | 0.8 |
| Accept threshold / low threshold | 0.60 / 0.30 |
| System prompt hash (SHA-256 prefix) | `754c50e5` |

The configurations were frozen on 2026-10-15 (3B) and 2026-10-18 (7B), before the single TEST run of each model. The ablation variants (rules + scoring, LLM + scoring) were tuned separately; their selected values are in each `frozen_config.json`.

## Software

Python 3.13.15, spaCy 3.8.16 (`en_core_web_sm` 3.8.0), transformers 5.18.0, PyTorch 2.11.0, networkx 3.7, pandas 2.2.3, accelerate 1.15.0, bitsandbytes 0.50.2 (7B only). See `environment.json` in each model folder (bitsandbytes is not recorded there).

## Data protection

Rating sheets contain only the ratings and optional free-text comments of two raters; names and contact details are not included. Keys that map sheet rows to elements are published after the study and are not needed to rate.

## Licenses

- Code: MIT (see `LICENSE`).
- Results, trace books, caches and rating data: CC BY 4.0.
- The benchmark and its user stories are the work of Bragilovski et al. and are distributed under their own CC BY 4.0 license; the cached LLM outputs quote short spans of those stories.

## How to cite

If you use this repository, please cite the paper and the archived release:

```
AUTHORS. TITLE OF THE PAPER. Software and Systems Modeling, 2026 (under review).
EXDOM prototype and evaluation, v1.0. Zenodo, https://doi.org/10.5281/zenodo.NNNNNNN
```

and the benchmark:

```
Bragilovski, M., van Can, A. T., Dalpiaz, F., Sturm, A. Leveraging machines to derive domain models
from user stories. Requirements Engineering 30(2):241-262, 2025. doi:10.1007/s00766-025-00442-9
```
