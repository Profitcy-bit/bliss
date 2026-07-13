# Evaluation — the ruler

The evaluation layer is the system's court of final appeal. Its design goal is
narrow and absolute: **make self-flattery structurally impossible.**

## 1. Point-in-time replay

The harness walks a candle series bar by bar. At bar *i*, the analysis and
grading pipeline receives `candles[:i+1]` — nothing else. The setup graded at
*i* is then labeled against the bars that follow. Two properties are pinned by
test rather than convention:

- **No-lookahead:** the read produced from a truncated series must be
  byte-identical to the read produced by slicing a longer series to the same
  point. Future data cannot reach backward.
- **Determinism:** identical inputs produce identical labels, always.

## 2. Outcome labeling — two models, one lesson

**Legacy labeler (retained as the pinned baseline):** assumes the planned
entry fills instantly; walks forward with stop-first same-bar resolution to
the best target reached.

**Execution-faithful labeler (the honest model):** mirrors the live position
monitor's pending-order semantics exactly:

| situation | resolution |
|---|---|
| No bar has traded through the entry | order is *pending* — nothing counts |
| Bar spans entry and stop together | filled and stopped: a loss (conservative ambiguity rule) |
| Target struck while still pending | **cancelled** — the trade never existed; excluded from hit rate |
| Pending past the working window | cancelled (expired) |
| Filled, target struck on a later bar | win banked at target (live exit semantics) |
| Filled, neither side resolves in budget | marked at the window's final close |

The symmetry between this labeler and the live monitor is deliberate: the
backtest and the runtime share one definition of reality.

## 3. Calibration

Per-grade statistics answer "do the grades mean anything?": hit rates with
Wilson score intervals, average realized R, edge versus the naive baseline,
Brier score, expected calibration error, and a monotonicity check — the tier
ladder must actually order outcomes (Prime ≥ Valid ≥ Watch ≥ Avoid), pooled
and per regime. Cancelled entries never enter hit rates. All reported
performance is reproducible from the journal.

## 4. Overfitting surveillance

Sharpe-like point estimates are treated as suspects: the rigor module reports
Probabilistic Sharpe Ratio, Deflated Sharpe Ratio (accounting for the number
of strategy variants tried), and Probability of Backtest Overfitting. A
candidate that wins its A/B but fails rigor is a NO-GO.

## 5. Case study — the phantom-fill subsidy

The project's defining incident. For weeks the legacy labeler's
entries-always-fill assumption looked innocent. The first live paper session
falsified it operationally: three resting limit orders, zero fills, price
never returning to the entries. Re-measuring the whole system under the
execution-faithful labeler on real broker data produced:

| metric (Prime setups) | assumed fills | honest fills |
|---|---|---|
| fill rate | 100% (by construction) | ≈ 30% |
| expectancy per attempt | +0.65R | ≈ 0.00R (statistically zero) |
| dominant cancel reason | — | target reached without a fill (≈ ⅔ of cancels) |

Diagnosis: the evaluation assumption specifically rewarded strategies subject
to maximal adverse selection — deep retracement entries that look brilliant
when fills are free and rarely fill in reality. Response: re-baseline
everything on the honest model; invalidate the ML layer's prior passing grade
pending recalibration on honest labels; redirect the roadmap from signal
accumulation to entry mechanics. The uncomfortable number was recorded and
published. **Assumptions are attack surface; test the ruler before trusting
what it measures.**

## 6. Companion incident — the silent daily-bars bug

The same investigation exposed a latent data defect: the broker adapter's
bar-size map was keyed with strings (`"5m"`, `"1H"`) that no caller ever
passed, so every intraday request silently fell through to daily aggregates —
the live monitor had been checking five-minute stop logic against daily OHLC
ranges without one raised error. Fix: canonical keys plus a full-coverage
test binding the timeframe enum to the map, converting any future drift into
a loud failure. Both incidents share a moral: *silent degradation is the
enemy; every fallback must either be honest or impossible.*

## 7. Candidate protocol (grade-before-hardcode)

```
research → sourced dossier → candidate implementation (default-OFF)
  → A/B on the fixed ruler (OFF vs ON, multi-symbol, day + week horizons,
    Prime-only cut, net of costs)
  → verdict table: hit rate AND R/attempt AND move-capture, both horizons
  → GO only if the lift is direction-consistent and survives the Prime cut
  → MIXED / NO-GO: recorded, retained, left OFF
```

A documented rejection is a successful result. The system's own liquidity
gate was graded NO-GO twice and remains off — the process working exactly as
designed.

## 8. The ML layer under the same law

The meta-labeler (deterministic logistic regression over as-of setup
features) is judged on calibration, not accuracy theater: ECE, Brier,
CI-disjoint tercile separation, PR-AUC on the loser class (the veto's actual
job), and a learning curve so "insufficient data" is a measured statement.
Its position-sizing authority — a conviction ladder over calibrated
confidence buckets — activates per bucket only at n ≥ 30 honest outcomes,
keys to the Wilson lower bound of realized (not claimed) reliability, caps at
a fraction of Kelly, and auto-reverts fleet-wide when live reliability decays
below band. Trust is rented, never owned.
