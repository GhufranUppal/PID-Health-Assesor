# Data model & fieldbus source

## Where the data comes from
The trend exports originate from a **building-automation fieldbus** historian. In most
sites this is **BACnet** (BACnet/IP or BACnet MS/TP); the same principles apply to
**Modbus** or **LonWorks** historians. A supervisory controller or BMS front-end
subscribes to points and logs them to a trend log, which is exported to CSV.

## Change-of-value (COV) logging — the core fact
Points are logged **on change of value**: a new row is written only when the value
changes by more than the COV increment, and that value is **held until the next row**.

Consequences the analysis must respect:
- **Forward-fill is mandatory.** To reconstruct the real signal at any instant, hold the
  last logged value forward (`value_at` / `held_values`). Never treat raw rows as evenly
  spaced samples.
- **Sparsity.** The command channel is sparse (median ≈ 1 sample / 2 min). For any
  window ≤ 5 min, **resample both channels to a fixed grid** (e.g. 10 s) before scoring,
  or reversal counts collapse to near zero.
- **Irregular timestamps.** Rows are microsecond-stamped and unevenly spaced; align by
  time, not by row index.

## Point mapping (confirm before attaching units)
| File | BACnet object (typical) | Meaning |
|---|---|---|
| `PID/Input.csv` | Analog Input / Analog Value | **Process variable** — the controlled measurement (flow, pressure, temperature, level, …) |
| `PID/PID out.csv` | Analog Output | **Command** — final-element position, 0–100 % |

There is **no setpoint (Analog Value setpoint) channel** in the export, which is why the
loop is graded from waveform *shape* rather than measured error.

## Saturated spans are excluded from tuning analysis
When the command sits pinned at a limit (e.g. 100 %) it is **not** the element in
control — a downstream stage or a separate loop may be trimming, or the equipment has
simply run out of capacity. Those spans carry little tuning information, so they are
flagged separately and excluded from the rolling-window tuning analysis; the assessment
concentrates on the **active** spans where the command is actually modulating.

## Exporting equivalent data from other fieldbuses
To reproduce this analysis on another loop, export two COV/trend logs — the PV and the
0–100 % command — as CSV with a timestamp column and a value column each, then drop them
in as `PID/Input.csv` and `PID/PID out.csv`. Capturing the **setpoint** too (if the
fieldbus exposes it) turns every shape-based screening flag into a provable diagnosis.
