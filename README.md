# Forecast Polymarket's next NVIDIA CEO prices minute by minute

Entry for the make.fun bounty
[`polymarket-nvidia-ceo-forecast-01`](https://make.fun/bounty/polymarket-nvidia-ceo-forecast-01):
forecast the Yes price of every market of Polymarket's event
"Next CEO of NVIDIA?" at every minute of the 4 hours after a checkpoint,
as five quantiles (0.05, 0.25, 0.5, 0.75, 0.95) per market per minute.

## How to run

```sh
python3 main.py
```

The grader runs `python3 /bot/main.py` with the case at `/input/<id>.in`
and expects `/output/<id>.out`. Outside the grader,
`MAKEFUN_INPUT_DIR` / `MAKEFUN_OUTPUT_DIR` stand in for `/input` /
`/output`:

```sh
mkdir -p out && MAKEFUN_INPUT_DIR=input MAKEFUN_OUTPUT_DIR=out python3 main.py
```

Python 3.13, standard library only, no network. One case takes well
under a second.

## Approach (v2)

- **Median: persistence.** The last observed price is carried forward.
  Prediction-market prices are (close to) martingales at this horizon:
  in backtests, every mean-reversion and trend variant of the median
  scored worse than the last price.
- **Spread: per-market, per-case, robust.** Volatility differs wildly
  across markets (a 0.30 favourite moves in whole cents; a 0.003
  longshot barely moves) and across regimes, so the spread is
  re-estimated from each case's own history: the 90th percentile of the
  absolute one-minute price changes over the last 3 and 12 hours,
  converted to a per-root-minute sigma (÷ 1.644854), averaged over the
  two windows and shrunk by 0.70, clamped to [0.0003, 0.006]. A plain
  standard deviation is dominated by the rare huge jumps (up to 0.48 in
  a minute in the early history) and makes the forecast far too wide;
  the robust scale tracks the typical minute instead.
- **Shape:** normal quantiles widening with `sqrt(horizon)`:
  `q_tau(h) = clamp(x + z_tau * sigma * sqrt(h), 0, 1)`. Heavier-tailed
  (Student-t) variants scored worse in backtests.
- **Cross-market correction (new in v2): sum-to-one.** The 24 markets
  are mutually exclusive outcomes of one question, so their true
  probabilities sum to 1 — but the observed sum of last prices drifts
  from 1 (0.81–1.09 across backtest windows) through market noise.
  Since the median carries most of the pinball loss, the medians are
  shrunk 40% of the way toward sum = 1: `f = 1 + 0.4 * (1/s - 1)` where
  `s` is the sum of last prices, and every quantile of a market is
  shifted by `x * (f - 1)` (multiplicative, so the spread is preserved
  and tiny-priced markets barely move). The correction is
  horizon-independent and is small exactly when the market is already
  coherent (calm windows, sum near 1), and large exactly when the
  market is incoherent (jumpy windows) — where it helps most.

## Backtest

Mean pinball loss (percentage points; lower is better) over eight
4-hour windows: practice cases p00–p04 scored against the prices in
the following practice case, plus p05 and the 06:00 / 08:00 UTC
checkpoints of 13 Nov 2026 scored against live Polymarket price
history fetched the same day (the 1-minute grid was rebuilt exactly as
the house builds it; the rebuild reproduced the house's own practice
prices minute-for-minute on the overlap).

| program | mean pinball loss |
|---|---|
| house baseline (fixed sigma 0.0025) | 0.333 |
| v1 (round 1 submission) | 0.231 |
| **v2 (this program)** | **0.225** |

v2 improves v1 by ~2.7% overall, concentrated where v1 was weakest:
the jumpy regime (p00 0.456 → 0.420, p02 0.362 → 0.352) and neutral
(≤ 0.001) in the calm windows. It beats the baseline in every one of
the eight windows and in every horizon bucket. Output is
deterministic: a pure function of the input file, byte-identical
across runs.

## License

MIT — see `LICENSE`.
