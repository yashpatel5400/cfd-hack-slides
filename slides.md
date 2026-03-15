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

<div class="mt-4 text-lg opacity-50 tracking-widest uppercase">
Adaptive solver selection for incompressible CFD
</div>

<img src="./images/cfd_airflow.png" class="mt-6 w-160 h-32 object-cover rounded-lg opacity-60" />

<div class="abs-b mb-10 text-sm tracking-widest uppercase opacity-30">
SemiAnalysis x Fluidstack Hackathon
</div>
</div>

---

# Motivation: The Poisson Equation Is Everywhere

<div class="grid grid-cols-2 gap-10 mt-3 text-sm">
<div>

<div class="text-xs tracking-widest uppercase opacity-40 mb-2">Ubiquity of Poisson Solves</div>

The Poisson equation $\nabla^2 u = f$ appears as a **core computational kernel** across science and engineering:

<div class="mt-2 space-y-1 text-[0.82rem] opacity-80">

- **CFD** — pressure projection in incompressible Navier–Stokes
- **Electrostatics** — electric potential from charge distributions
- **Gravitational physics** — potential fields in astrophysics, geodesy
- **Structural mechanics** — stress analysis, plate bending
- **Heat transfer** — steady-state temperature distributions
- **Image processing** — Poisson blending, inpainting, surface reconstruction

</div>

</div>
<div>

<div class="text-xs tracking-widest uppercase opacity-40 mb-2">The Neural Operator Dilemma</div>

Neural operators can approximate PDE solutions **orders of magnitude faster** than classical solvers.

But they offer **no convergence guarantees** — predictions may look plausible while being quantitatively wrong.

<div class="mt-3 pl-4 border-l-2 border-cyan-400 opacity-80">

**Our approach:** Route between classical iterative methods (with guarantees) and a neural operator (with speed). Use ML when it helps, fall back to classical when it doesn't, and **always converge**.

</div>

<div class="mt-3 pl-4 border-l-2 border-amber-400 opacity-80">

Accelerating the Poisson solve has **outsized practical impact** — it is the dominant bottleneck in projection-based CFD solvers and appears in virtually every branch of computational physics.

</div>

</div>
</div>

---

# Incompressible Navier–Stokes

<div class="grid grid-cols-2 gap-10 mt-3">
<div>

The governing equations for incompressible flow:

$$
\frac{\partial \mathbf{u}}{\partial t} + (\mathbf{u} \cdot \nabla)\mathbf{u} = -\frac{1}{\rho}\nabla p + \nu \nabla^2 \mathbf{u} + \mathbf{f}
$$

$$
\nabla \cdot \mathbf{u} = 0
$$

<div class="mt-2 text-sm opacity-70">

**Key challenge:** velocity and pressure are coupled; pressure has no independent evolution equation.

</div>

</div>
<div>

<div class="text-xs tracking-widest uppercase opacity-40 mb-3">Splitting Methods</div>

Projection / SIMPLE / PISO all split this into:

<div class="mt-2 space-y-2 text-sm">
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

<div class="mt-3 text-sm opacity-70">

The pressure Poisson solve is often the **dominant computational bottleneck**.

</div>

</div>
</div>

---

# Focus: The Pressure Poisson Equation

<div class="grid grid-cols-[1fr,1.3fr] gap-8 mt-3">
<div>

<div class="text-xs tracking-widest uppercase opacity-40 mb-2">In CFD Splitting Methods</div>

Projection / SIMPLE / PISO all require solving:

$$\nabla^2 p^{n+1} = \frac{\rho}{\Delta t} \nabla \cdot \mathbf{u}^*$$

<div class="mt-3 space-y-1 text-sm opacity-80">

- **Elliptic** — globally coupled, every grid point depends on every other
- **Solved every timestep** — often multiple times per outer iteration
- **Dominant cost** — typically 50–80% of total CFD solve time

</div>

</div>
<div>

<div class="text-xs tracking-widest uppercase opacity-40 mb-2">Our Test Problem</div>

<div class="rounded-lg p-4 bg-white/5 border border-cyan-400/30">

**2D Poisson with periodic boundary conditions**

$$-\nabla^2 u = f \quad \text{on } [0,1]^2$$

<div class="mt-2 space-y-1 text-sm opacity-80">

- Grid size $N = 31$ ($961$ unknowns)
- Forcing $f$ drawn from hierarchical Gaussian random fields
- Solutions unique up to a constant (mean-zero constraint)

</div>

</div>

<div class="mt-3 text-sm opacity-70">

This isolates the core challenge: **can adaptive solver routing accelerate convergence of the Poisson linear system?**

</div>

</div>
</div>

---

# Iterative Solvers: The Landscape

<div class="mt-2 text-sm opacity-80">

Each linear subproblem $A\mathbf{u} = \mathbf{b}$ is solved by repeated iteration:

</div>

<div class="mt-3">

| Solver | Per-step cost | Convergence | Strengths |
|--------|:---:|:---:|-----------|
| Jacobi($\omega$) | $O(N^2)$ | Slow | Parallelizable, tunable damping |
| Gauss-Seidel | $O(N^2)$ | ~2× Jacobi | Better for low-freq error |
| SOR($\omega$) | $O(N^2)$ | Tunable | Optimal $\omega$ can be much faster |
| Multigrid | $O(N^2)$ | $O(1)$ iters | Optimal, but complex |
| **FNO (ours)** | $O(N^2)$ | One-shot | Large initial correction |

</div>

<div class="mt-4 pl-4 border-l-2 border-cyan-400 text-sm opacity-80">

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
<div class="mt-4 text-lg opacity-50">Oracle greedy routing — SOR(1.0) + SOR(1.3) + SOR(1.6) + FNO</div>
</div>

---

# 2D Poisson: Convergence

<img src="./images/poisson_fno_convergence.png" class="w-full max-h-80 object-contain rounded-lg border-0" />

<div class="grid grid-cols-2 gap-6 mt-3 text-sm">
<div class="text-center p-3 rounded-lg bg-white/5 border border-white/10">

**Best Classical (SOR 1.3)**
<br><span class="font-mono text-xs">Final L2: 3.60 × 10⁻⁶</span>
<br><span class="font-mono text-xs">AUC: 0.395</span>

</div>
<div class="text-center p-3 rounded-lg bg-white/5 border border-cyan-400/40">

<span class="text-cyan-400">**Greedy + Unrolled FNO**</span>
<br><span class="font-mono text-xs">Final L2: 5.70 × 10⁻⁸</span>
<br><span class="font-mono text-xs text-cyan-400">AUC: 9.58 × 10⁻⁴ — 412× lower</span>

</div>
</div>

---

# 2D Poisson: Routing Patterns

<img src="./images/poisson_routing_comparison.png" class="w-full max-h-72 object-contain rounded-lg border-0" />

<div class="grid grid-cols-2 gap-6 mt-3 text-sm">
<div class="opacity-80">

**Oracle Greedy** — the upper bound:
- Highly adaptive per-sample routing
- **SOR(1.6)** dominates (~64%), **SOR(1.3)** ~27%, **SOR(1.0)** ~8%
- FNO used sparingly (~0.5%) for targeted corrections
- AUC: **8.8 × 10⁻⁴**

</div>
<div class="opacity-80">

**Learned LSTM Router** — trained to imitate:
- Captures SOR(1.3) dominance (~57%) and per-sample adaptation
- Uses FNO at matching rate (0.5%) to the oracle
- AUC: **0.052** — 7.6× better than best classical, but still 59× gap to oracle

</div>
</div>

---

# Results Summary

<div class="mt-4">

| | **Best Classical (SOR 1.3)** | **Greedy + Unrolled FNO** | **Improvement** |
|---|---|---|---|
| **Final L2 Error** | 3.60 × 10⁻⁶ | **5.70 × 10⁻⁸** | 63× lower |
| **AUC** | 0.395 | **9.58 × 10⁻⁴** | 412× lower |

</div>

<div class="mt-5 pl-4 border-l-2 border-cyan-400 opacity-80 text-sm">

**Greedy routing** with 3 SOR variants + an unrolled FNO achieves **412× lower AUC** than the best single classical solver on the 2D Poisson equation. The FNO is trained through unrolled trajectories so it learns to correct the residuals that *actually arise* mid-solve — not random i.i.d. residuals.

</div>


<style>
:root {
  --slidev-theme-primary: #22d3ee;
}
.slidev-layout {
  background: #000 !important;
  color: #e4e4e7 !important;
  overflow: hidden !important;
}
.slidev-page .slidev-layout {
  padding: 2rem 2.5rem !important;
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
