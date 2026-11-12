# Voxel-based 3D Fire Simulation

Also called: 3D Voxel Fire Simulation

## Overview
Voxel-based 3D Fire Simulation is a grid-based method that solves the incompressible Navier-Stokes equations on a structured 3D voxel grid to simulate the dynamics of fire, including velocity, smoke density, temperature, and fuel levels [S1, unofficial]. It distinguishes itself from general fluid solvers by explicitly coupling buoyancy forces driven by temperature gradients and vorticity confinement to restore curling smoke behavior lost in discrete simulations [S1, unofficial]. Unlike [GPU-based Fluid Simulation with Voxel Rendering](7400-gpu-based-fluid-simulation-with-voxel-rendering.md) which may focus on general fluid dynamics, this method specifically targets the thermodynamic properties required for fire rendering [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A 3D voxel grid of resolution $(N_x, N_y, N_z)$, time step $\Delta t$, and physical constants ($P, m, g, R, T_0, \epsilon$).
*   **Outputs:** Updated 3D textures for velocity ($u$), pressure ($p$), density ($\rho$), temperature ($T$), and fuel level.

**Data Structures:**
The simulation state is stored in dedicated 3D textures or arrays. Each voxel stores scalar fields for temperature, density, and fuel, and a vector field for velocity. The grid size used in the primary implementation is $128 \times 128 \times 128$ [S1, unofficial].

**Formulas:**
The core dynamics are governed by the Navier-Stokes equation for incompressible flow [S1, unofficial]:
$$
\frac{\partial u}{\partial t} = -(u \cdot \nabla) u - \frac{1}{\rho} \nabla p + f
$$
Where:
*   $u$ is the velocity vector.
*   $p$ is the pressure.
*   $\rho$ is the density of the molecule mass.
*   $f$ is the external force field.
*   The term $-(u \cdot \nabla) u$ represents advection.
*   The term $-\frac{1}{\rho} \nabla p$ represents pressure gradients generating force [S1, unofficial].

The external force $f$ consists of buoyancy and vorticity confinement [S1, unofficial]:
1.  **Buoyancy Force:**
    $$
    f_{buoyancy} = \frac{Pmg}{R} \left(\frac{1}{T_0} - \frac{1}{T}\right)z
    $$
    Where:
    *   $P$ is pressure (constant reference).
    *   $m$ is molecular mass.
    *   $g$ is gravitational acceleration.
    *   $R$ is the gas constant.
    *   $T_0$ is the reference temperature.
    *   $T$ is the current voxel temperature.
    *   $z$ is the vertical direction vector.
    Higher temperatures result in larger upward velocities [S1, unofficial].

2.  **Vorticity Confinement Force:**
    $$
    f_{vc} = \epsilon (\Psi \times \omega) \delta x
    $$
    Where:
    *   $\epsilon$ is a scaling factor.
    *   $\omega$ is the vorticity vector ($\nabla \times u$).
    *   $\delta x$ is the grid spacing.
    *   $\Psi$ is the normalized gradient of the vorticity magnitude:
        $$
        \Psi = \frac{\eta}{|\eta|}
        $$
    *   $\eta$ is the gradient of the magnitude of vorticity:
        $$
        \eta = \nabla |\omega|
        $$
    This force restores curling smoke behavior lost due to numerical dissipation [S1, unofficial].

**Algorithm Steps:**
1.  **Initialize:** Set initial values for velocity, pressure, temperature, and fuel in the 3D grid.
2.  **Compute Forces:** Calculate buoyancy force $f_{buoyancy}$ based on current temperature $T$ and vorticity confinement force $f_{vc}$ based on vorticity $\omega$ [S1, unofficial].
3.  **Advection:** Compute advection $-(u \cdot \nabla) u$ for velocity and other scalar attributes (smoke, fuel) based on interactions between neighboring voxels [S1, unofficial].
4.  **Pressure Projection:** Solve for pressure $p$ to enforce incompressibility (divergence-free velocity field) and compute the pressure gradient term $-\frac{1}{\rho} \nabla p$ [S1, unofficial].
5.  **Update Velocity:** Integrate the Navier-Stokes equation to update velocity $u$ using the computed forces and pressure [S1, unofficial].
6.  **Update Scalars:** Propagate changes in temperature, smoke density, and fuel levels corresponding to the new velocity field [S1, unofficial].
7.  **Render:** Pass the updated 3D textures to the rendering pipeline for ray marching or voxel rasterization [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def simulate_fire_step(grid, dt, params):
    # grid: object containing 3D arrays for u (velocity), p (pressure), 
    #       T (temperature), rho (density), fuel, omega (vorticity)
    # params: dict containing P, m, g, R, T0, epsilon, delta_x
    
    # 1. Compute Vorticity
    omega = curl(grid.u)
    
    # 2. Compute Vorticity Confinement Force
    eta = gradient(magnitude(omega))
    Psi = normalize(eta)
    f_vc = params.epsilon * cross(Psi, omega) * params.delta_x
    
    # 3. Compute Buoyancy Force
    # z is the vertical unit vector (0, 1, 0)
    buoyancy_factor = (params.P * params.m * params.g / params.R) * (1.0/params.T0 - 1.0/grid.T)
    f_buoyancy = buoyancy_factor * z
    
    # Total external force
    f = f_buoyancy + f_vc
    
    # 4. Advect Velocity
    # Advection term: -(u . nabla) u
    u_advected = grid.u - dt * convect(grid.u, grid.u)
    
    # 5. Pressure Projection
    # Solve Poisson equation for pressure to make velocity divergence-free
    # This step typically involves an iterative solver (e.g., Jacobi)
    p_new = solve_pressure_poisson(u_advected, grid.rho)
    
    # 6. Update Velocity
    # u_new = u_advected - (dt / rho) * nabla p
    grad_p = gradient(p_new)
    u_new = u_advected - (dt / grid.rho) * grad_p
    
    # 7. Update Scalars (Temperature, Fuel, Smoke)
    # Advect scalars using the new velocity field
    grid.T = advect_scalar(grid.T, u_new, dt)
    grid.fuel = advect_scalar(grid.fuel, u_new, dt)
    grid.smoke = advect_scalar(grid.smoke, u_new, dt)
    
    # 8. Update Velocity Field
    grid.u = u_new
    
    return grid
```

**Parameters:**
*   Voxel grid size: $128 \times 128 \times 128$ [S1, unofficial].
*   GPU block size: $16 \times 16 \times 16$ [S1, unofficial].
*   Threads per block: 512 [S1, unofficial].

**Pitfalls:**
*   **HLSL Global Variables:** In HLSL compute shaders, global variables cannot be assigned within the shader file using an assignment operator during declaration; this results in zero values without a syntax error. The variable must be manually assigned in the host driver script (e.g., C#) before each invocation of the compute kernels [S1, unofficial].
*   **Debugging:** Debugging attribute propagation across timestamps is difficult due to the lack of print debugging in shaders. Using debug shaders that fill 3D textures based on thresholds or external tools like RenderDoc is recommended [S1, unofficial].

## Subtopics

### Stam and Fiume Fire Animation
A variant that applies the diffusion method to the animation of fire and other gaseous phenomena [S2]. It is noted for its application to complex scenes, though the cost of progressive shooting of energy from surface patches can be expensive, potentially requiring a hierarchical approach [S2].

## Operating characteristics
*   **Performance:** The method achieved real-time performance (60+ fps) on an NVIDIA RTX 2060 graphics card with a $128^3$ voxel grid [S1, unofficial].
*   **Complexity:** The simulation involves solving the Navier-Stokes equations, which typically requires iterative solvers for pressure projection, making it computationally intensive per time step.
*   **Strengths:** Realistic fire dynamics through explicit coupling of temperature-driven buoyancy and vorticity confinement [S1, unofficial].
*   **Limitations:** Debugging is challenging due to the nature of GPU shaders [S1, unofficial]. The method relies on a structured grid, which may limit geometric complexity compared to particle-based methods, though it benefits from efficient voxel rendering [S1, unofficial].

## Background
The method is based on the Navier-Stokes equations for fluid dynamics [S1, unofficial]. The specific implementation described in the primary source was developed for a 3D fire simulation project, utilizing GPU compute shaders for real-time performance [S1, unofficial]. The use of diffusion methods for fire animation has been explored in earlier work, such as by Stam and Fiume [S2].

## Sources
[S1] Fire Simulation and Rendering - GitHub Pages, https://cs184-firesim.github.io/final-report/, fetched 2026-10-27 (unofficial)
[S2] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
