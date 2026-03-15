---
theme: default
title: "Greedy Multi-Solver Routing for PDE Linear Systems"
info: |
  Adaptive solver selection for the linear subproblems arising in incompressible CFD.
class: text-center
drawings:
  persist: false
transition: slide-left
mdc: true
---

<div class="h-full flex flex-col items-center justify-center">
<div class="text-5xl font-bold tracking-tight leading-tight">
Greedy Multi-Solver Routing<br>for PDE Linear Systems
</div>

<div class="mt-6 text-lg opacity-50 tracking-widest uppercase">
Adaptive solver selection for incompressible CFD
</div>

<div class="abs-b mb-10 text-sm tracking-widest uppercase opacity-30">
SemiAnalysis x Fluidstack Hackathon
</div>
</div>

---

# Motivation: Verification Is the Bottleneck

<div class="grid grid-cols-2 gap-10 mt-6">
<div>

<div class="text-xs tracking-widest uppercase opacity-40 mb-3">AI-Driven Engineering Design</div>

AI workflows for engineering design are rapidly maturing — generative models can propose geometries, materials, and configurations at unprecedented speed.

But **every AI-generated design must be verified** through physics simulation before it can be trusted.

<div class="mt-5 pl-4 border-l-2 border-red-400 text-sm opacity-80">

As generation gets faster, **verification (CFD/FEA simulation) becomes the dominant bottleneck** in the design loop.

</div>

</div>
<div>

<div class="text-xs tracking-widest uppercase opacity-40 mb-3">The ML Surrogate Dilemma</div>

ML surrogates (neural operators, GNNs) can approximate PDE solutions **orders of magnitude faster** than classical solvers.

But they offer **no convergence guarantees** — predictions may look plausible while being quantitatively wrong.

<div class="mt-5 pl-4 border-l-2 border-cyan-400 text-sm opacity-80">

**Our approach:** Route between classical methods (with guarantees) and ML surrogates (with speed). Use ML when it helps, fall back to classical when it doesn't, and **always converge**.

</div>

</div>
</div>

---

# Incompressible Navier–Stokes

<div class="grid grid-cols-2 gap-12 mt-6">
<div>

The governing equations for incompressible flow:

$$
\frac{\partial \mathbf{u}}{\partial t} + (\mathbf{u} \cdot \nabla)\mathbf{u} = -\frac{1}{\rho}\nabla p + \nu \nabla^2 \mathbf{u} + \mathbf{f}
$$

$$
\nabla \cdot \mathbf{u} = 0
$$

<div class="mt-4 text-sm opacity-70">

**Key challenge:** velocity and pressure are coupled; pressure has no independent evolution equation.

</div>

</div>
<div>

<div class="text-xs tracking-widest uppercase opacity-40 mb-4">Splitting Methods</div>

Projection / SIMPLE / PISO all split this into:

<div class="mt-2 space-y-3">
<div class="flex gap-3 items-start">
<span class="text-cyan-400 font-mono font-bold">01</span>
<span>**Momentum predictor** — convection-diffusion solve for $\mathbf{u}^*$</span>
</div>
<div class="flex gap-3 items-start">
<span class="text-cyan-400 font-mono font-bold">02</span>
<span>**Pressure Poisson solve** — enforce $\nabla \cdot \mathbf{u} = 0$</span>
</div>
<div class="flex gap-3 items-start">
<span class="text-cyan-400 font-mono font-bold">03</span>
<span>**Velocity correction** — project onto divergence-free space</span>
</div>
</div>

<div class="mt-5 text-sm opacity-70">

The pressure Poisson solve is often the **dominant computational bottleneck**.

</div>

</div>
</div>

---

# The Two Core Subproblems

<div class="grid grid-cols-2 gap-8 mt-6">
<div class="rounded-lg p-5 bg-white/5 border border-white/10">

<div class="text-cyan-400 text-xs tracking-widest uppercase mb-3">Momentum Predictor</div>

**Convection-Diffusion Equation**

$$-\nu \nabla^2 u_i + \mathbf{u} \cdot \nabla u_i = \text{source}$$

<div class="mt-4 space-y-1 text-sm opacity-80">

- Advances velocity in time
- Linearized using lagged velocity
- One solve per velocity component

</div>

</div>
<div class="rounded-lg p-5 bg-white/5 border border-white/10">

<div class="text-cyan-400 text-xs tracking-widest uppercase mb-3">Pressure Correction</div>

**Poisson Equation**

$$\nabla^2 p^{n+1} = \frac{\rho}{\Delta t} \nabla \cdot \mathbf{u}^*$$

<div class="mt-4 space-y-1 text-sm opacity-80">

- Elliptic, globally coupled
- Enforces incompressibility
- Often the main bottleneck

</div>

</div>
</div>

<div class="mt-8 text-center opacity-60">

Both subproblems are solved iteratively — can we accelerate them with learned routing?

</div>

---

# Iterative Solvers: The Landscape

<div class="mt-4 text-sm opacity-80">

Each linear subproblem $A\mathbf{u} = \mathbf{b}$ is solved by repeated iteration:

</div>

<div class="mt-6">

| Solver | Per-step cost | Convergence | Strengths |
|--------|:---:|:---:|-----------|
| Jacobi($\omega$) | $O(N^2)$ | Slow | Parallelizable, tunable damping |
| Gauss-Seidel | $O(N^2)$ | ~2× Jacobi | Better for low-freq error |
| SOR($\omega$) | $O(N^2)$ | Tunable | Optimal $\omega$ can be much faster |
| Multigrid | $O(N^2)$ | $O(1)$ iters | Optimal, but complex |
| **ML Surrogate** | $O(N^2)$ | One-shot | Large initial correction |

</div>

<div class="mt-6 pl-4 border-l-2 border-cyan-400 text-sm opacity-80">

**Key insight:** Different solvers damp different spectral modes of the error at different rates. No single solver is optimal at every stage of convergence.

</div>

---

# Greedy Multi-Solver Routing

<div class="grid grid-cols-2 gap-10 mt-4">
<div>

<div class="text-xs tracking-widest uppercase opacity-40 mb-3">The Idea</div>

At each iteration, **choose the solver** that gives the best immediate error reduction:

$$k^* = \arg\min_{k \in \{1,\ldots,K\}} \| u_k^{(t+1)} - u_{\text{true}} \|_2$$

<div class="mt-5 rounded-lg p-4 bg-white/5 border border-white/10 text-sm">

**Solver portfolio:**

<div class="mt-2 font-mono text-xs space-y-1">

- `SOR(ω=1.0)` — Gauss-Seidel
- `SOR(ω=1.3)` — moderate over-relaxation
- `SOR(ω=1.6)` — aggressive over-relaxation
- `DeepONet` — ML correction

</div>
</div>

</div>
<div>

<div class="text-xs tracking-widest uppercase opacity-40 mb-3">Why It Works</div>

Each SOR variant has a different spectral damping profile:

<img src="./images/eigenvalue_comparison.png" class="mt-2 rounded-lg border border-white/10" />

<div class="text-xs mt-2 opacity-50">
25% of modes have |λ<sub>Jacobi(0.67)</sub>| &lt; |λ<sub>GS</sub>|: different solvers excel on different modes.
</div>

</div>
</div>

---
class: text-center
---

<div class="h-full flex flex-col items-center justify-center">
<div class="text-xs tracking-widest uppercase opacity-30 mb-4">Results</div>
<div class="text-4xl font-bold tracking-tight">2D Poisson Equation</div>
<div class="mt-4 text-lg opacity-50">Oracle greedy with SOR portfolio — K = 4 solvers</div>
</div>

---

# 2D Poisson: Convergence

<img src="./images/poisson_sor_convergence.png" class="w-full rounded-lg border border-white/10" />

<div class="grid grid-cols-3 gap-6 mt-5 text-sm">
<div class="text-center p-3 rounded-lg bg-white/5 border border-white/10">

**SOR(1.0) Only**
<br><span class="font-mono text-xs">Final L2: 2.17 × 10⁻⁵</span>
<br><span class="font-mono text-xs">AUC: 0.485</span>

</div>
<div class="text-center p-3 rounded-lg bg-white/5 border border-white/10">

**HINTS**
<br><span class="font-mono text-xs">Final L2: 4.61 × 10⁻⁴</span>
<br><span class="font-mono text-xs">AUC: 0.377</span>

</div>
<div class="text-center p-3 rounded-lg bg-white/5 border border-cyan-400/40">

<span class="text-cyan-400">**Greedy (Oracle)**</span>
<br><span class="font-mono text-xs">Final L2: 2.45 × 10⁻⁷</span>
<br><span class="font-mono text-xs text-cyan-400">AUC: 0.115 — 4.2× better</span>

</div>
</div>

---

# 2D Poisson: Routing Pattern

<img src="./images/poisson_sor_routing.png" class="w-full rounded-lg border border-white/10" />

<div class="mt-4 text-sm opacity-80 space-y-2">

- **SOR(1.6)** dominates (~74%) — fastest for high-frequency error early on
- **SOR(1.3)** used ~25% throughout — better for certain mid-frequency modes
- **DeepONet** used sparingly (0.3%) — one-shot correction in first few iterations

</div>

---
class: text-center
---

<div class="h-full flex flex-col items-center justify-center">
<div class="text-xs tracking-widest uppercase opacity-30 mb-4">Results</div>
<div class="text-4xl font-bold tracking-tight">2D Convection-Diffusion</div>
<div class="mt-4 text-lg opacity-50">Oracle greedy with SOR portfolio — the more interesting case</div>
</div>

---

# 2D ConvDiff: Convergence

<img src="./images/convdiff_sor_convergence.png" class="w-full rounded-lg border border-white/10" />

<div class="grid grid-cols-3 gap-6 mt-5 text-sm">
<div class="text-center p-3 rounded-lg bg-white/5 border border-white/10">

**SOR(1.0) Only**
<br><span class="font-mono text-xs">Final L2: 1.84 × 10⁻⁸</span>
<br><span class="font-mono text-xs">AUC: 0.091</span>

</div>
<div class="text-center p-3 rounded-lg bg-white/5 border border-white/10">

**HINTS**
<br><span class="font-mono text-xs">Final L2: 1.03 × 10⁻⁴</span>
<br><span class="font-mono text-xs">AUC: 0.101</span>

</div>
<div class="text-center p-3 rounded-lg bg-white/5 border border-cyan-400/40">

<span class="text-cyan-400">**Greedy (Oracle)**</span>
<br><span class="font-mono text-xs">Final L2: 1.41 × 10⁻⁸</span>
<br><span class="font-mono text-xs text-cyan-400">AUC: 0.019 — 4.8× better</span>

</div>
</div>

---

# 2D ConvDiff: Routing Pattern

<img src="./images/convdiff_sor_routing.png" class="w-full rounded-lg border border-white/10" />

<div class="mt-4 text-sm opacity-80 space-y-2">

- Clear **phase transition** at iteration ~150–200
- Early: SOR(1.6) dominates (high-freq damping)
- Late: **SOR(1.0) rises to ~60%** as low-frequency modes dominate the residual
- The optimal relaxation parameter **shifts during convergence** — routing captures this

</div>

---

# Results Summary

<div class="mt-6">

| | **SOR(1.0) Only** | **HINTS** | **Oracle Greedy** | **vs. Best Baseline** |
|---|---|---|---|---|
| **Poisson — Final L2** | 2.17 × 10⁻⁵ | 4.61 × 10⁻⁴ | **2.45 × 10⁻⁷** | 88× lower |
| **Poisson — AUC** | 0.485 | 0.377 | **0.115** | 4.2× lower |
| **ConvDiff — Final L2** | 1.84 × 10⁻⁸ | 1.03 × 10⁻⁴ | **1.41 × 10⁻⁸** | 1.3× lower |
| **ConvDiff — AUC** | 0.091 | 0.101 | **0.019** | 4.8× lower |

</div>

<div class="mt-8 pl-4 border-l-2 border-cyan-400 opacity-80">

**Key finding:** Multi-solver greedy routing achieves **4–5× lower AUC** than any single solver or fixed schedule (HINTS). The gains come from adapting the solver choice to the current error spectrum.

</div>

---

# K=2 Baseline: Learned Router Also Works

<div class="text-sm opacity-70 mt-1">Prior results with Jacobi + DeepONet (K=2) and a learned LSTM router</div>

<div class="grid grid-cols-2 gap-8 mt-5 text-sm">
<div class="rounded-lg p-4 bg-white/5 border border-white/10">

<div class="text-cyan-400 text-xs tracking-widest uppercase mb-3">2D Poisson (K=2)</div>

| Strategy | Final L2 | AUC |
|---|---|---|
| Jacobi Only | 4.37 × 10⁻⁴ | 0.919 |
| HINTS | 7.29 × 10⁻⁴ | 0.523 |
| Oracle Greedy | 4.41 × 10⁻⁵ | 0.192 |
| **LSTM Router** | **1.05 × 10⁻⁴** | **0.329** |

<div class="mt-3 text-cyan-400 font-mono text-xs">LSTM Router: 2.8× better AUC than baseline</div>

</div>
<div class="rounded-lg p-4 bg-white/5 border border-white/10">

<div class="text-cyan-400 text-xs tracking-widest uppercase mb-3">2D ConvDiff (K=2)</div>

| Strategy | Final L2 | AUC |
|---|---|---|
| Jacobi Only | 1.55 × 10⁻⁴ | 0.337 |
| HINTS | 1.80 × 10⁻⁴ | 0.186 |
| Oracle Greedy | 1.29 × 10⁻⁵ | 0.077 |
| **LSTM Router** | **4.20 × 10⁻⁵** | **0.136** |

<div class="mt-3 text-cyan-400 font-mono text-xs">LSTM Router: 2.5× better AUC than baseline</div>

</div>
</div>

---

# Why Multi-Solver Routing?

<div class="grid grid-cols-2 gap-12 mt-6">
<div>

<div class="text-xs tracking-widest uppercase opacity-40 mb-3">Spectral Intuition</div>

Different solvers have different **eigenvalue amplification profiles**.

As convergence progresses:

<div class="mt-3 space-y-3">
<div class="flex gap-3 items-start">
<span class="text-cyan-400 font-mono font-bold text-xs">01</span>
<span class="text-sm">High-freq error dies fast → all solvers good</span>
</div>
<div class="flex gap-3 items-start">
<span class="text-cyan-400 font-mono font-bold text-xs">02</span>
<span class="text-sm">Mid-freq error remains → SOR(1.6) excels</span>
</div>
<div class="flex gap-3 items-start">
<span class="text-cyan-400 font-mono font-bold text-xs">03</span>
<span class="text-sm">Low-freq error dominates → SOR(1.0) wins</span>
</div>
</div>

<div class="mt-5 text-sm opacity-70">

The **optimal solver changes during convergence**.

</div>

</div>
<div>

<div class="text-xs tracking-widest uppercase opacity-40 mb-3">Connection to CFD</div>

Each CFD time step requires:

- Multiple Poisson solves (pressure)
- Multiple ConvDiff solves (momentum)

A greedy router that adapts per-iteration could **reduce total iteration count** across the entire simulation.

<div class="mt-5 rounded-lg p-4 bg-white/5 border border-white/10 text-sm">

The router is lightweight — <span class="font-mono text-cyan-400">~10K params</span> — compared to the solvers themselves.

</div>

</div>
</div>

---

# Next Steps

<div class="grid grid-cols-2 gap-12 mt-6">
<div>

<div class="text-xs tracking-widest uppercase opacity-40 mb-4">In Progress</div>

<div class="space-y-4 text-sm">

- **Improved ML surrogates** — larger DeepONet and FNO training in progress
- **Learned router for K=4 SOR portfolio** — LSTM training to replace oracle
- **Spectral analysis** of routing decisions

</div>

</div>
<div>

<div class="text-xs tracking-widest uppercase opacity-40 mb-4">Future Directions</div>

<div class="space-y-4 text-sm">

- **Multi-step lookahead** — RL-based planning instead of myopic greedy
- **Cyclic CFD integration** — stitch Poisson and ConvDiff routers in a projection-method loop
- **Variable coefficients** — per-sample optimal solvers for heterogeneous problems

</div>

</div>
</div>

<div class="mt-10 text-center">
<div class="pl-4 pr-4 py-3 border border-cyan-400/30 rounded-lg inline-block text-sm">

The key contribution: <span class="text-cyan-400">**adaptive, learned solver selection**</span> that exploits the complementary spectral properties of classical iterative methods and ML surrogates.

</div>
</div>

<style>
:root {
  --slidev-theme-primary: #22d3ee;
}
.slidev-layout {
  background: #000 !important;
  color: #e4e4e7 !important;
}
.slidev-layout h1 {
  color: #fff !important;
  font-weight: 700 !important;
  letter-spacing: -0.02em !important;
}
.slidev-layout h2, .slidev-layout h3 {
  color: #e4e4e7 !important;
}
.slidev-layout table {
  font-size: 0.8rem;
}
.slidev-layout th {
  background: rgba(255,255,255,0.05) !important;
  border-color: rgba(255,255,255,0.1) !important;
  color: #a1a1aa !important;
  font-weight: 500 !important;
  text-transform: uppercase;
  font-size: 0.7rem;
  letter-spacing: 0.05em;
}
.slidev-layout td {
  border-color: rgba(255,255,255,0.06) !important;
  color: #d4d4d8 !important;
}
.slidev-layout tr:hover td {
  background: rgba(255,255,255,0.03);
}
.slidev-layout strong {
  color: #fff;
}
.slidev-layout code {
  background: rgba(255,255,255,0.08) !important;
  color: #22d3ee !important;
  border: none !important;
}
.slidev-layout a {
  color: #22d3ee !important;
}
.katex {
  color: #e4e4e7 !important;
}
</style>
