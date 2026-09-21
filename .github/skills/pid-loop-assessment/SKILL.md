---
name: pid-loop-assessment
description: 'Diagnose HVAC / building-automation PID loops from change-of-value (COV) fieldbus historian CSV exports and produce color-coded HTML reports plus a numbered Jupyter notebook. Use when the user wants to analyze economizer/damper/valve loop data (PID/Input.csv = process variable, PID/PID out.csv = 0-100% command), detect stuck-at-100% saturation, hunting/oscillation, output-channel thrash, integral windup, P-too-high or sawtooth tuning faults, run rolling-window loop-health analysis, invent control-theory metrics, or assess how a loop is performing without a setpoint channel. Triggers: "PID hunt", "loop tuning", "economizer", "supply air", "rolling window", "integral windup", "P too high", "two-minute window", "assess the loop".'
argument-hint: 'Describe the window length, active-span filter, and metrics/fault signatures to assess'
---

# PID / Economizer Loop Assessment

Diagnose a building-automation control loop from sparse COV fieldbus exports and
deliver a reproducible analysis package: a Python module, a numbered notebook, HTML
reports, worked-example figures, and supporting CSVs — all color-coded and
self-contained.

## When to Use
- Analyzing `PID/Input.csv` (process variable, e.g. supply-air temperature / PID input)
  and `PID/PID out.csv` (0–100% economizer/damper/valve command).
- Detecting saturation (stuck at exactly 100%), hunting/oscillation, output-channel
  thrash, integral windup, P-too-high, or sawtooth signatures.
- Producing a plain-English loop verdict when there is **no setpoint channel** — the
  loop must be diagnosed from waveform *shape* and PV↔output coupling.

## Reference files (load as needed)
- [tools.md](./references/tools.md) — the tools the agent uses and when.
- [metrics.md](./references/metrics.md) — the full metric glossary.
- [fault-signatures.md](./references/fault-signatures.md) — the fault catalog + fixes.
- [data-model-and-fieldbus.md](./references/data-model-and-fieldbus.md) — COV /
  fieldbus data model and point mapping.

## Ground Truth (read first)
- [README-LOOP-TUNING.md](../../../README-LOOP-TUNING.md) — the field guide. §4/§5/§12
  describe windup, P/I faults, anti-windup, and shape-based diagnosis. Cite sections.
- [analyze_pid_two_minute.py](../../../analyze_pid_two_minute.py) — the **foundation
  module**. Always reuse its helpers instead of re-implementing them:
  - `load_cov`, `value_at`, `value_before`, `held_values`, `percent_at_value`
  - `hunting_metrics`, `window_verdict`, `continuous_stuck_runs`, `hunt_table_html`
  - constants: `ROOT`, `PID_DIR`, `INPUT_CSV`, `OUTPUT_CSV`, `WINDOW`,
    `SATURATION_VALUE=100.0`, `DEADBAND`, `TV_MIN`, `CYCLES_MIN`, `REVERSALS_MIN`,
    `SWING_CALLOUT_F`
- [analyze_pid_5min_loop.py](../../../analyze_pid_5min_loop.py) — the reference
  implementation of the rolling-window + invented-metrics + classifier pattern.
- [pid_worked_examples.py](../../../pid_worked_examples.py) — annotated example figures
  with the smoothed "windup swing" overlay.

## Tools the agent uses
| Tool | Role in this workflow |
|---|---|
| read / search | Read the field guide + foundation module; find helpers, episodes, prior notebooks. |
| edit | Create the analysis module, numbered notebook, and docs. |
| execute | Run the module; export notebooks with `nbconvert`. |
| todo | Track multi-step deliverables. |
| notebook tools | Notebook summary, insert/edit/run cells. |
| browser tools | Optionally open the HTML report to verify rendering. |

## Data Model — COV forward-fill (critical)
The exports come from a **fieldbus historian** (BACnet/Modbus) logging **on change of
value**: every logged value is **held until the next timestamp**. To reconstruct the
true signal you MUST forward-fill (`value_at` / `held_values`). The output channel is
sparse (median ≈ 1 sample / 2 min), so for any window ≤ 5 min **resample both channels
onto a fixed grid** (e.g. 10 s via `pd.date_range(..., freq="10s")` feeding `value_at`)
before computing metrics — otherwise reversal counts collapse. See
[data-model-and-fieldbus.md](./references/data-model-and-fieldbus.md).

## Environment (portable)
- Use any **Python 3.11+** with the packages in `requirements.txt` (pandas, numpy,
  matplotlib, jupyter, nbconvert, **jinja2**). The interpreter need not match the
  author's. `nbconvert` needs jinja2 to export notebooks.
- If your **notebook kernel** lacks jinja2, the pandas `.style` accessor /
  `background_gradient` FAIL inside notebooks. Use plain inline-HTML renderers that
  compute the gradient manually and return an HTML string (this repo already does).
- **pandas 3**: subtracting a nanosecond from microsecond timestamps raises "Cannot
  losslessly convert units" → use `value_before` with `searchsorted(side="left")`.
  Only `.round(3)` numeric columns, never datetime.
- Guard empty results before `sort_values` (build an explicit empty DataFrame).
- An oscillation period is only real with **≥ 3 mean-crossings** (≥ 2 gives a
  period=window-length artifact).

## Metric Toolbox (summary — full glossary in references/metrics.md)
Standard: `pv_range`, total variation, reversal count (sign changes past a deadband),
mean-crossing period (min/cycle), near-rail dwell (`out ≥ 95%`). Invented composites:
- **Hunting Index** = pv_reversals × pv_range
- **Aggression Index** = out_reversals × out_range
- **Windup Index** = near_rail_frac × pv_range, up-weighted when reversals are low
- **Span lag** = output→PV cross-correlation lag (minutes)

## Fault Classifier (data-driven thresholds — catalog in references/fault-signatures.md)
Derive thresholds from the data (medians / quantiles), then map each window to one of:
`Integral windup` (near-rail dwell + one-way PV, few reversals) ·
`P too high (fast oscillation)` · `P & I too high (sawtooth)` ·
`Too much integral (slow rolling)` · `Disturbance / step (not tuning)` · `Calm`.
Collapse consecutive same-verdict windows into **episodes**.

## Deliverable Pattern (follow every time)
1. **Python module** `analyze_pid_<topic>.py` — reuses `analyze_pid_two_minute`
   helpers, writes `PID/<topic>.html` (a standalone report with the Python source
   **embedded**) plus supporting `PID/<topic>_*.csv`.
2. **Numbered notebook** `0N_<topic>.ipynb` — `import analyze_pid_<topic> as xx`,
   render prose + color-coded tables + a base64 matplotlib chart + insights.
   Notebooks must NOT use `.style` (no jinja2); use inline-HTML renderers.
3. **Export** the notebook to a self-contained HTML alongside it.
4. Optionally add annotated **worked-example figures** (see `pid_worked_examples.py`).

## Procedure
1. Read the guide + foundation module; confirm the point mapping (Input = PV).
2. Load both CSVs with `load_cov`; compute the overlapping time range.
3. Restrict to active spans (output ≠ 100%) meeting the minimum window length.
4. Resample to the fixed grid; slide the rolling window; score metrics.
5. Classify → episodes → roll up a plain-English assessment (cite guide sections).
6. Build the report HTML (cards, prose, verdict-mix table, chart, episodes, worked
   examples, glossary, embedded source) and the notebook; run and verify both.
7. Always state the caveat: **no setpoint channel + COV sparsity → these are
   shape-based screening flags, not proof.**

## Run & Export Commands
Run from the repo root so local `import` resolves.
```
python run_analysis.py                 # report + CSVs + figures in one shot
python analyze_pid_<topic>.py          # a single analysis
python -m nbconvert --to html --execute --output 0N_<topic>.html 0N_<topic>.ipynb \
    --ExecutePreprocessor.timeout=300 --ExecutePreprocessor.kernel_name=python3
```

## Verify
- Report HTML exists and contains the expected verdict strings, an embedded
  `data:image/png;base64` chart, and the `.highlight` CSS class (proves source shown).
- Notebook exported HTML is non-trivial in size and every cell executed.
- No pandas unit-conversion errors or datetime-round warnings in the run log.

## Output to the User
Concise summary with markdown file links (NOT in backticks) to every deliverable, the
dominant fault + its percentage, key medians (period, lag), and the no-setpoint / COV
caveat.
