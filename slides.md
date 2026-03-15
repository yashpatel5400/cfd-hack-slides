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

<div class="mt-2 space-y-3 text-sm">
<div class="flex gap-3 items-start">
<span class="text-cyan-400 font-mono font-bold shrink-0">01</span>
<div><strong>Momentum predictor</strong> — convection-diffusion solve for <strong>u</strong>*</div>
</div>
<div class="flex gap-3 items-start">
<span class="text-cyan-400 font-mono font-bold shrink-0">02</span>
<div><strong>Pressure Poisson solve</strong> — enforce ∇ · <strong>u</strong> = 0</div>
</div>
<div class="flex gap-3 items-start">
<span class="text-cyan-400 font-mono font-bold shrink-0">03</span>
<div><strong>Velocity correction</strong> — project onto divergence-free space</div>
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

# The Routing Loop

<div class="flex justify-center">
<div class="relative" style="width: 780px; height: 310px;">

<svg viewBox="0 0 780 310" style="position:absolute;inset:0;width:100%;height:100%">
  <defs>
    <marker id="arrowC" markerWidth="8" markerHeight="8" refX="7" refY="4" orient="auto">
      <path d="M0,0 L8,4 L0,8 Z" fill="#22d3ee" opacity="0.4"/>
    </marker>
    <marker id="arrowW" markerWidth="8" markerHeight="8" refX="7" refY="4" orient="auto">
      <path d="M0,0 L8,4 L0,8 Z" fill="#fff" opacity="0.5"/>
    </marker>
    <marker id="arrowG" markerWidth="8" markerHeight="8" refX="7" refY="4" orient="auto">
      <path d="M0,0 L8,4 L0,8 Z" fill="#4ade80" opacity="0.7"/>
    </marker>
  </defs>
  <line x1="118" y1="105" x2="222" y2="105" stroke="#fff" stroke-width="1.5" opacity="0.3" marker-end="url(#arrowW)"/>
  <line x1="348" y1="105" x2="452" y2="105" stroke="#fff" stroke-width="1.5" opacity="0.3" marker-end="url(#arrowW)"/>
  <line x1="578" y1="105" x2="662" y2="105" stroke="#fff" stroke-width="1.5" opacity="0.15" stroke-dasharray="6,4" marker-end="url(#arrowW)"/>
  <path d="M 580,125 Q 620,250 390,265 Q 160,280 100,135" fill="none" stroke="#22d3ee" stroke-width="1.5" opacity="0.25" stroke-dasharray="6,4" marker-end="url(#arrowC)"/>
  <path d="M 390,220 L 390,170" stroke="#4ade80" stroke-width="1.5" opacity="0.5" marker-end="url(#arrowG)"/>
</svg>

<div class="absolute flex flex-col items-center" style="left: 40px; top: 76px;">
  <div class="px-4 py-2 rounded-lg bg-white/8 border border-white/15 text-center">
    <div class="font-mono text-base text-white">u<sup>(t)</sup></div>
    <div class="text-[10px] opacity-40 mt-1">current solution</div>
  </div>
</div>

<div class="absolute flex flex-col items-center" style="left: 215px; top: 50px;">
  <div class="text-[10px] tracking-widest uppercase opacity-30 mb-1">router picks k*</div>
  <div class="relative">
    <div class="w-28 h-24 rounded-xl bg-white/5 border border-cyan-400/30 flex items-center justify-center">
      <div class="text-center">
        <div class="text-[10px] tracking-widest uppercase text-cyan-400 opacity-70">action</div>
        <div class="text-xs mt-1 font-mono opacity-80">argmin<sub>k</sub> err</div>
      </div>
    </div>
    <div class="absolute text-[9px] font-mono" style="right: -80px; top: -4px;">
      <div class="px-2 py-0.5 rounded bg-amber-500/15 text-amber-300 border border-amber-400/20">SOR(1.0)</div>
    </div>
    <div class="absolute text-[9px] font-mono" style="right: -80px; top: 20px;">
      <div class="px-2 py-0.5 rounded bg-orange-500/15 text-orange-300 border border-orange-400/20">SOR(1.3)</div>
    </div>
    <div class="absolute text-[9px] font-mono" style="right: -80px; top: 44px;">
      <div class="px-2 py-0.5 rounded bg-red-500/15 text-red-300 border border-red-400/20">SOR(1.6)</div>
    </div>
    <div class="absolute text-[9px] font-mono" style="right: -80px; top: 68px;">
      <div class="px-2 py-0.5 rounded bg-cyan-500/15 text-cyan-300 border border-cyan-400/30">FNO ✦</div>
    </div>
  </div>
</div>

<div class="absolute flex flex-col items-center" style="left: 460px; top: 76px;">
  <div class="px-4 py-2 rounded-lg bg-white/8 border border-white/15 text-center">
    <div class="font-mono text-base text-white">u<sup>(t+1)</sup></div>
    <div class="text-[10px] opacity-40 mt-1">updated solution</div>
  </div>
</div>

<div class="absolute font-mono text-xl tracking-[0.4em] opacity-20" style="left: 620px; top: 93px;">···</div>

<div class="absolute" style="left: 670px; top: 78px;">
  <div class="px-3 py-2 rounded-lg bg-emerald-500/10 border border-emerald-400/20 text-center">
    <div class="font-mono text-sm text-emerald-400">u*</div>
    <div class="text-[10px] opacity-40 mt-0.5">converged</div>
  </div>
</div>

<div class="absolute" style="left: 250px; top: 225px;">
  <div class="px-5 py-2 rounded-lg bg-emerald-500/8 border border-emerald-400/20">
    <div class="text-[10px] tracking-widest uppercase text-emerald-400 opacity-70 mb-1">Unrolled training</div>
    <div class="text-xs opacity-70">FNO sees <span class="text-cyan-300 font-mono">real in-loop residuals</span></div>
    <div class="text-xs opacity-50 mt-0.5">∇<sub>θ</sub> Σ<sub>t</sub> ‖FNO(r<sup>(t)</sup>) − correction*‖²</div>
  </div>
</div>

<div class="absolute text-[10px] text-cyan-400 opacity-40 italic" style="left: 120px; top: 270px;">repeat until ‖r‖ &lt; ε</div>

</div>
</div>

<div class="grid grid-cols-2 gap-8 text-xs opacity-70">
<div>

**Inference:** At each iteration the router evaluates all K+1 candidates and picks the one minimising immediate error.

</div>
<div>

**Training:** The FNO is fine-tuned through unrolled trajectories so it learns to correct the residuals that *actually arise* mid-solve — not random i.i.d. residuals.

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
