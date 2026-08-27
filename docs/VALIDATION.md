# Engineering Validation

**Project 0 — Tennessee Eastman Process: anomaly detection, fault classification, attribution**

This document records what the system does, what it fails at, and **why** — in process
engineering terms rather than metric terms. It is the primary artefact of this project.

---

## 1. Headline results

| Metric | Value |
|---|---|
| Anomaly detector | PCA · Hotelling's T² + SPE · 99th percentile limits · trained on normal data only |
| **False alarm rate** | **2.11%** |
| Classifier | LightGBM · 200 trees · no tuning |
| **Accuracy** | **83.5%** |
| **Macro F1** | **84.2%** |
| Split | Training files vs Testing files — **separate simulation runs, no leakage** |
| Train / test rows | 384,000 / 640,000 |

> Accuracies above 95% are commonly reported on this dataset. They generally come from
> random row-level splitting, which places temporally adjacent samples from the *same*
> simulation run in both train and test. The numbers above use a run-level separation and
> are therefore lower and generalisable.

---

## 2. Finding 1 — Three faults are effectively invisible

| Fault | Description | Detection rate |
|:---:|---|:---:|
| 3 | D Feed temperature (Step) | **2.1%** |
| 9 | D Feed temperature (Random variation) | **2.2%** |
| 15 | Condenser cooling water valve (Sticking) | **2.4%** |

**These detection rates equal the false alarm rate (2.11%).**

That is the important observation. The detector is not *partially* detecting these faults —
it is detecting **nothing at all**. Every alarm raised during these faults is a baseline
false alarm that would have occurred on normal data too.

**Physical interpretation:** these three disturbances produce deviations that the plant-wide
control system fully compensates for. The controlled variables return to setpoint, and the
compensating action is small enough to stay inside the normal operating correlation structure.
There is no residual signature left in the measurements for a correlation-based detector to see.

**External validation:** faults 3, 9 and 15 are widely reported in the process monitoring
literature as the hardest — often undetectable — faults in this benchmark. Reproducing that
result independently, with a clean split, is evidence that the detector behaves correctly
rather than evidence that it is broken.

### Detection rate — full ranking

| Band | Faults | Detection rate |
|---|---|---|
| **Undetectable** | 3, 9, 15 | 0.02 |
| **Weak** | 19, 16, 5, 10 | 0.17 – 0.41 |
| **Moderate** | 20, 11 | 0.55 – 0.78 |
| **Strong** | 17, 18, 13, 8, 12, 2, 1 | 0.92 – 0.99 |
| **Complete** | 14, 4, 6, 7 | 1.00 |

The strong band is dominated by step disturbances in feed and cooling — large, persistent,
directly measured effects. The weak band is dominated by random variations, whose signature
is intermittent and partially absorbed by the controllers.

---

## 3. Finding 2 — The confusion cluster is the same three faults

| True → Predicted | Rate | Comment |
|---|:---:|---|
| 9 → 3 | 35.0% | Both are **D Feed temperature** — same variable, step vs random |
| 15 → 3 | 32.8% | |
| 15 → 9 | 27.7% | |
| 9 → 15 | 27.7% | |
| 3 → 15 | 22.5% | |
| 3 → 9 | 21.9% | Same pair as row 1, reversed |
| 12 → 18 | 11.0% | Condenser CW inlet temp vs an unknown step disturbance |

**Faults 3, 9 and 15 form a closed mutual-confusion cluster — and they are exactly the three
faults the detector cannot see.**

This is a single physical finding explaining both results:

> These three disturbances leave almost no measurable signature. The detector therefore cannot
> flag them, and the classifier cannot separate them from one another, because from the sensor
> data they all look like normal operation.

The 9 ⟷ 3 confusion is additionally **expected and acceptable**: both are disturbances in the
same physical variable (D feed temperature), differing only in whether the disturbance is a
step or a random variation. Distinguishing them requires temporal dynamics that a sample-level
classifier does not have access to.

**No confusion was observed between physically unrelated subsystems** — no feed fault was
confused with a cooling fault, and no reactor fault with a stripper fault. The error structure
is physically coherent.

---

## 4. Finding 3 — Attribution is physically consistent for detectable faults

| Fault | Top SHAP drivers | Expected mechanism | Verdict |
|:---:|---|---|:---:|
| **4** — Reactor CW inlet temp (Step) | Reactor CW Valve (9.84) · Reactor CW Outlet Temp (0.93) · Reactor Temp (0.66) | CW inlet temp rises → reactor heats → controller opens CW valve | ✅ |
| **6** — A Feed loss (Step) | A Feed (19.69) · A Feed Valve (6.36) | Loss of A feed appears directly in the A feed measurement and its valve | ✅ |
| **14** — Reactor CW valve (Sticking) | Reactor Temp (5.37) · Reactor CW Outlet Temp (5.05) · Reactor CW Valve (2.84) | Valve sticks → reactor temperature oscillates without a matching valve response | ✅ |
| **13** — Reaction kinetics (Slow drift) | Stripper Pressure (1.12) · Stripper Steam Flow (0.88) · Product Composition E (0.86) | Kinetics drift → conversion changes → product composition shifts → stripper duty changes | ✅ |

Two details worth noting:

**Fault 4 — the valve outranks the temperature.** The strongest signature is the *controller's
compensating action*, not the disturbed variable itself. This is characteristic of a
well-controlled plant: the manipulated variable carries the information that the controlled
variable no longer shows.

**Fault 14 — reactor temperature outranks the valve.** The reverse ordering, and it is the
correct one for a sticking valve: the temperature moves while the valve does not respond.
The diagnostic evidence is the *decoupling* between them.

**Attribution magnitudes are also informative.** Fault 6 has a dominant driver (19.7, nearly
3× the second) while fault 13 is diffuse (top value 1.12, spread across six variables). This
matches the physics: an abrupt feed loss has one clear cause; a slow kinetics drift expresses
itself weakly across the whole downstream section.

---

## 5. Finding 4 — Detector and classifier attributions disagree, and they should

Overlap between the classifier's SHAP drivers and the detector's SPE contributors:

| Fault | Agreement (top-6) |
|:---:|:---:|
| 4 | 2/6 |
| 6 | 1/6 |
| 13 | 2/6 |
| 14 | 1/6 |

**This is not a failure. The two methods answer different questions:**

| | Question answered |
|---|---|
| **SHAP on the classifier** | Which variables **distinguish this fault from the other 19**? |
| **SPE contribution** | Which variables **deviate most from the normal correlation structure**? |

A variable can deviate strongly under many different faults — the recycle valve (`xmv_5`) and
the reactor CW valve appear in the SPE top list for faults 6, 13 and 14 alike. They are
**general stress responders**, informative for *detection* and useless for *discrimination*.
The classifier correctly assigns them low importance.

Where the two do agree, the agreement is on the physically primary variable:
Reactor CW Valve for fault 4, Reactor CW Outlet Temp for fault 14.

### A methodological limitation of the current SPE attribution

Fault 6 (A feed loss) is the notable case: A Feed ranks **1st** in SHAP but only **5th** in SPE
contribution.

The likely reason is structural. SPE measures deviation in the **residual subspace** — what the
PCA model cannot explain. A large shift that remains *within* the principal component subspace
appears in **T²**, not in SPE. A clean feed-rate drop is highly correlated with its own valve
and with downstream flows, so it is largely captured inside the model subspace.

> **Correction to apply:** contribution should be computed for **both T² and SPE**, not SPE
> alone. The current attribution is incomplete for in-subspace faults. This is listed as a
> known limitation rather than silently corrected, because the finding itself is informative.

---

## 6. False alarm rate — engineering reading

| Quantity | Value |
|---|---|
| Control limits | 99th percentile of training normal data |
| Theoretical FAR at that limit | ≈ 1% |
| **Observed FAR on unseen normal runs** | **2.11%** |

The observed rate is roughly double the theoretical rate. The cause is not a coding error:
control limits were fitted on 40 normal simulation runs, and unseen normal runs contain
operating variability that those 40 runs do not fully span.

**This is itself a finding.** In a real plant, the equivalent statement is that a monitoring
model fitted on a limited window of "known good" operation will alarm more often than expected
once it meets genuinely new — but still normal — operating conditions.

**Operational meaning:** at 3 minutes per sample, 2.11% corresponds to roughly **one false
alarm every 2.4 hours**. Acceptable for an advisory system that ranks candidate issues for an
engineer. Not acceptable for an autonomous system that triggers action, without alarm
suppression logic (persistence filters, multi-sample confirmation).

---

## 7. Limitations

1. **Three faults are undetectable** (3, 9, 15) and mutually indistinguishable. This is a
   property of the process and the sensor set, not a tuning issue. No threshold change fixes it.
2. **Sample-level model.** No temporal dynamics are used, so step vs random variants of the
   same disturbance cannot be separated in principle.
3. **SPE-only attribution** misses in-subspace faults — see §5.
4. **False alarm rate is roughly 2×** the theoretical level on unseen normal runs.
5. **Simulation data.** No sensor dropouts, no calibration drift, no maintenance events, no
   simultaneous faults. Real plant data is harder.
6. **40 of 500 simulation runs used** per fault. Results are stable at this size but not
   maximal.

---

## 8. What this system is, and is not

**It is** an advisory monitoring layer: it flags a deviation, ranks the contributing variables,
and points the engineer toward a candidate physical mechanism.

**It is not** an autonomous safety or shutdown system. It cannot see three of the twenty
disturbances at all, and it raises a false alarm roughly every 2.4 hours.

**Correct deployment** is as a triage tool that reduces search time for an engineer who already
knows the plant — not as a replacement for that engineer.

---

## 9. Next steps

| Priority | Action |
|:---:|---|
| 1 | Add **T² contribution** alongside SPE — closes the gap identified in §5 |
| 2 | Add **alarm persistence logic** (N consecutive samples) and re-measure FAR vs detection delay |
| 3 | Add **temporal features** (rolling statistics) to attempt separation of step vs random variants |
| 4 | Test sensitivity to the number of training runs — does FAR fall as normal coverage grows? |
