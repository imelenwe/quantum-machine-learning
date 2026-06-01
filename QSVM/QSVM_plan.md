# QSVM vs Classical SVM — Project Plan

## 1. Goal
Build a **single notebook** (`QSVM/QSVM.ipynb`, currently empty) that compares:

1. **Classical SVM** — full Titanic data, the real baseline.
2. **QSVM on a simulator** — full data, exact statevector. The *real* quantum-vs-classical
   accuracy comparison.
3. **QSVM on real IBM hardware** — a small subsample, to demonstrate a genuine hardware run
   and visualize noise.

Across **two feature maps**: `ZZFeatureMap` with **linear** entanglement (shallower) and
**full** entanglement (deeper, higher expressivity). Binary classification: Titanic survival.

---

## 2. Environment (RESOLVED ✅)
- **Conda env:** `qcml-ibmqc` (user-selected, confirmed correct).
- **Versions:** qiskit `1.4.3`, scikit-learn `1.8.0`, qiskit-ibm-runtime `0.40.0`,
  qiskit-aer `0.14.2`, numpy `1.26.4`.
- **Action taken:** installed the only missing piece — `qiskit-machine-learning==0.8.2`
  (pinned to the 0.8.x line that targets qiskit 1.x; did not disturb qiskit 1.4.3).
- **Verified imports:** `ZZFeatureMap`, `FidelityStatevectorKernel`,
  `FidelityQuantumKernel`, `QSVC`, `QiskitRuntimeService`, `SamplerV2` all import cleanly.
- Why qiskit 1.x and not the 2.3.1 env: 1.x keeps the `ZZFeatureMap` class API; 2.x
  deprecates it. Also `qcml-ibmqc` already has sklearn + ibm_runtime; the 2.x env does not.

## 3. Data (RESOLVED ✅)
- `QSVM/1780212217603_train.npz` → `train_features (711, 7)`, `train_labels (711,)`
- `QSVM/1780212217603_test.npz`  → `test_features (178, 7)`,  `test_labels (178,)`
- 7 preprocessed numerical features → **7 qubits**. Labels are float64 (cast to int).

---

## 4. Approach — notebook sections  ⚠️ PROPOSED, pending sign-off
| # | Section | Data | Backend | Output |
|---|---|---|---|---|
| 0 | Setup: imports, version print, `random_state`, `shots` | — | — | env sanity |
| 1 | Load `.npz`, cast labels to int, sanity-check shapes | full | — | X/y train/test |
| 2 | **Classical SVM** baseline (`SVC`, RBF) | full | CPU | accuracy |
| 3 | Build both feature maps (linear + full), draw circuits, print depth/gate counts | — | — | circuits |
| 4 | **QSVM simulator**, both maps (`FidelityStatevectorKernel` → `QSVC`) | full | CPU | 2 accuracies |
| 5 | Stratified subsample helper (~30 train / 15 test, class-balanced) | subsample | — | small X/y |
| 6 | **QSVM simulator on subsample**, both maps (noise baseline) | subsample | CPU | 2 accuracies |
| 7 | **QSVM on IBM hardware**, both maps (`FidelityQuantumKernel` + `SamplerV2`, in a Session) — *fenced so Run All can't trigger it* | subsample | IBM | 2 accuracies + kernel matrices |
| 8 | **Results**: comparison table + bar chart; sim-vs-hardware kernel heatmaps (noise) | all | — | final comparison |

## 5. The 1-hour reality (important)
- Sections **0–6 and 8** run on CPU/simulator → fast, can be fully built & validated within the hour.
  **This is the real deliverable + the meaningful accuracy comparison.**
- Section **7 (hardware)** depends on IBM queue time. Even a 30/15 subsample = ~885 kernel
  circuits *per feature map* (~1770 total). That likely **will not finish inside one hour**.
  - Recommendation: lock & validate everything on the simulator first, then *launch* the
    hardware run and let it complete on its own schedule. Optionally shrink the subsample
    (e.g. 15 train / 8 test) for a faster but rougher hardware demo.

## 6. Decisions (LOCKED 2026-06-01)
- [x] Section plan (§4) approved.
- [x] Hardware subsample: **16 train / 8 test**.
- [x] `shots`: **4096**.
- [x] Classical baseline: **RBF only** (`SVC()` default).
- [x] Feature map depth: **`reps=1`**.
- [x] IBM credentials: **load token from `.env`** (gitignored). Needs `IBM_QUANTUM_TOKEN` in
      `QSVM/.env`; confirm `python-dotenv` is available (or parse manually).

## 6b. Finding — subsample is degenerate for accuracy (2026-06-01)
At 16 train / 8 test the QSVM predicts all-majority (acc = 0.625 = baseline) for **both**
feature maps — too few points to learn a boundary (confirmed: preds `[0 0 0 0 0 0 0 0]`;
~24+ train needed to be non-degenerate). **Decision: keep 16/8.** The hardware run is a
**noise demonstration**, not an accuracy contest:
- Headline accuracy comparison lives in the **full-data simulator** (Sec 4: classical 0.775
  vs QSVM 0.809 / 0.832).
- Section 7/8 deliverable = the **sim-vs-hardware kernel-matrix heatmap** (visualizing noise),
  plus proving it runs on real hardware (`ibm_fez`).
- The Aer dry-run is validated (pipeline correct end-to-end).

## 7. Risks
- **Kernel = O(n²) circuits** → never run full data on hardware (see §5).
- **Primitive wiring:** `FidelityQuantumKernel` + `ComputeUncompute` + `SamplerV2` is the
  fiddliest part across qiskit-machine-learning 0.8 / ibm_runtime 0.40 — pin down in §7 first.
- **Noise → non-PSD kernel matrix:** `SVC(kernel='precomputed')` may need a small eigenvalue
  clip / regularization.
- **Run-All hazard:** the hardware cell must be clearly fenced.

---

## 8. Workflow note
Per our agreement: I print code in responses for you to read & type; I don't edit
`QSVM.ipynb` until you say so. We go section by section, starting at Section 0.
