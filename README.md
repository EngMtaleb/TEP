# TEP — Anomaly Detection & Fault Diagnosis

Turning a Tennessee Eastman Process notebook into a validated engineering system —
with a leakage-free evaluation, unsupervised anomaly detection, and physically-grounded attribution.

**Research thread:** hybrid physics + ML modelling for process monitoring and degradation prediction.

---

## Headline results

| Metric | Value |
|---|---|
| **False alarm rate** | **2.11%** |
| **Accuracy** (20 fault classes) | **83.5%** |
| **Macro F1** | **84.2%** |
| Split | Training files vs Testing files — **separate simulation runs, no leakage** |
| Train / test rows | 384,000 / 640,000 |

> Accuracies above 95% are commonly reported on this benchmark. They generally come from
> random row-level splitting, which places temporally adjacent samples from the *same*
> simulation run in both train and test. The numbers here use run-level separation.

---

## What this project actually found

**Three faults are invisible — at exactly the false alarm rate.**

| Fault | Description | Detection rate |
|:---:|---|:---:|
| 3 | D Feed temperature (Step) | 2.1% |
| 9 | D Feed temperature (Random variation) | 2.2% |
| 15 | Condenser cooling water valve (Sticking) | 2.4% |

Detection rate equals the false alarm rate, meaning the detector sees **no signal at all**.
The plant-wide control system fully compensates these disturbances, leaving no residual
signature in the measurements. These three faults are widely reported in the process
monitoring literature as the hardest in this benchmark — reproducing that independently,
with a clean split, is external validation that the detector behaves correctly.

**The same three faults form a closed mutual-confusion cluster** in the classifier
(9→3: 35%, 15→3: 33%, 15→9: 28%). One physical finding explains both results: from the
sensor data, these disturbances look like normal operation.

**Attribution is physically consistent.** For fault 4 (cooling water inlet temperature step),
the strongest SHAP driver is the *cooling water valve* — the controller's compensating action,
not the disturbed variable. For fault 14 (sticking valve), reactor temperature outranks the
valve: the diagnostic evidence is the **decoupling** between them.

📄 **Full analysis: [`docs/VALIDATION.md`](docs/VALIDATION.md)**

---

## Architecture

Two layers, mirroring how monitoring works in a real plant:

```
Layer 1 — Anomaly detection     PCA · Hotelling T² + SPE · trained on normal data only
                                 ↓
Layer 2 — Fault classification   LightGBM · 200 trees · no tuning
                                 ↓
Layer 3 — Attribution            SHAP → contributing variables → candidate mechanism
```

Layer 1 requires no labelled faults, which is the realistic industrial case.

---

## Status

| Stage | Status |
|---|:---:|
| Baseline documented | ✅ |
| Temporal leakage fixed | ✅ |
| Anomaly detection + SHAP | ✅ |
| Engineering validation | ✅ |
| Production refactor (`src/`) | ⬜ |
| API + Docker + CI | ⬜ |
| Deployment | ⬜ |

---

## Known limitations

1. **Faults 3, 9 and 15 are undetectable** and mutually indistinguishable — a property of the
   process and sensor set, not a tuning issue.
2. **Sample-level model** — no temporal dynamics, so step vs random variants of the same
   disturbance cannot be separated in principle.
3. **SPE-only attribution** misses in-subspace faults; T² contribution should be added.
4. **False alarm rate is ~2× theoretical** on unseen normal runs.
5. **Simulation data** — no sensor dropouts, calibration drift, or simultaneous faults.

This is an **advisory monitoring layer**, not an autonomous safety system.

---

## Data

**Source:** Tennessee Eastman Process (Rieth et al.) — a public benchmark for fault detection
in chemical processes.

⚠️ Data files are not committed. Place them in `data/`, or use
[`afrniomelo/tep-csv`](https://www.kaggle.com/datasets/afrniomelo/tep-csv) on Kaggle.

---

## Structure

```
data/                    # not committed
notebooks/               # exploration and analysis
src/                     # production code
docs/
  ├── BASELINE.md        # results before any fixes
  ├── VALIDATION.md      # engineering validation ← start here
  ├── results.json
  └── confusion.png
tests/
```

---

## Setup

```bash
uv venv && source .venv/bin/activate
uv pip install -r requirements.txt
```
