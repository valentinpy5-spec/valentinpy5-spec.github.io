---
layout: page
title: "NACA0012 Flow Solver: Semi-Lagrangian / Pressure Projection"
description: A 2D incompressible Navier-Stokes solver around a NACA0012 airfoil, built on a MAC staggered grid with semi-Lagrangian advection and a matrix-free conjugate-gradient pressure projection.
img: assets/img/projects/naca0012_fluidsim_2d/naca0012_velocity.png
importance: 3
category: Personnal
github: https://github.com/valentinpy5-spec/aerodynamic-sim
---

<div class="mb-3">
  <a href="https://github.com/valentinpy5-spec/aerodynamic-sim" target="_blank" rel="noopener noreferrer" class="btn btn-outline-dark">
    <i class="fa-brands fa-github fa-lg"></i>&nbsp; View on GitHub
  </a>
</div>

## What this project is

A from-scratch incompressible Navier-Stokes solver for 2D flow around a NACA0012 airfoil, built directly on the classic **MAC (Marker-and-Cell) staggered grid** formulation rather than a collocated grid: pressure lives at cell centers, and each velocity component lives on its own face. This is the CPU predecessor to the GPU Lattice-Boltzmann version of this project ([see the LBM GPU solver]({{ '/projects/naca0012_lbm_gpu/' | relative_url }})), and the two take genuinely different numerical approaches to the same physical problem, which is part of why keeping both is useful: one solves Navier-Stokes directly through a pressure projection, the other never forms a pressure field at all.

The code is organized around `src/StaggeredGrid.cpp`, `src/PressureSolver.cpp`, and `src/FluidSimulator.cpp`. The overall structure (staggered-grid classes, the semi-Lagrangian/pressure-projection split, some method names and comments) follows [Doyub Kim's fluid simulation tutorial](https://unusualinsights.github.io/fluid_tutorial/#home), which was the starting point for this project; the airfoil geometry, 2D boundary conditions, and initial conditions, and the removal of the tutorial's FLIP/PIC particle layer in favor of a purely grid-based Eulerian solver, are specific to this project.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/naca0012_fluidsim_2d/naca0012_velocity.png" title="Velocity field around the NACA0012" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/naca0012_fluidsim_2d/naca0012_pressure.png" title="Pressure field around the NACA0012" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
  Left: velocity/dye field, showing the boundary layer and wake behind the airfoil. Right: the corresponding pressure field solved by conjugate gradient, showing the stagnation region at the leading edge and the low-pressure zone over the upper surface.
</div>

## The governing equations, split into three sub-problems

The solver targets the incompressible Navier-Stokes equations, but rather than discretizing them as one coupled system, it follows the classical **operator-splitting** strategy (Chorin's projection method, as formalized for graphics by Stam and by Bridson): each timestep is broken into an advection sub-step, a body-force sub-step, and a pressure/incompressibility sub-step, solved one after another rather than simultaneously.

<div class="row justify-content-sm-center">
  <div class="col-sm-6 mt-3 mt-md-0">
    {% include figure.liquid loading="eager" path="assets/img/projects/naca0012_fluidsim_2d/bridson_eq_ns_split.png" title="Navier-Stokes split into advection, body forces, and pressure projection" class="img-fluid rounded z-depth-1" %}
  </div>
</div>
<div class="caption">
  The three-way operator split (Bridson, eq. 2.8-2.10) this solver follows every timestep: advect a quantity along the flow ($Dq/Dt = 0$), apply body forces, then project the velocity field onto its divergence-free component by solving for pressure.
</div>

Concretely, `FluidSimulator`'s per-frame loop calls `AdvectVelocity` (the $Dq/Dt=0$ sub-step, applied to velocity itself), then adds gravity/external forces, then calls `ProjectPressure` (the $\partial\vec u/\partial t + \tfrac1\rho\nabla p = 0$ sub-step, solved so that the constraint $\nabla\cdot\vec u = 0$ holds afterward); each sub-step is solved to good accuracy on its own, and the splitting error this introduces is first-order in $\Delta t$, the standard trade-off of this family of methods.

## Why a staggered grid, and what problem it solves

Storing pressure and all velocity components at the same grid point (a collocated arrangement) leads to a well-known failure mode in incompressible solvers: the discrete pressure gradient and divergence operators can develop a null space, letting a checkerboard pressure pattern exist that produces zero force on the velocity field and is therefore invisible to the solver. The MAC grid sidesteps this entirely by never colocating pressure and velocity.

<div class="row justify-content-sm-center">
  <div class="col-sm-6 mt-3 mt-md-0">
    {% include figure.liquid loading="eager" path="assets/img/projects/naca0012_fluidsim_2d/bridson_fig2_1_mac_grid.png" title="The two-dimensional MAC grid" class="img-fluid rounded z-depth-1" %}
  </div>
</div>
<div class="caption">
  The 2D MAC grid (Bridson, fig. 2.1): pressure $p_{i,j}$ at cell centers, $u_{i\pm 1/2,j}$ on vertical faces, $v_{i,j\pm 1/2}$ on horizontal faces. `StaggeredGrid` implements exactly this layout as four separate grid arrays (`u_`, `v_`, `w_`, `p_`), each sized one larger than the cell grid in its own staggered direction, so the discrete divergence and gradient operators are natural adjoints of each other by construction.
</div>

## Semi-Lagrangian advection with a midpoint backtrace

Advection is handled by tracing each grid point backward through the velocity field to find where its value came from one timestep ago, rather than discretizing the advection term directly: the classic semi-Lagrangian approach, which is unconditionally stable regardless of the CFL number (unlike explicit upwinding).

<div class="row justify-content-sm-center">
  <div class="col-sm-6 mt-3 mt-md-0">
    {% include figure.liquid loading="eager" path="assets/img/projects/naca0012_fluidsim_2d/bridson_fig3_1_semilagrangian.png" title="Semi-Lagrangian backtrace" class="img-fluid rounded z-depth-1" %}
  </div>
</div>
<div class="caption">
  The semi-Lagrangian idea (Bridson, fig. 3.1): to find the new value at grid point $\vec x_G$, trace backward along the velocity field to the departure point $\vec x_P$ where that fluid parcel was one timestep ago, and interpolate the old field there.
</div>

`StaggeredGrid::AdvectVelocity` does this with a second-order **RK2 midpoint** backtrace rather than a single Euler step:

$$\vec x_{\text{mid}} = \vec x_G - \tfrac{1}{2}\Delta t\,\vec u(\vec x_G), \qquad \vec x_P = \vec x_G - \Delta t\,\vec u(\vec x_{\text{mid}})$$

i.e. it evaluates the velocity at the query point, steps half a timestep backward to get a midpoint, re-evaluates the velocity there, and only then takes the full backward step from that midpoint estimate. This is meaningfully more accurate than a naive one-step backtrace, at the cost of one extra trilinear interpolation per grid point per step. Solid cells (inside the airfoil) are excluded from the backtrace target via `ClampToNonSolidCells`, which snaps any backtraced position landing inside the airfoil to the nearest fluid cell center instead.

## Enforcing incompressibility: discretizing the pressure Poisson equation

After advection, the velocity field is generally not divergence-free, which is required for incompressible flow. Enforcing $\nabla\cdot\vec u=0$ after the fact means solving a discrete Poisson equation for pressure. Bridson's derivation makes both discretizations, divergence and Laplacian, explicit side by side:

<div class="row justify-content-sm-center">
  <div class="col-sm-9 mt-3 mt-md-0">
    {% include figure.liquid loading="eager" path="assets/img/projects/naca0012_fluidsim_2d/bridson_discretization_divergence_laplacian.png" title="Discretizing the divergence of velocity and the Laplacian of pressure" class="img-fluid rounded z-depth-1" %}
  </div>
</div>
<div class="caption">
  The velocity divergence at cell $(i,j,k)$ (blue) is a centered finite difference of the face velocities; the pressure Laplacian (red) is the standard 7-point stencil in 3D (5-point in the 2D case this project actually runs). Setting divergence-after-projection to zero and solving for $p$ gives exactly this equation.
</div>

Each pressure unknown only couples to its 4 immediate neighbors (2D) or 6 (3D), which is why the resulting linear system is extremely sparse: only the diagonal and a few off-diagonal bands are ever nonzero, visualized directly in Bridson's matrix form:

<div class="row justify-content-sm-center">
  <div class="col-sm-9 mt-3 mt-md-0">
    {% include figure.liquid loading="eager" path="assets/img/projects/naca0012_fluidsim_2d/bridson_linear_system.png" title="The sparse linear system for pressure" class="img-fluid rounded z-depth-1" %}
  </div>
</div>
<div class="caption">
  One row of the pressure system: a fluid cell's own pressure carries coefficient 6 (its number of neighbor faces, 2D case here is 4), each fluid neighbor carries -1, and the right-hand side is the (negated) velocity divergence at that cell, scaled by $\rho(\Delta x)^2/\Delta t$. This is exactly the system `PressureSolver` assembles implicitly, one row per fluid cell.
</div>

Each pressure unknown only appears in the equations for the cells that share a face with it, which is also visible directly on the grid: the coupling runs entirely through the four adjacent face velocities, and nowhere else.

<div class="row justify-content-sm-center">
  <div class="col-sm-6 mt-3 mt-md-0">
    {% include figure.liquid loading="eager" path="assets/img/projects/naca0012_fluidsim_2d/bridson_pressure_sample_stencil.png" title="Pressure sample and its four neighboring velocity faces" class="img-fluid rounded z-depth-1" %}
  </div>
</div>
<div class="caption">
  A single pressure sample $p$ and its four neighboring face velocities. Each of these faces contributes one term to the divergence at this cell, and, after projection, is corrected by the pressure difference across it: this is the local picture behind the global sparse system above.
</div>

## Solving the system: matrix-free conjugate gradient

The linear system above is symmetric positive-definite by construction (a discrete Laplacian restricted to fluid cells), which makes **conjugate gradient (CG)** the natural choice: it converges monotonically in the $A$-norm of the error, needs only matrix-vector products (no factorization), and for a well-conditioned sparse system like this one typically converges in far fewer than $N$ iterations. `PressureSolver::ProjectPressure` implements CG without ever assembling the matrix $A$ explicitly:

$$\mathbf r_0 = \mathbf b - A\mathbf p_0, \quad \mathbf d_0 = \mathbf r_0, \qquad
\alpha_k = \frac{\mathbf r_k^\top \mathbf r_k}{\mathbf d_k^\top A\mathbf d_k}, \qquad
\mathbf p_{k+1} = \mathbf p_k + \alpha_k \mathbf d_k, \qquad \mathbf r_{k+1} = \mathbf r_k - \alpha_k A\mathbf d_k$$

$$\beta_k = \frac{\mathbf r_{k+1}^\top \mathbf r_{k+1}}{\mathbf r_k^\top \mathbf r_k}, \qquad \mathbf d_{k+1} = \mathbf r_{k+1} + \beta_k \mathbf d_k$$

`ATimes` computes every matrix-vector product $A\mathbf d$ on the fly from a per-cell neighbor bitmask (`neighbors_`, built by `MakeNeighborMaterialInfo`): for each fluid cell it reads off which of its neighbors are also fluid, applies the corresponding stencil (a `6` or `4` on the diagonal depending on dimensionality, and a `-1` for each fluid neighbor, exactly as in the matrix above); `SOLID` neighbors simply drop out of the sum, which is how the immersed airfoil boundary enters the pressure solve without ever needing a boundary-fitted mesh. This avoids ever storing the (very sparse, but still large) system matrix, at the cost of recomputing the stencil pattern lookup every iteration, a standard trade-off in CFD pressure solvers where the matrix structure is fixed but large. Once $p$ converges, `SubtractPressureGradientFromVelocity` applies the correction $\vec u \mathrel{-}= \Delta t\,\nabla p/\rho$ face by face, which is what actually restores $\nabla\cdot\vec u = 0$.

## Handling the airfoil as an immersed boundary

The NACA0012 profile is not meshed; it is voxelized directly onto the Cartesian grid from the standard 4-digit NACA thickness formula (`StaggeredGrid::NACA0012`, `IsInsideNACA0012`), the same closed-form profile used in the GPU LBM version of this project. Every grid cell is classified `FLUID` or `SOLID` at initialization and after every angle-of-attack change (`ResetForAlpha`), and the no-slip condition on the airfoil surface is enforced by directly zeroing every velocity face adjacent to a `SOLID` cell (`EnforceSolidVelocities`), rather than through boundary-fitted mesh cells. This immersed-boundary approach is what makes the "change angle of attack live at runtime" interaction possible: changing α only requires re-voxelizing which cells are solid, never regenerating a mesh.

## Lift from circulation, not from a surface pressure integral

Unlike a typical panel-method or RANS solver, lift here is not computed by integrating pressure over the airfoil surface. `StaggeredGrid::ComputeLift` instead evaluates the circulation Γ around a rectangular contour enclosing the airfoil, by summing `u·dx` and `v·dy` along the four sides of that box, and applies the **Kutta-Joukowski theorem**:

$$C_l = \frac{2\Gamma}{U_\infty \cdot c}$$

This is an inviscid-flow result, strictly valid only for irrotational flow outside the boundary layer, appropriate here given the solver has no explicit turbulence closure, but worth stating plainly rather than implying the lift estimate accounts for viscous effects it does not model.

## What's next

The most direct next step, already explored in the GPU LBM rewrite of this same problem, is comparing the two solvers' drag and wake structure at matched Reynolds number: the LBM version computes forces through momentum exchange at the solid boundary (Ladd's method) rather than through circulation, so the two give genuinely independent estimates of the same physical quantity and are a natural cross-check of each other.

## References

- Bridson, R., *Fluid Simulation for Computer Graphics*, 2nd edition, CRC Press, 2015: MAC grid formulation, the advection/forces/pressure operator split, semi-Lagrangian advection, and the pressure Poisson discretization and matrix-free conjugate-gradient solve; this project follows Bridson's staggered-grid conventions throughout, and every boxed figure above is reproduced from this text.
- Kim, D., [*Fluid Simulation for Computer Graphics: A Practical Guide*](https://unusualinsights.github.io/fluid_tutorial/#home): the tutorial this project's overall code structure was originally built from, before being reworked for a 2D NACA0012 case without FLIP/PIC.
- Stam, J., *Stable Fluids*, SIGGRAPH 1999: semi-Lagrangian (backward-trace) advection scheme underlying `AdvectVelocity`.
- Anderson, J. D., *Fundamentals of Aerodynamics*, McGraw-Hill: Kutta-Joukowski circulation theorem used in `ComputeLift`.
