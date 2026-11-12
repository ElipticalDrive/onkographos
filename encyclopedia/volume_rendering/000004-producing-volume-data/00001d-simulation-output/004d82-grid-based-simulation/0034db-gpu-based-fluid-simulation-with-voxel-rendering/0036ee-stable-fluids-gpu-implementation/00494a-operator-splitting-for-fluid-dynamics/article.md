# Operator Splitting for Fluid Dynamics

Also called: Advection-Force-Projection Splitting, Dimensional Splitting (Fluids), Strang Splitting

## Overview
Operator splitting decomposes the complex dynamic equations of fluid dynamics into sequential, simpler sub-steps to facilitate numerical integration on a grid [S1, unofficial]. It solves the problem of enforcing incompressibility while applying external forces and advecting the fluid state. It sets itself apart from the parent [Stable Fluids (GPU Implementation)](7906-stable-fluids-gpu-implementation.md) by explicitly structuring the time integration into three distinct, ordered phases: advection, force application, and pressure projection [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: Velocity field $\mathbf{u}_{t-1}$ at time step $t-1$, external force field $\mathbf{f}$, time step size $\Delta t$, and volume field (density/temperature) $\phi_{t-1}$.
*   Output: Velocity field $\mathbf{u}_t$ and volume field $\phi_t$ at time step $t$.

**Data Structures:**
*   Velocity field: 3D grid of vectors $(u, v, w)$.
*   Volume field: 3D grid of scalars.
*   Pressure field: 3D grid of scalars (intermediate).

**Steps:**
1.  **Advection:** Move the fluid quantities (velocity and volume field) along the velocity field $\mathbf{u}_{t-1}$. This is performed in the Lagrangian view by back-tracing particles. For each grid cell, trace the particle position back by $\Delta t$ using $\mathbf{u}_{t-1}$, then interpolate the quantity from the source cells to the current cell [S1, unofficial].
2.  **Force Application:** Add the external force to the advected velocity field.
    $$ \mathbf{u}^* = \mathbf{u}_{advected} + \Delta t \cdot \mathbf{f} $$
    Where $\mathbf{u}^*$ is the intermediate velocity field.
3.  **Pressure Projection:** Rectify $\mathbf{u}^*$ to make it divergence-free ($\nabla \cdot \mathbf{u}_t = 0$). This involves solving for pressure $p$ such that:
    $$ \nabla^2 p = \frac{1}{\Delta t} \nabla \cdot \mathbf{u}^* $$
    Then update the velocity:
    $$ \mathbf{u}_t = \mathbf{u}^* - \Delta t \nabla p $$
    [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def operator_splitting_step(u_t_minus_1, f, phi_t_minus_1, dt):
    # Step 1: Advection
    # Back-trace positions using u_t_minus_1
    source_positions = get_source_positions(u_t_minus_1, dt)
    u_advected = interpolate_field(u_t_minus_1, source_positions)
    phi_t = interpolate_field(phi_t_minus_1, source_positions)

    # Step 2: Apply External Force
    u_star = u_advected + dt * f

    # Step 3: Pressure Projection
    # Compute divergence of u_star
    div_u_star = compute_divergence(u_star)
    
    # Solve Poisson equation for pressure p
    # Laplacian(p) = div_u_star / dt
    p = solve_poisson(div_u_star / dt)
    
    # Update velocity to be divergence-free
    grad_p = compute_gradient(p)
    u_t = u_star - dt * grad_p
    
    return u_t, phi_t
```

**Parameters:**
*   $\Delta t$: Time step size. Not given in the sources.
*   Interpolation method: Typically trilinear or bilinear. Not given in the sources.

**Edge Cases:**
*   If $\mathbf{u}_{t-1}$ is zero, advection does not move particles; values remain unchanged.
*   Boundary conditions for the pressure projection solver are critical for correctness. Not given in the sources.

## Subtopics

### [Neural Euler Fluid Simulation](12482-neural-euler-fluid-simulation.md)
Replaces the linear pressure projection solver with a ConvNet to regress velocity corrections directly from the intermediate velocity and volume field, enabling differentiable rendering and free-surface handling.

## Operating characteristics
*   **Time Complexity:** Dominated by the pressure projection step, which typically requires an iterative solver (e.g., Conjugate Gradient) or FFT. Not given in the sources.
*   **Memory Complexity:** Requires storage for velocity, volume, and pressure fields. Not given in the sources.
*   **Strengths:** Modular; allows independent optimization of advection, force, and projection steps. Stable for incompressible flows when projection is accurate.
*   **Limitations:** Splitting errors can accumulate if $\Delta t$ is large. The projection step is computationally expensive.
*   **When to use:** When simulating incompressible fluids where mass conservation is critical.
*   **When not to use:** For compressible flows or when real-time performance is required and approximate methods are acceptable.

## Background
Originated from the need to efficiently solve the Navier-Stokes equations on fixed grids. The specific application in neural fluid dynamics was detailed in "Inferring Fluid Dynamics via Inverse Rendering" [S1, unofficial]. It builds on the Stable Fluids framework [7906-stable-fluids-gpu-implementation.md].

## Sources
[S1] Inferring Fluid Dynamics via Inverse Rendering, https://doi.org/10.48550/arxiv.2304.04446, fetched 2026-10-26 (unofficial)
