# VQE from scratch — concept spec

A clean re-derivation of the lab's Variational Quantum Eigensolver. Goal: understand
the *essence* of VQE and the two classical optimizers (gradient descent, Nelder–Mead),
and implement both optimizers by hand. You write the code; this file is the map.

---

## Status (last updated 2026-06-18)

Implemented and verified in `VQE.ipynb`:

- ✅ `single_qubit_ansatz` — `Ry(θ)·Rz(φ)`, statevector-checked.
- ✅ `expectation_value(params, ansatz, H, method, shots)` — both `"exact"`
  (hand inner product `⟨ψ|P|ψ⟩`) and `"measure"` (basis rotation + parity rule).
  The two agree to shot-noise on a known point.
- ✅ `gradient_descent(init_params, cost_fn, step, eps, max_iters)` — symmetric
  finite-difference gradient, returns `(params, energy, trajectory)`, internal
  convergence threshold (`1e-6`) + `max_iters` guardrail.
- ✅ §6 exact-vs-measure experiment for GD (see updated findings below).

Still to do: Nelder–Mead (§5b), the GD-vs-NM comparison on the measure landscape,
and the 2-qubit capstone (§2b, §8.6).

---

## 0. The one-sentence idea

> Pick a knob-tunable quantum state, measure its energy, and let a classical optimizer
> twist the knobs until the energy bottoms out. That floor is an estimate of the
> ground-state energy.

Everything below is just the four machine parts that make that sentence runnable:
the **state** (ansatz), the **energy** (Hamiltonian + measurement), the **optimizer**,
and the **principle** that says the floor is meaningful.

---

## 1. The variational principle (why any of this works)

For a Hamiltonian `H` with ground-state energy `E₀`, **any** normalized state `|ψ⟩`
obeys

```
⟨ψ| H |ψ⟩ ≥ E₀
```

Equality only when `|ψ⟩` is the ground state. So if we parameterize a family of states
`|ψ(θ)⟩` and minimize the expectation value over `θ`, we can only ever approach `E₀`
from above — never undershoot it. That makes the minimization *safe*: a lower number is
always a better (or equal) answer, never a lie.

**The VQE loop:**

```
   θ ──► |ψ(θ)⟩ ──► measure ⟨H⟩(θ) ──► classical optimizer ──► new θ
   ▲                                                              │
   └──────────────────────────────────────────────────────────────┘
```

Quantum computer does the middle (prepare + measure). Classical computer does the
optimization. That split is the whole point of "variational."

---

## 2. The ansatz (the tunable state)

An ansatz is a parameterized circuit `U(θ)` applied to `|0…0⟩`. Two requirements:
it must be (a) cheap to run and (b) *expressive* enough to reach (or nearly reach) the
true ground state.

### 2a. Single qubit — 2 parameters

```
|ψ(θ, φ)⟩ = Rz(φ) · Ry(θ) · |0⟩
```

`Ry(θ)` sets the polar angle on the Bloch sphere (how much |0⟩ vs |1⟩), `Rz(φ)` sets
the azimuthal phase. Together they reach **every** single-qubit pure state — fully
expressive for 1 qubit. This is the workhorse for the core lesson.

### 2b. Two qubits — 8 parameters (the capstone)

A product of single-qubit rotations can only build *unentangled* states. The ground
state of a generic 2-qubit `H` is entangled, so we need an entangler:

```
single_qubit_ansatz(q0, θ0, φ0)      # rotate each qubit
single_qubit_ansatz(q1, θ1, φ1)
CNOT(q0 → q1)                         # entangle
single_qubit_ansatz(q0, θ4, φ4)      # rotate again
single_qubit_ansatz(q1, θ6, φ6)
```

8 parameters. The CNOT sandwiched between rotation layers is what lets the state span
entangled corners of the 2-qubit Hilbert space. (The lab notes 8 params already gets
"very close" to exact — that's the lesson: a *little* entanglement structure goes far.)

---

## 3. The Hamiltonian as a sum of Pauli strings

Represent `H` as a dict mapping Pauli strings → real coefficients:

```python
H = {'X': 3, 'Y': -2, 'Z': 1}        # 1-qubit:  H = 3X − 2Y + 1Z
H = {'XY': 3, 'ZZ': -2}              # 2-qubit:  H = 3·(X⊗Y) − 2·(Z⊗Z)
```

Why this form? Because expectation is **linear**:

```
⟨H⟩ = Σ_k  c_k · ⟨Pauli_string_k⟩
```

So we never measure `H` directly — we measure each Pauli term's expectation and take the
weighted sum. Each term is handled by the same routine in §4.

---

## 4. Measuring ⟨Pauli⟩ — the one genuinely quantum trick

Hardware can only measure in the **Z (computational) basis**. So to measure `⟨X⟩` or
`⟨Y⟩`, rotate the axis we care about onto Z *before* measuring.

### 4a. Basis-change rotations (apply after the ansatz, before measurement)

| Pauli | rotation to apply | reason |
|-------|-------------------|--------|
| `Z`   | none              | already in Z basis |
| `X`   | `Ry(−π/2)`        | maps X-eigenbasis → Z-eigenbasis |
| `Y`   | `Rx(+π/2)`        | maps Y-eigenbasis → Z-eigenbasis |

### 4b. From counts to expectation (parity rule)

After rotating, measure all qubits. For a measured bitstring, its contribution is the
**parity**: `+1` if it has an even number of `1`s, `−1` if odd.

```
⟨Pauli_string⟩ ≈ (1/shots) · Σ_bitstrings  (−1)^(number of 1s) · count(bitstring)
```

Single-qubit special case: `⟨Z⟩ = (N₀ − N₁) / shots`.

### 4c. The exact "wavefunction" method (no shots)

For comparison, compute `⟨H⟩` exactly from the statevector — no sampling, no noise.
Cleanest route: build the Pauli operator and evaluate `⟨ψ| P |ψ⟩` directly (e.g. apply
the Pauli to a copy of the state and take the inner product, or use Qiskit's
`SparsePauliOp` / `Statevector.expectation_value`). This is the *noise-free reference*
we'll need in §6.

> **Design note:** write `expectation_value(params, ansatz, H, method)` **once** with
> `method ∈ {"measure", "exact"}`, and have *both* optimizers call it. The lab
> duplicated this logic inside the gradient-descent function — don't.

---

## 5. The classical optimizers (implement both by hand)

Both minimize the same scalar function `f(θ) = ⟨H⟩(θ)`. They differ in *how* they
decide where to step.

### 5a. Gradient Descent (finite-difference)

No analytic gradient — estimate each partial derivative with a symmetric difference:

```
∂f/∂θ_i ≈ [ f(θ + ε·e_i) − f(θ − ε·e_i) ] / (2ε)
θ_new   = θ − step · ∇f
```

For 1 qubit that's 4 expectation-value evaluations per step (±ε on θ and on φ).
Loop until the step barely moves the energy. Parameters to expose: `ε` (probe size),
`step` (learning rate), max iterations, convergence tolerance.

**Conceptual weak spot (this is the lesson):** the gradient is a *difference of two
noisy numbers divided by a small ε*. Shot noise in `f` gets **amplified by 1/(2ε)**.
This is `eps`-dependent and is the heart of the §6 finding — keep it in mind.

### 5b. Nelder–Mead (derivative-free simplex)

Keeps a **simplex** of `N+1` points (3 for 1 qubit, 9 for 8 params) and crawls downhill
by reshaping it — no derivatives at all. Each iteration:

1. **Order** the vertices by `f`. Identify worst (`hi`), and the rest.
2. **Centroid** `x̄` = mean of all vertices *except* the worst.
3. **Reflect** the worst through the centroid: `x_r = x̄ + α(x̄ − x_hi)`, `α=1` (lab uses 2).
   - If `x_r` is *middling* (better than worst-of-rest, not better than best): **accept reflect**.
   - If `x_r` is the *new best*: try **expand** further `x_e = x̄ + γ(x_r − x̄)`; keep whichever of `x_e`, `x_r` is lower.
   - If `x_r` is still *bad*: **contract** toward the better of `x_r`/`x_hi` by `ρ=0.5`. If contraction helps, accept it; else **shrink** the whole simplex toward the best vertex by `σ=0.5`.

Terminate when the best value stops improving by more than `δ` for several iterations in
a row.

**Why it's here:** no derivatives → no `1/ε` noise amplification → far more robust to
shot noise than GD. It's slower per "real" gradient, but it actually converges on the
measured landscape.

> **Cleanup vs. lab:** the lab hand-rolled `Calculate_MinMax`, `Compute_Centroid`,
> `Reflection_Point`. Use `np.argmin`/`np.argmax`, `np.mean(axis=0)`, and plain vector
> arithmetic (`xbar + alpha*(xbar - x_hi)`). Keep the *algorithm* explicit and readable;
> let numpy do the bookkeeping.

---

## 6. The punchline experiment (do not skip — it's the real lesson)

Run the **same** 1-qubit problem `H = {'X':3,'Y':-2,'Z':1}` and watch how GD behaves on
the exact vs measured landscape, sweeping `eps`. **What we actually found (be honest in
the writeup — it's more interesting than "GD fails"):**

| Setting | Result |
|---------|--------|
| `exact`, `eps=1e-6` | converges to E₀ in ~20 iters, error `0.0` |
| `measure`, `eps=1e-6` | **random walk — no descent at all** (see below) |
| `measure`, `eps ∈ [0.05, 1.0]` | descends to ≈ E₀; this toy is forgiving |

**The `eps=1e-6` failure is dramatic and worth showing.** At a fixed point the true
gradient is `~[-2.5, -1.6]` (steps of ~0.2 rad). The *measured* gradient with `eps=1e-6`
comes out as `[1300, 21300]`, `[-12600, -34300]`, … — random sign, magnitude in the
**thousands** — because shot noise `~0.01` divided by `2·1e-6` ≈ 5000. Multiplied by the
step size, each "step" jumps **hundreds of radians**: the parameters teleport randomly
every iteration. There is no descent; the trajectory is a random walk.

**But — and this is the honest catch — with a sane `eps` (0.05–1.0) GD on this toy
converges fine.** So the lesson is *not* "GD can't optimize noisy functions." It's:

- **You can't tune `eps` without knowing the answer.** We only know 0.05–1.0 is the safe
  band because we have E₀ to check against. In a real problem you don't; too-small `eps`
  silently degenerates into the random walk above.
- **GD never self-terminates under noise.** The `|Δenergy| < tol` check can't fire when
  the energy jitters by ~0.02 forever, so GD always burns `max_iters`.
- **GD's reported "answer" is a single noisy evaluation.** It can even land *below* E₀
  (we saw `-3.7628 < -√14`), which is physically impossible — a tell that the number is
  shot noise, not a converged value. The variational principle (§1) is a free noise
  detector here.

**Conclusion:** on this easy 1-qubit landscape GD is adequate once `eps` is tuned.
Nelder–Mead earns its place by removing the fragile parts — no `eps` to guess, a real
convergence signal (the simplex physically shrinks), and fewer evaluations per step.
Those advantages are mild here and become decisive on harder/larger landscapes, which is
why noisy quantum optimization tends to reach for derivative-free methods (Nelder–Mead,
COBYLA, SPSA) over vanilla finite-difference GD.

---

## 7. The exact answer to check against

For a 1-qubit `H = Σ c_k P_k`, the ground-state energy is `−‖c‖₂`:

```
E₀ = −√(cx² + cy² + cz²)
```

For `{'X':3,'Y':-2,'Z':1}` that's `−√14 ≈ −3.742`. Use this to *grade* both optimizers.
For the 2-qubit capstone, get `E₀` as the smallest eigenvalue of the assembled matrix
(e.g. `np.linalg.eigvalsh` on the `SparsePauliOp.to_matrix()`).

---

## 8. Build order (you write each step)

1. ✅ `single_qubit_ansatz(qc, qubit, params)` — `Ry`, `Rz`. Statevector sanity-checked.
2. ✅ `expectation_value(params, ansatz, H, method)` — both `"measure"` and `"exact"`,
   verified to agree within shot noise.
3. ✅ `gradient_descent(...)` — returns `(params, energy, trajectory)`; convergence plot
   on the exact landscape lands on E₀.
4. ⬜ `nelder_mead(...)` — same return shape (params, energy, trajectory).
5. 🔶 **§6 experiment:** GD exact-vs-measure done (findings above). NM half pending —
   add GD-vs-NM on the measure landscape once NM exists.
6. ⬜ `two_qubit_ansatz(qc, q, params)` + run an optimizer on `{'XY':3,'ZZ':-2}`; compare
   to the exact eigenvalue.

**Conventions to hold the line on:** one expectation function shared by both optimizers;
numpy for vector bookkeeping; every optimizer returns its trajectory; each section states
its known-exact answer and checks against it.
