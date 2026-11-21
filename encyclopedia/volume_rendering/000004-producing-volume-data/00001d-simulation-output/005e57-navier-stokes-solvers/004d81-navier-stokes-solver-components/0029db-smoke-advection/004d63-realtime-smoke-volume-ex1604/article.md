# Realtime Smoke Volume (Ex1604)

## Overview
Realtime Smoke Volume (Ex1604) is a hybrid rendering technique that combines low-resolution velocity projection with high-resolution density advection to visualize smoke in real-time [S1]. It solves the performance bottleneck of simulating and rendering dense volumetric smoke by decoupling the fluid dynamics resolution from the visual density resolution [S1]. Unlike standard full-resolution Navier-Stokes solvers, it projects velocity on a coarse grid to maintain stability and mass conservation, then advects a finer density field using the same MacCormack routine as velocity, finally ray marching the density in a volume pixel shader [S1].

## How it works
**Inputs and Outputs:**
- Input: Low-resolution velocity field (projected), high-resolution density field, external sources (optional).
- Output: Rendered smoke volume via ray marching in a pixel shader.

**Data Structures:**
- Velocity Field: Stored on a low-resolution 3D grid. Fields for $u, v, w$ components.
- Density Field: Stored on a high-resolution 3D grid. Field for scalar density $\rho$.
- Texture3D: Both fields are typically uploaded to GPU textures for compute shader access and ray marching sampling.

**Steps:**
1. **Velocity Projection:** Perform a projection step on the low-resolution velocity field to ensure divergence-free conditions. This maintains fluid stability at a lower computational cost [S1].
2. **Density Advection:** Advect the high-resolution density field using the low-resolution velocity field. The advection uses the MacCormack scheme, consistent with the velocity advection method [S1].
3. **Ray Marching:** In the volume pixel shader, ray march through the high-resolution density field to compute the final color and opacity [S1].

**Pitfalls:**
- **DiffUpSample Misconception:** The `DiffUpSample` technique reflects low-resolution differences to the high-resolution field before or after projection. It is **not** a physical diffusion solve [S1]. Implementers must not treat it as a substitute for actual diffusion if physical accuracy is required.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Realtime Smoke Volume (Ex1604)
# Based on [S1]

def realtime_smoke_volume_step(low_res_velocity, high_res_density, dt):
    """
    Performs one step of the Realtime Smoke Volume simulation.
    
    Args:
        low_res_velocity: 3D array of velocity vectors (u, v, w) on low-res grid.
        high_res_density: 3D array of scalar density values on high-res grid.
        dt: Time step.
        
    Returns:
        Updated high_res_density field.
    """
    
    # Step 1: Low-resolution velocity projection
    # Ensure velocity is divergence-free on the low-res grid
    low_res_velocity = project_velocity(low_res_velocity)
    
    # Step 2: High-resolution density advection
    # Advect density using the low-res velocity field via MacCormack scheme
    # Note: The velocity field must be interpolated to the high-res grid coordinates
    high_res_density = advect_density_mac_cormack(
        high_res_density, 
        low_res_velocity, 
        dt
    )
    
    # Step 3: Return updated density for ray marching
    return high_res_density

def project_velocity(velocity_field):
    """
    Projects velocity field to be divergence-free.
    Uses a standard projection method (e.g., Gauss-Seidel relaxation for pressure solve).
    [G] Standard projection algorithm assumed as not detailed in [S1].
    """
    # Placeholder for projection logic
    # 1. Compute divergence of velocity
    # 2. Solve Poisson equation for pressure
    # 3. Subtract gradient of pressure from velocity
    return velocity_field

def advect_density_mac_cormack(density, velocity, dt):
    """
    Advects density using the MacCormack scheme.
    [G] MacCormack scheme involves predictor and corrector steps.
    """
    # Predictor step
    density_pred = density - dt * divergence(density * velocity)
    
    # Corrector step
    # Interpolate velocity and density_pred to appropriate staggered grids
    density_corrected = 0.5 * (density + density_pred - dt * divergence(velocity * density_pred))
    
    return density_corrected
```

## Subtopics

### Smoke Simulation Step
A generic simulation step that updates the smoke density field by adding external sources, optionally diffusing, advecting, and applying a fixed per-cell decay [S2, unofficial]. It is a more complete physical simulation step compared to the hybrid approach of Ex1604, which skips diffusion and decay in its core description [S1]. Use this when physical accuracy (diffusion, decay) is required, rather than just visual advection [S2, unofficial].
- **Definition:** Updates density field via source addition, optional diffusion, optional advection, and fixed decay [S2, unofficial].
- **Steps:**
    1. Add external sources: `AddSource(sd, _density)` [S2, unofficial].
    2. If compiled with `DIFFUSE`, swap `_densityTmp` and `_density`, then diffuse: `Diffuse(0, _densityTmp, _density, diffusion)` [S2, unofficial].
    3. If compiled with `ADVECT`, swap `_densityTmp` and `_density`, then advect using velocity components: `Advect(0, _densityTmp, _density, _velX, _velY, _velZ)` [S2, unofficial].
    4. Iterate over all cells, subtract 0.001 from density, and clamp to zero if negative: `FOR_ALL_CELL { _density[_I(k,j,i)] -= 0.001; if(_density[_I(k,j,i)] < 0) _density[_I(k,j,i)] = 0; } END_FOR` [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Dominated by the low-resolution velocity projection and high-resolution density advection. The projection is $O(N_{low})$ where $N_{low}$ is the number of low-res cells. The advection is $O(N_{high})$ where $N_{high}$ is the number of high-res cells. Ray marching is $O(N_{rays} \times N_{steps})$.
- **Memory Complexity:** Requires storage for both low-res velocity and high-res density fields.
- **Measured Performance:** Specific performance metrics are not provided in the sources [S1]. Storyboard times are given for Ex1604 (4.025s, 13.417s, 22.808s frames), but these include rendering and are not isolated simulation times [S1].
- **Strengths:** Real-time performance due to low-res velocity projection. High visual fidelity due to high-res density advection.
- **Limitations:** Not physically accurate for diffusion or decay. Relies on the quality of the low-res velocity field for stability.
- **When to Use:** Real-time visualization of smoke where performance is critical and physical accuracy of diffusion/decay is not required.
- **When Not to Use:** When physical accuracy of diffusion or decay is required.

## Background
The Realtime Smoke Volume technique is presented as part of a fluid simulation demo series (Ex1604) in a graphics study archive [S1]. It builds on the MacCormack Advection Scheme [4952-maccormack-advection-scheme.md] for advection and the general Navier-Stokes Solver Components [4639-smoke-advection.md] for velocity projection [S1]. The technique is designed to balance performance and visual quality for real-time applications [S1].

## Sources
[S1] Part4 Chapter16 Fluid Simulation Demo, https://github.com/Razria93/Graphics_Study_Archive/issues/33, fetched 2026-10-26
[S2] nevermoe/SmokeSimulation, https://github.com/nevermoe/SmokeSimulation, fetched 2026-10-26 (unofficial)
