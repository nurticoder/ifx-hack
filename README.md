# IFX Hack

An AI-driven dynamic index prototype that adjusts portfolio weights when the
market appears to be entering a high-volatility or crisis regime. The project
compares a risk-adjusted portfolio with an equal-weight index using historical
returns from five Hong Kong-listed technology companies.

> **Disclaimer:** This is a research and demonstration project, not financial
> advice. Backtests and the illustrative stress scenario do not guarantee
> future performance.

## What it does

The Streamlit dashboard in [`app.py`](./app.py) provides:

- A walk-forward backtest of the AI portfolio versus an equal-weight index.
- Total return, maximum drawdown, and crisis-period return comparisons.
- A configurable AI crisis-sensitivity parameter.
- A configurable allocation-aggressiveness parameter.
- An illustrative 2008-style shock replay based on observed asset betas.
- Current portfolio allocations and a correlation-network visualization.

The command-line engine in [`run_engine.py`](./run_engine.py) provides a
lightweight, non-interactive version of the analysis. It prints:

- Random Forest feature importances.
- Systemic centrality scores for each ticker.
- Naive equal-weight and AI risk-adjusted portfolio weights.

## Data

[`fetch_data.py`](./fetch_data.py) downloads adjusted closing prices and
trading volumes from Yahoo Finance using `yfinance`. The configured tickers are:

| Ticker | Exchange/company |
| --- | --- |
| `0700.HK` | Tencent |
| `9988.HK` | Alibaba |
| `3690.HK` | Meituan |
| `1810.HK` | Xiaomi |
| `9618.HK` | JD.com |

The download range is from `2023-01-01` through `2026-10-01` (the end date is
exclusive in `yfinance`). It creates:

- `price_matrix.csv` — adjusted closing prices by trading day and ticker.
- `volume_matrix.csv` — trading volume by trading day and ticker.

The dashboard currently reads `price_matrix.csv`. The command-line engine reads
both CSV files.

## Requirements

- Python 3.13 or newer
- [`uv`](https://docs.astral.sh/uv/) recommended for environment and dependency
  management
- Internet access when refreshing data from Yahoo Finance

Dependencies are declared in [`pyproject.toml`](./pyproject.toml), including
Streamlit, pandas, NumPy, scikit-learn, SciPy, NetworkX, Matplotlib, and
`yfinance`.

## Setup

From the repository root:

```bash
uv sync
```

If you do not use `uv`, create a Python 3.13+ environment and install the
dependencies declared in `pyproject.toml` with your preferred package manager.

## Refresh market data

To download the configured historical prices and volumes:

```bash
uv run python fetch_data.py
```

This overwrites `price_matrix.csv` and `volume_matrix.csv` in the repository
root. Refreshing the data is optional if the checked-in CSV files are suitable
for the analysis.

## Run the dashboard

Start the interactive Streamlit application with:

```bash
uv run streamlit run app.py
```

Streamlit will print a local URL, normally
`http://localhost:8501`. Use the sidebar controls to adjust:

- **AI crisis sensitivity** — how strongly predicted volatility increases the
  shift toward the risk-adjusted allocation.
- **Allocation Aggressiveness** — how strongly lower-centrality assets are
  favored in the risk-adjusted allocation.

## Run the command-line engine

To run the non-interactive analysis:

```bash
uv run python run_engine.py
```

The engine expects both `price_matrix.csv` and `volume_matrix.csv` in the
current working directory. Run it from the repository root or update the
working-directory assumptions in the code before invoking it elsewhere.

## Method overview

### Dashboard backtest

For each rebalance point, the dashboard:

1. Calculates market returns as the cross-sectional mean of the five ticker
   returns.
2. Builds rolling five-day volatility, rolling 20-day volatility, and
   five-day momentum features.
3. Trains a small `RandomForestRegressor` on data available before that
   rebalance.
4. Predicts the next five-day volatility regime.
5. Builds a weighted correlation graph over the trailing 60 trading days.
6. Computes weighted degree (network strength) as a systemic-centrality score.
7. Allocates more weight to lower-centrality assets using the selected
   aggressiveness.
8. Blends that allocation with the equal-weight portfolio according to the
   predicted crisis level.

The backtest rebalances every 20 trading days and uses a minimum training
history of 120 observations.

### Command-line engine

The command-line path uses the same general centrality and inverse-centrality
allocation idea, but its features differ slightly: it includes average trading
volume momentum and returns feature importances rather than running the
dashboard's walk-forward simulation.

## Repository layout

```text
.
├── app.py             # Streamlit dashboard and walk-forward backtest
├── fetch_data.py      # Yahoo Finance data downloader
├── run_engine.py      # Command-line feature and allocation engine
├── price_matrix.csv   # Historical adjusted-close input data
├── volume_matrix.csv  # Historical volume input data
├── pyproject.toml     # Project metadata and dependencies
└── uv.lock            # Locked dependency versions
```

## Notes and limitations

- Yahoo Finance availability, ticker coverage, and revised historical data can
  change over time.
- The application uses only the configured five-ticker universe; it is not a
  diversified market index.
- The “2008-style” section is a synthetic scenario, not an actual 2008
  backtest. The selected tickers do not all have price history from 2008.
- Transaction costs, slippage, taxes, liquidity constraints, and execution
  delays are not modeled.
- The crisis label is a heuristic based on realized volatility relative to a
  rolling market-volatility quantile.
- The CSV files are local inputs and are not validated against a remote data
  source before analysis.
