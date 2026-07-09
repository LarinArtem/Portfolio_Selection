# Regime-Aware Multi-Factor Portfolio Selection (Bachelor's Final Project)

An end-to-end quantitative stock-selection system: it builds a clean US equity universe from raw API data, scores every stock on six investment factors, adapts the factor weights to the current volatility regime using a Hidden Markov Model and an ARCH model, and finally assembles low-correlation portfolios diversified across all 11 GICS sectors.

## Overview

The project (`final_bachlors_project.ipynb`) answers a practical question: *given the entire US mid-cap+ universe, which 15–20 stocks should be in the portfolio right now?* Instead of a static factor blend, the system recognizes that some factors (e.g., momentum) behave differently in calm vs. turbulent markets, so it detects the current volatility regime and re-weights the factors accordingly.

## Pipeline design

### 1. Universe construction
- Pulls the full listing of stocks and ETFs from the **Polygon API**.
- Persists everything in a local **SQLite** database with helper functions for reads/writes, so API calls are made once and cached.
- Filters down to a high-quality universe: **common stock only**, **US-domestic** companies (no ADRs), **mid-cap and above** (market cap ≥ $2B), and an **IPO date at least one year old** (so a full year of returns/volatility exists).
- Sector classification is retained so that valuation comparisons are made *within* sectors — "apples to apples."

### 2. Factor model
Each stock receives a z-score-standardized score on six factors:

| Factor | Inputs | Weight |
|---|---|---|
| Quality (Q) | ROE, ROA, profit margin | 0.25 |
| Size (S) | Market cap | 0.10 |
| Value (V) | P/E, PEG, P/B, EV/EBITDA, EV/Revenue | 0.20 |
| Dividend (D) | Dividend yield | 0.05 |
| Momentum (M) | 12-1 month return (skipping the last month to avoid short-term reversal bias) | 0.20 |
| Low Volatility (VOL) | 1-year daily-return standard deviation (lower is better) | 0.20 |

Z-scores are computed per sector, then combined into a single composite score.

### 3. Volatility-regime detection (HMM + ARCH)
This is the project's core innovation:

- A **Gaussian Hidden Markov Model** (`hmmlearn`) is fit on market returns to learn distinct volatility states.
- An **ARCH model** (`arch`) estimates current conditional volatility, identifying which HMM state the market is in *today*.
- The average of the recent state sequence is used as a high/low volatility indicator (above 0.5 → high-vol regime).
- Factor weights in the composite score are then adjusted by regime — defensive factors (quality, low-vol) get more weight in high-volatility regimes, momentum/value more in calm regimes.

### 4. Ranking and portfolio assembly
- Stocks are ranked by the regime-adjusted combined score; the top names per sector are shortlisted.
- The final step generates candidate portfolios of **15–20 stocks spanning all 11 sectors**, sampling combinations and preferring those with **low pairwise return correlation** — diversification is enforced explicitly rather than assumed from sector spread alone.

### 5. Supporting infrastructure
Separate SQLite tables (with dedicated helper functions) store fundamentals, technical indicators, and historical prices, keeping the notebook reproducible and fast to re-run.

## Tech stack

Python, `pandas`, `numpy`, `yfinance`, Polygon API, `financedatabase`, `sqlite3`, `hmmlearn` (Gaussian HMM), `arch` (ARCH/GARCH), `scikit-learn`, `scipy`, `matplotlib`, `seaborn`.

## How to run

1. Install dependencies:
   ```bash
   pip install pandas numpy yfinance financedatabase hmmlearn arch scikit-learn scipy matplotlib seaborn requests
   ```
2. Add your Polygon API key where the API client is configured.
3. Run the notebook top to bottom. First run populates the SQLite database (slow); subsequent runs read from cache.

## Limitations & future work

- Factor weights outside the regime adjustment are fixed heuristics; they could be learned via backtesting.
- The correlation-based portfolio search is randomized sampling; a mean-variance or risk-parity optimizer would make it exact.
- Adding transaction-cost-aware rebalancing and an out-of-sample backtest would complete the research loop.
