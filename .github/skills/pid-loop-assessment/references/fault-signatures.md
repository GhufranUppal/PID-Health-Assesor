# Fault-signature catalog

Each rolling window is mapped to exactly one signature. The discriminators are the
**command's dwell near the 100 % rail**, the **PV amplitude**, and — crucially — the
**PV reversal rate** (frequency). Consecutive same-signature windows are collapsed into
**episodes**.

| Signature | Fingerprint (data rule) | Physical meaning | Fix (see SKILL Part I) |
|---|---|---|---|
| **Integral windup** | near-rail dwell ≥ 0.5, big amplitude, **reversals < 3** | Output saturated; integral keeps accumulating, overshoots and unwinds late → one big slow swing | Enable anti-windup / integral clamping; verify output limits |
| **P too high (fast oscillation)** | big amplitude, **reversals ≥ 4**, short period | Proportional gain too high → fast, tight, symmetric ripple | Lower `Kp`; add small `Kd` |
| **P & I too high (sawtooth)** | high sawtooth skew, high total variation, reversals ≥ 2 | Ramp-then-drop that keeps accelerating | Lower P first, then lengthen reset |
| **Too much integral (slow rolling)** | big amplitude, reversals ≥ 2, **long** period | Reset too fast → slow rolling overshoot | Lengthen integral time |
| **Disturbance / step (not tuning)** | big amplitude, **reversals < 2**, command often **not** pinned | Load/mode change the loop is responding to | Not a tuning fault; correlate with load and mode changes |
| **Calm** | none of the above | Loop is behaving | — |

## The key discriminator
- **Windup vs P-too-high** is decided by **frequency**, not the command: both may show a
  pinned command, but windup is a **slow big swing** (few reversals) while P-too-high is
  **fast tight chatter** (many reversals).
- **Disturbance** stands apart because the **command is actually modulating** (low
  near-rail dwell) — the loop has authority and is chasing a load change.

## Integral windup — the signature in detail
As in the anti-windup discussion (SKILL Part I §7): *"If the command saturates, the
integral keeps accumulating error it can't act on, then overshoots badly when it finally
unwinds."* In the data this reads as: **command pinned near its limit** while the PV
makes a **large, low-frequency** excursion and reverses **late** (few reversals).

## Always attach the caveat
No setpoint channel + COV sparsity → these are **shape-based screening flags**, not
proof. Confirm with a high-resolution PV + setpoint + output capture during a live
tuning session.
