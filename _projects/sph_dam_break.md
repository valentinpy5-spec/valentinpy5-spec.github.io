---
layout: page
title: "Weakly-Compressible SPH: Dam Break"
description: A CPU smoothed-particle hydrodynamics solver (WCSPH, cubic spline kernel, Tait equation of state) validated on the classic 3D dam-break benchmark.
img: assets/img/projects/sph_dam_break/poster.png
importance: 4
category: Personnal
github: https://github.com/valentinpy5-spec/SPH
---

<div class="mb-3">
  <a href="https://github.com/valentinpy5-spec/SPH" target="_blank" rel="noopener noreferrer" class="btn btn-outline-dark">
    <i class="fa-brands fa-github fa-lg"></i>&nbsp; View on GitHub
  </a>
</div>

## What this project is

A Lagrangian fluid solver, the other way around from the grid-based projects on this page: instead of a fixed mesh with a pressure Poisson solve, the fluid is a set of particles carrying mass, position, and velocity, and pressure emerges from an equation of state applied to each particle's locally estimated density: **Weakly-Compressible Smoothed Particle Hydrodynamics (WCSPH)**. The code is on GitHub at [SPH](https://github.com/valentinpy5-spec/SPH) (C++17, Eigen for the physics, OpenGL for rendering, Dear ImGui for the live UI), validated on the classic dam-break benchmark below.

<div class="row justify-content-sm-center">
  <div class="col-sm-8 mt-3 mt-md-0">
    {% assign poster_url = "assets/img/projects/sph_dam_break/poster.png" | relative_url %}
    {% include video.liquid path="assets/video/projects/sph_dam_break_demo.mp4" class="img-fluid rounded z-depth-1" controls=true poster=poster_url %}
  </div>
</div>
<div class="caption">
  4500 fluid particles released under gravity in a unit-cube domain, captured directly from a live run, including the ImGui performance panel (physics ≈ 21-31 ms/frame, render under 1 ms, confirming the physics update, not rendering, is the actual cost).
</div>

## Why particles instead of a grid

The other fluid solvers on this page discretize space with a fixed grid and track field quantities at fixed locations (an Eulerian description). SPH instead discretizes *mass*: any field quantity (density, pressure) is estimated at a particle's location as a weighted sum over its nearby neighbors, using a smoothing kernel. A dam break, with its collapsing front and fragmenting splash, has no fixed shape to mesh, which is exactly the regime SPH is suited for and a fixed-grid method is not.

## Kernel and density

Density at each particle is a sum of neighboring masses weighted by a smoothing kernel $W$, and pressure-force/viscosity computations need its gradient. Rather than using three different specialized kernels for these three tasks (the common Müller et al., 2003 convention), this solver uses a single **cubic spline kernel** for all of them, following the more recent recommendation in Bender et al. (2017) that mathematical consistency outweighs the simplicity of a three-kernel trio:

$$W(q) = \sigma \begin{cases} 6(q^3 - q^2) + 1 & 0 \le q \le \tfrac12 \\ 2(1-q)^3 & \tfrac12 < q \le 1 \\ 0 & q > 1 \end{cases}, \qquad q = \frac{r}{h}, \quad \sigma = \frac{8}{\pi h^3}$$

The gradient is the analytical derivative of this same piecewise form, keeping the pressure-force and viscosity terms mathematically consistent with the density estimate they're built from.

## Pressure from density: the Tait equation of state

Pressure is not solved for to enforce incompressibility as in the grid-based solvers on this page; it is computed directly, per particle, from how far the local density has drifted from the rest density $\rho_0$, via the **Tait equation of state**:

$$p = \frac{\rho_0 c_s^2}{7}\left[\left(\frac{\rho}{\rho_0}\right)^7 - 1\right]$$

The seventh-power dependence gives a much sharper pressure response than the simpler linear equation of state $p = k(\rho - \rho_0)$, which is what keeps the fluid "weakly compressible" in practice without ever solving a global pressure system. The resulting pressure force, $\nabla p / \rho \approx \sum_j m_j \left(\tfrac{p_i}{\rho_i^2} + \tfrac{p_j}{\rho_j^2}\right)\nabla W_{ij}$, is symmetric in $i$ and $j$ by construction, so Newton's third law holds exactly between every particle pair and momentum is conserved to machine precision.

## Time integration and neighbor search

Each frame runs a fixed symplectic-Euler-style substep sequence: velocity is first updated by the non-pressure forces (gravity and viscosity), density and pressure are then estimated from the resulting configuration, and only then is the pressure-force contribution applied before updating position, at a timestep of `dt = 4×10⁻⁴ s`. Every density, pressure-force, and viscosity computation needs each particle's neighbors within the kernel support radius $h$; done naively that is an $O(N^2)$ pass every substep, so a spatial hash buckets particles into a uniform grid of cell size $h$ and only checks neighboring cells, which is what keeps this 4500-particle dam break at 30-45 FPS.

## What's next

The most direct next step is replacing the naive per-frame spatial hash rebuild with an incremental one to scale particle count further, then porting the solver itself to GPU/OpenCL, the same CPU-to-GPU trajectory already completed for the [NACA0012 flow solver]({{ '/projects/naca0012_lbm_gpu/' | relative_url }}) elsewhere on this page.

## References

- Monaghan, J. J., *Smoothed particle hydrodynamics*, Reports on Progress in Physics, 2005: foundational SPH formulation, kernel interpolation, and the pressure-force symmetrization used throughout this solver.
- Müller, M., Charypar, D., Gross, M., *Particle-Based Fluid Simulation for Interactive Applications*, SCA 2003: the three-kernel (poly6/spiky/viscosity) convention this project deliberately departs from.
- Bender, J. et al., *Smoothed Particle Hydrodynamics Techniques for the Physics Based Simulation of Fluids and Solids*, Eurographics Tutorial, 2017: single consistent cubic-spline-kernel recommendation adopted here.
- Becker, M., Teschner, M., *Weakly compressible SPH for free surface flows*, SCA 2007: Tait equation of state and WCSPH stability considerations.
