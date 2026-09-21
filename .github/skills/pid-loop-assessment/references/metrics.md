# Metric glossary

All metrics are computed on the **forward-filled, resampled** signal (a fixed 10 s
grid inside each active span), never on raw COV rows.

## Standard metrics (per rolling window)
| Metric | Definition | What it tells you |
|---|---|---|
| `pv_range` | max − min of the PV in the window | raw amplitude of the PV swing |
| `pv_tv` (total variation) | Σ |Δpv| past a deadband | "odometer" travel / restlessness |
| `pv_reversals` | count of sign changes of Δpv past a deadband | how often the PV turns around → **frequency** |
| `pv_period_min` | 2 × window ÷ mean-crossings (only if ≥ 3 crossings) | implied oscillation period (min/cycle) |
| `pv_sawtooth` | \|skew\| of the PV increments | slow-ramp-then-fast-drop asymmetry |
| `out_range` / `out_tv` / `out_reversals` | same three, for the command channel | how hard/often the output thrashes |
| `out_near_rail_frac` | fraction of the window with `out ≥ 95%` | dwell against the 100 % saturation rail |

## Invented composites
| Metric | Formula | Purpose |
|---|---|---|
| **Hunting Index** | `pv_reversals × pv_range` | single number for limit-cycling (cycles × amplitude) |
| **Aggression Index** | `out_reversals × out_range` | output thrash — high = over-driving proportional gain |
| **Windup Index** | `near_rail_frac × pv_range`, ×1.0 if reversals < 3 else ×0.35 | ranks suspected integral-windup windows (rail dwell × amplitude, up-weighted when slow) |
| **Span lag** | argmax over lag of corr(Δoutput, later Δpv) | effective loop delay (minutes) the loop must be detuned against |

## Reading the fingerprint
Plot **amplitude (`pv_range`) vs reversal-rate (`pv_reversals`)**, colored by verdict:
- **low reversals + high amplitude** → integral windup / disturbance corner.
- **high reversals** → proportional gain too high (fast oscillation).
- A **period near 2–6× the span lag** is consistent with limit-cycling driven by
  excess gain against that lag.

## Thresholds
Thresholds are **data-driven**, derived from the dataset itself (medians / quantiles of
`pv_range`, `pv_period_min`, `pv_sawtooth`) so the classifier adapts to each loop rather
than using fixed magic numbers.
