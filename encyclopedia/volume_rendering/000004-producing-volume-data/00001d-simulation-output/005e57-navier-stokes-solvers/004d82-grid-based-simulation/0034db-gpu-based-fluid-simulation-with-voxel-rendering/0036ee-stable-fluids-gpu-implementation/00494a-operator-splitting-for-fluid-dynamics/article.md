# Operator Splitting for Fluid Dynamics

Also called: Advection-Force-Projection Splitting, Dimensional Splitting (Fluids), Strang Splitting

## Overview
Operator splitting decomposes the complex dynamic equations of fluid dynamics into sequential, simpler sub-steps to facilitate numerical integration on a grid [S1, unofficial]. It solves the problem of enforcing incompressibility while applying external forces and advecting the fluid state [S1, unofficial]. It sets itself apart from the parent [Stable Fluids (GPU Implementation)](7906-stable-fluids-gpu-implementation.md) by explicitly structuring the time integration into three distinct, ordered phases: advection, force application, and pressure projection [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: Velocity field $\mathbf{u}_{t-1}$ at time step $t-1$, external force field $\mathbf{f}$, time step size $\Delta t$, and volume field $\phi_{t-1}$ [G].
*   Output: Velocity field $\mathbf{u}_t$ and volume field $\phi_t$ at time step $t$ [G].

**Data Structures:**
*   Velocity field: 3D grid of vectors $(u, v, w)$ [G].
*   Volume field: 3D grid of scalars [G].
*   Pressure field: 3D grid of scalars (intermediate) [G].

**Steps:**
1.  **Advection:** Move the fluid quantities (velocity and volume field) along the velocity field $\mathbf{u}_{t-1}$ [S1, unofficial]. This is performed by back-tracing particle positions using $\mathbf{u}_{t-1}$ and interpolating quantities from the source cells to the current cell [S1, unofficial].
2.  **Force Application:** Add the external force to the advected velocity field [S1, unofficial].
3.  **Pressure Projection:** Rectify the velocity field to make it divergence-free ($\nabla \cdot \mathbf{u}_t = 0$) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def operator_splitting_step(u_t_minus_1, f, phi_t_minus_1, dt, grid_shape):
    # Step 1: Advection
    # Back-trace positions: source_pos = current_grid_coords - u_t_minus_1 * dt
    source_pos = np.indices(grid_shape).transpose(1,2,3,0).astype(float) - u_t_minus_1 * dt
    # Interpolate u_t_minus_1 and phi_t_minus_1 at source_pos using trilinear interpolation [G]
    u_advected = trilinear_interpolate(u_t_minus_1, source_pos)
    phi_t = trilinear_interpolate(phi_t_minus_1, source_pos)

    # Step 2: Force Application
    u_star = u_advected + dt * f

    # Step 3: Pressure Projection
    # Compute divergence of u_star using central finite differences [G]
    div_u_star = compute_divergence(u_star)
    # Solve Poisson equation Laplacian(p) = div_u_star / dt using an iterative solver [G]
    p = solve_poisson(div_u_star / dt)
    # Compute gradient of p using central finite differences and subtract from u_star [G]
    grad_p = compute_gradient(p)
    u_t = u_star - dt * grad_p

    return u_t, phi_t
```

**Parameters:**
*   $\Delta t$: Not given in the sources.
*   Interpolation method: Not given in the sources.

**Edge Cases:**
*   If $\mathbf{u}_{t-1}$ is zero, advection does not move particles; values remain unchanged [G].
*   Boundary conditions for the pressure projection solver are critical for correctness [G].

## Subtopics

### [Neural Euler Fluid Simulation](12482-neural-euler-fluid-simulation.md)
Replaces the linear pressure projection solver with a ConvNet to regress velocity corrections directly from the intermediate velocity and volume field, enabling differentiable rendering and free-surface handling.

## Operating characteristics
Not given in the sources.

## Background
The specific application in neural fluid dynamics was detailed in "Inferring Fluid Dynamics via Inverse Rendering" [S1, unofficial].

## Sources
[S1] Inferring Fluid Dynamics via Inverse Rendering, https://doi.org/10.48550/arxiv.2304.04446, fetched 2026-10-26 (unofficial)
