# Equity Walk-Forward Backtest (US Equities)

A research-style, walk-forward backtesting notebook for simple equity signals using daily US equity/ETF data. The project focuses on leakage-safe feature construction, realistic next-day execution, transaction costs, and out-of-sample evaluation.

## What this project does
- Downloads daily OHLCV data from Stooq for a small universe of liquid US tickers
- Builds rolling features (returns, rolling mean/std, z-scores)
- Generates trading signals (mean reversion) and simulates next-day execution
- Runs a rolling walk-forward evaluation (train window → test window)
- Applies per-side transaction costs based on turnover
- Reports out-of-sample performance metrics and plots equity/drawdown

## Data
- Source: Stooq daily CSV
- Universe (20 tickers): SPY, AAPL, MSFT, AMZN, NVDA, GOOGL, META, JPM, XOM, JNJ, PG, KO, PEP, WMT, COST, HD, V, MA, UNH, LLY
- Date range: 1970-01-02 to 2026-01-15

## Strategy (Mean Reversion)
### Features
- Daily returns: `r_t = P_t / P_{t-1} - 1`
- Rolling z-score over a window `w`:
  - `z_t = (r_t - mean_w(r)) / std_w(r)`

### Entry-only variant
- Enter long when `z_t < entry_z`
- Position is applied next day to avoid lookahead: `pos_{t+1} = signal_t`

### Entry + Exit variant
A stateful rule to reduce tail risk / overholding:
- Enter long when `z_t < entry_z`
- Exit when `z_t > exit_z` (fixed `exit_z = -1` in current comparison)
- Next-day execution: positions are shifted by 1 day

## Backtest assumptions
- Execution: next trading day (signals computed on day *t*, positions applied on day *t+1*)
- Portfolio: equal-weight across tickers each day
- Costs: 5 bps per side (0.0005) applied when a sleeve changes position (turnover-based)

## Walk-forward validation
Rolling walk-forward backtest:
- Train window: 504 trading days (~2 years)
- Test window: 63 trading days (~3 months)
- Step size: 63 trading days
- Parameter selection on each train window using Sharpe (net of costs)
- Test features computed using train + test history (realistic availability), but performance is measured on test dates only

## Results (Out-of-sample, with costs)
Comparison from the current run:

| Variant | Sharpe (OOS) | Max Drawdown (OOS) | Avg Daily Turnover (OOS) |
|---|---:|---:|---:|
| Entry-only | 0.223 | -0.255 | 0.044 |
| Entry + Exit (exit_z = -1) | 0.353 | -0.122 | 0.039 |

Interpretation:
- Adding an exit rule improved risk-adjusted performance and roughly halved max drawdown while slightly reducing turnover.

## How to run
- Open notebooks/01_stooq_walkforward.ipynb and run top-to-bottom:
  1) download prices
  2) compute features/signals
  3) run walk-forward backtests
  4) produce plots and metrics

Dependencies are listed in `requirements.txt`.

## Next improvements
- Add benchmark comparisons (SPY buy & hold, equal-weight buy & hold)
- Add parameter sweeps for exit thresholds and windows
- Add volatility targeting / position sizing
- Move notebook logic into `src/` and create a single `scripts/run_backtest.py` runner
- Add unit tests for turnover/cost logic and leakage checks
