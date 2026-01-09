# Donchian–Keltner Leverage Rotation Backtest 

A rules-based backtest is implemented for a long-only Nasdaq regime strategy that combines (i) a Donchian breakout entry, (ii) a Keltner-band risk exit, and (iii) a volatility-conditioned “leverage rotation” between 3×, 2×, 1× exposure (or cash). The design goal has been to test whether leveraged equity exposure can be held for longer horizons when market participation is gated by trend/risk filters, with the primary objective of reducing maximum drawdown relative to a continuously levered buy-and-hold allocation.

The motivation has been aligned with the view that long-horizon underperformance in daily-reset leverage is driven primarily by realized volatility and return path dependency (the “constant leverage trap”), rather than by an inevitable mechanical “decay” in all environments. 

## Strategy overview

A single “signal” price series is used to determine whether the market is risk-on or risk-off. When the regime is risk-on, an exposure level is selected from a small set of instruments based on realized volatility. When the regime is risk-off, cash is held.

The strategy is composed of two layers:

1. **Regime layer (risk-on / risk-off).**  
   A long regime is entered after a Donchian breakout and is exited after a downside Keltner-band breach. Donchian and Keltner channel constructions are standard tools in systematic trend-following and breakout timing. 

2. **Leverage selection layer (3× / 2× / 1× / cash).**  
   When risk-on, the held instrument is selected by comparing annualized realized volatility to fixed thresholds. Lower volatility is mapped to higher leverage; higher volatility is mapped to lower leverage and, beyond a threshold, to cash. This implementation is directionally consistent with the idea that leverage has been most viable in low-volatility environments and should be avoided in high-volatility regimes. 
## Data and instruments

- **Signal series.**  
  The signal series is downloaded with `yfinance` and used to compute OHLC-based indicators and daily returns.

- **Trade instruments.**  
  The strategy rotates between `TQQQ` (3×), `QLD` (2×), and `QQQ` (1×). A cash proxy can be enabled (for example `BIL`), otherwise cash is modeled with zero return.

**Synthetic leverage mode (long history).**  
The trade layer of this strategy is explicitly expressed as a rotation between `TQQQ` (3×), `QLD` (2×), and `QQQ` (1×). A hard practical constraint is that **`TQQQ` only has live price history back to its inception on February 9, 2010**; therefore, any backtest that uses the *actual* `TQQQ` time series cannot begin before 2010. (For additional context, `QLD` launched in 2006 and `QQQ` in 1999, so the 3× leg is the binding limitation when you require a consistent instrument set.)
To evaluate the same regime/rotation logic across a wider range of market environments than the post-2010 sample allows, an optional **synthetic long-history mode** is provided.

When synthetic leverage is enabled, the trade instruments are reconstructed as daily-reset constant-leverage series from the underlying signal’s daily close-to-close returns (used as a proxy for `QQQ`-like exposure over a longer sample). Concretely:

- The underlying simple return is computed as `r[t] = Close[t] / Close[t-1] - 1`.
- The leveraged return is modeled as `r_L[t] = L * r[t]` with a clamp on extreme negative days to prevent non-positive prices.
- The synthetic price is compounded as `Price_L[t] = Price_L[t-1] * (1 + r_L[t])`.

This reconstruction captures daily compounding and volatility drag mechanically, but it does not reproduce real ETF tracking error, financing costs, fee drag, dividend treatment, borrow constraints, or rebalance frictions. The volatility/path-dependence mechanism that motivates regime filtering is discussed in the supporting literature.

## Signal construction

### Donchian breakout entry
- A Donchian breakout level is computed as the rolling maximum of the **prior** `DONCHIAN_LEN` highs (today is excluded by shifting the series by one day).
- A risk-on regime is entered when `Close` exceeds that Donchian breakout level.

Donchian channels are commonly used to identify breakout points and trend conditions.

### Keltner-band exit
- A Keltner basis is computed as an EMA of the close with length `KELTNER_EMA_LEN`.
- An ATR-style volatility estimate is computed from OHLC using `KELTNER_ATR_LEN`.
- A downside band is formed as `basis − KELTNER_MULT × ATR`.
- The position is exited when `Close` crosses below the downside Keltner band.

Keltner channels are frequently specified as an EMA plus/minus a volatility band, commonly defined via ATR.

### Cooldown
After an exit, a `COOLDOWN_DAYS` period is enforced during which the regime is forced to risk-off. This is intended to reduce immediate re-entry during whipsaw conditions.

### Execution lag
A one-day execution lag (`LAG_DAYS`) is applied so that signals generated at the close are assumed to be executed on the next trading day. This lag is intended to avoid implicit look-ahead from same-bar execution.

## Leverage rotation rule

When the regime is risk-on, annualized realized volatility is computed from the signal return series using a rolling window (`VOL_LEN`). The following mapping is applied:

- If vol < `VOL_TH1`, the 3× instrument is held.
- Else if vol < `VOL_TH2`, the 2× instrument is held.
- Else if vol < `VOL_TH3`, the 1× instrument is held.
- Else, cash is held.

This mapping is intended to operationalize the principle that leverage is most defensible when realized volatility is sufficiently low, and that it becomes fragile during high-volatility “seesaw” regimes. 

## Transaction cost model

A simple proportional commission rate (`COMMISSION_RATE`) is applied based on “legs”:

- Cash → risk asset: 1 leg (buy).
- Risk asset → cash: 1 leg (sell).
- Risk asset → risk asset: 2 legs (sell + buy).

Slippage, bid–ask spread, market impact, borrow/financing costs, and tax effects are not modeled unless explicitly added.

## Outputs

The notebook produces:

- A daily time series of:
  - regime state, desired exposure, held instrument, realized volatility,
  - gross returns, costs, net returns,
  - equity curve and drawdown.
- Summary metrics:
  - total return, CAGR, Sharpe, Sortino, maximum drawdown, turnover proxy, and average time spent in each instrument.
- A performance dashboard including:
  - equity curve, drawdown curve, return heatmap, rolling Sharpe/volatility, return distribution histogram,
  - instrument price curves and instrument max drawdowns.

## Key assumptions and limitations

- No investment advice is provided, and no claims of live tradability are made.
- Synthetic leverage series are daily-reset constant-leverage reconstructions and are not equal to realized ETF total returns. Fee drag, financing costs, tracking error, and operational frictions are excluded by design.
- The signal series may not represent total-return performance (dividends may be omitted depending on the chosen underlying), and this can materially affect long-horizon comparisons.
- Transaction costs are simplified to a commission-only model; slippage and market impact are not included.
- Parameter values are illustrative and are not presented as optimal. Robustness across regimes and parameter sensitivity should be treated as a required extension rather than an optional enhancement.
- The channel logic implemented here is a simplified variant of academic channel/timing specifications (for example, in the use of a single Donchian entry rule and a single Keltner lower-band exit rule), and results should be interpreted accordingly.

## Parameter selection (fixed vs. iterative)

Two parameter-selection modes are supported.

1. Fixed parameters (default).  
   All strategy parameters are set explicitly in the “Parameters” cell at the top of the notebook. The backtest is then run using those fixed values. This mode is intended for transparent, repeatable evaluation of a single specification.

2. Iterative parameter search (optional).  
   A grid search can be enabled by setting `RUN_GRID_SEARCH = True`. When enabled, combinations of channel lengths, band multipliers, cooldown length, and volatility thresholds are iterated on a train-only subset of the data. Invalid combinations (for example, non-ordered volatility thresholds) are excluded. An optional maximum drawdown constraint can be enforced during the iteration via `MAX_DD_ALLOWED`, and a minimum risk-on exposure constraint can be enforced via `MIN_RISK_ON_DAYS` (if used).

After the search is completed, the best-performing parameter set (by the selected objective metric in the grid-search cell) is written back into the global parameter variables and can be used for a subsequent run. The grid-search block is intended as an exploratory tool and is not required for the core backtest.


## How to run

1. Install dependencies:
   - `numpy`, `pandas`, `matplotlib`, `yfinance`, `tqdm`
2. Open the notebook locally (or in Colab).
3. Set parameters in the “Parameters” cell (tickers, windows, thresholds, costs, and dates).
4. Run all cells to download data, build synthetic instruments (if enabled), run the backtest, and render the dashboard.

## References (papers included in this repository)

- “Leverage for the Long Run: A Systematic Approach to Managing Risk and Magnifying Returns.”
- “Does Trend Following Still Work on Stocks?”
- Additional background papers on generalized momentum and allocation frameworks are included for context.

