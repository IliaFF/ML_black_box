<!-- Description: Full guide on running an AI-agent-assisted ML competition project.
     Covers Claude Code configuration, CLAUDE.md setup, project conventions, and ML workflow.
     Related: [[CLAUDE.md]], [[status.md]], [[history.md]], [[early_stages_full.md]] -->

# Агентный ML-проект: полный гайд

Как организован этот проект и как повторить такой же с нуля.

---

## 1. Что это за подход

Обычно в соревнованиях участник сам пишет код, запускает эксперименты, анализирует результаты. Здесь иначе: **Claude Code (AI-агент) — соавтор**. Он пишет скрипты, запускает их, читает логи, предлагает следующий шаг. Участник задаёт цели и принимает решения.

Результат проекта Dz5 ML: **13 дней, 345 Python-скриптов, 130+ проверенных семейств формул, финальный скор 88.735 (1-е место).**

Ключевые преимущества подхода:
- Не нужно писать шаблонный код — агент делает это мгновенно
- Параллельный запуск субагентов для исследования кода
- Полный аудит-трейл: каждый эксперимент задокументирован
- Агент помнит все предыдущие гипотезы и результаты внутри сессии

---

## 2. Установка Claude Code

### 2.1 Установка

```powershell
# Windows — скачать установщик с claude.ai/code
# Или через npm (требует Node.js 18+)
npm install -g @anthropic/claude-code
```

Запуск:
```powershell
claude                          # интерактивный режим
claude "сделай это"             # одна задача и выход
claude --model sonnet           # явный выбор модели
```

### 2.2 Глобальные настройки: `~/.claude/settings.json`

Этот файл управляет поведением Claude во всех проектах.

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node \"C:/Users/USERNAME/.claude/hooks/caveman-activate.js\"",
            "timeout": 5,
            "statusMessage": "Loading caveman mode..."
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node \"C:/Users/USERNAME/.claude/hooks/caveman-mode-tracker.js\"",
            "timeout": 5
          }
        ]
      }
    ]
  },
  "statusLine": {
    "type": "command",
    "command": "powershell -ExecutionPolicy Bypass -File \"C:/Users/USERNAME/.claude/hooks/caveman-statusline.ps1\""
  },
  "enabledPlugins": {
    "caveman@caveman": true,
    "co-researcher@co-researcher-marketplace": true
  },
  "extraKnownMarketplaces": {
    "caveman": {
      "source": { "source": "github", "repo": "JuliusBrussee/caveman" }
    },
    "co-researcher-marketplace": {
      "source": { "source": "github", "repo": "poemswe/co-researcher" }
    }
  },
  "env": {
    "CLAUDE_CODE_USE_POWERSHELL_TOOL": "1"
  },
  "autoUpdatesChannel": "latest",
  "theme": "dark",
  "model": "sonnet"
}
```

**Важно:** `CLAUDE_CODE_USE_POWERSHELL_TOOL=1` — заставляет агента использовать PowerShell вместо bash на Windows. Без этого команды падают.

### 2.3 Плагины

Установка плагина из GitHub:
```
/install-plugin github:JuliusBrussee/caveman
/install-plugin github:poemswe/co-researcher
```

**Caveman** — сжатый стиль ответов (убирает воду, оставляет суть). Ускоряет работу в ~2 раза за счёт более коротких ответов. Управление: `/caveman lite|full|ultra`, отключение: `stop caveman`.

**Co-Researcher** — академические навыки (literature review, hypothesis testing, methodology). Полезно при анализе физических формул и структуры данных.

### 2.4 Hooks

Хуки — shell-скрипты, которые запускаются при событиях Claude Code. Используются для:
- Инъекции контекста в начале сессии (`SessionStart`)
- Проверки / трансформации каждого запроса (`UserPromptSubmit`)
- Кастомного статус-бара в терминале

Расположение: `~/.claude/hooks/`

---

## 3. Конфигурация проекта: CLAUDE.md

**CLAUDE.md** — главный файл конфигурации проекта. Лежит в корне репозитория. Автоматически читается агентом при каждом запуске. Это «конституция» проекта — агент обязан следовать ей.

### 3.1 Что обязательно писать в CLAUDE.md

#### Окружение
```markdown
## Environment
All Python runs via conda (not `.conda_env\python.exe` directly — DLL paths break numpy):
```powershell
D:\Miniforge\condabin\conda.bat run -p D:\PROJECT\.conda_env python ...
```

Без этой заметки агент будет пытаться вызвать python.exe напрямую — numpy падает из-за DLL.

#### Ключевые команды
```markdown
## Key Commands
# Smoke test
conda run -p .conda_env python -m scripts.collect --dry-run

# Production (requires explicit flag)
conda run -p .conda_env python -m scripts.collect --allow-production
```

Агент будет знать "безопасную" и "опасную" команды. Барьер `--allow-production` — критичен для проектов с квотами/API.

#### Протокол экспериментов
```markdown
## Experiment Protocol
Before any production run:
1. Dry-run (no quota)
2. Smoke: 1 request, verify save
3. Bayes smoke: verify fit/predict path
```

Без протокола агент может случайно исчерпать квоту.

#### Файлы отслеживания
```markdown
## Tracking Files
- status.md — full snapshot, OVERWRITE every update
- history.md — APPEND-ONLY change log, dated sections
- assumptions.md — working hypotheses
```

Два типа файлов намеренно разные: `status.md` всегда актуален (перезаписывается полностью), `history.md` — вечный лог (только добавление).

#### Логирование долгих скриптов
```markdown
## Logging
Every long script must write results/<script_name>.log (buffering=1).
Log every 500 iterations. Include: iteration, loss/RMSE, elapsed time.
Monitor: Get-Content results/script.log -Wait -Tail 20
```

Без этого правила агент пишет скрипты, которые молчат 30 минут, — невозможно понять, работают ли они.

#### Язык ответов
```markdown
## Language
All responses to user in Russian. Code, comments, variable names in English.
```

Или любой нужный язык. Агент строго соблюдает.

#### Заголовки файлов
```markdown
## File Headers
Every generated file starts with docstring stating:
1. What the file does
2. Related files as [[wiki-links]]

Python:
```python
"""Short description.
Related:
  [[scripts/config.py]] - constants
  [[results/output.csv]] - output
"""
```

Wiki-ссылки позволяют агенту быстро находить связанные файлы без полного обхода дерева.

#### Fallback policy
```markdown
## Fallback Policy
Silent fallbacks forbidden. Any fallback must state:
- what failed
- what was used instead
- whether it changes result quality
```

Без этого агент тихо меняет метод/библиотеку — результаты несравнимы.

### 3.2 Полный шаблон CLAUDE.md

```markdown
# CLAUDE.md

## Environment
[Как запускать Python/julia/R — точные команды с путями]
[Credentials через env vars, никогда в репо]

## Key Commands
[3-5 ключевых команд: dry-run, smoke, production, jupyter]

## Experiment Protocol
[Порядок проверок перед production]
[Явный gate для опасных операций]

## Architecture
[Цель проекта в одном предложении]
[Таблица: модуль → роль]
[Текущий статус: что уже известно о структуре данных]

## Tracking Files
[status.md — правило перезаписи]
[history.md — правило дополнения]
[assumptions.md, summary.md]
[Когда обновлять каждый]

## Logging and Progress Tracking
[Паттерн log-файла]
[Как смотреть в реальном времени]

## Fallback Policy
[Явные fallbacks запрещены/обязательно с описанием]

## Language
[Язык ответов пользователю]
[Язык кода/комментариев]

## File Headers
[Шаблон docstring для Python]
[Шаблон для Markdown]

## Style
[Отступы, именование, прочие конвенции]
```

---

## 4. Структура проекта

```
PROJECT_ROOT/
├── CLAUDE.md                    ← конфигурация агента (ОБЯЗАТЕЛЬНО)
├── status.md                    ← текущий статус (перезапись)
├── history.md                   ← лог изменений (дополнение)
├── assumptions.md               ← гипотезы
├── AGENT_GUIDE.md               ← этот файл
│
├── scripts/
│   ├── config.py                ← константы, имена фич
│   ├── collect.py               ← точка входа пайплайна
│   ├── api_client.py            ← REST клиент
│   ├── storage.py               ← I/O, дедупликация
│   ├── summary.md               ← инвентарь скриптов
│   │
│   ├── architecture_formula_round1_*.py   ← эксперименты (нумерованные)
│   ├── architecture_formula_round2_*.py
│   ├── ...
│   ├── tmp_*.py                 ← временные прототипы (125+ штук)
│   ├── generate_*.py            ← генераторы визуализаций/отчётов
│   └── scan_*.py                ← целевые исследования данных
│
├── results/
│   ├── dataset.parquet          ← основной датасет
│   ├── requests_log.jsonl       ← лог API-запросов
│   ├── model_u_v2_final_params.npy  ← параметры формулы
│   ├── summary.md               ← инвентарь результатов
│   │
│   ├── eda/                     ← 1009 EDA-артефактов
│   ├── models/                  ← сохранённые модели
│   ├── campaigns/               ← активное обучение
│   ├── presentation/            ← PNG-графики для доклада
│   └── interactive/             ← HTML-интерактивы
│
├── report_final.ipynb           ← финальный Jupyter-ноутбук
├── presentation_speech.md       ← текст доклада
├── early_stages_full.md         ← полная история гипотез
│
└── .conda_env/                  ← conda-окружение (не в git)
```

### Правила именования

| Паттерн | Назначение |
|---------|-----------|
| `architecture_formula_roundN_*.py` | Пронумерованные раунды — легко отсортировать по прогрессу |
| `tmp_*.py` | Быстрые прототипы — не беспокоиться о качестве кода |
| `scan_*.py` | Целевые зондирования конкретного аспекта данных |
| `generate_*.py` | Скрипты без состояния, создающие артефакты |

---

## 5. Рабочий процесс с агентом

### 5.1 Начало сессии

Первое, что делать в начале каждой сессии:
```
прочитай status.md и history.md, расскажи что сейчас известно и что дальше
```

Агент восстанавливает контекст и предлагает следующий шаг. Без этого каждая сессия начинается "с нуля".

### 5.2 Паттерны эффективной работы

**Делегировать рутину, решать стратегию:**
- Агент пишет шаблонный код, парсит логи, строит графики
- Человек решает: продолжать эту гипотезу или попробовать другую

**Декомпозиция задачи:**
```
разбей задачу на: 1) исследование данных, 2) формула-baseline, 3) бустинг-ансамбль
```

**Параллельные субагенты** (для исследования кода):
```
запусти параллельно: изучи что пробовали в rounds 1-15, и изучи что пробовали в rounds 16-32
```

**Реальное время логов:**
```powershell
Get-Content D:\PROJECT\results\current_experiment.log -Wait -Tail 20
```

Пока скрипт считается, можно ставить следующие задачи.

### 5.3 Итеративный цикл эксперимента

```
1. Гипотеза → агент пишет tmp_hypothesis_N.py
2. Запуск → агент читает лог
3. Результат плохой → агент предлагает 2-3 объяснения
4. Выбор следующего шага → переход к 1
5. Результат хороший → перенести в architecture_formula_roundN.py
6. Обновить status.md и history.md
```

### 5.4 Команды для отладки

```
прочитай лог results/script.log и объясни что пошло не так
```
```
сравни результаты round12 и round15, что изменилось
```
```
найди все скрипты где используется BW-формула и посмотри как менялись параметры
```

---

## 6. ML-пайплайн: как именно велась эта задача

Задача: реверс-инжиниринг чёрного ящика `y = f(x)` с 13 физическими признаками через активное обучение.

### 6.1 Фаза 1: EDA и понимание данных (1-2 дня)

```python
# Ключевые открытия первых 2 дней:
# 1. 11 из 13 фич — бинарные гейты (порог ≈ ±9.5)
# 2. Активная зона: все |xi| < 9 для нефизических фич
# 3. В активной зоне: y ≈ F(mu) * sin(2.5*theta) + H(mu)
```

Инструменты: SHAP, корреляционные матрицы, целевые сканы по каждой оси.

### 6.2 Фаза 2: Поиск формулы (5-7 дней)

Проверялись 130+ семейств функций. Порядок от простого к сложному:

```
Синусоиды → Фурье → Breit-Wigner → Damped harmonic →
→ Режимные модели → OOD Ensemble → Latent/Factorized → Kernel/RFF
```

Инфраструктура поиска:
```python
# Каждая гипотеза = отдельный скрипт tmp_*.py с:
# 1. Функцией predict_formula(x, params)
# 2. Оптимизатором Nelder-Mead (5 стартов × 50K итераций)
# 3. Метриками: RMSE, R², зоновый анализ
# 4. Сравнением с предыдущим лучшим

# Формат лога:
# Start 1/5  RMSE=12.34  elapsed=23s
# *** New best: RMSE=8.99  params=[...]
```

Критический прорыв: формула Breit-Wigner (физика резонанса):
```
F(mu) = A * mu² / ((|mu| - a)² + Γ²)
```
RMSE упал с ~15 до 8.996.

### 6.3 Фаза 3: LGBM на остатках (2-3 дня)

После нахождения аналитической формулы — бустинг на остатках:

```python
residuals = y - formula_v2(X)
lgbm = LGBMRegressor(**best_params)
lgbm.fit(features, residuals)
y_pred = formula_v2(X) + lgbm.predict(features)
```

Признаки для LGBM:
- 16 компонент формулы (F(mu), H(mu), sin(k*theta), G_clip_i, ...)
- 7 зональных индикаторов (|mu|>8, theta<0, |tc|>5, ...)
- Исходные 13 физических признаков

### 6.4 Фаза 4: Итеративное уточнение

```
for iteration in range(2):
    1. LGBM 10-fold CV на остатках формулы
    2. Переобучить формулу на (y - lgbm_oof)  ← чище сигнал
    3. Повторить LGBM
```

Prогресс скора:
| Этап | OOF RMSE | Kaggle |
|------|----------|--------|
| Formula K4 (BW) | ~8.99 | — |
| LGBM Round 1 | 3.33 | — |
| LGBM Round 6 | 2.97 | — |
| Formula V2 + 16 компонент | 1.72 | — |
| Optuna (150 проб) | 1.44 | — |
| Зоновая оптимизация | 1.435 | — |
| Итеративное уточнение | 1.420 | 88.735 |

---

## 7. Конфигурация Python-окружения

```powershell
# Установка Miniforge (рекомендуется вместо Anaconda)
winget install Miniforge3

# Создание окружения прямо в проекте
D:\Miniforge\condabin\conda.bat create -p D:\PROJECT\.conda_env python=3.11

# Установка пакетов
D:\Miniforge\condabin\conda.bat run -p D:\PROJECT\.conda_env pip install `
    numpy pandas scipy scikit-learn lightgbm optuna `
    matplotlib plotly jupyterlab pyarrow

# Запуск (ОБЯЗАТЕЛЬНО через conda.bat run, не python.exe напрямую)
D:\Miniforge\condabin\conda.bat run -p D:\PROJECT\.conda_env python script.py
```

**Почему conda.bat run, а не python.exe напрямую:**
Windows conda-окружения требуют активации для правильных DLL-путей numpy/scipy. Без активации импорт numpy падает с ошибкой DLL.

**Проблема Cyrillic в консоли:**
conda.bat перехватывает stdout и конвертирует в cp1251. Кириллица в `print()` = `UnicodeEncodeError`. Решение: все `print()` в Python-скриптах — только на английском.

---

## 8. Паттерны кода

### 8.1 Структура экспериментального скрипта

```python
"""One-line description of what this script tests.

Related:
  [[scripts/config.py]] - feature names
  [[results/dataset.parquet]] - data source
  [[scripts/architecture_formula_roundN_prev.py]] - previous version
"""
from pathlib import Path
import numpy as np
import pandas as pd
from scipy.optimize import minimize
import time

REPO = Path("D:/PROJECT")
LOG = open(str(REPO / "results" / "script_name.log"), "w", buffering=1)

def log(msg):
    print(msg, flush=True)
    LOG.write(msg + "\n")
    LOG.flush()

# --- Load data ---
df = pd.read_parquet(REPO / "results" / "dataset.parquet")

# --- Model ---
def predict(X, params):
    ...

def loss(params):
    y_pred = predict(X_train, params)
    return np.sqrt(np.mean((y_pred - y_train)**2))

# --- Optimization: 5 starts ---
best_loss = np.inf
best_params = None
p0 = np.load(...)  # warm start

for i, sigma in enumerate([0, 0.01, 0.05, 0.1, None]):
    if sigma is None:
        p_start = p0_cold
    elif sigma == 0:
        p_start = p0.copy()
    else:
        p_start = p0 * (1 + np.random.normal(0, sigma, len(p0)))

    log(f"Start {i+1}/5  sigma={sigma}")
    res = minimize(loss, p_start, method="Nelder-Mead",
                   options={"maxiter": 100_000, "xatol": 1e-8})

    if res.fun < best_loss:
        best_loss = res.fun
        best_params = res.x
        log(f"  *** New best: RMSE={best_loss:.4f}")
    else:
        log(f"  RMSE={res.fun:.4f}")

log(f"Final RMSE={best_loss:.4f}")
np.save(REPO / "results" / "model_params.npy", best_params)
```

### 8.2 Стандарт логирования

```python
# Начало скрипта
log(f"=== Script name  {time.strftime('%Y-%m-%d %H:%M:%S')} ===")
log(f"Data: {len(df)} rows")

# Каждые N итераций (в callback или обёртке)
if iteration % 500 == 0:
    elapsed = time.time() - t_start
    log(f"iter={iteration}  loss={current_loss:.4f}  elapsed={elapsed:.0f}s")

# Новый рекорд
if loss < best_loss:
    log(f"*** New best at iter={iteration}: {loss:.4f} (was {best_loss:.4f})")

# Финал
log(f"Done. Best RMSE={best_loss:.4f}  params saved to {out_path}")
```

### 8.3 Зональный анализ (обязателен для отчёта)

```python
# После каждого эксперимента проверять зоны отдельно
zones = {
    "active_all":  (np.abs(df[gate_cols]).max(axis=1) < 9),
    "active_mu8":  (np.abs(df["muon_flux"]) > 8) & active_mask,
    "theta_neg":   (df["cerenkov_angle"] < 0) & active_mask,
    "hard_slice":  (df["muon_flux"] > 0) & (df["drift_bias"] < -8),
}
for name, mask in zones.items():
    if mask.sum() > 10:
        rmse = np.sqrt(np.mean((y_pred[mask] - y_true[mask])**2))
        log(f"Zone {name}: n={mask.sum()}  RMSE={rmse:.4f}")
```

---

## 9. Управление контекстом агента

Проблема: контекстное окно конечно (~200K токенов). При длинных сессиях контекст сжимается.

**Стратегии:**

1. **Внешняя память через файлы** — всё важное сохраняется в `status.md`, `history.md`, `assumptions.md`. При новой сессии агент читает эти файлы.

2. **Атомарные скрипты** — каждый эксперимент в отдельном файле. Агент не держит весь код в голове, читает нужный файл.

3. **Явные ссылки в docstring** — wiki-links `[[file.py]]` позволяют агенту за одно чтение найти все связанные файлы.

4. **Сжатые сессии** — клод Code автоматически сжимает историю при переполнении. Файлы отслеживания гарантируют непрерывность.

5. **Агенты для исследования** — длинные обходы кода запускать через субагенты, чтобы не засорять основной контекст.

---

## 10. Типичные ошибки и решения

| Проблема | Причина | Решение |
|----------|---------|---------|
| `DLL load failed` при импорте numpy | Прямой вызов `.conda_env\python.exe` | Всегда `conda.bat run -p .conda_env python` |
| `UnicodeEncodeError: charmap cp1251` | Кириллица в `print()` через conda | Только английский в print/log |
| Агент "забыл" предыдущий эксперимент | Контекст сжался | Обновить `status.md`, попросить его прочитать |
| Скрипт молчит 30 минут | Нет логирования | Логировать каждые 500 итераций |
| Агент переписывает правильный код | Нет описания в docstring | Добавить docstring с `Related:` |
| Эксперимент расходует API квоту | Нет barrier | `--allow-production` gate в CLAUDE.md |
| Несравнимые результаты между запусками | Нет фиксации seed | `np.random.seed(42)` в начале каждого скрипта |

---

## 11. Чек-лист для старта нового проекта

```
[ ] Установить Miniforge, создать .conda_env в корне проекта
[ ] Установить Claude Code (npm install -g @anthropic/claude-code)
[ ] Настроить ~/.claude/settings.json (env, model, theme)
[ ] Установить плагин caveman (для быстрых ответов)
[ ] Написать CLAUDE.md (окружение, команды, протокол, язык)
[ ] Создать status.md и history.md (пустые)
[ ] Создать scripts/config.py (константы, имена фич)
[ ] Создать scripts/collect.py с --dry-run и --allow-production
[ ] Создать scripts/storage.py (I/O датасета)
[ ] Протестировать: conda run python -m scripts.collect --dry-run
[ ] Зафиксировать первый статус в status.md
[ ] Начать сессию: claude → "прочитай CLAUDE.md и status.md"
```

---

## 12. Советы по взаимодействию с агентом

**Конкретность важнее вежливости:**
```
# Плохо
помоги с формулой

# Хорошо
добавь в predict_U третий член: Gaussian по theta около theta=6, амплитуда ~0.3
```

**Указывать файлы и строки:**
```
в scripts/tmp_formula_v2_pipeline.py строка 145, функция predict_U — добавь...
```

**Явно запрашивать зональный анализ:**
```
запусти эксперимент и покажи RMSE отдельно для зон: theta<0, |mu|>8, hard_slice
```

**Просить объяснение перед большим изменением:**
```
прежде чем менять архитектуру, объясни что именно сломано в текущей
```

**Фиксировать удачные инсайты немедленно:**
```
это важно — запиши в assumptions.md: theta=0 — граница двух режимов
```

---

*Проект Dz5 ML, май 2026. 13 дней, 1-е место, 88.735.*
