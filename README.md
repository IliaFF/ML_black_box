# Dz5 ML — 1st Place Solution

**Score: 88.735 · 1st place**

Physics regression: reverse-engineer black-box signal voltage `y = f(x)` from 13 particle detector features via active learning and formula discovery.

---

## Problem

- **Target:** predict signal voltage `y` from 13 detector physics features
- **Key features:** `muon_flux` (μ), `cerenkov_angle` (θ), `track_curvature` (tc)
- **Active zone:** rows where `max|xᵢ| < 9.07` — 91,685 test rows (of ~100K total)
- **Outside active zone:** `y = 0` (gate functions clip to zero)
- **Training data:** 44,604 rows collected via active learning (API, ~29K in active zone)

---

## Approach

Two-stage model:

```
y_pred = Formula_V2(μ, θ, tc, gates)  +  LightGBM(formula_components)
```

### Stage 1 — Physics Formula V2 (44 parameters)

Discovered by iterative curve fitting, starting from `y ≈ A·sin(θ)` and expanding:

```
Fv(μ)   = A_bw · μ² / ((|μ| − a_bw)² + gW²)         # Breit-Wigner resonance
Hv(μ)   = AH · [1/((μ−mH)²+gH²) − 1/((μ+mH)²+gH²)] # antisymmetric BW

gθ(θ)   = rational(θ) + BT·θ·exp(−eT·(θ−cT)²)        # piecewise: θ<0 / θ≥0
C(θ)    = C₀ + A·sinh(k₂·|θ|/n) + D·θ·exp(−e·(θ−c)²) # piecewise: θ<0 / θ≥0

cos_c   = cos(k₂·μ + tc²/β_tc)                         # phase modulation
G_clip  = clip(10 − |x|, 0, 1)                          # locality gate (each feature)

y = [Fv·(1 + α·tc²)·sin(k₁·θ) + Hv + gθ·cos_c + μ·tc·C(θ)] · ∏G_clip
```

Key constants: `k₁ = 2.500`, `k₂ = 1.799`

Optimized with Nelder-Mead (5 restarts × 100K iterations) + Powell polish.

### Stage 2 — LightGBM Residual Correction

**Features (23 total):**
- 16 formula components: `μ, θ, tc, tc², |tc|, tc·μ, tc·θ, Fv, Hv, sin_th, gθ, cos_c, C, Fv·sin_th, μ·tc·C, full_core`
- 7 zone indicators: `floor(θ), floor(|tc|/2), |tc|>5, |tc|>7, |μ|>7, θ≥0, floor(|μ|/3)`

**Target:** `y − Formula_V2(x)` (residuals)

**Training:** 10-fold CV with early stopping (patience=300)

**Hyperparameters** (Optuna, 150 trials):
```python
num_leaves=16, learning_rate=0.019, reg_lambda=1.39,
reg_alpha=1.54, subsample=0.946, colsample_bytree=0.446
```

**Iterative refinement:** alternate formula refit on `y − lgbm_oof` and LGBM retrain on `y − formula`. One iteration: OOF 1.4346 → 1.4199.

---

## Results

| Stage | RMSE |
|-------|------|
| Baseline (sin only) | 88.0 |
| Formula V2 (44 params) | 9.88 |
| LGBM on 3 raw features | 2.44 |
| LGBM on 16 formula components | 1.72 |
| + Optuna tuning | 1.44 |
| + Zone features | 1.44 |
| **Iterative formula↔LGBM (final)** | **1.42** |
| **Kaggle (public leaderboard)** | **88.735 · 1st place** |

---

## Key Findings

1. **Formula components as features** beat raw inputs by 0.6+ RMSE — using Fv, Hv, gθ, cos_c, C as LGBM inputs, not μ/θ/tc directly
2. **Zone features need small trees** — `floor(θ)`, `|tc|` bands only help with `num_leaves=16` (Optuna found this)
3. **Pseudo-labeling is dangerous** — OOF improves but Kaggle score worsens (encodes model bias)
4. **Two-zone splitting doesn't help** — single 10-fold model beats separate easy/hard models
5. **Active threshold = 9.07** — found by 7 Kaggle submissions; 9.065 and 9.075 both worse
6. **Iterative refinement works** — one iteration of formula↔LGBM cycle improves OOF from 1.4346 → 1.4199

---

## Discovery Timeline

10 key discoveries from `y ≈ A·sin(θ)` to the final ensemble:

| # | Discovery | RMSE |
|---|-----------|------|
| 1 | `y ≈ A·sin(k₁·θ)` — baseline oscillation | 88.0 |
| 2 | Breit-Wigner poles F(μ), H(μ) | 53.8 |
| 3 | Locality: G_clip gates, noise floor | ~36 |
| 4 | Cosine phase term `cos(k₂·μ + tc²/β)` | 36.7 |
| 5 | `gθ(θ)` rational + Gaussian structure | ~15 |
| 6 | `C(θ)` sinh + Gaussian exponentials | 9.47 |
| 7 | LGBM residual correction (3 features) | 2.44 |
| 8 | 5-fold → 10-fold ensemble | 1.72 |
| 9 | 23 formula-component features + Optuna | 1.44 |
| 10 | Active threshold optimization (9.07) | **1.42 · 88.735** |

---

## Interactive Visualizations

Open HTML files locally in a browser. Start from [index.html](index.html) — navigation hub for all 15 visualizations.

### Key Dashboards

| File | Description |
|------|-------------|
| [index.html](index.html) | Navigation hub — all 15 visualizations |
| [discoveries_timeline.html](discoveries_timeline.html) | 10-step discovery chronology with formulas and algorithms |
| [score_timeline.html](score_timeline.html) | RMSE/R² progression across all 3 phases |
| [formula_terms.html](formula_terms.html) | 6-panel chart of each formula term: Fv, Hv, sin, gθ, C, G_clip |

### Interactive 3D Formula Surfaces

| File | Description |
|------|-------------|
| [formula_realtime.html](formula_realtime.html) | All three surfaces — 3 global sliders (μ, θ, tc) update all simultaneously |
| [formula_mu_theta.html](formula_mu_theta.html) | y(μ, θ) — slider for tc |
| [formula_mu_tc.html](formula_mu_tc.html) | y(μ, tc) — slider for θ |
| [formula_theta_tc.html](formula_theta_tc.html) | y(θ, tc) — slider for μ |

### SHAP & Explainability

| File | Description |
|------|-------------|
| [shap_beeswarm.html](shap_beeswarm.html) | SHAP beeswarm — feature importance across all rows |
| [shap_scatter.html](shap_scatter.html) | SHAP scatter — nonlinearities and interactions per feature |

### Data & Discovery Plots

| File | Description |
|------|-------------|
| [3d_data_raw.html](3d_data_raw.html) | 3D scatter of raw y(μ, θ) data — locality visible immediately |
| [plot_20a_y_mu_theta.html](plot_20a_y_mu_theta.html) | First view of y(μ, θ) heatmap — Breit-Wigner × sin structure |
| [plot_20b_y_vs_theta_slices.html](plot_20b_y_vs_theta_slices.html) | y vs θ slices by μ — sin(k₁θ) structure confirmed |
| [plot_20c_y_vs_mu_slices.html](plot_20c_y_vs_mu_slices.html) | y vs μ slices by θ — F(μ) Breit-Wigner extraction |
| [plot_21_theta_func.html](plot_21_theta_func.html) | gθ(θ) structure — rational + Gaussian, asymmetric branches |
| [plot_3d_residuals.html](plot_3d_residuals.html) | 3D residuals after Formula V2 — what LGBM corrects |

---

## Repository Files

| File | Description |
|------|-------------|
| [report_final.ipynb](report_final.ipynb) | Final analysis notebook — full pipeline, zone analysis, results |
| [work_report_final.md](work_report_final.md) | Final work report — methodology and findings |
| [early_stages_full.md](early_stages_full.md) | Full log of early-stage experiments before formula discovery |
| [formula_v2.md](formula_v2.md) | Formula V2 reference — all 44 parameters and derivation |
| [presentation_speech.md](presentation_speech.md) | Presentation script for the competition results |

---

## Experiment Log Summary

From BigPC final experiments (10-fold, OOF RMSE):

| Exp | Features | OOF RMSE | Notes |
|-----|----------|----------|-------|
| baseline | 3 raw | 2.437 | leaves=63, λ=10 |
| Exp1 | 11 engineered | 2.369 | +tc², sin_th, tc·μ |
| Exp2 | **16 formula comps** | **1.723** | key breakthrough |
| Exp2-opt | Exp2 + Optuna 150 | 1.444 | leaves=26, lr=0.013 |
| zone-opt | 23 feat + Optuna | 1.435 | leaves=16, lr=0.019 |
| iterative | formula↔LGBM | **1.420** | **final submission** |

Experiments that failed: pseudo-labeling (OOF 1.399 but Kaggle worse), two-zone split (1.657), sample weighting `|tc|>5` x2 (1.438).

---

## Active Learning Data Collection

Data collected via AIM API over ~2 weeks, 44,604 points total:

- Initial uniform grid: 500 points in [−10, 10]¹³
- Bayesian active learning (UCB acquisition) targeting high-uncertainty regions
- Focused scans on formula residual peaks (high-|μ| zones, boundary regions)
- Final round: 1,320 points in worst θ×|tc| zones

---

## Stack

- **Formula optimization:** `scipy.optimize.minimize` — Nelder-Mead + Powell
- **Gradient boosting:** `lightgbm` — 10-fold CV with early stopping
- **Hyperparameter search:** `optuna` — TPE sampler, 150 trials
- **Visualizations:** `plotly`, `bokeh`, SHAP
- **Data:** `pandas`, `pyarrow` (Parquet)
- **Active learning:** custom Bayesian linear regression (closed-form, RFF features)
