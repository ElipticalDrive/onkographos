# Smoke Advection
Also called: MacCormack routine for smoke, Tracking additional quantities like density and temperature

## Overview
Smoke advection tracks scalar quantities such as density, temperature, or color that are transported by a fluid velocity field to visualize volumetric effects [S1, unofficial]. It solves the advection equation for these scalars using the same numerical scheme employed for the velocity field, typically the MacCormack method [S1, unofficial]. Unlike velocity, which drives the dynamics, these quantities are primarily visual; however, temperature can influence dynamics via buoyancy forces [S1, unofficial]. It is distinguished from pure velocity advection by the need to allocate separate storage for each scalar and potentially inject source terms, such as Gaussian splats, to define the smoke's appearance [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Velocity field $\mathbf{u}$, current scalar field $\phi$ (e.g., density, temperature, color), time step $\Delta t$, spatial grid dimensions $(N_x, N_y, N_z)$ [S1, unofficial].
- **Output:** Updated scalar field $\phi^{n+1}$ [S1, unofficial].

**Data Structures:**
- For each additional quantity $\phi$, allocate a texture or grid with the same dimensions as the velocity grid [S1, unofficial].
- The velocity field $\mathbf{u}$ is stored as a vector field over the grid [G].

**Method:**
The evolution of the scalar quantity $\phi$ is governed by the advection equation:
$$ \frac{\partial \phi}{\partial t} = -(\mathbf{u} \cdot \nabla) \phi $$
[S1, unofficial]
This equation is solved numerically using the MacCormack advection routine, a two-step predictor-corrector scheme [S2, unofficial].

1. **Predictor Step:** Calculate a temporary predicted value $\phi^*$ using forward differencing [S2, unofficial].
2. **Corrector Step:** Calculate the final corrected value $\phi^{n+1}$ using backward differencing on the predicted values [S2, unofficial].
3. **Source Injection:** To visualize smoke, a source term is added [S1, unofficial]. Each frame, a 3D Gaussian "splat" is injected into the color or density texture [S1, unofficial]. This provides a localized source of smoke that is then advected by the velocity field [S1, unofficial]. The injected values have no physical significance for density/temperature unless coupled with buoyancy, but create visual patterns [S1, unofficial].
4. **Buoyancy (Optional):** If temperature $T$ is tracked, it influences the velocity field via a buoyant force [S1, unofficial]. This requires coupling the scalar advection with the momentum equation [G]. The buoyant force is proportional to the temperature difference from a reference value [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def advect_scalar(phi, u, dt, grid_dims, dx, dy, dz):
    """
    Advect scalar field phi using MacCormack scheme with velocity u.
    phi: 3D array of shape (Nx, Ny, Nz)
    u: 3D vector field of shape (Nx, Ny, Nz, 3)
    dt: time step
    grid_dims: (Nx, Ny, Nz)
    dx, dy, dz: grid spacing
    """
    Nx, Ny, Nz = grid_dims
    phi_star = [[phi[i][j][k] for k in range(Nz)] for j in range(Ny)] for i in range(Nx)]  # Copy phi
    
    # Predictor step: forward differencing
    # Compute gradient of phi using forward differences
    dphi_dx = [[(phi[i+1][j][k] - phi[i][j][k]) / dx for k in range(Nz)] for j in range(Ny)] for i in range(Nx-1)]
    dphi_dy = [[(phi[i][j+1][k] - phi[i][j][k]) / dy for k in range(Nz)] for j in range(Ny-1)] for i in range(Nx)]
    dphi_dz = [[(phi[i][j][k+1] - phi[i][j][k]) / dz for k in range(Nz-1)] for j in range(Ny)] for i in range(Nx)]
    
    # Boundary fill (backward extrapolation)
    for j in range(Ny):
        for k in range(Nz):
            dphi_dx[Nx-1][j][k] = dphi_dx[Nx-2][j][k]
    for i in range(Nx):
        for k in range(Nz):
            dphi_dy[i][Ny-1][k] = dphi_dy[i][Ny-2][k]
    for i in range(Nx):
        for j in range(Ny):
            dphi_dz[i][j][Nz-1] = dphi_dz[i][j][Nz-2]
            
    # Flux: u . grad(phi)
    for i in range(Nx):
        for j in range(Ny):
            for k in range(Nz):
                flux = u[i][j][k][0] * dphi_dx[i][j][k] + u[i][j][k][1] * dphi_dy[i][j][k] + u[i][j][k][2] * dphi_dz[i][j][k]
                phi_star[i][j][k] -= dt * flux
                
    # Corrector step: backward differencing on phi_star
    dphi_star_dx = [[(phi_star[i][j][k] - phi_star[i-1][j][k]) / dx for k in range(Nz)] for j in range(Ny)] for i in range(1, Nx)]
    dphi_star_dy = [[(phi_star[i][j][k] - phi_star[i][j-1][k]) / dy for k in range(Nz)] for j in range(1, Ny)] for i in range(Nx)]
    dphi_star_dz = [[(phi_star[i][j][k] - phi_star[i][j][k-1]) / dz for k in range(1, Nz)] for j in range(Ny)] for i in range(Nx)]
    
    # Boundary fill (forward extrapolation)
    for j in range(Ny):
        for k in range(Nz):
            dphi_star_dx[0][j][k] = dphi_star_dx[1][j][k]
    for i in range(Nx):
        for k in range(Nz):
            dphi_star_dy[i][0][k] = dphi_star_dy[i][1][k]
    for i in range(Nx):
        for j in range(Ny):
            dphi_star_dz[i][j][0] = dphi_star_dz[i][j][1]
            
    # Flux for predicted state and final update
    phi_new = [[0.0]*Nz for _ in range(Ny)] for _ in range(Nx)]
    for i in range(Nx):
        for j in range(Ny):
            for k in range(Nz):
                flux_star = u[i][j][k][0] * dphi_star_dx[i][j][k] + u[i][j][k][1] * dphi_star_dy[i][j][k] + u[i][j][k][2] * dphi_star_dz[i][j][k]
                phi_new[i][j][k] = 0.5 * (phi[i][j][k] + phi_star[i][j][k] - dt * flux_star)
                
    return phi_new

def inject_smoke_source(texture, center, intensity, sigma, grid_dims, dx, dy, dz):
    """
    Inject a 3D Gaussian splat into the texture.
    """
    Nx, Ny, Nz = grid_dims
    cx, cy, cz = center
    rx = int(3 * sigma / dx)
    ry = int(3 * sigma / dy)
    rz = int(3 * sigma / dz)
    
    for i in range(max(0, int(cx/dx) - rx), min(Nx, int(cx/dx) + rx + 1)):
        for j in range(max(0, int(cy/dy) - ry), min(Ny, int(cy/dy) + ry + 1)):
            for k in range(max(0, int(cz/dz) - rz), min(Nz, int(cz/dz) + rz + 1)):
                dist_sq = ((i*dx - cx)**2 + (j*dy - cy)**2 + (k*dz - cz)**2)
                texture[i][j][k] += intensity * math.exp(-dist_sq / (2 * sigma**2))
```

**Parameters:**
- $\Delta t$: Time step [S1, unofficial].
- Grid resolution: $(N_x, N_y, N_z)$ [S1, unofficial].
- Gaussian parameters for source injection: position, intensity, spread ($\sigma$) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** Proper handling of boundaries is crucial for both the predictor and corrector steps [G].
- **Precision:** Floating-point precision can affect the accumulation of errors over many time steps [G].

## Subtopics

### [MacCormack Advection Scheme](4952-maccormack-advection-scheme.md)
The MacCormack advection scheme is a higher-order numerical method used to advect scalar fields (such as smoke density or temperature) and vector fields (velocity) in fluid simulations, designed to reduce the numerical smoothing inherent in standard semi-Lagrangian advection. It achieves this by combining a forward semi-Lagrangian step with a backward correction step, allowing low-resolution grids to retain visual detail comparable to significantly higher-resolution grids using standard first-order methods. Unlike the unconditionally stable semi-Lagrangian scheme, the MacCormack scheme is conditionally stable.

### Parallel Navier-Stokes Solver (MacCormack Method)
An explicit, two-step, three-time level scheme used to solve unsteady compressible Navier-Stokes equations, second-order accurate in both time and space, chosen for ease of parallelization [S2, unofficial]. The predictor step calculates a temporary predicted value using forward differencing, and the corrector step determines the final corrected value using backward differencing [S2, unofficial]. Forward and backward differencing are alternated between predictor and corrector steps to avoid bias and achieve second-order accuracy [S2, unofficial]. The data domain is subdivided into blocks, and at each time level, solution values at the outermost boundary layers of each block are exchanged between nearest neighbors after each predictor and corrector step [S2, unofficial]. An application programmer writes a `Prepare Data` routine to calculate flow properties to be visualized and store them in the appropriate format for rendering [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ per time step, where $N$ is the number of grid cells [G].
- **Memory Complexity:** $O(N)$ for the scalar field, with additional memory required for each tracked quantity [S1, unofficial].
- **Performance:** The method is efficient for real-time applications due to its simplicity and parallelizability [S2, unofficial].
- **Strengths:** Second-order accuracy in space and time, reduces numerical diffusion compared to first-order methods, easy to parallelize [S2, unofficial].
- **Limitations:** Conditionally stable, requires careful boundary handling [G].
- **When to use:** Suitable for real-time fluid simulations where visual quality is important and computational resources are limited [S1, unofficial].
- **When not to use:** Not suitable for highly compressible flows with strong shocks without modifications [G].

## Background
The concept of tracking additional quantities like density and temperature to visualize smoke was introduced by Fedkiw et al. (2001) [S1, unofficial]. The MacCormack method, originally developed for solving hyperbolic partial differential equations, has been adapted for fluid simulation due to its ease of implementation and parallelization [S2, unofficial]. It is widely used in computer graphics for real-time fluid simulation [S1, unofficial].

## Sources
[S1] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S2] DTIC ADA302573: Runtime Volume Visualization for Parallel CFD., https://archive.org/details/DTIC_ADA302573, fetched 2026-10-26 (unofficial)
