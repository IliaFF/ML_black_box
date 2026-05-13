# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Environment

All Python runs via conda (not `.conda_env\python.exe` directly — DLL paths break numpy):

```powershell
D:\Miniforge\condabin\conda.bat run -p D:\Dz5_ML\.conda_env python ...
```

Credentials via env vars: `AIM_USERNAME`, `AIM_PASSWORD`, `AIM_API_URL` (default `https://aim.bioml.ru`). Never stored in repo.

## Key Commands

```powershell
# Dry-run (no quota)
D:\Miniforge\condabin\conda.bat run -p D:\Dz5_ML\.conda_env python -m scripts.collect --dry-run

# Smoke checks only (tiny quota)
D:\Miniforge\condabin\conda.bat run -p D:\Dz5_ML\.conda_env python -m scripts.collect

# Production (requires explicit flag; uses quota)
D:\Miniforge\condabin\conda.bat run -p D:\Dz5_ML\.conda_env python -m scripts.collect --allow-production --max-requests 50 --max-points 5000

# JupyterLab
D:\Miniforge\condabin\conda.bat run -p D:\Dz5_ML\.conda_env jupyter lab
```

Minimum verification: dry-run completes without errors.

## Experiment Protocol

Before any production run, complete 3 smoke checks in order:
1. Dry-run (no quota)
2. API smoke: 1 request, ≤5 points, verify save to `results/`
3. Bayes smoke: 1 request, ≤5 points, verify Bayesian linear fit/predict path

Production gate: pipeline blocks production requests unless `--allow-production` is passed.

## Architecture

**Goal:** Reverse-engineer black-box `y = f(x)` over 13 physics features via active learning + formula search.

**Data:** `results/dataset.parquet` (15 431 rows as of 2026-05-07), `results/requests_log.jsonl`. Features defined in `scripts/config.py` (`N_FEATURES = 13`; key features: `cerenkov_angle`, `muon_flux`, `drift_bias`).

**Pipeline (`scripts/`):**

| Module | Role |
|--------|------|
| `collect.py` | Entry point; orchestrates smoke → production; enforces `cerenkov_angle ∈ [-π/2, π/2]` for API queries |
| `api_client.py` | REST API wrapper (auth, quota, retry) |
| `bayes_linear.py` | Bayesian linear regression — main surrogate (mean/std output) |
| `sampling.py` | Candidate generation (Gaussian + box mix) |
| `storage.py` | Dataset I/O, deduplication, hashing |

**Analysis scripts** live in `scripts/` alongside the pipeline — 32+ architecture benchmark rounds (`architecture_formula_round{N}_*.py`), EDA reports, active-learning campaigns. Outputs go to `results/eda/`, `results/models/`, `results/campaigns/`.

**Known structure (as of 2026-05-07):**
- `y = F(mu)·sin(2.5·θ) + H(mu)` in the active zone
- `F(mu)`: Breit-Wigner power law, peak at |mu|≈9.05, R²=0.9978
- `H(mu)`: odd BW resonance, peak at mu≈7.69, R²=0.9990
- All 11 non-physics features are pure threshold gates (threshold ≈ ±9.5)
- Hard slice (`muon_flux > 0 ∧ drift_bias < -8`) remains unsolved (best hard_r2=+0.077)

## Tracking Files

**Rules:**

- `status.md` — **full project snapshot, regenerated from scratch on every update**. Always overwrite completely. Must reflect: dataset size, quota, best model metrics, open problems, key scripts, next steps.
- `history.md` — **append-only change log**. Never rewrite or delete prior entries. Add a new dated section (`## YYYY-MM-DD (topic)`) for every significant change: new scripts, new model results, new discoveries, dataset updates, bug fixes. One section per logical change, not per file edited.
- `assumptions.md` — working hypotheses (free-form)
- `scripts/summary.md`, `results/summary.md` — per-directory inventories

**When to update:**
- `status.md`: at start of each session (after reviewing current state) and after any significant result
- `history.md`: after running experiments, creating new scripts, discovering new physics, or any other non-trivial change

## Logging and Progress Tracking

Every long-running calculation script must:
1. **Write a log file** to `results/<script_name>.log` (line-buffered, `buffering=1`) in addition to stdout.
2. **Log progress periodically** — at minimum every 500 optimizer iterations or every completed start/epoch. Include: iteration number, current loss/RMSE, elapsed time.
3. **Log each major milestone** — start of each run, best result updates (`***`), ETA, final result summary.
4. **Flush immediately** after each log line (`flush=True` on print, `.flush()` on file).

Standard log pattern:
```python
LOG = open(str(REPO_ROOT / "results" / "script_name.log"), "w", buffering=1)

def log(msg):
    print(msg, flush=True)
    LOG.write(msg + "\n")
    LOG.flush()
```

This allows real-time monitoring via:
```powershell
Get-Content D:\Dz5_ML\results\script_name.log -Wait -Tail 20
```

## Fallback Policy

Silent fallbacks are forbidden. Any fallback must state: what failed, what was used instead, and whether it changes result semantics/quality.

## Language

All responses to the user must be in **Russian**. Code, comments, variable names, file contents, and commit messages remain in English.

## File Headers

Every generated file (Python script, Markdown, CSV description, notebook) must start with a header that:
1. States what the file does (1–3 sentences)
2. Lists related files as wiki-links

**Python scripts** — use a module docstring:
```python
"""Short description of what this script does.

Related:
  [[scripts/config.py]] - feature names and constants
  [[results/eda/output_file.csv]] - output written by this script
  [[scripts/other_script.py]] - upstream dependency
"""
```

**Markdown files** — use a comment block at the top:
```markdown
<!-- Description: what this file tracks/reports.
     Related: [[status.md]], [[history.md]], [[scripts/relevant.py]] -->
```

Wiki-links format: `[[path/to/file]]` — use paths relative to repo root. This allows fast navigation across the project.

## Style

4 spaces, `snake_case`. Keep notebook cells short. PowerShell automation: prefer `powershell -NoProfile` to avoid `PSSecurityException`.
