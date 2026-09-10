TEP — Anomaly Detection & Fault Diagnosis

Turning a Tennessee Eastman Process capstone notebook into a validated engineering system — with a leakage-free evaluation, unsupervised anomaly detection, physics-informed residuals, and physically-grounded attribution.

Research thread: hybrid physics + ML modelling for process monitoring and degradation prediction.

⚠️ Correction notice

The original capstone in this repository was evaluated with a temporal data leakage. It split rows randomly, but consecutive rows come from the same simulation run and are strongly correlated — so near-identical samples appeared in both training and test sets.

The reported accuracy was inflated and did not measure generalisation.

The original notebook is preserved unchanged in notebooks/01_original_capstone.ipynb. Everything below uses run-level separation. The full account, including four further methodological errors found during the correction, is in docs/VALIDATION.md.

Headline results
Metric	Part 1 · raw	Part 2 · enriched
Detector features	52	112
False alarm rate	2.11%	1.42%
Accuracy (20 fault classes)	83.5%	90.9%
Macro F1	84.2%	91.2%

Split: Training files vs Testing files — separate simulation runs, no leakage. 384,000 training rows / 640,000 test rows.

Accuracies above 95% are commonly reported on this benchmark. They generally come from random row-level splitting, which places temporally adjacent samples from the same simulation run in both train and test.

Part 2 improved detection and reduced false alarms simultaneously — the rolling statistics suppress sample-level noise, producing a cleaner definition of normal rather than higher fault sensitivity.

⭐ What this project found
Three faults are invisible — at exactly the false alarm rate
Fault	Description	Detection rate
3	D feed temperature (step)	2.1%
9	D feed temperature (random variation)	2.2%
15	Condenser cooling water valve (sticking)	2.4%

Detection rate equal to the false alarm rate means the detector sees no signal at all — every alarm raised during these faults is a baseline false alarm that would have occurred on normal data too.

Why: the disturbed variable is not measured. The 22 process measurements contain no feed temperature, and for the sticking valve only the command is recorded, not the actual stem position. This is an observability limit, not a modelling one.

Physics-informed residuals and temporal features were tested against this. The hypothesis was rejected — all three remain at the noise floor in every feature set.

The recommendation is instrumentation, not algorithms: a D feed temperature transmitter, and valve position feedback on the condenser cooling water valve.

Detectability and discriminability are different properties

The classifier improved sharply on exactly those three faults — fault 3 recall rose from 0.30 to 0.71 with temporal features.

This does not contradict the above. The detector asks "is this outside normal?"; the classifier asks "which of the 20 faults is it?" The disturbances sit inside the normal envelope yet remain statistically distinguishable between fault types.

⚠️ And the system-level consequence: the detector gates the classifier. In deployment these faults never raise an alarm, so the classifier is never invoked on them. The 71% recall is a component metric, not operational capability.

Attribution is physically consistent

For fault 4 (cooling water inlet temperature step), the strongest SHAP driver is the cooling water valve — the controller's compensating action, not the disturbed variable.

For fault 14 (sticking valve), reactor temperature outranks the valve: the diagnostic evidence is the decoupling between them.

Residuals fail on controlled variables — and that is the finding
Residual	Target	R²
Stripper temperature	floating	0.941
Compressor work	floating	0.851
Reactor temperature	controlled	0.398
Reactor pressure	controlled	0.099
Mass balance	controlled	0.001

Under closed-loop control a controlled variable has almost no variance. A regression fitted on it has nothing to explain, regardless of how sound the physics is.

Design rule: build residuals on floating or manipulated variables — never on controlled ones. The controller has already removed the information you are trying to measure.

📄 Full analysis: docs/VALIDATION.md

Architecture
Layer 1 — Anomaly detection     PCA · Hotelling T² + SPE · trained on normal data only
                                        ↓
Layer 2 — Fault classification  LightGBM · 200 trees · no tuning
                                        ↓
Layer 3 — Attribution           SHAP → contributing variables → candidate mechanism

Layer 1 requires no labelled faults, which is the realistic industrial case.

Part 2 adds: six physics-informed residuals (mass and energy balances, duty relations) and rolling statistics computed within each simulation run.

Status
Stage	
Baseline documented	✅
Temporal leakage fixed	✅
Anomaly detection + SHAP	✅
Physics residuals + temporal features	✅
Engineering validation	✅
Production refactor (src/)	⬜
API + Docker + CI	⬜
Deployment	⬜
Known limitations
Faults 3, 9 and 15 are undetectable by any feature set tested — a property of the process and sensor set. Fixing it requires instrumentation.
Their classifier recall is not operational capability — the detector gates the classifier and never fires on them.
Residuals on controlled variables are near-useless (R² ≈ 0.001–0.10). Redesign on floating or manipulated variables.
SPE-only attribution misses in-subspace faults; T² contribution should be added.
False alarm rate is ~2× theoretical on unseen normal runs (Part 1), improved to ~1.4× with windowed features.
Simulation data — no sensor dropouts, calibration drift, or simultaneous faults.
40 of 500 simulation runs used per fault.

This is an advisory monitoring layer, not an autonomous safety system.

Data

Source: Tennessee Eastman Process (Rieth et al.) — a public benchmark for fault detection in chemical processes.

⚠️ Data files are not committed. Place them in data/, or use afrniomelo/tep-csv on Kaggle.

Structure
data/                              # not committed
notebooks/
  ├── 01_original_capstone.ipynb   # as submitted - kept for the record
  └── 02_corrected.ipynb           # leakage-free evaluation + Part 2
src/                               # production code
docs/
  ├── BASELINE.md                  # results before any fixes
  ├── VALIDATION.md                # where it fails, and why ← start here
  ├── results.json
  └── confusion.png
tests/
index.html                         # interactive dashboard
Setup
bash
uv venv && source .venv/bin/activate
uv pip install -r requirements.txt
Author

Mohammed Taleb — chemical engineer and instructor, building AI systems for industrial process problems.

Implementation assisted by AI tools. Engineering framing, interpretation and validation are the author's.
