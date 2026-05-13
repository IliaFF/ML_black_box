<!-- Description: полная документация ранних этапов проекта — все идеи, гипотезы, модели, статистика по файлам.
     Related: [[work_report_final.md]], [[history.md]], [[assumptions.md]], [[presentation_speech.md]] -->

# Ранние этапы: все идеи и проверки

---

## 0. Сводная статистика по файлам

| Категория | Файлов | Смысл |
|-----------|--------|-------|
| **Всего .py в scripts/** | **345** | весь проект |
| `architecture_formula_round*.py` | 39 | архитектурные бенчмарки (раунды 2–32) |
| `tmp_*.py` | 125 | быстрые прототипы, разовые эксперименты |
| `scan_*.py` | 24 | прецизионное зондирование функции через API |
| `*_report.py` | 51 | полные аналитические отчёты |
| `*_benchmark.py` | 32 | сравнительные бенчмарки семейств моделей |
| `*_sampler*.py` | 13 | версии активного сэмплера (v1–v5) |
| `eda_*.py` | 5 | разведочный анализ |
| **EDA артефактов в results/eda/** | **1009** | графики, CSV, markdown отчёты |
| Лог-файлов | 26 | прогресс вычислений |

**Уникальных именованных модельных семейств** (по grep в бенчмарках):

| Файл | Семейств |
|------|---------|
| round7 (structural probe) | 21 |
| round12 (full core) | 20 |
| round23 (damped vs piecewise) | 19 |
| round28 (unified latent) | 24 |
| round29 (non-neural reset) | 38 |
| raw_feature_only_structure | 12 |
| **Итого уникальных семейств** | **~130+** |

---

## 1. Исходные гипотезы (01 мая, до данных)

Из `assumptions.md` — первые записи до сбора данных:

```
## 2026-05-01 — Target Function Form
- y ~ sin(cerenkov_angle)    ← доминантная гипотеза
- y ~ tan(cerenkov_angle)    ← альтернатива
- y симметрично по muon_flux (|mu| или mu²)
- y аналитически порождён с аддитивным/мультипликативным шумом
```

**Все 4 гипотезы оказались частично верны:**
- sin(θ) — основа (k₁=2.5, не просто sin)
- tan — нет
- |mu|²-симметрия через BW — да
- Аналитическая структура — да, 44 параметра

---

## 2. Что зондировалось через API (24 scan-файла)

Каждый scan-файл — это специально сконструированный батч запросов к API, где все признаки фиксированы, кроме одного (или двух). Цель: измерить точную форму функции по одному признаку.

| Scan | Цель | Ключевая находка |
|------|------|-----------------|
| `theta_structure` | sweep θ от −9 до +9, остальные = 0 | Основная синусоидальная структура, период ≈ 2.51 рад |
| `mu_fshape` | sweep μ при θ=−8.5 | F(μ) форма — BW-резонанс, пик вне зоны |
| `core_dense` | dense μ range, все ворота=0 | Точное измерение H(μ) и F(μ) |
| `fmu_dense2` | wide + fine + narrow μ scan | Резонанс при |μ|≈11.83, уточнение a_bw |
| `gate_structure` | одиночный gate 0→9 | G_clip(x) = clip(10−|x|, 0, 1) — чистый порог |
| `gate_xi` | gate 0→20 при μ=9, θ=0.5 | Подтверждение: пороговая функция, обнуление при |x|>10 |
| `gates_mu_nonzero` | gates при μ≠0 | Gate не взаимодействует с F(μ)·sin — просто множитель |
| `gate_dense` | scint_alpha плотно | Подтверждение G_clip для первого gate-признака |
| `tc_theta_map` | tc при разных θ | C(θ) = slope(dy/dtc)/μ — θ-зависимый коэффициент |
| `tc_vs_mu` | tc при μ=0,2,5,8 | Линейность по μ·tc подтверждена |
| `track_curvature` | tc от 0 до 9 | G_sym_track форма — такой же clip |
| `high_tc` | зона больших |tc| | Худшие зоны: θ[+6,+8)×|tc|[7,9), RMSE до 16 |
| `db_negthet` | drift_bias при θ<0 | db работает только как gate, не как сигнальный признак |
| `db_gate_interaction` | db + gate cross | Нет взаимодействия между db и gates |
| `components_big` | изоляция компонент (~1000 pts) | Измерение всех 4 аддитивных членов |
| `components_dense2` | второй dense scan | Уточнение компонент (~1570 pts) |
| `dense_components` | финальное точное измерение | Прецизионные значения g_θ, C(θ) |
| `verify_components` | прямое измерение g_θ, H, F, C, G_tc | Подтверждение всей формулы |
| `verify_signflip` | знаковый переворот | Проверка антисимметрии H(μ) |
| `big_neg_theta` | отрицательные θ (~1000 pts) | g_θ(θ<0) рациональная + гауссиан |
| `dirty_region` | θ[−7,0) + C_pos (~2000 pts) | Верификация C_pos с 3 гауссианами |
| `neg_gate_cross` | gates × g_θ cross | Нет кросс-членов gate×g_θ |
| `hard_slice` | θ sweep при drift=−9 | Hard slice: структура неустойчива |
| `combination_test` | multiple non-zero gates + db | Мультипликативный, не аддитивный эффект gates |

**Итого через scan-файлы:** ~20 000 API-точек потрачено на прецизионное зондирование структуры формулы.

---

## 3. Ранние аналитические идеи (до формулы)

### 3.1 EDA-фаза (01–02 мая)

Запускались через `eda_report.py`, `eda_bokeh.py`, `eda_plotly_3d.py`:

| Анализ | Что сделано | Результат |
|--------|-------------|-----------|
| Корреляционная матрица | Pearson/Spearman по всем 13 признакам | cerenkov_angle и muon_flux — главные |
| PCA | 13D → 2D проекция | Нет чёткой кластеризации |
| SHAP scatter | TreeSHAP по LightGBM на всех данных | cerenkov_angle доминирует, gates ≈ 0 |
| 3D scatter y(θ, μ) | Plotly 3D | Виден синусоидальный узор по θ |
| Режимный анализ | 9 gate-признаков раздельно | Все gates — пороги, не градиентные |
| |y|>50 subset | `eda_high_abs_y.py` | Структура сильнее при больших |y| |
| Частотный анализ | FFT + low-pass filter | Доминантная частота ≈ k₁/2π по θ |
| UMAP кластеризация | `custom_tools_bokeh.py` | Нет чётких кластеров в признаковом пространстве |

### 3.2 Первые модельные идеи (01–02 мая)

Из `history.md` и `*_report.py` файлов:

#### Синусоидальные модели

| Модель | Формула | Результат |
|--------|---------|-----------|
| Простой синус | `y = c₀ + A·sin(θ + φ)` | R²≈−0.003 (глобально) |
| Синус с контекстом | `y = c₀(x) + A(x)·sin(θ + φ(x))` | R²=0.27 (локально, weak) |
| Взвешенный SinCos | `y = A·sin(θ) + B·cos(θ)` амплитудная потеря | Не помогло, перефитинг σ |
| Гетероскедастичный SinCos | Гауссовский p(y|x) с σ(x) | Σ модель переобучается |
| Fourier 1-3 гармоники | `y = Σ aₙ·sin(nθ) + bₙ·cos(nθ)` | fourier_gbm R²=0.338 на mid-band |
| Синус с ML-амплитудой | `A_hat = ML(x), g_hat = ML(x)` | Разбирается на линейную проекцию |

#### Core+Gates модели

| Модель | Идея | Результат |
|--------|------|-----------|
| Fourier123 core + additive gates | `y = f_fourier(θ,μ) + Σ gᵢ(xᵢ)` | core_mu_gates best, R²<0 на test |
| Nonlinear core + gates | LightGBM на core, отдельно на gates | Тяжёлый перефитинг (test R²<0) |
| Angle phase model | Фазово-сдвинутый синус с linear phase driver | Недостаточно мощи |
| Amplitude×Phase hybrid | Отдельно учим A(x) и φ(x) | R²=0.877 для A(x) отдельно! |

**Ключевое открытие этой фазы:** отдельное моделирование амплитуды A(x) через kNN локальный Fourier дало R²=0.877 — первый настоящий прорыв.

#### Режимные и полосовые гипотезы

| Гипотеза | Что тестировалось | Вердикт |
|----------|------------------|---------|
| Три режима | noise / weak-signal / strong-signal mixture | Подтверждено, но weak = hard slice |
| Low-band только шум | |y|<300 — это шум без структуры | **Верно** — noise-only |
| Mid-band структура | 300<|y|<4500 имеет Fourier-структуру | Fourier R²=0.338 |
| Hard slice (μ>0, db<−8) | Отдельная структура | R²=−163 — **катастрофический** перефитинг |
| Band transfer | Mid-band модель на low-band | Отрицательный перенос |

---

## 4. Архитектурные раунды: все 32 (+ подраунды)

### Раунды 2–8: Деноизинг, остатки, маршрутизация, ветвления

| Раунд | Гипотеза | Протестированные семейства | Вывод |
|-------|---------|---------------------------|-------|
| **2** (amplitude restoration) | Деноизинг помогает MoE | hybrid_moe_monotone_denoised + 5 вариантов | hybrid_moe проходит feasibility |
| **2** (amplitude boost) | Безопасный гамма-множитель | trusted_zone_gamma × 5 вариантов | γ улучшает MSE 1617→1574k |
| **3** (core benchmark) | Raw-only core семейства | additive, product, rational, latent, MoE frozen | latent_product > legacy на residual |
| **3b** (raw only) | Residual corrector сырыми признаками | latent_product × 3 варианта | Улучшение baseline на val |
| **4** (residual core) | Guard residual routing | cap/gate/weighted/ungated × latent_product | latent_product > additive |
| **4b** (feasibility) | 7 политик маршрутизации | ungated → sign-consistency, 7 шагов | trusted_zone_gate лучший |
| **5** (outer routing) | Разделение unsafe/safe зон | soft_damping + safe_residual × 3 варианта | soft damping + safe residual |
| **6** (singularity branch) | Ветвления на сингулярностях | piecewise_singularity × 4 семейства | Ветвление закрывает gap |

### Раунды 7–9: Структурные зонды

| Раунд | Протестированные формульные семейства | Результат |
|-------|--------------------------------------|-----------|
| **7** (structural probe) | **21 семейство:** csc_clip, double_root_log, inverse_power, inverse_sinc, latent_product, latent_resonance, latent_step_response, log_abs_sinc, log_sin, pole_log, rational_single_denominator, sec_clip, second_order_transfer, shifted_inverse_power, sinc, single_step_response + 5 вспомогательных | Подтверждение BW-полюса, sinc-мотив |
| **8** (preservation) | Сужение round7 + stable-mid preservation | Закрепление лучших из round7 |

### Раунды 10–18: Teacher-Oracle, Multitask, Продукты, Факторизация

| Раунд | Ключевая идея | Что нового |
|-------|---------------|------------|
| **10** (teacher oracle) | Oracle на режимах → student | Feasibility gap 161→5.3 |
| **11** (recalibration) | Frozen guard + γ calibrator | Лучший fit на stable_mid |
| **12** (full core) | **20 семейств core:** inverse_trig_sec, periodic_context_product, periodic_first_harmonic, periodic_fourier_small, periodic_multi_alpha, raw_angle_additive, raw_angle_latent_product, raw_angle_low_rank_product, raw_angle_piecewise, raw_angle_rational, resonance_transfer, sinc_core + гибриды | Улучшение над round11 |
| **13** (angle extrapolation) | Wrapped-angle periodic surrogate | Bounded detector-first wrapper |
| **14** (dual target) | Раздельное обучение на y и y_denoised | Ранжировать по raw y |
| **15** (multitask) | Совместный raw_y + physical_y_denoised | Maintain raw y ranking |
| **16** (product angle) | Angle harmonics × power envelopes × denominators | Multiplicative > additive |
| **17** (factorized support) | Three-zone splits, denominator-first | Learned support partitions help |
| **18** (latent argument) | Explicit vs latent vs hybrid products | Explicit products strong |

### Раунды 19–32: OOD, Конфликты, Затухание, Аудит

| Раунд | Гипотеза | Семейства | Вывод |
|-------|---------|-----------|-------|
| **19** (OOD ensemble) | Замороженный банк моделей + routers | Conservative ensemble + OOD routers | Ensembles > singles on OOD |
| **20** (practical OOD) | Round11-centric caps + routers | round20_round11_plus_disagreement_cap | Лучший на real/synthetic |
| **21** (OOD collection) | Таргетированный сбор 13D | Батч у boundary policy gap | Подготовка к round27 |
| **22** (periodic conflict) | Конфликтная геометрия | Soft/hard conflict × policies | Conflict-only улучшает |
| **23** (damped vs piecewise) | **19 семейств:** env_exp_abs, env_exp_sq, env_gauss, env_power, harmonic_mix + piecewise × режимы | Gaussian-damped > piecewise |
| **24** (damped tan) | Safe clipped-tan, phase-shift | tan_clip, tan_phase_shifted, damped × 3 | Variants > piecewise baseline |
| **25** (gaussian refinement) | **Harmonic complexity** + bounded phase + denominators | Гауссовские затухающие гармоники | Winner direct core |
| **26** (gauss-line collection) | Батч у round20/round25 disagreement | Targeted live batch | Подготовка round27 |
| **27** (OOD reset) | Rebuilt support/conflict geometry | Расширенные данные | New OOD leaderboard |
| **28** (unified latent) | **24 семейства:** latent_oscillation_single_axis, oscillation_only, gaussian, gaussian_rational, gauss_amp, gauss_env, gauss_rational, harmonic×envelope комбо + soft_regime × positional_alignment | Latent heads заменяют external policy |
| **29** (non-neural reset) | **38 семейств:** boosted_structured, kernel_rff, structured_additive_ensemble, raw_harmonic, raw_harmonic_envelope, raw_x_harmonic + все комбо | Structured ensembles competitive |
| **30** (two-stage residual) | One-model two-stage, internal residual heads | boosted base + residual heads | Two-stage within one model |
| **31** (contract audit) | Forensics: target/split/duplicate | Validation protocol guardrail | — |
| **32** (legacy integration) | Trusted raw-only latent-product + heuristic unsafe guard | Финальная интеграция | Legacy + heuristics |

---

## 5. Версии активного сэмплера (13 файлов)

| Версия | Ключевое изменение | Аллокация |
|--------|-------------------|-----------|
| v1 (active_signal_sampler) | Mixture: signal value + regime disagreement + boundary + novelty | — |
| **v2** | Stable/boundary/hard/coverage mix | — |
| **v3** | Transition-first + extrapolation balanced | 40/20/25/10/5 |
| **v4** | Hybrid-gate boundary-first (heavier boundary discovery) | — |
| **v5** | Regime-exploration outside mean±std envelope, [-100,100] expansion | exploration-first |
| `hard_positive_mu_sampler v1/v2` | Hard slice (μ>0, db<−8) focused batches | 100% hard slice |
| `hard_positive_mu_cycle_experiment` | 10-cycle + 5-cycle focused experiments | — |

**Итог сэмплинга:** hard slice (μ>0, drift<−8) остался **структурно неустойчивым** даже после 1000+ точечного таргетированного сбора.

---

## 6. Validation-First Framework

Отдельная методологическая инфраструктура (файлы `validation_first_model_sweep.py`, `validation_first_model_sweep_dashboards.py`):

- Locked train/val/test split (единожды, не пересоздаётся)
- Top-30 legacy pool для сравнения
- Denoising-aware варианты
- Feasibility score: low/hard/noise = 55.1 / 66.1 / 65.0

**Победитель:** `hybrid__arch_moe_monotone__denoised_features`

---

## 7. Полный список протестированных формульных семейств

### Тригонометрические / синусоидальные
- `sin(θ)`, `cos(θ)`, `sin(2.5θ)` — базовые
- Fourier 1, 2, 3 гармоники
- `sin(k·θ)` с оптимизируемым k
- `sinc(θ)` = sin(θ)/θ
- `log_sin(θ)` = log|sin(θ)|
- `csc(θ)` = 1/sin(θ) (с clip)
- `sec(θ)` = 1/cos(θ) (с clip)
- Damped sinusoid: `A·sin(θ)·exp(−γ|θ|)`
- Gaussian-damped: `A·sin(θ)·exp(−γθ²)`
- Phase-shifted: `A·sin(θ + φ(x))`
- Latent argument: `A·sin(f(x))` где f — ML-функция

### Резонансные / рациональные
- Breit-Wigner: `A·μ²/((|μ|−a)²+Γ²)` ← **выигрышная**
- Double-root log: `log((μ−a₁)(μ−a₂))`
- Pole log: `log|μ−a|`
- Shifted inverse power: `1/(μ−a)^n`
- Inverse power: `1/|μ|^n`
- Rational single denominator: `A/(μ²+b²)`
- Second-order transfer: `1/(s²+2ζωs+ω²)` (передаточная функция)
- Latent resonance: `A(x)/((f(x))²+Γ²)`

### Гиперболические
- `sinh(k·θ/n)` ← **в C(θ) победитель**
- `cosh(θ)`, `tanh(θ)`
- `log_abs_sinc` = log|sinh|/θ

### Степенные / полиномиальные
- `μ^n` (n=1,2,3)
- `|μ|^n`
- `product_power_log` = `x^a·log(x)^b`
- `product_trig_power` = `sin(x)·x^a`

### Гауссианы и огибающие
- `exp(−e·(θ−c)²)` ← **в g_θ и C победитель**
- `exp(−γ|θ|)` (экспоненциальное затухание)
- `exp(−γθ^n)` (степенное в показателе)
- `gauss_rational` = Gaussian × rational

### Ансамблевые архитектуры
- MoE (Mixture of Experts): gate_network → N_experts
- Hybrid_moe_monotone_denoised
- Conservative ensemble (frozen bank)
- Round11-centric caps
- Structured additive ensemble
- Kernel RFF + Ridge (Random Fourier Features)
- Two-stage internal residual heads
- Latent oscillation + envelope

### Пороговые / кусочные
- Piecewise (ручные точки разрыва)
- Step response: `Heaviside(x−a)`
- Single step response
- Three-regime (noise/weak/strong)

---

## 8. Что не сработало: полный список с причинами

| Идея | Файл(ы) | Причина провала |
|------|---------|-----------------|
| Глобальный синус `c₀+A·sin(θ+φ)` | explicit_sinusoid_global_report | R²≈−0.003 — недостаточная структура без BW |
| Probabilistic SinCos | probabilistic_sincos_report | σ-модель переобучается, не улучшает среднее |
| LightGBM напрямую на y | nonlinear_core_gated_model_report | test R²<0 — перефитинг без формульной основы |
| Hard-slice модели (μ>0, db<−8) | hard_positive_mu_* | R²≈−163 — структурно несовместимо со стабильным законом |
| Piecewise amplitude correction | piecewise_amplitude_correction_report | Амплитуда корректируется, но фаза ломается |
| Joint multitask Fourier | joint_multitask_fourier_report | Совместная оптимизация коллапсирует structure |
| Non-neural surrogates (Ridge/RF) | nonlinear_core_gated_report | test R²<0, не могут учить residuals |
| ML-аргумент синуса (latent φ) | round18_latent_argument_product | Сводится к линейной проекции — нет новой геометрии |
| Two-zone LGBM (easy/hard) | submission_training | OOF +0.223 — деление теряет взаимодействие зон |
| Pseudo-labeling (91K теста) | submission_training/EXPERIMENTS2 | OOF −0.036 но Kaggle провал — bias из текущей модели |
| Weighted SinCos (amplitude imbalance) | amplitude_weighted_sincos_report | Не исправляет shrinkage |
| Differential Evolution (DE) | submission_training/EXPERIMENTS2 | Прерван — нет улучшения над NM+Powell |
| Режим-ориентированные predictors | regime_local_predictor_report | Experts переобучаются на своих режимах |
| Band transfer (mid→low) | target_band_model_report | Отрицательный трансфер: другая природа сигнала |
| Angle wrapping к [-π/2, π/2] | history.md (01 мая) | Потеря информации о знаке — откат |
| Sample weighting |tc|>7 ×4 | EXPERIMENTS.md | Нестабильность, OOF хуже |
| Box-Cox transform | EXPERIMENTS2.md | Запланировано, не реализовано |

---

## 9. Что сработало: хронология находок

| Дата | Находка | Метрика |
|------|---------|---------|
| 01 мая | Gate-признаки — только пороги (SHAP) | Качественно |
| 01 мая | Amplitude A(x) отдельно через kNN | R²=0.877 |
| 01–02 мая | Fourier3 на mid-band | R²=0.338 |
| 02 мая | Три режима: noise/weak/strong | Качественно |
| 02 мая | Low-band = только шум (|y|<300) | Верифицировано |
| 07 мая | Breit-Wigner F(μ)·sin(2.5θ) + H(μ) | RMSE→8.996 |
| 08 мая | LGBM на residuals | RMSE 8.996→3.33 |
| 09–10 мая | Рост данных + больше фолдов | RMSE 3.33→2.75 |
| 10 мая | Scan-файлы уточняют все компоненты | Формула V2 |
| 11 мая | 16 компонент формулы как LGBM-признаки | OOF 2.437→1.723 |
| 12 мая | Optuna: num_leaves=16 оптимально | OOF→1.444 |
| 12 мая | Zone features + Optuna | OOF→1.435 |
| 12 мая | Порог 9.07 (7 сабмитов) | Kaggle 88.735 |
| 13 мая | Итерационное уточнение formula↔LGBM | OOF→1.420 |

---

## 10. Инфраструктурные находки (недооценённые)

Помимо модельных идей, ранние этапы дали инфраструктурные решения:

1. **Validation-first framework** — фиксированный split раз и навсегда, что предотвратило data leakage через 32 раунда бенчмарков.

2. **Adaptive sampler v5** — расширенный диапазон [-100,100] для OOD exploration, режимно-разделённое исследование. Собрал данные для формульных раундов 23–32.

3. **Feasibility scoring** — помимо R², отдельный feasibility score (low/hard/noise зоны), который не позволял выбрать модель хорошую на easy-зоне но плохую на hard.

4. **Active learning с Bayesian UCB** — позволил собрать 29K ценных точек при минимальной квоте (вместо случайных).

5. **Scan-протокол** — прецизионное зондирование по одному признаку за раз позволило установить точную аналитическую форму F(μ), H(μ), G_clip(x), C(θ), g_θ(θ) через ~20K API-точек.

---

## 11. Хронология раундов: summary

```
Раунды 2–6:  Структура + маршрутизация + ветвления (без явной формулы)
Раунды 7–9:  Зондирование структуры: pole, sinc, resonance, step-response
Раунды 10–13: Oracle + recalibration + extrapolation
Раунды 14–18: Multitask + products + factorized + latent argument
Раунды 19–22: OOD robustness + conflict geometry
Раунды 23–27: Damped harmonics + Gaussian-damped победитель
Раунды 28–30: Unified latent + non-neural reset + two-stage
Раунды 31–32: Audit + legacy integration

→ Параллельно: scan_*.py файлы (24 шт.) зондируют точную форму функции
→ Параллельно: tmp_*.py (125 шт.) — быстрые прототипы идей

После round32: переход к явной физической формуле (Formula K4, затем V2)
→ Все архитектурные раунды → LGBM без явной формулы → тупик
→ Scan-данные → явная формула → LGBM на residuals → прорыв
```

---

## 12. Нерешённые проблемы (на момент завершения)

| Проблема | Что пробовалось | Статус |
|----------|----------------|--------|
| **Hard slice** (μ>0, db<−8) | 1000+ таргетированных точек, 5 модельных подходов, cycle experiments | Нерешено, R²=0.077 |
| **Высокий |tc|** зоны (θ[+6,+8)×|tc|[7,9)) | sample weighting, two-zone LGBM | RMSE до 4.85 |
| **Точность 2-й итерации** уточнения | 1 итерация (OOF 1.435→1.420) | Вероятно ещё 1–2 итерации дадут ещё |
| **Box-Cox / target transform** | Запланировано в EXPERIMENTS2 | Не реализовано до исчерпания квоты |
