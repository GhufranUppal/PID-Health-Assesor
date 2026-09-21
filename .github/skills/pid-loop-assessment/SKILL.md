---
name: pid-loop-assessment
description: 'Diagnose any PID control loop (flow, pressure, temperature, level; valve, damper, pump/VFD) from change-of-value (COV) fieldbus historian CSV exports and produce color-coded HTML reports plus a numbered Jupyter notebook. Combines control-theory foundations (P/I/D behaviour, why loops oscillate, tuning methods, anti-windup) with an automated rolling-window loop-health assessment. Use to analyze a process-variable trace (PID/Input.csv) against its 0-100% command (PID/PID out.csv), detect saturation, hunting/oscillation, output-channel thrash, integral windup, P-too-high or sawtooth tuning faults, invent control-theory metrics, or assess a loop with no setpoint channel. Triggers: "PID hunt", "loop tuning", "hunting detection", "rolling window", "integral windup", "P too high", "assess the loop".'
argument-hint: 'Describe the window length, active-span filter, and metrics/fault signatures to assess'
---

# PID Loop Health Assessment

Diagnose **any** feedback control loop from sparse change-of-value (COV) fieldbus
exports and deliver a reproducible analysis package — a Python module, a numbered
notebook, color-coded HTML reports, and supporting CSVs. The method is loop-agnostic:
it grades a loop from the **shape** of its process variable (PV) and command traces,
so it works on flow, pressure, temperature, level, valve, damper, or pump/VFD loops,
with or without a recorded setpoint. This skill fuses two bodies of knowledge — the
**control-theory field guide** (what P/I/D do, why loops oscillate, how to tune) and
the **automated assessment procedure** (COV data model, rolling-window metrics, fault
classifier, deliverables).

## Author
Ghufran Uppal

## When to use
- You have a PV trace (`PID/Input.csv`) and its 0–100% command trace
  (`PID/PID out.csv`) and want a health verdict on the loop.
- You need to detect saturation (command pinned at a limit), hunting/oscillation,
  output-channel thrash, integral windup, P-too-high, or sawtooth tuning faults.
- You must diagnose a loop with **no setpoint channel** — grading from waveform
  *shape* and PV↔command coupling alone.
- You want a plain-English, color-coded, reproducible report plus a notebook a
  colleague can re-run.

## Reference files (load as needed)
- [tools.md](./references/tools.md) — the tools the agent uses and when.
- [metrics.md](./references/metrics.md) — the full metric glossary.
- [fault-signatures.md](./references/fault-signatures.md) — the fault catalog + fixes.
- [data-model-and-fieldbus.md](./references/data-model-and-fieldbus.md) — COV /
  fieldbus data model and point mapping.

## Operating principles (how a capable agent works here)
- **Ground every claim in the data.** Never assert a fault you did not measure; run
  the numbers and cite them.
- **Read the command and the PV together.** The command channel usually settles an
  ambiguous PV shape (e.g. windup vs disturbance).
- **Reuse, don't reinvent.** Import shared helpers; don't re-implement loaders/metrics.
- **Change one thing at a time** when recommending tuning fixes, and re-test.
- **Be honest about limits.** Without a setpoint and with sparse COV data, verdicts
  are shape-based *screening flags*, not proof — always say so.

---

# Part I — Control-theory foundations

## 1. The feedback loop and the PID law
A feedback loop compares a **setpoint (SP)** — the value you want — against the
**process variable (PV)** — the value you measure — and drives a **command (CV/MV)** —
a valve, damper, or pump/VFD — to close the gap:

$$e(t) = \mathrm{SP} - \mathrm{PV}(t)$$

A PID controller forms its command from that error using three terms:

$$u(t) = K_p\,e(t) + K_i\!\int_0^t e(\tau)\,d\tau + K_d\,\frac{d\,e(t)}{dt}$$

**Tuning** is choosing $K_p$, $K_i$, $K_d$ so the loop is **stable** (does not
oscillate without bound — non-negotiable), **responsive** (corrects disturbances and
reaches new setpoints quickly), and **smooth** (does not hunt, overshoot excessively,
or hammer the actuator).

## 2. What P, I, D each do
| Term | Responds to | Effect when increased | Failure mode |
| --- | --- | --- | --- |
| **P** — Proportional | The **present** error | Faster response, smaller offset | Too high → overshoot, then sustained/growing oscillation |
| **I** — Integral (reset) | The **accumulated past** error | Removes steady-state offset; drives PV *exactly* to SP | Too much → slow rolling overshoot; **integral windup** |
| **D** — Derivative (rate) | The **predicted future** error (rate of change) | Damps overshoot, improves settling | Amplifies **noise**; often left off in noisy loops |

Intuition: **P is a spring** (harder push the further from SP, but a pure-P loop
settles with an offset); **I is patience** (keeps adding correction until any error is
gone, but its momentum can carry PV past SP); **D is anticipation** (brakes early on
fast error changes, but magnifies measurement noise). Many robust loops run **PI only**
with a modest P and gentle I.

## 3. Gain vs proportional band vs reset vs rate
The same physics is written two ways depending on the vendor — know both:

| Parallel (gain) form | Standard (time) form | Relationship |
| --- | --- | --- |
| $K_p$ — proportional gain | **Proportional Band** (PB), % of range | $\text{PB} = 100 / K_p$ |
| $K_i$ — integral gain | **Reset** / integral time $T_i$ | $K_i = K_p / T_i$ |
| $K_d$ — derivative gain | **Rate** / derivative time $T_d$ | $K_d = K_p \, T_d$ |

**Proportional band** is the span of PV (as % of range) over which the command travels
0→100%. Narrow band = high gain → aggressive, prone to oscillation. Wide band = low
gain → sluggish; the actuator "isn't trying hard" and may not reach full stroke until
the PV is far off target. A common conservative starting point is a moderate P with
light reset and light rate — deliberately quiet so it can burn in on live equipment,
then be tightened from trend evidence.

## 4. Why loops oscillate — excess gain + lag (the key mechanism)
Three regimes:

```
Stable (decaying)        Marginally stable         Unstable (growing)
   /\                        /\    /\    /\             /\      /\
  /  \    /\  __            /  \  /  \  /  \           /  \    /  \    /
 /    \  /  \/             /    \/    \/    \         /    \  /    \  /
------\/-------- SP    ---------------------- SP  ---------\/------\/-- SP
 amplitude shrinks         amplitude constant         amplitude grows
 → GOOD                    → borderline / hunting     → DANGEROUS
```

A loop reacts to **error now** and moves the command **now**, but the **effect** of
that move appears in the PV only after a **lag** (thermal mass, transport time,
actuator stroke, sensor response). So the controller always acts on stale
information. **Gain** is how hard it shoves for a given error. Turn gain up too far and
the lag turns that aggression into a cycle:

1. PV below SP → error positive → command opens the actuator.
2. High gain opens it *a lot* — a big shove.
3. Lag means the effect keeps arriving even after PV reaches SP — the correction is
   "still in the pipe."
4. PV overshoots → error flips → the loop slams the command the other way, again too
   hard.
5. That correction also lands late → PV overshoots the other way. Repeat →
   **sustained oscillation (hunting)**.

**Analogy — steering a car with a delay:** gentle inputs drift smoothly into the lane
(stable); yanking the wheel makes you cross the centre line before the car responds, so
you yank back and swerve (oscillation).

**Same disease, two flavours:**
- **Too much P** → each shove is too big → *fast, symmetric* oscillation (short period).
- **Too much I** → the integral keeps accumulating during the lag, then dumps all the
  stored correction at once → *slow, rolling* overshoot that winds up the other way.

**Fixes follow the cause:** lower gain ($K_p$) / lengthen reset ($T_i$) so corrections
are gentle enough to land before the error reverses; add derivative ($K_d$) to ease off
early; if the lag is genuine **dead time**, detune more (no PID cleverness beats waiting
for delayed feedback).

## 5. Reading the trend — waveform → what to change
The shape of the wave tells you which term is wrong:

| Waveform on the trend | What it means | What to change |
| --- | --- | --- |
| **Sawtooth / ramp that keeps accelerating** | **P and I both too high**; the command is over-driving | **Lower P first**, then lengthen reset (lower I) |
| **Fast, symmetric oscillation** (short period) | **Proportional gain too high** (band too narrow) | Reduce $K_p$; optionally add a little $K_d$ |
| **Slow, rolling overshoot** that settles | **Too much integral** (reset too fast) | Lengthen integral time $T_i$ (reduce $K_i$) |
| **Sluggish; large lasting offset** | **Gain too low / band too wide**, or not enough reset | Raise $K_p$; add/increase $I$ to kill the offset |
| **Spiky, jittery command tracking noise** | **Derivative amplifying measurement noise** | Reduce/disable $K_d$; add a derivative low-pass filter |
| **Constant-amplitude limit cycle** even detuned | Likely **valve stiction / backlash**, not tuning | Fix the actuator; consider an output deadband |

Target response for most loops: a **quarter-amplitude decay** (each overshoot ≈ ¼ of
the previous) — fast enough to reject disturbances, damped enough not to hunt.

## 6. Classic tuning methods
**Manual (closed-loop):** (1) set $K_i=K_d=0$ and find P-only behaviour — it settles
with an offset; (2) raise $K_p$ until the PV *just* sustains a constant-amplitude
oscillation — the **ultimate gain $K_u$** with period $T_u$; (3) back $K_p$ off to
about half for a quarter-amplitude decay; (4) add $K_i$ (shorten reset) until the
offset is gone but *before* it rolls; (5) add a little $K_d$ only if needed and the PV
is not noisy.

**Ziegler–Nichols (ultimate-gain)** from $K_u$, $T_u$:

| Controller | $K_p$ | $T_i$ | $T_d$ |
| --- | --- | --- | --- |
| P | $0.5\,K_u$ | — | — |
| PI | $0.45\,K_u$ | $T_u/1.2$ | — |
| PID | $0.6\,K_u$ | $T_u/2$ | $T_u/8$ |

Z–N is a fast start but aggressive — detune for smoother control. **Relay
(Åström–Hägglund) auto-tuning** oscillates the loop with a bang-bang output and derives
$K_u = 4b/(\pi a)$ (output step $b$, amplitude $a$) — safer than pushing a real loop to
instability. **Model-based (IMC / lambda)** fits a first-order-plus-dead-time model
from an open-loop step test and computes gains — best for slow loops.

## 7. Anti-windup, bumpless transfer, derivative filtering
- **Integral anti-windup.** If the command saturates, the integral keeps accumulating
  error it can't act on, then overshoots when it unwinds. Clamp or back-calculate the
  integral so it never winds up beyond the output limits.
- **Bumpless transfer.** When the loop is switched into control (or gains change), seed
  the integral to the current command so the actuator doesn't jump.
- **Derivative filtering / derivative-on-PV.** Differentiate the *PV* (not the error) to
  avoid a spike on setpoint changes, and low-pass the derivative ($3 \le N \le 10$) so
  it doesn't amplify noise.

## 8. Not all oscillation is a tuning fault
Before re-tuning, rule out external causes: **upstream/interacting loops**,
**disturbances & staging** (a pump starting, a load step, a mode changeover), **sensor
problems** (noise, lag, placement), **mechanical faults** (stiction, backlash,
over/undersized final elements), and **process nonlinearity** (fine at full load,
unstable at low load → gain scheduling). Overlay the loop's PV/command with neighbouring
points that could "pop it out" of its stable region before calling it a tuning fix.

## 9. COV resolution bounds what you can detect
You can only tune out a wave you can *see*. Coarse COV increments or slow polling hide
small, fast hunts. **While tuning, raise resolution**: tighten the COV increment on PV
and command, raise the trend rate, and capture PV *and* command together so you see
cause and effect. An automated detector is only as good as its input granularity —
the COV increment you choose directly sets the smallest hunt you can catch.

---

# Part II — Data model (COV fieldbus)
Exports come from a **fieldbus historian** (BACnet / Modbus / LonWorks) logging **on
change of value**: a row is written only when the value moves past the COV increment,
and that value is **held until the next row**. The analysis must respect this:

- **Forward-fill is mandatory.** Reconstruct the signal at any instant by holding the
  last logged value forward (`value_at` / `held_values`). Never treat raw rows as
  evenly spaced samples.
- **Sparsity → resample.** The command channel is sparse (often ≈ 1 sample / 2 min);
  for any window ≤ 5 min, **resample both channels to a fixed grid** (e.g. 10 s) before
  scoring, or reversal counts collapse to near zero.
- **Irregular timestamps.** Rows are microsecond-stamped and unevenly spaced; align by
  time, not row index.
- **No setpoint channel.** The export is PV + command only, which is why the loop is
  graded from waveform *shape* and PV↔command coupling rather than measured error.

Point mapping (confirm before attaching units): `PID/Input.csv` = **process variable**;
`PID/PID out.csv` = **0–100% command**. See
[data-model-and-fieldbus.md](./references/data-model-and-fieldbus.md).

---

# Part III — Automated assessment (rolling-window)

## Metric toolbox (full glossary in references/metrics.md)
All metrics are computed on the **forward-filled, resampled** signal, never on raw COV
rows. Standard: `pv_range`, total variation, reversal count (sign changes past a
deadband), mean-crossing period (min/cycle, only real with **≥ 3 mean-crossings**),
near-rail dwell (fraction with the command near its limit). Invented composites:
- **Hunting Index** = `pv_reversals × pv_range` — one number for limit-cycling.
- **Aggression Index** = `out_reversals × out_range` — output thrash / over-driven P.
- **Windup Index** = `near_rail_frac × pv_range`, up-weighted when reversals are low.
- **Span lag** = output→PV cross-correlation lag (minutes) the loop must be detuned
  against.

## Hunting detector (simple, explainable counting rule)
You do not need ML to catch most hunting. Relative to a baseline (the setpoint or a
slow moving average of the PV): an **overshoot** is a peak with `PV − baseline > x`; an
**undershoot** a trough with `baseline − PV > x`. If **N consecutive *alternating***
excursions each exceed `x`, flag **potential hunting**.

```text
streak := 0; last_side := none
for each local extremum (peak/trough):
    dev := extremum - baseline
    if   dev >  x: side := 'over'
    elif dev < -x: side := 'under'
    else: streak := 0; continue          # inside band → not hunting
    if side != last_side: streak += 1     # require alternation
    else:                 streak := 1
    last_side := side
    if streak >= N: FLAG('potential hunting', amplitude=dev, time=extremum.time)
```

Make `x` **relative** so one detector serves many loops: scale it to the COV increment,
`x = k × COV` (e.g. `k = 3–4`), or to a multiple of the PV's standard deviation. Start
around `N = 2–3`. The **alternation requirement** is what separates real hunting from a
single healthy overshoot after a setpoint change.

## Fault classifier (data-driven thresholds — catalog in references/fault-signatures.md)
Derive thresholds from the data itself (medians / quantiles), then map each rolling
window to exactly one signature. The key discriminators are **near-rail dwell**, **PV
amplitude**, and — crucially — **PV reversal rate (frequency)**:

`Integral windup` (near-rail dwell + one-way PV, few reversals) ·
`P too high (fast oscillation)` (big amplitude, many reversals, short period) ·
`P & I too high (sawtooth)` (high skew, high total variation) ·
`Too much integral (slow rolling)` (big amplitude, long period) ·
`Disturbance / step (not tuning)` (big amplitude, few reversals, command **not**
pinned) · `Calm`. Collapse consecutive same-verdict windows into **episodes**.

> **Windup vs P-too-high** is decided by **frequency**, not the command (both may pin
> the command): windup is a *slow big swing* (few reversals), P-too-high is *fast tight
> chatter* (many reversals). **Disturbance** stands apart because the command is
> actually modulating — the loop has authority and is chasing a load change.

---

# Part IV — Deliverables, environment & verification

## Environment (portable)
- Any **Python 3.11+** with pandas, numpy, matplotlib, jupyter, nbconvert, **jinja2**
  (list them in a `requirements.txt`). The interpreter need not match the author's;
  do **not** hardcode an interpreter path. `nbconvert` needs jinja2 to export.
- If the **notebook kernel** lacks jinja2, the pandas `.style` accessor /
  `background_gradient` fail — use plain inline-HTML renderers that compute the gradient
  manually and return an HTML string.
- **pandas 3**: subtracting a nanosecond from microsecond timestamps raises "Cannot
  losslessly convert units" → use a `value_before` helper with
  `searchsorted(side="left")`. Only `.round(3)` numeric columns, never datetime.
- Guard empty results before `sort_values` (build an explicit empty DataFrame).

## Reusable foundation module
Keep shared helpers in one importable module (e.g. `pid_core.py`) and **reuse them**
instead of re-implementing loaders/metrics per analysis: `load_cov`, `value_at`,
`value_before`, `held_values`, `percent_at_value`, `hunting_metrics`, `window_verdict`,
`continuous_stuck_runs`, and constants (`WINDOW`, `SATURATION_VALUE`, `DEADBAND`, the
minimum total-variation / cycles / reversals thresholds).

## Deliverable pattern (follow every time)
1. **Python module** `analyze_pid_<topic>.py` — reuses the foundation helpers, writes a
   standalone `<topic>.html` report with the Python source **embedded**, plus supporting
   `<topic>_*.csv`.
2. **Numbered notebook** `0N_<topic>.ipynb` — `import analyze_pid_<topic> as xx`, render
   prose + color-coded tables + a base64 matplotlib chart + insights. Notebooks must NOT
   use `.style` (no jinja2); use inline-HTML renderers.
3. **Export** the notebook to a self-contained HTML alongside it.
4. Optionally add annotated **worked-example figures**.
5. **Never overwrite** prior numbered notebooks/reports — increment the number.

## Procedure (end to end)
1. Confirm the point mapping (Input = PV, PID out = command).
2. Load both CSVs with `load_cov`; compute the overlapping time range.
3. Restrict to **active spans** (command not pinned at a limit) meeting the minimum
   window length — pinned spans carry little tuning information; flag them separately.
4. Resample to the fixed grid; slide the rolling window; score metrics.
5. Classify → episodes → roll up a plain-English assessment (cite the foundations above).
6. Build the report HTML (cards, prose, verdict-mix table, chart, episodes, glossary,
   embedded source) and the notebook; run and verify both.
7. Always state the caveat: **no setpoint channel + COV sparsity → shape-based
   screening flags, not proof.**

## Run & export commands
Run from the repo root so local `import` resolves.
```
python analyze_pid_<topic>.py          # a single analysis (report + CSVs + figures)
python -m nbconvert --to html --execute --output 0N_<topic>.html 0N_<topic>.ipynb \
    --ExecutePreprocessor.timeout=300 --ExecutePreprocessor.kernel_name=python3
```

## Verify
- Report HTML exists and contains the expected verdict strings, an embedded
  `data:image/png;base64` chart, and the `.highlight` CSS class (proves source shown).
- Notebook exported HTML is non-trivial in size and every cell executed.
- No pandas unit-conversion errors or datetime-round warnings in the run log.

## Constraints (DO NOT)
- **Do not assume a setpoint exists** — diagnose from waveform shape and PV↔command
  coupling, and always state that caveat.
- **Do not compute metrics on raw COV samples** for windows ≤ 5 min — forward-fill and
  resample first, or reversal counts collapse.
- **Do not re-implement** helpers that already exist in the foundation module — import
  and reuse them.
- **Do not use** the pandas `.style` accessor in notebooks if the kernel lacks jinja2.
- **Do not hardcode** an interpreter path, and **do not overwrite** prior numbered
  notebooks/reports.

## Output to the user
Concise summary with markdown file links (NOT in backticks) to every deliverable, the
dominant fault + its percentage, key medians (oscillation period, loop lag), and the
explicit no-setpoint / COV-sparsity caveat. Keep prose tight; put the depth in the
report and notebook.
