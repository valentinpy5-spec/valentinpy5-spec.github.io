---
layout: page
title: "NACA0012 Flow Solver: Lattice-Boltzmann on GPU"
description: Real-time GPU Lattice-Boltzmann solver (D2Q9/BGK, OpenCL) for 2D flow around a NACA0012 airfoil, with live angle-of-attack control and OpenGL visualization.
img: assets/img/projects/naca0012_lbm_gpu/lbm_converged.png
importance: 2
category: Internships
github: https://github.com/valentinpy5-spec/LBM_NACA0012
---

<div class="mb-3">
  <a href="https://github.com/valentinpy5-spec/LBM_NACA0012" target="_blank" rel="noopener noreferrer" class="btn btn-outline-dark">
    <i class="fa-brands fa-github fa-lg"></i>&nbsp; View on GitHub
  </a>
</div>

## What this project is

An entirely GPU-resident CFD solver: the 2D incompressible Navier-Stokes equations around a NACA0012 airfoil, solved with the **Lattice-Boltzmann Method (LBM)** and rendered live in OpenGL, with no CPU/GPU round-trip in the simulation loop itself. This was my first real GPU programming project, and the main point of it: LBM's locality (every cell only ever talks to its immediate neighbors) makes it a nearly ideal problem to learn OpenCL on, since the parallelization strategy is obvious from the physics itself rather than something to bolt on afterward.

The code is on GitHub at [LBM_NACA0012](https://github.com/valentinpy5-spec/LBM_NACA0012).

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/naca0012_lbm_gpu/lbm_t0.png" title="LBM NACA0012, initial condition" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/naca0012_lbm_gpu/lbm_converged.png" title="LBM NACA0012, converged flow field" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
  Left: uniform inflow at t = 0. Right: converged velocity-magnitude field around the airfoil at zero angle of attack (Re ≈ 435), captured directly from a live run: symmetric boundary layers and a settled wake, consistent with the low-Reynolds, attached-flow regime this BGK scheme targets.
</div>

## Why Lattice-Boltzmann instead of solving Navier-Stokes directly

Rather than discretizing the Navier-Stokes equations directly and solving a pressure Poisson equation at every step, LBM evolves a set of particle distribution functions on a discrete velocity lattice, here **D2Q9**: each cell stores 9 distribution values, one per discrete velocity direction, updated each step by streaming (propagating each distribution to its neighbor) and collision (relaxing each distribution toward a local equilibrium). Macroscopic density and velocity are recovered as simple moments of these 9 values; there is no separate pressure solve, since the weakly-compressible LBM formulation enforces incompressibility implicitly through the collision step itself.

## The BGK collision operator and its stability limits

The full lattice Boltzmann equation with single-relaxation-time (BGK) collision, combining streaming and collision into one update per discrete velocity direction $i$, is:

$$f_i(\mathbf{x} + \mathbf{e}_i \Delta t,\ t + \Delta t) = f_i(\mathbf{x}, t) + \omega\left[f_i^{\text{eq}}(\mathbf{x}, t) - f_i(\mathbf{x}, t)\right]$$

where the equilibrium distribution $f_i^{\text{eq}}$ is a truncated expansion of the Maxwell-Boltzmann distribution in the local macroscopic velocity $\mathbf{u}$:

$$f_i^{\text{eq}} = w_i \rho \left[1 + \frac{\mathbf{e}_i \cdot \mathbf{u}}{c_s^2} + \frac{(\mathbf{e}_i \cdot \mathbf{u})^2}{2c_s^4} - \frac{\mathbf{u} \cdot \mathbf{u}}{2c_s^2}\right]$$

with $w_i$ the lattice weights and $c_s$ the lattice speed of sound. Collision uses this single-relaxation-time **BGK** operator: each distribution relaxes toward $f_i^{\text{eq}}$ at a rate set by a single parameter ω. This determines the lattice viscosity, and with it the Reynolds number reachable at a given grid resolution:

$$Re = \frac{U_{\text{lu}} \cdot N_{\text{chord}}}{\nu_{\text{lu}}}, \qquad \nu_{\text{lu}} = \frac{1}{3}\left(\frac{1}{\omega} - \frac{1}{2}\right)$$

BGK is stable only for ω < 2: as ω approaches 2, the lattice viscosity vanishes and the scheme loses the numerical dissipation it needs near the airfoil's leading edge, where velocity gradients are sharpest. This is why the solver is restricted to a moderate Reynolds number (≈ 435 at the default configuration) rather than a fully turbulent regime; going further needs a scheme that decouples physical viscosity from numerical stability, such as **TRT** (Two-Relaxation-Time).

## Geometry, boundary conditions, and aerodynamic forces

The NACA0012 profile is voxelized directly on the lattice from the standard 4-digit NACA thickness formula, rotated live by the angle of attack. No-slip on the airfoil surface comes from full-way bounce-back (a solid cell reflects every incoming distribution back the way it came), the inlet velocity is imposed with the **Zou-He** method (Zou & He, 1997), and the outlet uses a simple zero-gradient condition. Lift and drag are not obtained from a pressure field, since LBM never maintains one directly, but from the **momentum-exchange method** (Ladd, 1994): summing the momentum carried by every reflected distribution at the airfoil surface gives Cl and Cd directly from the same bounce-back mechanism that already enforces no-slip.

## What building this taught me about GPU programming

This is where the basics of GPU programming actually clicked: structuring a physical update as a small set of independent kernels, thinking in terms of what each thread reads and writes rather than a sequential loop, and keeping data resident on the GPU across the whole simulation loop instead of shuttling it back and forth every step. LBM's per-cell locality also made memory access patterns concrete: getting the streaming step to read and write efficiently is a direct lesson in how GPU memory bandwidth actually works, in a way that's hard to learn from a purely CPU-side project. Coupling the simulation to OpenGL rendering through a shared context, so the result can be visualized without ever leaving the GPU, was the other half of that lesson: understanding where the CPU/GPU boundary actually is, and how much slower everything gets the moment you cross it unnecessarily.

## What's next

The direct next steps are a **TRT** collision operator (Ginzburg & d'Humières, 2008) to reach higher Reynolds numbers at the same resolution, and a 3D **D3Q19** extension for genuinely three-dimensional wingtip effects that a 2D slice cannot capture.

## References

- Krüger, T. et al., *The Lattice Boltzmann Method: Principles and Practice*, Springer, 2017: primary reference for the D2Q9/BGK scheme implemented here.
- Ladd, A. J. C., *Numerical simulations of particulate suspensions via a discretized Boltzmann equation*, Journal of Fluid Mechanics, 1994: momentum-exchange method used for the aerodynamic forces.
- Zou, Q. and He, X., *On pressure and velocity boundary conditions for the lattice Boltzmann BGK model*, Physics of Fluids, 1997: inlet velocity boundary condition.
- Lehmann, M., *Computational Fluid Dynamics with the Lattice Boltzmann Method: Fluids, Solids, and Bubbles*, PhD thesis, 2022 (FluidX3D): TRT and high-Reynolds considerations referenced for the solver's next steps.
- Ginzburg, I. and d'Humières, D., *Two-relaxation-time Lattice Boltzmann scheme: About parametrization, velocity, pressure and mixed boundary conditions*, Advances in Water Resources, 2008: TRT parametrization planned as the stability extension beyond BGK.
