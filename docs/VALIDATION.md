# Engineering Validation

**Project 0 — Tennessee Eastman Process: anomaly detection, fault diagnosis, physics-informed residuals**

This document records what the system does, what it fails at, and **why** — in process
engineering terms rather than metric terms. It is the primary artefact of this project.

---

## 1. Headline results

| Metric | Part 1 (raw) | Part 2 (enriched) |
|---|:---:|:---:|
| Detector features | 52 | 112 |
| **False alarm rate** | 2.11% | **1.42%** |
| Classifier accuracy | 83.5% | **90.9%** |
| Macro F1 | 84.2% | **91.2%** |
| Training time | 330 s | 773 s |

Split: **Training files vs Testing files — separate simulation runs, no leakage.**
384,000 training rows / 640,000 test rows (40 of 500 simulation runs per fault).

> Accuracies above 95% are commonly reported on this benchmark. They generally come from
> random row-level splitting, which places temporally adjacent samples from the *same*
> simulation run in both train and test. The numbers here use run-level separation.

**Part 2 improved detection and reduced false alarms simultaneously** — an uncommon result,
explained in §4.

---

## 2. The hypothesis, and its outcome

### The hypothesis

Part 1 found faults 3, 9 and 15 undetectable — detection rate equal to the false alarm rate,
meaning zero signal. Inspection of the measurement set showed why:

| Fault | Disturbed variable | Measured? |
|:---:|---|:---:|
| 3 | D feed temperature | ❌ not among the 22 measurements |
| 9 | D feed temperature | ❌ |
| 15 | *Actual position* of condenser CW valve | ❌ — only the command `xmv_11` is measured |

An **observability** problem, not a sensitivity problem.

The hypothesis: physics-informed residuals reconstruct unmeasured quantities from measured
ones, and might therefore surface these faults.

### ❌ The hypothesis was rejected

| Fault | A (raw) | B (residuals) | C (raw+resid) | D (all+temporal) |
|:---:|:---:|:---:|:---:|:---:|
| **FAR** | 0.021 | 0.022 | 0.022 | **0.014** |
| 3 | 0.021 | 0.021 | 0.023 | 0.015 |
| 9 | 0.022 | 0.022 | 0.023 | 0.015 |
| 15 | 0.024 | 0.031 | 0.026 | 0.021 |

All three remain **at the false alarm rate in every feature set**. Fault 15 reaches 1.5× FAR
in the best case — not a detection.

**Conclusion:** this is a genuine physical detection limit, not a modelling gap. The
disturbance, after control compensation, is smaller than the noise floor of the energy and
duty balances themselves. Adding features does not create information the sensors never
captured.

> **A negative result, stated with confidence:** faults 3, 9 and 15 are undetectable from this
> measurement set by correlation-based *or* balance-based monitoring. Detecting them requires
> **new instrumentation** — a D feed temperature transmitter, and valve position feedback on the
> condenser cooling water valve — not a better algorithm.

That recommendation is actionable, and it is the kind of answer a plant engineer can use.

---

## 3. ⭐ The unexpected finding — detectability and discriminability are different

The classifier improved sharply on exactly the three faults the detector still cannot see:

| Fault | Recall — Part 1 | Recall — Part 2 |
|:---:|:---:|:---:|
| 3 | 0.30 | **0.71** |
| 15 | 0.25 | **0.59** |
| 9 | 0.28 | **0.45** |

At first reading this contradicts §2. It does not — the two layers answer different questions:

| Layer | Question | Trained on |
|---|---|---|
| **Detector** | Is this outside the **normal** operating envelope? | normal data only |
| **Classifier** | Given a fault is present, **which of the 20** is it? | faulty data only |

The disturbances in faults 3, 9 and 15 leave a signature that is **inside the normal envelope**
(so no alarm) yet **statistically distinguishable between fault types** once temporal structure
is available (so classifiable).

**Detectability and discriminability are separate properties of a fault.**

### ⚠️ And the system-level consequence

The architecture is sequential: the detector gates the classifier. In deployment, faults 3, 9
and 15 **never raise an alarm**, so the classifier is never invoked on them.

> **The classifier's 71% recall on fault 3 is not operational capability.** It is measured on
> data pre-filtered to contain only faults. The system as a whole remains blind to these three.

Reporting the classifier metric without this caveat would be misleading. It is the difference
between a component metric and a system metric.

**It also revises a limitation stated in Part 1.** That version claimed step and random variants
of the same disturbance could not be separated *in principle* by a sample-level model. Temporal
features separated them — fault 3 recall more than doubled. The word "in principle" was wrong;
the correct statement is that a *sample-level* model cannot separate them, and windowed features
can.

---

## 4. Where Part 2 did work — and why FAR fell

Six faults moved from weak or moderate to near-complete detection:

| Fault | Description | A (raw) | C (+residuals) | D (+temporal) |
|:---:|---|:---:|:---:|:---:|
| 19 | Unknown (Random variation) | 0.174 | 0.667 | **1.000** |
| 5 | Condenser CW inlet temp (Step) | 0.272 | 0.280 | **1.000** |
| 16 | Unknown (Random variation) | 0.271 | 0.373 | **0.786** |
| 10 | C Feed temperature (Random) | 0.406 | 0.797 | **0.976** |
| 20 | Unknown (Random variation) | 0.548 | 0.647 | **0.968** |
| 11 | Reactor CW inlet temp (Random) | 0.775 | 0.792 | **0.999** |

**The two contributions are separable, and they act on different faults:**

- **Residuals dominate** for faults 19 (+0.49) and 10 (+0.39) — these disturbances break a
  physical balance while individual variables stay in range.
- **Temporal features dominate** for faults 5 (+0.72) and 16 (+0.41) — these carry a
  time-signature that a single sample cannot express.

Neither alone would have produced the result.

### Why the false alarm rate went *down*

Adding features normally loosens the normal model and raises false alarms. Here FAR fell from
2.11% to 1.42%.

**Cause:** the rolling statistics average out sample-level sensor noise. Normal operation becomes
a tighter cluster in the enriched space, so spurious single-sample excursions no longer cross the
control limits. The gain is not from better fault sensitivity — it is from a **cleaner definition
of normal**.

This is the practical argument for windowed monitoring in real plants: it suppresses noise and
improves both axes at once.

---

## 5. Residual quality — and what the failures teach

Fitted on normal training data only:

| Residual | Target | R² | Reading |
|---|---|:---:|---|
| `r_stripper` | Stripper temperature | **0.941** | Strong physical relation |
| `r_compressor` | Compressor work | **0.851** | Strong |
| `r_condenser_duty` | Separator temperature | 0.606 | Moderate |
| `r_reactor_energy` | Reactor temperature | 0.398 | Weak |
| `r_reactor_pressure` | Reactor pressure | 0.099 | Very weak |
| `r_mass_balance` | Stripper underflow | **0.001** | Failed |

### ⭐ The pattern — and it is a control insight, not a modelling one

The three weak residuals all target **tightly controlled variables**: reactor pressure, reactor
temperature and stripper underflow are all held at setpoint by the plant-wide control system.

> **Under closed-loop control a controlled variable has almost no variance. A regression fitted
> on it has almost nothing to explain, so R² collapses — regardless of how sound the physics is.**

The two strong residuals target variables that are allowed to **float**: compressor work and
stripper temperature respond to conditions rather than being driven to a setpoint.

**Design rule for future residual-based monitoring:**

> Build residuals on **floating** variables, or on the **manipulated** variables the controller
> moves. Do not build them on controlled variables — the controller has already removed the
> information you are trying to measure.

`r_mass_balance` failed for exactly this reason: all four feeds and the product flow are
flow-controlled, so under normal operation there is no variation to relate.

---

## 6. ⭐ Six physics residuals versus fifty-two raw variables

Set **B** uses only the six residuals — **one ninth of the feature count** — and beats the raw
52-variable detector on several faults:

| Fault | A (52 raw) | B (6 residuals) |
|:---:|:---:|:---:|
| 19 | 0.174 | **0.613** |
| 10 | 0.406 | **0.845** |
| 20 | 0.548 | **0.661** |
| 16 | 0.271 | **0.328** |

And loses badly on others:

| Fault | A (52 raw) | B (6 residuals) |
|:---:|:---:|:---:|
| 7 | 1.000 | **0.344** |
| 17 | 0.922 | **0.497** |
| 14 | 1.000 | **0.762** |

**Reading:** the residuals encode a *specific* physical view — energy and duty balances around
the reactor, condenser, compressor and stripper. Faults that break those balances are caught with
six numbers. Faults outside that view are missed almost entirely; fault 7 is a header pressure
loss, outside every balance defined here.

> **Physics-informed features are a lens, not a compression.** They see their own subsystem
> extremely well and are blind outside it. Coverage comes from choosing the right set of balances,
> not from adding more of the same.

This is also why set **C** (raw + residuals) outperforms both: raw variables provide coverage,
residuals provide sensitivity.

---

## 7. Attribution — SHAP vs SPE contribution

Overlap between the classifier's SHAP drivers and the detector's SPE contributors (top 6):

| Fault | Agreement |
|:---:|:---:|
| 4 | 2/6 |
| 6 | 1/6 |
| 13 | 2/6 |
| 14 | 1/6 |

**Low agreement is expected**, because the two answer different questions — the same distinction
as §3:

| | Question answered |
|---|---|
| **SHAP on the classifier** | Which variables **distinguish this fault from the other 19**? |
| **SPE contribution** | Which variables **deviate from the normal correlation structure**? |

Variables such as the recycle valve (`xmv_5`) and the reactor CW valve appear in the SPE top list
for faults 6, 13 and 14 alike — **general stress responders**, informative for detection and
useless for discrimination. The classifier correctly assigns them low importance.

Where the two agree, they agree on the physically primary variable: Reactor CW Valve for fault 4,
Reactor CW Outlet Temp for fault 14.

### Physical consistency of the SHAP attributions

| Fault | Top drivers | Verdict |
|:---:|---|:---:|
| **4** — Rx CW inlet temp (Step) | Reactor CW Valve (9.84) · CW Outlet Temp (0.93) · Reactor Temp (0.66) | ✅ |
| **6** — A Feed loss (Step) | A Feed (19.69) · A Feed Valve (6.36) | ✅ |
| **14** — Rx CW valve (Sticking) | Reactor Temp (5.37) · CW Outlet Temp (5.05) · CW Valve (2.84) | ✅ |
| **13** — Reaction kinetics (Drift) | Stripper Pressure (1.12) · Steam Flow (0.88) · Product Comp. E (0.86) | ✅ |

Two details worth stating:

**Fault 4 — the valve outranks the temperature.** The strongest signature is the controller's
*compensating action*, not the disturbed variable. Characteristic of a well-controlled plant: the
manipulated variable carries the information the controlled variable no longer shows.

**Fault 14 — reactor temperature outranks the valve.** The reverse ordering, and the correct one
for a sticking valve: the temperature moves while the valve does not respond. The diagnostic
evidence is the **decoupling** between them.

### A methodological limitation

Fault 6 (A feed loss): A Feed ranks 1st in SHAP but only 5th in SPE contribution.

SPE measures deviation in the **residual subspace** — what PCA cannot explain. A large shift that
stays *within* the principal subspace appears in **T²**, not SPE. A clean feed drop is highly
correlated with its own valve and downstream flows, so it is largely captured inside the model
subspace.

> **Correction to apply:** compute contributions for **both T² and SPE**. The current attribution
> is incomplete for in-subspace faults. Listed as a known limitation rather than silently fixed,
> because the finding is itself informative.

---

## 8. False alarm rate — engineering reading

| | Part 1 | Part 2 |
|---|:---:|:---:|
| Control limits | 99th percentile of training normal | same |
| Theoretical FAR | ≈ 1% | ≈ 1% |
| **Observed FAR** | **2.11%** | **1.42%** |

Part 1's rate was roughly double the theoretical value — not a coding error. Limits were fitted on
40 normal simulation runs, and unseen normal runs contain variability those 40 do not span.

**In plant terms:** a monitoring model fitted on a limited window of "known good" operation will
alarm more often than expected once it meets genuinely new — but still normal — conditions.

Part 2's windowed features close most of that gap by suppressing sample-level noise.

**Operational meaning:** at 3 minutes per sample, 1.42% is roughly **one false alarm every
3.5 hours** (Part 1: every 2.4 hours). Acceptable for an advisory system that ranks candidate
issues. Not acceptable for autonomous action without alarm-suppression logic.

---

## 9. Detection delay

Median samples from fault onset to first alarm (1 sample = 3 minutes), raw detector:

| Delay | Faults |
|---|---|
| **Immediate** (0 samples) | 4, 5, 6, 7, 14 |
| **Under 30 min** | 1, 11, 12, 19 |
| **30–90 min** | 2, 8, 10, 16, 17, 20, 13 |
| **Over 90 min** | 3, 9, 15, 18 |

The four slowest include the three undetectable faults — their "delay" is an artefact of
occasional false alarms, not real detection. Fault 18 at 114 minutes is a genuine slow detection.

---

## 10. Limitations

1. **Faults 3, 9 and 15 are undetectable** by any feature set tested. A measurement limitation,
   not a modelling one. Fix requires instrumentation (§2).
2. **The classifier's recall on those faults is not operational capability** — the detector gates
   it and never fires (§3).
3. **Residuals on controlled variables are near-useless** (R² ≈ 0.001–0.10). Redesign on floating
   or manipulated variables (§5).
4. **Residual coverage is narrow** — fault 7 detection collapses from 1.00 to 0.34 in the
   residuals-only set, because no balance defined here covers the header pressure path (§6).
5. **SPE-only attribution** misses in-subspace faults (§7).
6. **Cost:** 112 features and 773 s training versus 52 features and 330 s, for +7.4 points of
   accuracy. Justified here, but it is a real cost.
7. **Simulation data.** No sensor dropouts, calibration drift, maintenance events, or simultaneous
   faults.
8. **40 of 500 simulation runs** used per fault.

---

## 11. What this system is, and is not

**It is** an advisory monitoring layer: it flags a deviation, ranks contributing variables, and
points the engineer toward a candidate physical mechanism.

**It is not** an autonomous safety or shutdown system. Three of twenty disturbances are invisible
to it, and it raises a false alarm roughly every 3.5 hours.

**Correct deployment** is as a triage tool that shortens the search for an engineer who already
knows the plant.

**And it produces one instrumentation recommendation:** add a D feed temperature transmitter and
condenser CW valve position feedback. Those three faults become detectable only then.

---

## 12. Next steps

| Priority | Action | Rationale |
|:---:|---|---|
| 1 | Add **T² contribution** alongside SPE | Closes the gap in §7 |
| 2 | **Redesign residuals on floating / manipulated variables** | §5 — the three failures share one cause |
| 3 | **Add balances covering the header and purge paths** | §6 — fault 7 exposes the coverage gap |
| 4 | **Alarm persistence logic** (N consecutive samples); re-measure FAR vs detection delay | Moves toward autonomous use |
| 5 | Test sensitivity to number of training runs | Does FAR fall further as normal coverage grows? |
