# QAOA / MaxCut — reference pages

**→ [Open the styled hub page](https://imelenwe.github.io/quantum-machine-learning/QAOA/learn/index.html)** to browse all four pages with descriptions and a suggested order.

Or use the plain list below: four standalone pages, meant to be read in this
order the first time — each one assumes the one before it. Links go to the
live rendered versions (GitHub shows raw `.html` files as source code, not as
pages, so use these rather than the files above).

## Part 1 — Understanding the problem itself

1. **[MaxCut, by hand](https://imelenwe.github.io/quantum-machine-learning/QAOA/learn/maxcut_demo.html)** — interactive puzzle. Click nodes, cut edges, try to beat 7. Builds the core intuition before any of the theory.
2. **[MaxCut: Three Real Representations](https://imelenwe.github.io/quantum-machine-learning/QAOA/learn/maxcut_explanations.html)** — reference, with diagrams. The same coloring game wearing three different costumes: magnets that want to disagree, party guests split from their rivals, and WiFi routers avoiding shared channels.

## Part 2 — How QAOA actually solves it

3. **[QAOA Theory: From Magnets to Circuit](https://imelenwe.github.io/quantum-machine-learning/QAOA/learn/qaoa_theory.html)** — theory, with diagrams. The full chain: Ising energy → MaxCut cost → the exponentiated cost operator → why QAOA alternates a "hard" step with an "easy" one, p times.
4. **[The Phase Gate Proof, From the Exponential Down](https://imelenwe.github.io/quantum-machine-learning/QAOA/learn/qaoa_C(x)_to_operator_proof.html)** — proof, with diagrams. Six baby steps deriving the CNOT–Rz–CNOT gate directly from exponentiating the cost operator — circuit diagram, phase wheels, matrix included.

---

The actual code lives in `QAOA_MaxCut.ipynb`, and the build plan in
`QAOA_SPEC.md`, one folder up in [`QAOA/`](..).
