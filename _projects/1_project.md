---
layout: page
title: "1D Euler Solver: Exact Riemann Solver (FVM)"
description: Finite volume solver for the compressible Euler equations, with an exact Riemann solver (Toro) and RK2 time integration, validated on the Sod shock tube.
img: assets/img/projects/euler_fvm/sod_t05.png
importance: 1
category: Personnal
github: https://github.com/valentinpy5-spec/FVM-Euler
---

<div class="mb-3">
  <a href="https://github.com/valentinpy5-spec/FVM-Euler" target="_blank" rel="noopener noreferrer" class="btn btn-outline-dark">
    <i class="fa-brands fa-github fa-lg"></i>&nbsp; View on GitHub
  </a>
</div>

## What this project is

A finite-volume solver for the 1D compressible Euler equations, built as one step of a personal progression toward a full Navier-Stokes solver, starting from scalar linear advection, then Burgers' equation, and now a genuine hyperbolic **system** with three coupled conservation laws. The goal was not just to get a shock tube picture that "looks right", but to implement and understand an **exact** Riemann solver from first principles, rather than reaching directly for an approximate one (Roe, HLLC).

The code is on GitHub at [FVM-Euler](https://github.com/valentinpy5-spec/FVM-Euler) (`FVGrid1DEuler`, `EulerRiemannProblem`, `EulerState`): C++, no external numerical library, only `<cmath>`.

## The physics: Euler equations in conservative form

The 1D compressible Euler equations for an inviscid, compressible flow are a hyperbolic system of conservation laws:

$$\frac{\partial \mathbf U}{\partial t} + \frac{\partial \mathbf F(\mathbf U)}{\partial x} = 0, \qquad \mathbf U = \begin{pmatrix}\rho \\ \rho u \\ \rho E\end{pmatrix}, \qquad \mathbf F(\mathbf U) = \begin{pmatrix}\rho u \\ \rho u^2 + p \\ (\rho E + p)u\end{pmatrix}$$

closed with the ideal-gas equation of state $p = (\gamma-1)\rho e$, using $\gamma = 1.4$ (diatomic gas, air). This is exactly the conservative formulation developed in **Toro's *Riemann Solvers and Numerical Methods for Fluid Dynamics*** (chapter 3): the code keeps the same notation (`RiemannPState` stores $\rho, u, p, c$; `EulerState1D` stores the conservative triple) to make the correspondence between implementation and textbook derivation as direct as possible.

<div class="row justify-content-sm-center">
  <div class="col-sm-8 mt-3 mt-md-0">
    {% include figure.liquid loading="eager" path="assets/img/projects/euler_fvm/sod_t00.png" title="Sod shock tube, initial condition" class="img-fluid rounded z-depth-1" %}
  </div>
</div>
<div class="caption">
  Sod shock tube, initial discontinuity (t ≈ 0): a density/pressure jump at x = 0.5, at rest on both sides, the standard test case from Sod (1978), reused throughout Toro's book (§4.3.3) precisely because its exact solution is known analytically and it exercises the three characteristic families simultaneously.
</div>

## Why a finite-volume face flux needs a Riemann problem

The core idea of the finite volume method, already explored in the earlier scalar-advection step of this project, is that a numerical flux at a cell interface fully determines the update of the cell average on each side. For a **linear** scalar equation, that flux is simple (e.g. plain upwinding). For the Euler system, the situation at each interface is structurally different: two constant states, $\mathbf{U}_L$ and $\mathbf{U}_R$, meet at $x = 0$, and the exact evolution of that discontinuity is itself a small, self-similar initial value problem: the **Riemann problem**. Solving it exactly at every face, at every timestep, is what an *exact* Riemann solver does (as opposed to an approximate one like HLLC, which sacrifices exactness for speed).

The Riemann problem has a self-similar structure: the solution only depends on the ratio $x/t$, never on $x$ and $t$ separately. It always resolves into at most three waves: two non-linear waves (each either a shock or a rarefaction, never both) straddling a linearly degenerate **contact discontinuity** in the middle, across which pressure and velocity are continuous but density jumps. Between the two non-linear waves lie two constant "star" states, sharing the same pressure and the same velocity by construction. Finding this structure exactly comes down to finding that shared star pressure, written $p_{\ast}$ throughout this project. Everything else then follows analytically: the density in the star region, the shock speed, the rarefaction fan profile.

<div class="row justify-content-sm-center">
  <div class="col-sm-8 mt-3 mt-md-0">
    {% include figure.liquid loading="eager" path="assets/img/projects/euler_fvm/toro_fig14_1_riemann_solution.png" title="Typical solution to the Riemann problem for the Euler equations" class="img-fluid rounded z-depth-1" %}
  </div>
</div>
<div class="caption">
  The general wave-fan structure of the Riemann problem (Toro, fig. 14.1): a rarefaction wave and a shock bound the fan on each side, separated by the contact discontinuity in the middle. The two star states share $p^\ast$ and $u^\ast$ but differ in density, exactly the structure `SampleSolution` reconstructs at every face.
</div>

## Solving for the star pressure: a root-finding problem, not a closed form

Because the shock relations (Rankine-Hugoniot) and the rarefaction relations (isentropic, from the Riemann invariants) have fundamentally different functional forms, there is no closed-form expression for $p_{\ast}$. Toro reduces the problem to finding the root of a single scalar equation, implemented directly as `EulerRiemannProblem::SolveStarPressure`:

$$F(p_{\ast}) = f_L(p_{\ast}) + f_R(p_{\ast}) + (u_R - u_L) = 0, \qquad f_s(p_{\ast}) = \begin{cases} \dfrac{2c_s}{\gamma-1}\!\left[\left(\dfrac{p_{\ast}}{p_{s}}\right)^{\frac{\gamma-1}{2\gamma}} - 1\right] & p_{\ast} \le p_{s} \ \ \text{(rarefaction)} \\[2mm] (p_{\ast}-p_{s})\left[\dfrac{2/((\gamma+1)\rho_s)}{p_{\ast} + \frac{\gamma-1}{\gamma+1}p_{s}}\right]^{1/2} & p_{\ast} > p_{s} \ \ \text{(shock)} \end{cases}$$

The solver in this project uses **Newton-Raphson**, not bisection, and this is a deliberate choice rather than a default. The function $F$ is smooth and strictly monotonic in the star pressure. This guarantees a unique root. Toro also provides the analytical derivative $F'$ in closed form, so Newton's quadratic convergence is essentially free: it typically resolves the star pressure to a tolerance of $10^{-10}$ in only 3 to 5 iterations, versus 30 to 40 for bisection at the same tolerance. The one subtlety implemented explicitly is the initial guess. A naive average of the two pressures can fail for strong pressure jumps, so the code uses Toro's physically motivated estimate below.

$$p_{\ast}^{(0)} = \max\!\left(10^{-6},\ \tfrac12(p_L+p_R) - \tfrac18(u_R-u_L)(\rho_L+\rho_R)(c_L+c_R)\right)$$

This is combined with a guard against Newton overshooting into negative pressure, where the fractional powers in $f_s$ become undefined: `if (x_new <= 0.0) x_new = 0.5 * x;` in `NewtonRaphson`.

<div class="row justify-content-sm-center">
  <div class="col-sm-8 mt-3 mt-md-0">
    {% include figure.liquid loading="eager" path="assets/img/projects/euler_fvm/toro_fig4_3_pressure_function.png" title="Strategy for solving the Riemann problem via a pressure function" class="img-fluid rounded z-depth-1" %}
  </div>
</div>
<div class="caption">
  The strategy behind `SolveStarPressure` (Toro, fig. 4.3): the particle velocity on each side is tied to the (unknown) star pressure through the functions $f_L$ and $f_R$. Newton-Raphson iterates on $p_{\ast}$ until $f_L(p_{\ast}) = f_R(p_{\ast})$, i.e. until both sides agree on a common particle velocity $u_{\ast}$.
</div>

## Sampling the solution at the face: shock vs. rarefaction, star vs. outer state

Once $p_{\ast}$ and the associated star velocity are known, `EulerRiemannProblem::SampleSolution` reconstructs the state that actually sits at the interface $x/t = 0$, by walking through the possible regions of the wave fan. First, which side of the contact discontinuity we're on, from the sign of the star velocity. Then, whether that side's non-linear wave is a shock, handled by `shock_solution` using the Rankine-Hugoniot star density and shock speed (Toro eq. 4.50, 4.52, 4.57, 4.59), or a rarefaction. If it's a rarefaction, the code further distinguishes whether the queried point lies ahead of the wave, inside the star region, or **inside the fan itself** (Toro eq. 4.56 and 4.63): this last case requires the full self-similar isentropic profile rather than a single constant state, and is the one most solvers skip or approximate. Implementing it explicitly is what makes this solver *exact* rather than merely "good enough" for a mild shock tube.

The resulting face state feeds directly into the physical flux $\mathbf F(\mathbf U)$, which is exactly Godunov's method: the numerical flux at each face **is** the physical flux evaluated at the exact Riemann solution, not an ad-hoc numerical construction.

## Time integration and validation

Spatial reconstruction is first-order (piecewise constant per cell, no MUSCL/slope limiter yet), but time integration uses **second-order SSP Runge-Kutta (RK2/Heun)** rather than plain forward Euler. The timestep respects the acoustic CFL condition below, with CFL = 0.5:

$$\Delta t \le \mathrm{CFL} \cdot \frac{\Delta x}{\max_i(|u_i|+c_i)}$$

<div class="row justify-content-sm-center">
  <div class="col-sm-8 mt-3 mt-md-0">
    {% include figure.liquid loading="eager" path="assets/img/projects/euler_fvm/toro_fig15_1_godunov_interfaces.png" title="Solving the Riemann problems at each interface for Godunov's method" class="img-fluid rounded z-depth-1" %}
  </div>
</div>
<div class="caption">
  Why the CFL restriction matters for Godunov's method (Toro/LeVeque, fig. 15.1): with Courant number below 1/2 (a), waves from neighboring Riemann problems never interact before reaching the next face, so the piecewise-constant flux assumption stays exact for the full timestep. Above 1/2 (b), waves from adjacent interfaces start to cross before the timestep ends, which is why the solver caps CFL at 0.5 rather than the theoretical stability limit of 1.
</div>

The test case is the classical Sod shock tube, run on 512 cells to $t = 0.2$ with $\gamma = 1.4$. Left state: density 1, velocity 0, pressure 1. Right state: density 0.125, velocity 0, pressure 0.1.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/euler_fvm/sod_t03.png" title="Sod shock tube, intermediate time" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/euler_fvm/sod_t05.png" title="Sod shock tube, final time" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
  Left: t ≈ 0.12, the wave pattern has fully formed. Right: t = 0.2 (converged final state). Left-to-right: the rarefaction fan (smooth, continuous decrease in ρ, u, p), the contact discontinuity (the small kink near x ≈ 0.5, a density/temperature jump with continuous p and u, exactly as the theory predicts), and the right-moving shock (the sharp jump near x ≈ 0.85, satisfying Rankine-Hugoniot). This is the qualitative structure Toro uses throughout the book as the reference validation case for any new Euler solver.
</div>

The three-wave structure predicted by the theory is visible directly in the output: a smooth rarefaction on the left, a small but genuine jump at the contact discontinuity, and a sharp right-moving shock, with no spurious oscillations, consistent with using the *exact* solution at every face rather than an approximate/dissipative one. The small overshoot visible right at the contact discontinuity in the density and velocity plots is a first-order spatial reconstruction artifact, expected, since the solver does not yet include a slope limiter (planned next step, see below).

## What's next for this solver

The roadmap for this project (documented in the repository) treats HLLC and MUSCL reconstruction as the natural next steps once RK2 was in place: the exact Riemann solver built here is deliberately kept as the **reference solution**, expensive per face (a Newton iteration at every single interface, every timestep) but useful precisely because it has no approximation to validate against. The next steps (in this order, each isolating a single new difficulty): a slope limiter (minmod/van Leer) with MUSCL reconstruction to remove the contact-discontinuity overshoot and reach second-order accuracy in space, then HLLC as the practical solver used in the rest of the Navier-Stokes progression, with the exact solver kept for validation.

## References

- Toro, E. F., *Riemann Solvers and Numerical Methods for Fluid Dynamics: A Practical Introduction*, Springer, 2009: chapters 3–4, the primary source for the exact Riemann solver implemented here (star region equations, Newton-Raphson iteration, sampling procedure).
- LeVeque, R. J., *Finite Volume Methods for Hyperbolic Problems*, Cambridge University Press, 2002: general finite volume / Godunov-method framework this project is built on.
- Sod, G. A., *A Survey of Several Finite Difference Methods for Systems of Nonlinear Hyperbolic Conservation Laws*, Journal of Computational Physics, 1978: original shock tube test case.
