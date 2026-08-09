# QAOA from scratch — concept spec

A clean re-derivation of the lab's Quantum Approximate Optimization Algorithm, applied
to MaxCut. Goal: understand the *mechanism* — how a cost function becomes a quantum
circuit, how that circuit gets nudged toward good answers, and how a classical
optimizer closes the loop — not just reproduce the lab's cells. You write the code;
this file is the map.

---

## Status: COMPLETE (last updated 2026-08-09)

`QAOA_MaxCut.ipynb` is done, rerun clean end to end on the lab's 6-node graph.

- ✅ §3 cost function + brute-force ground truth — max cut = 7, reached by exactly 2
  colorings (`011001` / `100110`, exact bitwise complements — the graph's Z2 symmetry)
- ✅ §2a phase operator (`problem_hamiltonian_operator`, CX-Rz-CX per edge)
- ✅ §2b mixing operator (`mixer_hamiltonian_operator`, Rx(β) per qubit)
- ✅ §2c full circuit builder (`create_qaoa_circuit`)
- ✅ §4 expectation value, both `exact` (statevector) and `measured` (8192-shot sampled)
- ✅ §5 classical optimizer — went further than planned: compared Nelder-Mead, SPSA, and
  COBYLA under real shot noise (not just Nelder-Mead), found COBYLA cheapest but flaky,
  and fixed that with a warm-started, 3-restart recipe across depths 1–10 — clean
  monotonic convergence 5.32 → 6.93 (true max 7), ~3,200 circuit executions total
- ✅ §6 p sweep — done as depths 1–10 (beyond the planned p=1..3), both exact and measured
- ➕ not in original scope, added along the way: per-depth circuit depth/gate-count/execution
  instrumentation, and a most-frequently-measured-bitstring tracker — checked across all 30
  depth/optimizer runs, the top bitstring was correct (`011001` or `100110`) every single
  time, even when the average expectation value looked unconvincing
- ⬜ §7 scaling experiment (~20 qubits) — **descoped**, not done
- ⬜ §8 hardware run — **descoped**, not done

§7 and §8 are being superseded rather than picked back up here: the next step is IBM's own
Qiskit Global Summer School 2026 Lab 4b, which already covers both — real Heron r2 hardware,
scaling to 160 and 1,600 nodes via Pauli Correlation Encoding, and a full error-mitigation
stack — at a level this from-scratch build was never aiming to reach. See
[`conclusion.html`](conclusion.html) for the full verdict on what this attempt did and
didn't establish, cross-checked against classical MaxCut solvers and IBM's own published
hardware numbers.

---

## 0. The one-sentence idea

> Encode the problem you want to solve as phases on a quantum state, mix those phases
> into interference, measure, and let a classical optimizer tune two knobs (γ, β) until
> measurement gives you good answers more often than bad ones.

Everything below is just the machine parts that make that sentence runnable: the
**cost function** (what "good" means), the **phase operator** (encodes it), the
**mixer** (turns phases into interference), the **expectation value** (what we're
actually optimizing), and the **classical optimizer** (closes the loop).

---

## 1. Why this needs two operators (there's no free lunch like VQE's bound)

VQE had a hard guarantee: `⟨ψ|H|ψ⟩ ≥ E₀` always, so minimizing can only approach the
truth from a safe direction. **QAOA has no equivalent guarantee.** There's no inequality
saying our expectation value can only improve — instead the whole method rests on
*steering probability mass* toward good answers through interference. That makes the
two operators genuinely necessary, not just convenient:

- **Phase operator `U(C,γ)` alone** — Starting from the equal superposition `|s⟩`
  (every coloring equally likely), applying only `U(C,γ)` changes *phases*, not
  probabilities. A measurement right after would still show every coloring with
  identical likelihood. Phases you can't measure are invisible — so this step alone
  accomplishes nothing observable.
- **Mixing operator `U(B,β)` alone** — Without phase information encoding the cost
  function, mixing just redistributes amplitude with no bias toward "good" — equivalent
  to blind noise.
- **Both together, repeated `p` times** — the phase step "tags" good colorings with a
  distinct phase, and the mixing step lets those tagged amplitudes interfere
  constructively with each other and destructively with bad ones. Whether that
  interference actually favors good states depends on `γ` and `β` — which is exactly
  what the classical optimizer searches for.

```
   |0…0⟩ ─ H^⊗n ─► |s⟩ ─► [U(C,γ) → U(B,β)] × p ─► measure
                              ▲                        │
                              └── classical optimizer ◄─┘
                                   (tune γ, β)
```

---

## 2. Circuit anatomy, for MaxCut specifically

### 2a. Phase operator `U(C,γ)`

MaxCut's cost function is a sum over edges: `C(z) = Σ_(i,j)∈E  ½(1 − z_i z_j)`. Dropping
the constant term (it only adds a global phase — no measurable effect, same argument the
lab makes), the operator to implement is `exp(-iγ Σ Z_i Z_j)`, which — because all the
`Z_i Z_j` terms commute — factors into one small gadget per edge:

```
exp(-iγ Z_i Z_j)  ⟹  CX(i→j) · Rz(2γ)(j) · CX(i→j)      [derive the angle factor together]
```

We'll derive *why* the CNOT-sandwich implements a `ZZ` rotation rather than copying it
from the lab — that's the actual "how does the quantum part solve it" question. One
thing worth confirming ourselves: whether the angle passed to `Rz` should be `γ` or `2γ`
depends on the sign/scale convention we pick for `C(z)` — don't assume, derive it against
a small (2-qubit) matrix check.

### 2b. Mixing operator `U(B,β)`

`Rx(β)` on every qubit, one layer, circuit depth 1 — this is the standard "transverse
field" mixer, same one the lab uses (`Ub_Mixer1`).

### 2c. Full circuit

```
H^⊗n |0…0⟩ → [U(C,γ_1) → U(B,β_1)] → [U(C,γ_2) → U(B,β_2)] → … → [p rounds] → measure
```

Graph (`n`, edge list) is an input from the start — not hardcoded — so §7's scaling
experiment is a change of input, not a rewrite.

---

## 3. Cost function + brute-force ground truth

Two things to write, both graph-generic:

- `maxcut_cost(edges, bitstring)` — score a single coloring.
- `all_energies(n, edges)` — brute-force enumerate all `2ⁿ` colorings and their costs
  (feasible up to `n≈20`, exactly like the lab's `MaxCut_Energy`). This is simultaneously
  (a) the answer key for grading, and (b) the ingredient for the exact expectation value
  in §4.

First validation target: the lab's own 6-node graph (`Vert=[0..5]`,
`Edge=[[0,1],[0,2],[0,5],[1,2],[1,3],[2,3],[2,4],[3,5]]`), known max cut = **7**, reached
by exactly 2 colorings (verified by brute force in this session already).

---

## 4. Expectation value F(γ,β) — exact and measured

Same duality as `VQE_SPEC.md` §2: one function, two methods.

- **`exact`** — weight each brute-force energy by its statevector probability
  `|amplitude|²` and sum. No sampling, no noise — the noise-free reference.
- **`measured`** — sample the circuit, and for each observed bitstring look up its cost
  directly. **Simpler than VQE here**: MaxCut's cost is already diagonal in the
  computational (Z) basis, so a plain measurement is all that's needed — no basis-
  rotation gymnastics like VQE's X/Y Pauli terms required. Worth noticing explicitly
  when we build it: this is a real structural simplification, not a coincidence.

> **Design note:** write `expectation_value(params, p, edges, n, method)` once with
> `method ∈ {"exact","measured"}`, shared by every optimizer call — same discipline as
> the VQE build.

---

## 5. Classical optimizer

`scipy.optimize.minimize(f, x0, method="Nelder-Mead")` over the `2p`-length vector
`[β_1..β_p, γ_1..γ_p]`, minimizing `-F(γ,β)` (we want to *maximize* cut count). No
hand-rolled simplex this time — you already built and understand Nelder-Mead by hand in
the VQE spec; the new territory here is the circuit side, so scipy carries the optimizer
load and effort goes into §2-4 instead.

---

## 6. p sweep — does more rounds actually help?

Run p=1, then p=2, then (optionally) p=3 on the known 6-node graph. For p=1 on
**3-regular graphs specifically**, Farhi et al.'s original QAOA paper proves an
approximation ratio of at least **0.6924** — a real number to grade against, not just
"the histogram looks better." (Note: the lab's 6-node graph isn't 3-regular, so that
exact bound won't apply there — it becomes relevant once §7 moves to 3-regular
instances.) Reproduce the lab's qualitative finding (higher p → better optimum) but via
the optimizer converging, not a full `(γ,β)` grid scan.

---

## 7. Scaling — same code, bigger graphs

Generate random 3-regular graphs (regular = every node has exactly 3 edges — the
standard QAOA benchmark family, and the one the 0.6924 bound applies to) at increasing
`n`, stepping up toward **~20 nodes**. Track:

- approximation ratio (`F_opt / C_max` from brute force) as `n` grows
- optimizer wall-clock / iteration count as `n` grows

Still simulation-only (`n=20` → `2^20 ≈ 1M` statevector entries, entirely tractable).
This is where "extensible to 20 qubits" gets exercised — nothing about §2-6 should need
to change, only the input graph.

---

## 8. Hardware run — gated, not automatic

Once the sim pipeline is validated end-to-end, run **one** small instance (matching the
6-node graph or a modest 3-regular instance — not the full n=20 scaling sweep) on IBM
Heron via Sampler, and compare the measured top bitstrings against sim.

**Before submitting anything:** check the current IBM budget ledger, estimate the QPU
seconds this will cost, and get explicit go-ahead — per the standing QPU safety rule.
This step does not happen just because the sim code runs; it's a deliberate, separate
decision.

---

## Conventions to hold the line on

- Graph (`n`, edge list) is a function argument from `§3` onward — never hardcoded —
  so `§7`'s scaling is a change of input, not a rewrite.
- One `expectation_value(...)` function shared by every optimizer call, `method ∈
  {"exact","measured"}`, mirroring the VQE build.
- scipy for the optimizer (already understood from VQE); hand-write the circuit-building
  and cost-function logic ourselves — that's the actual new material here.
- Every experiment states its known answer (brute force here, no closed form like VQE's
  `-‖c‖`) and checks against it.
- No hardware job runs without an explicit go-ahead and a stated estimated cost first.
