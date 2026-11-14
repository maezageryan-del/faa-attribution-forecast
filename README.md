# FAA outage attribution forecast — empirical change-quantile entry

`main.ts` (Node 22, standard library only, no network) forecasts, for every
market of Polymarket's FAA outage attribution family (attributed to Iran /
a foreign actor / a hack / an AI sandbox escape), the five quantiles
(0.05 / 0.25 / 0.5 / 0.75 / 0.95) of the Yes price at each of the 240 minutes
after a case's checkpoint.

The design carries over from the twin "Next CEO of NVIDIA?" forecast
contest (same harness, same scorer), with one adaptation: the FAA markets
were created on 2026-11-13, so the final's cases hold far less history
than the practice's — the first final case gives each market only about
1,000 minutes. The estimator is built to stay honest on histories that
short.

## How it works

- **Per-market empirical change distribution.** For each market, the bot
  collects the overlapping h-minute price changes from the trailing ~18
  hours of the case's own history (or all of it, when there is less) and
  takes their empirical quantiles at knot horizons (1, 5, 15, 30, 60, 120,
  240 minutes), interpolated in sqrt(h). This captures what a Gaussian
  random walk cannot: minutes that do not move at all, tick-sized steps,
  jumps, and the strong mean reversion of these prices at minute scale.
- **Recency weighting.** Change samples are weighted by exp(-age / 150
  min): the market's behavior in the last few hours says much more about
  the next four hours than its launch minutes do — doubly true for a
  market whose first hour contained repricings of 20 points.
- **Volatility-regime scaling.** Each market's band is scaled by a shrunk
  (x0.75), clamped (0.7–1.4) ratio of its recent 360-minute
  30-minute-change width to its full-window width, so a market that has
  just woken up gets wider bands and one that has gone quiet gets narrower
  ones than the case-wide calibration alone would give it.
- **Persistence median, with a dwell correction.** The quantile shape is
  centered so the median sits on the last price; the inner pair is
  narrowed slightly (x0.9) and the outer pair starts from a x1.3 widening,
  both then case-calibrated. On top of that, a last price that changed
  only a few minutes before the checkpoint is treated as an unconfirmed
  print (order-book midpoint bounce): the center blends partway back
  toward the previous price level, with weight falling off as the new
  level ages (5/(dwell+5), strength 0.45, ramped in over 45 minutes).
- **Pooled width floor.** Every market's band is widened additively by
  0.25x the case's pooled cross-market quantile deviations, so a dormant
  market that suddenly wakes up is not forecast with a near-zero band just
  because its own recent history is flat.
- **Per-case calibration, adapted to young markets.** Before forecasting,
  the same estimator is backtested at pseudo-checkpoints inside the
  case's history (forecasts made only from data up to each checkpoint,
  scored against the realized 30/120/240-minute moves). The backtest
  picks one band multiplier for short horizons, one for long horizons,
  and one outer-quantile widening for the whole case. On a market whose
  history is too short for the usual checkpoint grid (start at minute
  480, every 240), the backtest samples from minute 240 every 120
  minutes instead, so calibration still engages on a ~1,000-minute
  history rather than silently falling back to defaults; on long
  histories the grid — and the result — is unchanged. In truncated-history
  tests on the practice data (600-minute histories), this sampling was
  worth about 0.010 of loss on average and never cost more than 0.003 on
  any single case.
- **Fallbacks.** A market with too little history falls back to a Gaussian
  shape with that market's own RMS minute change (floored/capped), or to
  the house baseline's sigma when even that is unavailable.

Everything is deterministic: no randomness, no clock, no network — two
runs of a case produce byte-identical output, and a single case runs in
well under a second (about 0.2 s on a four-market case).

On the kit's six practice cases (cut from the NVIDIA event, as the rules
explain) this scores a mean pinball loss of 0.268992, against the house
baseline's 0.363589 on the same cases (per case: 0.4799 / 0.2994 / 0.3467
/ 0.2453 / 0.1994 / 0.1088). The rules are explicit that the practice score
is no guide to the final; it only verifies the machinery end to end.
