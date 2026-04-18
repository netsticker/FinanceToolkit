# FinanceToolkit — Comprehensive Handover

> **Read-this-first concept:** each section tells you *exactly which file to open next* and *why*,
> building your mental model from the outside in before diving into implementation details.

---

## Table of Contents

1. [What Is This Project?](#1-what-is-this-project)
2. [Read This First — 10-File Quick Map](#2-read-this-first--10-file-quick-map)
3. [Repository Layout](#3-repository-layout)
4. [Architecture Deep-Dive](#4-architecture-deep-dive)
   - 4.1 [The MVC Pattern Used Here](#41-the-mvc-pattern-used-here)
   - 4.2 [Data Flow From User to Result](#42-data-flow-from-user-to-result)
   - 4.3 [Data Sources and Fallback Logic](#43-data-sources-and-fallback-logic)
5. [Module-by-Module Reference](#5-module-by-module-reference)
   - 5.1 [Core / Root Level](#51-core--root-level)
   - 5.2 [Ratios](#52-ratios)
   - 5.3 [Models](#53-models)
   - 5.4 [Options](#54-options)
   - 5.5 [Risk](#55-risk)
   - 5.6 [Performance](#56-performance)
   - 5.7 [Technicals](#57-technicals)
   - 5.8 [Fixed Income](#58-fixed-income)
   - 5.9 [Economics](#59-economics)
   - 5.10 [Discovery](#510-discovery)
   - 5.11 [Portfolio](#511-portfolio)
   - 5.12 [Utilities](#512-utilities)
   - 5.13 [Normalization](#513-normalization)
6. [Key Technical Patterns](#6-key-technical-patterns)
7. [Data Structures You Will Encounter](#7-data-structures-you-will-encounter)
8. [Dependencies and Toolchain](#8-dependencies-and-toolchain)
9. [Testing Strategy](#9-testing-strategy)
10. [CI/CD Pipeline](#10-cicd-pipeline)
11. [How to Run Locally](#11-how-to-run-locally)
12. [Configuration and Caching](#12-configuration-and-caching)
13. [Common Patterns When Adding New Functionality](#13-common-patterns-when-adding-new-functionality)
14. [Known Constraints and Gotchas](#14-known-constraints-and-gotchas)
15. [Useful External Links](#15-useful-external-links)

---

## 1. What Is This Project?

**FinanceToolkit** (`pip install financetoolkit`) is an open-source Python library that provides
150+ financial ratios, indicators and performance measurements — all computed transparently from raw
financial statements and price data.

**Why it exists:** Financial data providers (Yahoo, Morningstar, Bloomberg, etc.) all report the same
ratio at different values because they use different calculation methods, most of which are hidden
behind paywalls. FinanceToolkit solves this by publishing every formula in plain Python so users can
audit exactly what they are getting.

**Version at time of handover:** `2.0.7` (see `pyproject.toml`).

**Entry-point import:**

```python
from financetoolkit import Toolkit          # main class — equities, ETFs, crypto, currencies
from financetoolkit import Discovery        # search / screen for instruments
from financetoolkit import Portfolio        # personal portfolio analysis
from financetoolkit import Economics        # macro-economic indicators (OECD / GMDB)
from financetoolkit import FixedIncome      # bonds, derivatives, central bank data
```

---

## 2. Read This First — 10-File Quick Map

Work through these ten files **in order**. Each one teaches you the concepts needed to understand
the next.

| # | File | What you learn |
|---|------|----------------|
| 1 | `README.md` (lines 1–115) | The *why*, installation, and a complete usage example covering every module. Skip the long tables; focus on the code snippets. |
| 2 | `CONTRIBUTING.md` | The MVC pattern used throughout the codebase and the developer workflow. Read the "Structure" section carefully — it is the key to navigating everything else. |
| 3 | `financetoolkit/__init__.py` | Five lines. Shows exactly what is exported as the public API. |
| 4 | `financetoolkit/toolkit_controller.py` (lines 70–565) | The `Toolkit.__init__` — all configuration options, how the object is wired up, and what state it carries. |
| 5 | `financetoolkit/toolkit_controller.py` (lines 565–710) | The `Toolkit.ratios` property — understand the lazy-loading pattern used by every sub-module property. |
| 6 | `financetoolkit/fmp_model.py` (lines 37–180) | `get_financial_data()` — the single function that talks to the FinancialModelingPrep API. Learn the error sentinel columns (`LIMIT REACH`, `INVALID API KEY`, etc.) used everywhere. |
| 7 | `financetoolkit/historical_model.py` (lines 37–130) | `get_historical_data()` — threading across tickers, FMP→YahooFinance fallback, OHLCV enrichment. |
| 8 | `financetoolkit/ratios/ratios_controller.py` (lines 33–230) | A full controller class — how it receives pre-loaded DataFrames and delegates to model functions. |
| 9 | `financetoolkit/ratios/profitability_model.py` | A full model file — one function per metric, pure pandas arithmetic, zero side effects. This is the gold standard for adding new calculations. |
| 10 | `financetoolkit/utilities/error_model.py` | The `@handle_errors` decorator and `check_for_error_messages()`. Understand these or you will be confused by silently-returned empty Series. |

---

## 3. Repository Layout

```
FinanceToolkit/
├── financetoolkit/                  ← main package
│   ├── __init__.py                  ← public API exports
│   ├── toolkit_controller.py        ← Toolkit class (the hub)
│   ├── fmp_model.py                 ← FinancialModelingPrep API layer
│   ├── yfinance_model.py            ← Yahoo Finance API layer
│   ├── historical_model.py          ← OHLCV + treasury data retrieval
│   ├── fundamentals_model.py        ← financial statement retrieval
│   ├── normalization_model.py       ← statement normalisation + CSV formats
│   ├── normalization_model.py       ← statement normalisation
│   ├── normalization/               ← CSV column-mapping files (FMP + YF flavours)
│   │   ├── balance.csv / balance_yf.csv
│   │   ├── income.csv  / income_yf.csv
│   │   ├── cash.csv    / cash_yf.csv
│   │   └── statistics.csv
│   ├── currencies_model.py          ← currency conversion helpers
│   ├── helpers.py                   ← shared DataFrame utilities
│   ├── ratios/                      ← Financial Ratios module
│   ├── models/                      ← Financial Models (DuPont, WACC, etc.)
│   ├── options/                     ← Options pricing + Greeks
│   ├── risk/                        ← VaR, CVaR, GARCH, etc.
│   ├── performance/                 ← Sharpe, Beta, CAPM, etc.
│   ├── technicals/                  ← Technical indicators
│   ├── fixedincome/                 ← Bonds, FED, ECB, FRED data
│   ├── economics/                   ← OECD / GMDB macro data
│   ├── discovery/                   ← Instrument search + screener
│   ├── portfolio/                   ← Personal portfolio analysis
│   └── utilities/
│       ├── cache_model.py           ← pickle / pandas cache helpers
│       ├── error_model.py           ← @handle_errors decorator + error reporting
│       └── logger_model.py          ← centralised logger setup
├── tests/                           ← pytest suite (mirrors package structure)
│   ├── conftest.py                  ← Recorder fixture (snapshot testing)
│   ├── datasets/                    ← pre-pickled DataFrames used by tests
│   ├── csv/ json/                   ← recorded expected outputs
│   ├── ratios/ models/ risk/ ...    ← per-module test folders
│   └── test_toolkit_controller.py   ← integration tests for every module
├── examples/                        ← Jupyter notebooks (one per module)
├── pyproject.toml                   ← build config, deps, linter settings
├── uv.lock                          ← locked dependency graph
├── .pre-commit-config.yaml          ← black, ruff, mypy, codespell hooks
└── .github/workflows/
    ├── linting.yml                  ← black + ruff + pylint on PRs
    └── unit_tests.yml               ← pytest on PRs
```

---

## 4. Architecture Deep-Dive

### 4.1 The MVC Pattern Used Here

The project follows a **Controller + Model** split (no View — this is a library, not an app).

```
User
 │
 ▼
Toolkit (toolkit_controller.py)          ← Controller (hub)
 │  Lazy-loads sub-controllers on demand
 ├─► Ratios (ratios_controller.py)       ← Sub-controller
 │    └─► profitability_model.py         ← Model (pure functions)
 │         get_gross_margin(revenue, cogs) → Series
 │
 ├─► Models (models_controller.py)
 │    └─► dupont_model.py
 │
 ├─► Risk (risk_controller.py)
 │    └─► var_model.py / cvar_model.py / garch_model.py
 │
 └─► ... (same pattern for every module)
```

**Rule:** Model files contain **only** pure calculation functions.  
**Rule:** Controller files handle data wiring, caching, and growth-rate wrappers.  
**Rule:** `toolkit_controller.py` handles data acquisition and state; sub-controllers receive
pre-loaded DataFrames.

### 4.2 Data Flow From User to Result

```
1. Toolkit(tickers=["AAPL","MSFT"], api_key="...", start_date="2020-01-01")
       ↓
2. __init__ validates inputs, sets _start_date / _end_date / _quarterly
       ↓
3. fmp_model.get_financial_data() probes API plan ("Free" vs "Premium")
       ↓
4. normalization_model.initialize_statements_and_normalization()
   loads CSV normalization maps and optionally loads statements from cache
       ↓
5. toolkit.ratios  ← @property
       ├─ calls get_balance_sheet_statement() if not already loaded
       │     └─ fundamentals_model.collect_financial_statements()
       │         └─ fmp_model.get_financial_data() [threaded per ticker]
       │             → fallback: yfinance_model if FMP fails
       │
       ├─ calls get_historical_data(period="yearly") if not loaded
       │     └─ historical_model.get_historical_data() [threaded]
       │         → helpers.enrich_historical_data()
       │             adds: Return, Volatility, Excess Return, Cumulative Return
       │
       └─ returns Ratios(tickers, historical, balance, income, cash)
              ↓
6. toolkit.ratios.collect_profitability_ratios()
       └─ profitability_model.get_gross_margin(revenue, cogs) → Series
       └─ ... (50+ individual functions, concat'd into a DataFrame)
       └─ .round(rounding)
```

### 4.3 Data Sources and Fallback Logic

```
enforce_source param
        │
        ├─ "FinancialModelingPrep"  →  FMP only (raises on failure)
        ├─ "YahooFinance"           →  Yahoo only (raises on failure)
        └─ None (default)
                │
                ├─ api_key present?
                │       YES → try FMP first
                │             FMP returns error sentinel? → fall back to Yahoo Finance
                │       NO  → use Yahoo Finance directly
```

Error sentinels are **column names** in the returned DataFrame (e.g. `"LIMIT REACH"`,
`"INVALID API KEY"`, `"NO DATA"`). The `check_for_error_messages()` function in
`utilities/error_model.py` iterates over all tickers' DataFrames, logs appropriate messages,
and removes failed tickers from the dictionary.

---

## 5. Module-by-Module Reference

### 5.1 Core / Root Level

| File | Role |
|------|------|
| `toolkit_controller.py` | `Toolkit` class. Hub for all data loading and sub-module access. Every sub-module is a `@property` that instantiates its controller on first access. |
| `fmp_model.py` | HTTP client for [FinancialModelingPrep](https://financialmodelingprep.com). Handles retries (up to 12), rate-limit sleep, error sentinel columns, and threading. |
| `yfinance_model.py` | Thin wrapper around `yfinance` for historical OHLCV and financial statements. |
| `historical_model.py` | Orchestrates multi-ticker historical data collection using threads (`threading.Thread`). Converts daily data to weekly/monthly/quarterly/yearly. Fetches US Treasury yields for risk-free rate. |
| `fundamentals_model.py` | `collect_financial_statements()` — threads per ticker, applies normalization CSV maps, resolves FMP/Yahoo differences, returns multi-index DataFrames. |
| `normalization_model.py` | Loads normalization CSV files and maps raw API column names to standardised names (e.g. `"netIncome"` → `"Net Income"`). |
| `helpers.py` | Shared utilities: `calculate_growth()`, `combine_dataframes()`, `enrich_historical_data()`, `convert_isin_to_ticker()`. |
| `currencies_model.py` | Currency conversion helpers used when financial statements are in a different currency than the stock price. |

### 5.2 Ratios

**Path:** `financetoolkit/ratios/`

50+ ratios across five categories:

| File | Ratios included |
|------|----------------|
| `profitability_model.py` | Gross Margin, Operating Margin, Net Profit Margin, ROA, ROE, ROIC, ROCE, etc. |
| `liquidity_model.py` | Current Ratio, Quick Ratio, Cash Ratio, Operating Cash Flow Ratio, etc. |
| `solvency_model.py` | Debt-to-Equity, Debt-to-Assets, Interest Coverage, etc. |
| `efficiency_model.py` | Asset Turnover, Inventory Turnover, Days Sales Outstanding, etc. |
| `valuation_model.py` | P/E, P/B, P/S, EV/EBITDA, Dividend Yield, etc. |
| `ratios_controller.py` | `Ratios` class. `collect_*` methods return all ratios in a category; `get_*` methods return a single ratio. Supports `growth=True` and `lag=N` for growth-rate wrappers. |
| `helpers.py` | `map_period_data_to_daily_data()` for aligning fundamental data with daily price history. |

**Key concept:** Every `collect_*` method checks internal state (`if self._profitability_ratios.empty`)
and only computes on first call — results are cached in the controller instance.

### 5.3 Models

**Path:** `financetoolkit/models/`

Well-known financial valuation models:

| File | Model |
|------|-------|
| `dupont_model.py` | DuPont Analysis (3-factor) and Extended DuPont (5-factor → ROE) |
| `wacc_model.py` | Weighted Average Cost of Capital |
| `enterprise_model.py` | Enterprise Value breakdown |
| `altman_model.py` | Altman Z-Score (bankruptcy predictor) |
| `piotroski_model.py` | Piotroski F-Score (financial health) |
| `intrinsic_model.py` | Discounted Cash Flow / Intrinsic Value |
| `growth_model.py` | Revenue and earnings growth projections |
| `models_controller.py` | `Models` class — orchestrates the above, computes "within-period" Beta for WACC. |

### 5.4 Options

**Path:** `financetoolkit/options/`

| File | Content |
|------|---------|
| `black_scholes_model.py` | Black-Scholes pricing for calls and puts |
| `greeks_model.py` | All 20 Greeks: Δ Delta, Γ Gamma, Θ Theta, Vega, Ρ Rho (1st order); Vanna, Charm, Vomma, Vera, Veta (2nd order); Speed, Zomma, Color, Ultima (3rd order) |
| `binomial_trees_model.py` | Binomial tree option pricing |
| `options_model.py` | Fetches live option chains from Yahoo Finance; implied volatility via scipy.optimize |
| `options_controller.py` | `Options` class. `collect_all_greeks()`, `get_black_scholes_model()`, `get_option_chains()`, Breeden–Litzenberger implied distribution, etc. |

**Note:** Greeks are computed across a matrix of (strike prices × expiration months) for each
ticker, so results are multi-indexed DataFrames with 3 levels.

### 5.5 Risk

**Path:** `financetoolkit/risk/`

| File | Content |
|------|---------|
| `var_model.py` | Value at Risk: Historical, Gaussian, Student-t, Cornish-Fisher distributions |
| `cvar_model.py` | Conditional Value at Risk (Expected Shortfall) |
| `evar_model.py` | Entropic Value at Risk |
| `garch_model.py` | GARCH(1,1) and EWMA volatility models |
| `risk_model.py` | Max Drawdown, Ulcer Index, correlations, skewness, kurtosis |
| `risk_controller.py` | `Risk` class. Accepts any `period` (`intraday`/`daily`/`weekly`/`monthly`/`quarterly`/`yearly`). |

### 5.6 Performance

**Path:** `financetoolkit/performance/`

| File | Content |
|------|---------|
| `performance_model.py` | Sharpe Ratio, Sortino Ratio, Treynor Ratio, Jensen's Alpha, Beta, CAPM, Information Ratio, R-Squared, Factor-Asset Correlations (Fama-French 5 factors) |
| `performance_controller.py` | `Performance` class — same period flexibility as Risk. Downloads Fama-French factor data directly from Ken French's website when needed. |

### 5.7 Technicals

**Path:** `financetoolkit/technicals/`

30+ indicators across four categories:

| File | Indicators |
|------|-----------|
| `momentum_model.py` | RSI, MACD, Stochastic, Williams %R, Average Directional Index, CCI, etc. |
| `overlap_model.py` | SMA, EMA, DEMA, TEMA, WMA, Ichimoku Cloud, VWAP, etc. |
| `volatility_model.py` | Bollinger Bands, ATR, Keltner Channels, Donchian Channels |
| `breadth_model.py` | On-Balance Volume, CMF, Chaikin Oscillator, Force Index, etc. |
| `technicals_controller.py` | `Technicals` class — supports intraday data when `Toolkit(intraday_period="5min")`. |

### 5.8 Fixed Income

**Path:** `financetoolkit/fixedincome/`

No FMP API key needed — all data comes from public sources:

| File | Data source |
|------|------------|
| `fred_model.py` | FRED (Federal Reserve Economic Data) — ICE BofA bond indices, effective yields, OAS spreads |
| `fed_model.py` | Federal Reserve — federal funds rate, balance sheet, quantitative easing data |
| `ecb_model.py` | European Central Bank — key interest rates, inflation |
| `euribor_model.py` | EURIBOR rates |
| `bond_model.py` | Bond valuation: pricing, duration, convexity, Black model (swaptions/caps/floors) |
| `derivative_model.py` | Swap pricing, forward rates |
| `fixedincome_controller.py` | `FixedIncome` class — can be used standalone (`FixedIncome(start_date="2020-01-01")`). |

### 5.9 Economics

**Path:** `financetoolkit/economics/`

| File | Data source |
|------|------------|
| `oecd_model.py` | OECD API — GDP, CPI, Unemployment Rate, Interest Rates, Trade Balance, etc. |
| `gmdb_model.py` | World Bank GMDB — alternative source for macro data |
| `economics_controller.py` | `Economics` class — standalone use: `Economics(start_date="2010-01-01").get_consumer_price_index()`. |

### 5.10 Discovery

**Path:** `financetoolkit/discovery/`

| File | Content |
|------|---------|
| `discovery_model.py` | Calls FMP to search instruments by name, symbol, ISIN, CIK, CUSIP |
| `discovery_controller.py` | `Discovery` class — `search_instruments()`, `get_stock_screener()`, `get_stock_list()`, `get_etf_list()`, full sector/industry screens. Requires FMP API key. |

### 5.11 Portfolio

**Path:** `financetoolkit/portfolio/`

| File | Content |
|------|---------|
| `portfolio_model.py` | Reads Excel/CSV transaction ledger, calculates cost basis, P&L, IRR |
| `overview_model.py` | Summary statistics and per-ticker overview tables |
| `portfolio_controller.py` | `Portfolio` class — loads transaction data, internally creates a `Toolkit` instance for historical price data, exposes all Toolkit metrics scoped to portfolio holdings. |
| `config.yaml` | Column name configuration (date, ticker, price, volume, costs) — customisable for different broker exports. |
| `helpers.py` | YAML config reader, data validation |
| `example_datasets/` | Template `portfolio_template.xlsx` / `.csv` for demonstration |

**How it connects to Toolkit:** `Portfolio` instantiates a `Toolkit` internally and exposes a
`.toolkit` property, giving access to ratios, performance, risk, etc. scoped to the portfolio's
holdings.

### 5.12 Utilities

**Path:** `financetoolkit/utilities/`

| File | Content |
|------|---------|
| `logger_model.py` | `setup_logger()` / `get_logger()` — single named logger (`"financetoolkit"`). All user-visible messages (INFO, WARNING, ERROR) go through here. |
| `error_model.py` | `@handle_errors` decorator — catches `KeyError`, `ValueError`, `AttributeError`, `ZeroDivisionError`, `IndexError`; logs them; returns `pd.Series(dtype="object")` instead of crashing. Also `check_for_error_messages()` for batch ticker error reporting. |
| `cache_model.py` | `load_cached_data()` / `save_cached_data()` — thin wrappers around `pd.read_pickle` and `pickle.dump`. Cache location is configurable at `Toolkit(use_cached_data="my_cache_folder")`. |

### 5.13 Normalization

**Path:** `financetoolkit/normalization/`

Six CSV files that map raw API field names to standardised names:

| File | Maps |
|------|------|
| `balance.csv` | FMP balance sheet columns → standardised names |
| `balance_yf.csv` | Yahoo Finance balance sheet columns → same standardised names |
| `income.csv` / `income_yf.csv` | Income statement (FMP and YF flavours) |
| `cash.csv` / `cash_yf.csv` | Cash flow statement |
| `statistics.csv` | FMP statistics (shares outstanding, market cap, etc.) |

These files are loaded at `Toolkit.__init__` time by `normalization_model.initialize_statements_and_normalization()`.
You can supply custom normalization files via `Toolkit(format_location="my_format_files/")`.

---

## 6. Key Technical Patterns

### Lazy-loading sub-modules

Every sub-module is a `@property` on `Toolkit`. It fetches any missing data and then returns a
freshly constructed controller object. **There is no singleton** — calling `toolkit.ratios` twice
creates two `Ratios` instances, but the underlying DataFrames on `Toolkit` are cached as instance
variables (`self._balance_sheet_statement`, etc.).

### Threading for multi-ticker requests

`historical_model.get_historical_data()` and `fundamentals_model.collect_financial_statements()`
spin up one `threading.Thread` per ticker. Results are written into a shared dictionary (safe
because each thread writes to a different key). `tqdm` shows progress when `progress_bar=True`.

### The `@handle_errors` decorator

All public methods on sub-controllers are wrapped with `@handle_errors` (imported from
`utilities/error_model.py`). It catches calculation errors and returns an empty `pd.Series` rather
than crashing. If you add new model functions and call them from a controller, always decorate the
controller method.

### Growth rates

`helpers.calculate_growth(dataset, lag=1, axis="columns")` wraps `pd.DataFrame.pct_change()`.
Every controller method that returns a metric also accepts `growth=True` and `lag: int | list[int]`.
When `lag` is a list, results gain an extra index level (`"Lag 1"`, `"Lag 2"`, etc.).

### Multi-index DataFrames

Results for multiple tickers are returned as multi-index DataFrames:

- **Fundamental data** (ratios, models): index is `(ticker, metric)`, columns are dates.
- **Historical data** (price, return, etc.): columns are `(field, ticker)`.
- Access a single ticker: `.loc["AAPL"]` or `.xs("AAPL", level=1, axis=1)`.

### Period flexibility

Most controllers accept `period` in `["intraday", "daily", "weekly", "monthly", "quarterly", "yearly"]`.
The controller selects the right historical slice from the dict passed to it.

---

## 7. Data Structures You Will Encounter

### Historical DataFrame (multi-column)

```
Columns:  (Open, AAPL), (Open, MSFT), (High, AAPL), ..., (Adj Close, AAPL), ...
          (Return, AAPL), (Volatility, AAPL), (Excess Return, AAPL), ...
Index:    DatetimeIndex (or PeriodIndex for non-daily)
```

### Financial Statements (multi-index rows)

```
Index level 0: ticker  (AAPL, MSFT)
Index level 1: metric  (Revenue, Net Income, ...)
Columns:        dates  (2019, 2020, 2021, ...)
```

### Risk-free rate DataFrame

```
Single column: "Adj Close"
Index: DatetimeIndex (daily)
```

### Error sentinel DataFrame

```
Single column: "LIMIT REACH" (or other error name)
```
Used as a signal — presence of this column means the API call failed for that ticker.

---

## 8. Dependencies and Toolchain

### Runtime dependencies (`pyproject.toml`)

| Package | Purpose |
|---------|---------|
| `pandas >= 2.2` | All data structures and calculations |
| `scikit-learn >= 1.6` | Used in GARCH / statistical models |
| `requests >= 2.32` | HTTP calls to FMP API |
| `yfinance` | Yahoo Finance data (historical + financials fallback) |
| `openpyxl >= 3.1` | Read/write Excel files for Portfolio module |
| `tqdm >= 4.67` | Progress bars |

### Development toolchain

| Tool | Config location | Purpose |
|------|----------------|---------|
| `uv` | `uv.lock` | Dependency locking and virtual env management |
| `hatchling` | `pyproject.toml [build-system]` | Package build backend |
| `pytest` | `pyproject.toml [tool.pytest]` | Test runner |
| `black` | `pyproject.toml` | Code formatter (line length 122) |
| `ruff` | `pyproject.toml [tool.ruff]` | Fast linter (E/W/F/Q/S/UP/I/PD/SIM/PLC/PLE/PLR/PLW) |
| `pylint` | `pyproject.toml [tool.pylint]` | Additional static analysis |
| `mypy` | `pyproject.toml [tool.mypy]` | Type checking (lenient — several error codes disabled) |
| `codespell` | `pyproject.toml [tool.codespell]` | Spell checking |
| `pre-commit` | `.pre-commit-config.yaml` | Runs black, ruff, mypy, codespell on every commit |

**Python version support:** 3.10, 3.11, 3.12, 3.13.

---

## 9. Testing Strategy

Tests live in `tests/` and mirror the package structure.

### Snapshot / recorder pattern

The project uses a custom snapshot-testing approach (inspired by OpenBB Terminal) implemented in
`tests/conftest.py`.

```python
# A typical test looks like:
def test_profitability_ratios(recorder):
    toolkit = Toolkit(tickers=["AAPL", "MSFT"], balance=balance_dataset, ...)
    recorder.capture(toolkit.ratios.collect_profitability_ratios())
```

`recorder.capture()` serialises the result (DataFrame → CSV, dict → JSON, string → txt) and
compares it to a previously-stored file in `tests/csv/` or `tests/json/`. On first run (no file),
it writes the file. On subsequent runs, it compares and fails if different.

**Floating point:** JSON comparisons use `math.isclose(rel_tol=1e-9)` to avoid spurious failures
across Python versions.

### Pre-loaded fixtures

Tests do **not** make real API calls. All data comes from pickled DataFrames in `tests/datasets/`:

```
tests/datasets/
├── balance_dataset.pickle
├── income_dataset.pickle
├── cash_dataset.pickle
├── historical_dataset.pickle
├── risk_free_rate.pickle
└── treasury_data.pickle
```

### Updating snapshots

```bash
# After a formula change, regenerate the expected outputs:
pytest tests --record-mode=rewrite
```

### Running tests

```bash
pytest tests                     # run all
pytest tests/ratios/             # run one module's tests
pytest tests -k "profitability"  # run matching test names
```

---

## 10. CI/CD Pipeline

Both workflows trigger on PRs and pushes to `main`/`develop`.

### `.github/workflows/linting.yml`

1. `black --diff --check financetoolkit` — format check
2. `ruff check financetoolkit` — fast lint
3. `pylint financetoolkit` — deeper static analysis
4. `markdown-lint` — link checker for README/CONTRIBUTING

### `.github/workflows/unit_tests.yml`

1. Install deps via Poetry
2. `pytest` — runs the full snapshot test suite

**Note:** The workflows still reference `poetry install` (legacy). The dev toolchain has since
migrated to `uv`, but the CI has not been updated yet. Both work for installing the package.

---

## 11. How to Run Locally

### Quick start

```bash
# 1. Clone
git clone https://github.com/JerBouma/FinanceToolkit.git
cd FinanceToolkit

# 2. Set up environment with uv (recommended)
pip install uv
uv venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
uv sync

# 3. Install pre-commit hooks
pip install pre-commit
pre-commit install

# 4. Run tests
pytest tests

# 5. Run linters
black --check financetoolkit
ruff check financetoolkit
```

### Use in a notebook

```python
from financetoolkit import Toolkit

# Free usage (Yahoo Finance only, limited data)
toolkit = Toolkit(tickers=["AAPL", "MSFT"], start_date="2020-01-01")

# Full usage (requires FMP API key)
toolkit = Toolkit(
    tickers=["AAPL", "MSFT"],
    api_key="YOUR_FMP_KEY",
    start_date="2020-01-01",
    quarterly=False,
)

# Get data
df = toolkit.get_historical_data()
ratios = toolkit.ratios.collect_profitability_ratios()
var = toolkit.risk.get_value_at_risk(period="weekly")
```

### Jupyter notebooks

The `examples/` directory contains one notebook per module:

| Notebook | Module |
|----------|--------|
| `Finance Toolkit - 1. Getting Started.ipynb` | Toolkit basics |
| `Finance Toolkit - 2. Discovery Module.ipynb` | Discovery |
| `Finance Toolkit - 3. Ratios Module.ipynb` | Ratios |
| `Finance Toolkit - 4. Models Module.ipynb` | Models |
| `Finance Toolkit - 5. Options Module.ipynb` | Options |
| `Finance Toolkit - 6. Technicals Module.ipynb` | Technicals |
| `Finance Toolkit - 7. Risk Module.ipynb` | Risk |
| `Finance Toolkit - 8. Performance Module.ipynb` | Performance |
| `Finance Toolkit - 9. Economics Module.ipynb` | Economics |
| `Finance Toolkit - 10. Fixed Income Module.ipynb` | Fixed Income |
| `Finance Toolkit - 11. Portfolio Module.ipynb` | Portfolio |
| `Finance Toolkit - Using External Datasets.ipynb` | Custom data |

---

## 12. Configuration and Caching

### Caching

Enable caching to avoid repeated API calls:

```python
toolkit = Toolkit(
    tickers=["AAPL"],
    api_key="...",
    use_cached_data=True,           # uses ./cached/ directory
    # use_cached_data="my_folder",  # or a custom path
)
```

On first run, data is stored as pickle files. On subsequent runs with the same configuration,
data is loaded from cache. Configuration (tickers, dates, etc.) is also persisted as
`configurations.pickle` and will override any different values passed to `Toolkit()`.

### Custom financial statement format

Supply your own normalization CSV files:

```python
toolkit = Toolkit(
    tickers=["AAPL"],
    format_location="path/to/my_normalization_files/",
)
```

Files must follow the same column structure as `financetoolkit/normalization/*.csv`.

### External datasets

Pass pre-loaded DataFrames directly (no API key needed for calculations):

```python
import pandas as pd

toolkit = Toolkit(
    tickers=["AAPL"],
    historical=pd.read_csv("my_prices.csv", index_col=0),
    balance=pd.read_csv("my_balance.csv", index_col=[0, 1]),
    income=pd.read_csv("my_income.csv", index_col=[0, 1]),
    cash=pd.read_csv("my_cash.csv", index_col=[0, 1]),
)
```

See `examples/Finance Toolkit - Using External Datasets.ipynb` for the required column names and
index structure.

---

## 13. Common Patterns When Adding New Functionality

### Adding a new ratio

1. Add the formula to the appropriate `*_model.py` under `financetoolkit/ratios/`:
   ```python
   def get_my_new_ratio(numerator: pd.Series, denominator: pd.Series) -> pd.Series:
       """Docstring with formula, args, returns."""
       return numerator / denominator
   ```

2. Add a `get_my_new_ratio()` method to `ratios_controller.py` that:
   - Pulls the required line items from `self._income_statement` / `self._balance_sheet_statement`
   - Calls the model function
   - Wraps with `@handle_errors`
   - Supports `growth`, `lag`, `trailing`, `rounding` parameters (follow existing examples)

3. Optionally add it to `collect_*_ratios()` if it belongs to a category.

4. Add a test in `tests/ratios/test_<category>_model.py` using the `recorder` fixture.

### Adding a new module

1. Create `financetoolkit/<my_module>/` with:
   - `__init__.py`
   - `<my_module>_model.py` (pure functions)
   - `<my_module>_controller.py` (controller class)
   - `helpers.py` if needed

2. Export the controller class from `financetoolkit/__init__.py`.

3. Add a `@property` to `Toolkit` in `toolkit_controller.py` that instantiates the controller
   (follow the `ratios` property as a template).

4. Add tests under `tests/<my_module>/`.

---

## 14. Known Constraints and Gotchas

| Issue | Detail |
|-------|--------|
| **Free FMP plan = US stocks only** | The free plan from FinancialModelingPrep is limited to US exchange stocks and 5 years of data with 250 requests/day. Non-US tickers fall back to Yahoo Finance. |
| **Yahoo Finance rate limits** | Yahoo Finance has undocumented rate limits. When hit, the toolkit returns a `"YFINANCE RATE LIMIT REACHED"` sentinel and logs a warning. Adding short sleeps or using FMP can help. |
| **Empty Series instead of exceptions** | All controller methods decorated with `@handle_errors` return `pd.Series(dtype="object")` on error instead of raising. Check for empty returns before chaining operations. |
| **`toolkit.ratios` is not cached** | Every call to `toolkit.ratios` creates a new `Ratios` instance. Assign it to a variable: `ratios = toolkit.ratios`. |
| **Currency conversion** | By default, `convert_currency=True` only on FMP Premium plans. On Free plans currency conversion is skipped, which can cause cross-ticker inconsistencies when comparing e.g. a USD stock against a EUR stock. |
| **RuntimeWarning suppressed** | `warnings.filterwarnings("ignore", category=RuntimeWarning)` is set in several modules because division-by-zero is expected in financial calculations (e.g. P/E when earnings are negative). |
| **`tqdm` optional** | The code does `try: from tqdm import tqdm; ENABLE_TQDM = True except ImportError: ENABLE_TQDM = False` — progress bars only show when tqdm is installed. |
| **`TICKER_LIMIT = 20`** | Defined in `toolkit_controller.py`. A warning is shown (but not enforced) when more than 20 tickers are provided. |
| **Intraday requires FMP Premium** | `Toolkit(intraday_period="5min")` only works with a Premium FMP subscription. |
| **Snapshot tests write on first run** | If a test file doesn't exist yet, `recorder.capture()` writes it and the test passes. On the next run it compares. Make sure you review generated snapshots before committing. |

---

## 15. Useful External Links

| Resource | URL |
|----------|-----|
| Official documentation | https://www.jeroenbouma.com/projects/financetoolkit |
| Code docs (auto-generated) | https://www.jeroenbouma.com/projects/financetoolkit/docs |
| FMP API key (15% discount affiliate) | https://www.jeroenbouma.com/fmp |
| Finance Database (300k+ symbols) | https://github.com/JerBouma/FinanceDatabase |
| PyPI package | https://pypi.org/project/FinanceToolkit/ |
| Getting Started notebook | https://www.jeroenbouma.com/projects/financetoolkit/getting-started |
| External datasets notebook | https://www.jeroenbouma.com/projects/financetoolkit/external-datasets |
| GitHub Issues | https://github.com/JerBouma/FinanceToolkit/issues |
| Ken French factor data | https://mba.tuck.dartmouth.edu/pages/faculty/ken.french/data_library.html |

---

*Handover document generated for FinanceToolkit v2.0.7 — April 2026.*
