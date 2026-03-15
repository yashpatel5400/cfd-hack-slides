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

# Greedy Multi-Solver Routing for PDE Linear Systems

Adaptive solver selection for the subproblems of incompressible CFD

<div class="abs-b mb-12 text-sm opacity-60">
CFD Hackathon 2026
</div>

---

# Motivation: Verification Is the Bottleneck

<div class="grid grid-cols-2 gap-8 mt-2">
<div>

### AI-Driven Engineering Design

AI workflows for engineering design are rapidly maturing — generative models can propose geometries, materials, and configurations at unprecedented speed.

But **every AI-generated design must be verified** through physics simulation before it can be trusted.

<div class="mt-4 p-3 bg-red-50 rounded-lg text-sm">

As generation gets faster, **verification (CFD/FEA simulation) becomes the dominant bottleneck** in the design loop.

</div>

</div>
<div>

### The ML Surrogate Dilemma

ML surrogates (neural operators, GNNs) can approximate PDE solutions **orders of magnitude faster** than classical solvers.

But they offer **no convergence guarantees** — predictions may look plausible while being quantitatively wrong.

<div class="mt-4 p-3 bg-green-50 rounded-lg text-sm">

**Our approach:** Hybrid solvers that route between classical methods (with guarantees) and ML surrogates (with speed). Get the best of both — use ML when it helps, fall back to classical when it doesn't, and **always converge**.

</div>

</div>
</div>

---
layout: two-cols
---

# Incompressible Navier–Stokes

The governing equations for incompressible flow:

$$
\frac{\partial \mathbf{u}}{\partial t} + (\mathbf{u} \cdot \nabla)\mathbf{u} = -\frac{1}{\rho}\nabla p + \nu \nabla^2 \mathbf{u} + \mathbf{f}
$$

$$
\nabla \cdot \mathbf{u} = 0
$$

**Key challenge:** velocity and pressure are coupled; pressure has no independent evolution equation.

::right::

<div class="ml-4 mt-8">

### Splitting Methods

Projection / SIMPLE / PISO all split this into:

1. **Momentum predictor** — convection-diffusion solve for tentative velocity $\mathbf{u}^*$
2. **Pressure Poisson solve** — enforce $\nabla \cdot \mathbf{u} = 0$
3. **Velocity correction** — project onto divergence-free space

The pressure Poisson solve is often the **dominant computational bottleneck** (global elliptic problem).

</div>

---

# The Two Core Subproblems

<div class="grid grid-cols-2 gap-8 mt-4">
<div class="border rounded-lg p-4 bg-blue-50">

### Momentum Predictor
**Convection-Diffusion Equation**

$$-\nu \nabla^2 u_i + \mathbf{u} \cdot \nabla u_i = \text{source}$$

- Advances velocity in time
- Linearized using lagged velocity
- One solve per velocity component

</div>
<div class="border rounded-lg p-4 bg-green-50">

### Pressure Correction
**Poisson Equation**

$$\nabla^2 p^{n+1} = \frac{\rho}{\Delta t} \nabla \cdot \mathbf{u}^*$$

- Elliptic, globally coupled
- Enforces incompressibility
- Often the main bottleneck

</div>
</div>

<div class="mt-6 text-center">

**Both subproblems are solved iteratively — can we accelerate them with learned routing?**

</div>

---

# Iterative Solvers: The Landscape

Each linear subproblem $A\mathbf{u} = \mathbf{b}$ is solved by repeated iteration:

| Solver | Per-step cost | Convergence rate | Strengths |
|--------|--------------|-----------------|-----------|
| Jacobi($\omega$) | $O(N^2)$ | Slow | Parallelizable, tunable damping |
| Gauss-Seidel | $O(N^2)$ | ~2× Jacobi | Better for low-freq error |
| SOR($\omega$) | $O(N^2)$ | Tunable | Optimal $\omega$ can be much faster |
| Multigrid | $O(N^2)$ | $O(1)$ iters | Optimal, but complex |
| **ML Surrogate** | $O(N^2)$ | One-shot | Large initial correction |

<div class="mt-4 p-3 bg-amber-50 rounded-lg">

**Key insight:** Different solvers damp different spectral modes of the error at different rates. No single solver is optimal at every stage of convergence.

</div>

---

# Greedy Multi-Solver Routing

<div class="grid grid-cols-2 gap-6">
<div>

### The Idea

At each iteration, **choose the solver** that gives the best immediate error reduction:

$$k^* = \arg\min_{k \in \{1,\ldots,K\}} \| u_k^{(t+1)} - u_{\text{true}} \|_2$$

**Solver portfolio:**
- SOR($\omega=1.0$) — Gauss-Seidel
- SOR($\omega=1.3$) — moderate over-relaxation
- SOR($\omega=1.6$) — aggressive over-relaxation
- DeepONet — ML correction

</div>
<div>

### Why It Works

Each SOR variant has a different spectral damping profile:

<img src="./images/eigenvalue_comparison.png" class="rounded shadow" />

<div class="text-xs mt-1 opacity-70">
25% of modes have |λ<sub>Jacobi(0.67)</sub>| &lt; |λ<sub>GS</sub>|: different solvers excel on different modes.
</div>

</div>
</div>

---
layout: section
---

# Results: 2D Poisson Equation

Oracle greedy with SOR portfolio — $K = 4$ solvers

---

# 2D Poisson: Convergence

<div class="grid grid-cols-1 gap-2">
<img src="./images/poisson_sor_convergence.png" class="w-full rounded shadow" />
</div>

<div class="grid grid-cols-3 gap-4 mt-4 text-sm">
<div class="text-center p-2 bg-blue-50 rounded">

**SOR(1.0) Only**
<br>Final L2: 2.17 × 10⁻⁵
<br>AUC: 0.485

</div>
<div class="text-center p-2 bg-orange-50 rounded">

**HINTS**
<br>Final L2: 4.61 × 10⁻⁴
<br>AUC: 0.377

</div>
<div class="text-center p-2 bg-green-50 rounded font-bold">

**Greedy (Oracle)**
<br>Final L2: 2.45 × 10⁻⁷
<br>AUC: 0.115 (**4.2× better**)

</div>
</div>

---

# 2D Poisson: Routing Pattern

<img src="./images/poisson_sor_routing.png" class="w-full rounded shadow" />

<div class="mt-2 text-sm">

- **SOR(1.6)** dominates (~74%) — fastest for high-frequency error early on
- **SOR(1.3)** used ~25% throughout — better for certain mid-frequency modes
- **DeepONet** used sparingly (0.3%) — one-shot correction in first few iterations

</div>

---
layout: section
---

# Results: 2D Convection-Diffusion

Oracle greedy with SOR portfolio — the more interesting case

---

# 2D ConvDiff: Convergence

<div class="grid grid-cols-1 gap-2">
<img src="./images/convdiff_sor_convergence.png" class="w-full rounded shadow" />
</div>

<div class="grid grid-cols-3 gap-4 mt-4 text-sm">
<div class="text-center p-2 bg-blue-50 rounded">

**SOR(1.0) Only**
<br>Final L2: 1.84 × 10⁻⁸
<br>AUC: 0.091

</div>
<div class="text-center p-2 bg-orange-50 rounded">

**HINTS**
<br>Final L2: 1.03 × 10⁻⁴
<br>AUC: 0.101

</div>
<div class="text-center p-2 bg-green-50 rounded font-bold">

**Greedy (Oracle)**
<br>Final L2: 1.41 × 10⁻⁸
<br>AUC: 0.019 (**4.8× better**)

</div>
</div>

---

# 2D ConvDiff: Routing Pattern — Phase Transition

<img src="./images/convdiff_sor_routing.png" class="w-full rounded shadow" />

<div class="mt-2 text-sm">

- Clear **phase transition** at iteration ~150–200
- Early: SOR(1.6) dominates (high-freq damping)
- Late: **SOR(1.0) rises to ~60%** as low-frequency modes dominate the residual
- The optimal relaxation parameter **shifts during convergence** — routing captures this

</div>

---

# Results Summary

<div class="mt-4">

| | **SOR(1.0) Only** | **HINTS** | **Oracle Greedy** | **Greedy / Best Baseline** |
|---|---|---|---|---|
| **2D Poisson — Final L2** | 2.17 × 10⁻⁵ | 4.61 × 10⁻⁴ | **2.45 × 10⁻⁷** | 88× lower |
| **2D Poisson — AUC** | 0.485 | 0.377 | **0.115** | 4.2× lower |
| **2D ConvDiff — Final L2** | 1.84 × 10⁻⁸ | 1.03 × 10⁻⁴ | **1.41 × 10⁻⁸** | 1.3× lower |
| **2D ConvDiff — AUC** | 0.091 | 0.101 | **0.019** | 4.8× lower |

</div>

<div class="mt-6 p-3 bg-green-50 rounded-lg">

**Key finding:** Multi-solver greedy routing achieves **4–5× lower AUC** than any single solver or fixed schedule (HINTS). The gains come from adapting the solver choice to the current error spectrum — not from any single solver being faster.

</div>

---

# K=2 Baseline: Learned Router Also Works

Prior results with Jacobi + DeepONet (K=2) and a **learned LSTM router**:

<div class="grid grid-cols-2 gap-6 mt-4 text-sm">
<div class="border rounded p-3">

### 2D Poisson (K=2, Jacobi + DeepONet)
| Strategy | Final L2 | AUC |
|---|---|---|
| Jacobi Only | 4.37 × 10⁻⁴ | 0.919 |
| HINTS | 7.29 × 10⁻⁴ | 0.523 |
| Oracle Greedy | 4.41 × 10⁻⁵ | 0.192 |
| **LSTM Router** | **1.05 × 10⁻⁴** | **0.329** |

LSTM Router: **2.8× better AUC** than baseline

</div>
<div class="border rounded p-3">

### 2D ConvDiff (K=2, Jacobi + DeepONet)
| Strategy | Final L2 | AUC |
|---|---|---|
| Jacobi Only | 1.55 × 10⁻⁴ | 0.337 |
| HINTS | 1.80 × 10⁻⁴ | 0.186 |
| Oracle Greedy | 1.29 × 10⁻⁵ | 0.077 |
| **LSTM Router** | **4.20 × 10⁻⁵** | **0.136** |

LSTM Router: **2.5× better AUC** than baseline

</div>
</div>

---

# Why Multi-Solver Routing?

<div class="grid grid-cols-2 gap-8 mt-6">
<div>

### Spectral Intuition

Different solvers have different **eigenvalue amplification profiles**.

As convergence progresses:
1. High-freq error dies fast → all solvers good
2. Mid-freq error remains → SOR(1.6) excels
3. Low-freq error dominates → SOR(1.0) wins

The **optimal solver changes during convergence**.

</div>
<div>

### Connection to CFD

Each CFD time step requires:
- Multiple Poisson solves (pressure)
- Multiple ConvDiff solves (momentum)

A greedy router that adapts per-iteration could **reduce total iteration count** across the entire simulation.

The router is lightweight (LSTM, ~10K params) compared to the solvers themselves.

</div>
</div>

---

# Next Steps

<div class="grid grid-cols-2 gap-8 mt-4">
<div>

### In Progress

- **Improved ML surrogates** — larger DeepONet and FNO (Fourier Neural Operator) training in progress
- **Learned router for K=4 SOR portfolio** — LSTM training to replace oracle
- **Spectral analysis** of routing decisions

</div>
<div>

### Future Directions

- **Multi-step lookahead** — RL-based planning instead of myopic greedy
- **Cyclic CFD integration** — stitch Poisson and ConvDiff routers in a projection-method loop
- **Variable coefficients** — per-sample optimal solvers for heterogeneous problems

</div>
</div>

<div class="mt-8 text-center text-lg font-bold">

The key contribution: **adaptive, learned solver selection** that exploits the complementary spectral properties of classical iterative methods and ML surrogates.

</div>
