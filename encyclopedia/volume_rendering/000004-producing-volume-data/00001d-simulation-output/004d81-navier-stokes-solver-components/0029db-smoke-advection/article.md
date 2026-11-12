# Smoke Advection

Also called: MacCormack routine for smoke, Tracking additional quantities like density and temperature

## Overview
Smoke advection tracks scalar quantities (such as density, temperature, or color) that are transported by a fluid velocity field to visualize smoke effects [S1, unofficial]. It solves the problem of visualizing fluid motion, which is otherwise invisible when only velocity is rendered [S1, unofficial]. It is distinguished from general fluid updates by focusing on the transport of passive (or buoyant) scalars rather than the momentum and pressure solves required for velocity projection [S1, unofficial]. It builds on [Navier-Stokes Solver Components](13501-navier-stokes-solver-components.md).

## How it works
**Inputs:**
*   Velocity field $\mathbf{u}$ (vector field).
*   Scalar field $\phi$ (e.g., density, temperature, color) to be advected.
*   Source terms for $\phi$ (e.g., smoke injection).

**Outputs:**
*   Updated scalar field $\phi$ at the next time step.

**Data Structures:**
*   For each additional quantity $\phi$, allocate a texture/grid with the same dimensions as the velocity grid [S1, unofficial].
*   The scalar field $\phi$ is stored in a 3D texture or grid structure compatible with the simulation grid [S1, unofficial].

**Steps:**
1.  **Source Injection:** Inject a source into the scalar field. For visual smoke, this is often a 3D Gaussian "splat" into a color texture each frame [S1, unofficial].
2.  **Advection:** Advect the scalar field $\phi$ using the velocity field $\mathbf{u}$ [S1, unofficial]. The evolution is governed by the advection equation:
    $$ \frac{\partial \phi}{\partial t} = -(\mathbf{u} \cdot \nabla) \phi $$
    [S1, unofficial]
3.  **Numerical Scheme:** Use the same advection routine used for velocity, typically the MacCormack scheme [S1, unofficial].
    *   **Predictor Step:** Calculate a temporary predicted value using forward differencing [S2, unofficial].
    *   **Corrector Step:** Determine the final corrected value using backward differencing [S2, unofficial].
    *   **Alternation:** Alternate forward and backward differencing between predictor and corrector steps to avoid bias and achieve second-order accuracy in time and space [S2, unofficial].
4.  **Buoyancy (Optional):** If temperature $T$ is tracked, it influences dynamics via buoyant force, causing hot smoke to rise and cool smoke to fall [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def advect_smoke(phi, u, dt, grid_dims):
    """
    Advect scalar field phi using velocity u via MacCormack scheme.
    phi: 3D array of scalar values (density/temperature/color)
    u: 3D vector field (velocity)
    dt: time step
    grid_dims: (nx, ny, nz)
    """
    # Predictor step: forward differencing
    # Calculate temporary predicted value phi_star
    # phi_star = phi - dt * (u . grad(phi)) using forward differences
    phi_star = predictor_step(phi, u, dt, grid_dims)

    # Corrector step: backward differencing
    # Use phi_star to calculate final corrected value phi_new
    # phi_new = 0.5 * (phi + phi_star - dt * (u . grad(phi_star)) using backward differences)
    phi_new = corrector_step(phi, phi_star, u, dt, grid_dims)

    return phi_new

def predictor_step(phi, u, dt, grid_dims):
    nx, ny, nz = grid_dims
    phi_star = copy(phi)
    # Forward differencing for gradient
    # dphi_dx = (phi[i+1,j,k] - phi[i,j,k]) / dx
    # dphi_dy = (phi[i,j+1,k] - phi[i,j,k]) / dy
    # dphi_dz = (phi[i,j,k+1] - phi[i,j,k]) / dz
    # u_dot_grad_phi = u_x * dphi_dx + u_y * dphi_dy + u_z * dphi_dz
    # phi_star[i,j,k] = phi[i,j,k] - dt * u_dot_grad_phi[i,j,k]
    # Note: Boundary conditions must be handled for i+1, j+1, k+1
    return phi_star

def corrector_step(phi, phi_star, u, dt, grid_dims):
    nx, ny, nz = grid_dims
    phi_new = copy(phi)
    # Backward differencing for gradient of phi_star
    # dphi_star_dx = (phi_star[i,j,k] - phi_star[i-1,j,k]) / dx
    # dphi_star_dy = (phi_star[i,j,k] - phi_star[i,j-1,k]) / dy
    # dphi_star_dz = (phi_star[i,j,k] - phi_star[i,j,k-1]) / dz
    # u_dot_grad_phi_star = u_x * dphi_star_dx + u_y * dphi_star_dy + u_z * dphi_star_dz
    # phi_new[i,j,k] = 0.5 * (phi[i,j,k] + phi_star[i,j,k] - dt * u_dot_grad_phi_star[i,j,k])
    # Note: Boundary conditions must be handled for i-1, j-1, k-1
    return phi_new
```

**Parameters:**
*   Time step $dt$: Must satisfy stability conditions for the MacCormack scheme (conditionally stable) [S1, S2, unofficial].
*   Grid resolution: Higher resolution reduces numerical smoothing inherent in first-order methods [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Stability:** The MacCormack scheme is conditionally stable and requires a limiter to prevent numerical blow-up [S1, unofficial].
*   **Bias:** Alternating forward and backward differencing is required to avoid bias from one-sided differencing [S2, unofficial].
*   **Physical Plausibility:** Color values have no physical significance, but temperature does, affecting buoyancy [S1, unofficial].

## Subtopics

### [MacCormack Advection Scheme](4952-maccormack-advection-scheme.md)
The MacCormack advection scheme is a higher-order numerical method used to advect scalar fields (such as smoke density or temperature) and vector fields (velocity) in fluid simulations, designed to reduce the numerical smoothing inherent in standard semi-Lagrangian advection. It achieves this by combining a forward semi-Lagrangian step with a backward correction step, allowing low-resolution grids to retain visual detail comparable to significantly higher-resolution grids using standard first-order methods. Unlike the unconditionally stable semi-Lagrangian scheme, the MacCormack scheme is conditionally stable and requires a limiter to prevent numerical blow-up.

### Parallel Navier-Stokes Solver (MacCormack Method)
An explicit, two-step, three-time level scheme used to solve unsteady compressible Navier-Stokes equations, second-order accurate in both time and space, chosen for ease of parallelization [S2, unofficial]. It subdivides the data domain into blocks, exchanging solution values at the outermost boundary layers of each block between nearest neighbors after each predictor and corrector step [S2, unofficial]. A 'Prepare Data' routine calculates flow properties to be visualized and stores them in the appropriate format for rendering [S2, unofficial].

### Realtime Smoke Volume (Ex1604)
Combines low-resolution velocity projection with high-resolution density advection, then ray marches density in a volume pixel shader [S3]. A pitfall is that `DiffUpSample` reflects low-resolution difference to high-resolution field before/after projection; it is not a physical diffusion solve [S3].

## Operating characteristics
*   **Time Complexity:** Second-order accurate in time and space [S2, unofficial].
*   **Memory Complexity:** Requires additional texture/grid storage for each scalar quantity $\phi$ with the same dimensions as the velocity grid [S1, unofficial].
*   **Performance:** The MacCormack method is easier to parallelize and implement than more sophisticated CFD codes [S2, unofficial].
*   **Strengths:** Reduces numerical smoothing compared to first-order methods, allowing low-resolution grids to retain visual detail [S1, unofficial].
*   **Limitations:** Conditionally stable, requiring careful time step selection and limiters [S1, unofficial]. Not unconditionally stable like semi-Lagrangian schemes [S1, unofficial].
*   **When to Use:** When visual detail is important and computational resources are limited, or when parallelization is a priority [S1, S2, unofficial].
*   **When Not to Use:** When unconditional stability is required or when high physical accuracy in diffusion is needed (e.g., `DiffUpSample` is not a physical diffusion solve) [S1, unofficial][S3].

## Background
The concept of tracking additional quantities like density and temperature to visualize smoke was introduced by Fedkiw et al. in 2001 [S1, unofficial]. The MacCormack method, used for this advection, is a classical finite-difference method [S2, unofficial]. It has been applied in parallel CFD simulations on architectures like the Intel Paragon [S2, unofficial] and in real-time graphics demos combining low-resolution velocity with high-resolution density [S3].

## Sources
[S1] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S2] DTIC ADA302573: Runtime Volume Visualization for Parallel CFD., https://archive.org/details/DTIC_ADA302573, fetched 2026-10-26 (unofficial)
[S3] Part4 Chapter16 Fluid Simulation Demo, https://github.com/Razria93/Graphics_Study_Archive/issues/33, fetched 2026-10-26
