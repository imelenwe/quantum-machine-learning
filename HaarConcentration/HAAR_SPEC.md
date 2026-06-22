# Haar-random states & concentration of measure — experiment spec

Follow-on to the VQE lab — reuses its Estimator/Sampler primitive patterns.

---

## Status (last updated 2026-06-22) — CORE COMPLETE

Pivoted from the original circuit + hardware plan to an **exact classical
statevector** study, which is where the real result lives. Implemented in
`HaarConcentration.ipynb`:

- ✅ Random-circuit state prep + depth sweep — finding: a feasible-depth circuit
  (linear CZ ladder + Ry/Rz) does **not** reach a true Haar state at n = 10
  (std 0.031 at depth 20 vs theory 0.0099). Preparable ≠ Haar.
- ✅ True Haar baseline via `random_statevector` — std 0.00985 matches theory
  1/√(n·2ⁿ) = 0.00988 exactly. (Haar states are entangled by construction and
  have no efficient circuit, so this only works in simulation.)
- ✅ Shot-noise limitation (Aer): at 500 shots the shot floor (~0.014) already
  exceeds the concentration spread (~0.010) — can't distinguish the spread from
  sampling noise. More shots recovers it at n = 10, but the needed shots grow ~2ⁿ.
- ✅ **n-scaling figure (the deliverable):** spread falls as 1/√(2ⁿ) while the
  shot floor (1/√shots) stays nearly flat; they cross at n ≈ log₂(shots). At
  156 qubits the spread (~10⁻²⁵) is ~20 orders of magnitude below any realistic
  floor.

**Conclusion:** random states on a large machine are useless for finding
low-energy states — every state gives ≈ the mean energy, and the differences
that matter sit forever below the measurement floor (the barren-plateau wall,
seen as a measurement problem).

**Hardware: DROPPED.** Not run, and shouldn't be — a true Haar state cannot be
prepared on hardware (exponential gate count), and even if it could, the spread
is below the shot/device-noise floor. The RUN CONFIGURATION below is the
original plan, kept for the record but **superseded**.

---

## RUN CONFIGURATION (original plan — SUPERSEDED, hardware dropped)

Fixed for the hardware run; Aer sim uses the **identical** N and shots.

- **n = 10 qubits**, **N = 20** random state instances, **shots = 500** per state.
- Total = 20 × 500 = **10,000 circuit executions**.
- Submit as a **single batched job** — all 20 circuits in one Estimator call.
- Estimated hardware time: **~1.5–2 min** (QPU budget: ~4 min remaining on ibm_fez,
  so this stays within budget).

**Why these numbers:**
- `H_local = (1/n) Σᵢ Zᵢ` has concentration spread ~1/√(2^n) ≈ 0.03. Shot noise
  must sit below that, so ideally shots > ~1000; **500 is an accepted compromise**
  given the budget.
- N = 20 is enough to *visually* see the clustering — not enough for a pretty
  histogram, but enough to make the point.

**Hard rule:** run Aer sim first with identical N and shots. Only submit to
ibm_fez once Aer looks correct.

**Expected hardware effect:** noise on ibm_fez will squeeze the distribution even
tighter toward 0 than Aer (mixing toward the maximally mixed state). This is
expected and fine — not a bug.

---

## GOAL
Empirically verify concentration of measure for Haar-random quantum states on
IBM hardware (ibm_fez), connect it to the barren plateau phenomenon, and check
whether it differs for global vs. local cost functions / Hamiltonians.

## BACKGROUND
For a fixed Hamiltonian H on n qubits, Haar-random states |ψ⟩ = U|0⟩ (U Haar-
random) satisfy: ⟨ψ|H|ψ⟩ concentrates near Tr(H)/2^n with exponentially small
spread (Levy's lemma / concentration of measure). Rare states far from this
average (e.g. near the ground-state energy) exist but have exponentially small
measure, so a random draw essentially never lands near one.

Established in discussion: fixing n = 10 qubits and sampling many random
state instances is sufficient to observe the concentration effect directly —
you do NOT need to vary n to see it. (Varying n / depth is a secondary,
optional extension below, not the core ask.)

## TASKS

### 1. Random state preparation
- Fix n = 10 qubits (per prior discussion — see Background).
- Build a parameterized random circuit deep enough to approximate a
  Haar-random / 2-design state: alternating layers of random single-qubit
  rotations (random angles ~ Uniform(0, 2π) on RX/RY/RZ) and a fixed
  entangling layer (CNOT or CZ ladder).
- Depth should be a tunable parameter, deep enough by default to plausibly
  approximate Haar-randomness at n=10 (this can be checked empirically —
  see Task 4 optional extension).

### 2. Hamiltonian — define BOTH a global and a local version
- **Global H**: H_global = I − |0⟩⟨0|^⊗n  (projector-style, depends on the
  joint state of all n qubits at once).
- **Local H**: H_local = (1/n) Σᵢ Zᵢ, or reuse the Pauli-string Hamiltonian
  already used in the VQE lab if more convenient/comparable.
- For both: compute Tr(H)/2^n analytically as the predicted Haar mean.
  (For a traceless Pauli-sum H_local this mean is 0; for the projector
  H_global it's 1 − 1/2^n ≈ 1.)
- Also compute the ground-state energy of each H classically (diagonalization)
  for reference on the plots.

### 3. Sampling loop
- For n = 10, generate N random circuit instances with fresh random angles
  each time. **Final values (see Run Configuration): N = 20, shots = 500.**
  (Sim development may use a larger N freely; the hardware run is fixed at
  N = 20 / 500 shots to fit the QPU budget.)
- For each instance, two-layer measurement structure:
  - INNER loop: repeated shots on that one fixed random state to estimate
    ⟨ψ|H|ψ⟩ (same Estimator-primitive approach as the VQE lab).
  - OUTER loop: fresh random states, to build up the distribution of
    ⟨ψ|H|ψ⟩ across instances.
- Do this separately for H_global and H_local, same set of random states if
  convenient (so both are evaluated on identical circuit draws).

### 4. Compare hardware vs simulator
- Run the same sampling loop on a noiseless simulator (Aer) and on ibm_fez,
  at n = 10.
- Plot both distributions together for each of H_global and H_local.
  Expect hardware distributions to be shifted/narrowed further toward the
  predicted mean due to noise-induced mixing toward the maximally mixed
  state, compared to the ideal simulator distribution.

### 5. Visualization
- Histogram of ⟨ψ|H|ψ⟩ across the N random instances, with vertical lines
  marking: (a) Tr(H)/2^n (predicted Haar mean), (b) the true ground-state
  energy (for scale/reference).
- Separate histograms (or faceted/overlaid plot) for H_global vs H_local,
  and for simulator vs hardware.
- Compare spread (std dev) of H_global vs H_local distributions at the same
  n and depth — test the slide's claim that local cost functions resist
  concentration more than global ones at limited depth.

## OPTIONAL EXTENSIONS (secondary, only if primary n=10 result is clean)
- Vary n (e.g. 4, 6, 8, 10) at fixed depth to see the spread narrow with n,
  and check scaling against the P(non-zero gradient) ~ O(1/n^ℓ) result from
  lecture (ℓ likely = number of layers — confirm against lecture notes).
- Vary depth at fixed n to find the minimum depth at which the random
  circuit already behaves like a 2-design (distribution stops changing
  with further depth increases).
- Layer-wise training comparison (separate follow-on experiment, not part
  of this Haar-sampling task): compare training a VQE ansatz layer-by-layer
  (train shallow, freeze/partially-freeze, add a layer, retrain) against
  full-depth random initialization, to empirically test whether layer-wise
  training avoids the plateau as the lecture claims.

## CONSTRAINTS
- IBM Quantum hardware budget is limited (~10 min/month, partially used) —
  default to Aer simulation for development/iteration; only run a small,
  deliberately chosen N configuration on ibm_fez once the pipeline is
  verified on the simulator, at the fixed n=10.
- Reuse Estimator/Sampler primitive patterns already established in the
  VQE lab code rather than introducing new patterns.

## DELIVERABLE
A notebook/script that: builds random circuits at n=10, computes ⟨H⟩ for
both H_global and H_local on simulator and hardware, and produces histograms
showing concentration around Tr(H)/2^n — with global vs. local and
simulator vs. hardware comparisons clearly laid out side by side.
