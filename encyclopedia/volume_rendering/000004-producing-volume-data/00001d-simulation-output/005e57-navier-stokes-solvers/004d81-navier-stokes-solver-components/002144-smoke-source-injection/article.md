# Smoke Source Injection

Also called: GenerateSmoke, SmokeSourceVolumetric

## Overview
Smoke Source Injection initializes a volumetric smoke source at a fixed grid location by setting density and velocity values within a defined spatial region [S1, S2, unofficial]. It solves the problem of introducing initial mass and momentum into the Navier-Stokes solver domain [S2, unofficial]. It differs from [Smoke Advection](4639-smoke-advection.md) by creating new data rather than moving existing data, and from the [General Fluid Simulation Update Loop](13472-general-fluid-simulation-update-loop.md) by being a boundary/initial condition step rather than a time-integration step [G].

## How it works
**Inputs:**
- Grid resolution $RES$ [S1, unofficial].
- Source center coordinates $(centerY, centerZ)$ [S1, unofficial].
- Source radius threshold $RES/10$ [S1, unofficial].
- Density amount $dens$ [S1, S2, unofficial].
- Velocity vector $\mathbf{v}_{init}$ [S1, S2, unofficial].
- Simulation time step $dt$ [S2, unofficial].

**Outputs:**
- Updated density field $\rho$ [S1, S2, unofficial].
- Updated velocity field $\mathbf{u}$ [S1, S2, unofficial].

**Data Structures:**
- 3D arrays for density and velocity components (e.g., `_density`, `_velX`) indexed by grid coordinates $(i, j, k)$ [S1, unofficial].
- In Unity implementations, these are `RenderTexture` or `ComputeBuffer` objects accessed via GPU kernels [S2, unofficial].

**Steps:**
1. Iterate over grid indices $i$ from $1$ to $N$ and $j$ from $1$ to $N$ [S1, unofficial].
2. Calculate the 2D distance from the center in the Y-Z plane: $d = \text{hypot}(i - centerY, j - centerZ)$ [S1, unofficial].
3. If $d < RES/10$, inject the source values at grid index $(5, i, j)$ [S1, unofficial].
4. Set the density value: $dens = (\text{rand()} \% 1000) / 1000.0f$ for range $[0.0, 1.0)$ [S1, unofficial], or use a user-defined `smoke_amount` parameter [S2, unofficial].
5. Set the velocity value: apply a constant X-velocity of $2.0$ [S1, unofficial], or use a user-defined `smoke_initial_velocity` vector [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def inject_smoke_source(grid, center_y, center_z, radius_threshold, density_amount, velocity_vector):
    """
    Injects smoke density and velocity into the grid.
    Implements the logic from [S1, unofficial] and [S2, unofficial].
    """
    res_y = grid.height
    res_z = grid.depth
    
    for i in range(1, res_y + 1):
        for j in range(1, res_z + 1):
            dist = math.hypot(i - center_y, j - center_z)
            
            if dist < radius_threshold:
                grid.density[5, i, j] = density_amount
                grid.vel_x[5, i, j] = velocity_vector.x
                grid.vel_y[5, i, j] = velocity_vector.y
                grid.vel_z[5, i, j] = velocity_vector.z
```

**Parameters:**
- `centerY`: Y-axis center of the smoke source. Typical value: $RES/4$ [S1, unofficial].
- `centerZ`: Z-axis center of the smoke source. Typical value: $RES/2$ [S1, unofficial].
- `smoke_amount`: Amount of density to add. Type: float [S2, unofficial].
- `smoke_initial_velocity`: Velocity vector to add. Type: Vector3 [S2, unofficial].
- `active`: Boolean flag. Defined in [S2, unofficial] but noted as not used in the provided snippet [S2, unofficial].

**Edge Cases:**
- Precision: Random density generation in [S1, unofficial] uses integer modulo arithmetic, which may introduce bias if not handled carefully [S1, unofficial].
- Boundaries: The source region must be clamped to the grid boundaries to avoid out-of-bounds access [G].

## Subtopics

### Smoke Advection
- **What it is:** Moves existing smoke density and velocity fields through the grid.
- **Difference:** Transports data; does not create new mass.
- **When to choose:** For time-stepping after injection.
- **Link:** [Smoke Advection](4639-smoke-advection.md)

### General Fluid Simulation Update Loop
- **What it is:** The complete time-step cycle including advection, diffusion, and pressure projection.
- **Difference:** Orchestrates the entire physics update; injection is a sub-step or initial condition.
- **When to choose:** For the main simulation loop.
- **Link:** [General Fluid Simulation Update Loop](13472-general-fluid-simulation-update-loop.md)

## Operating characteristics
- Time and memory complexity: Not given in the sources.
- Measured performance: GPU implementations (e.g., Unity Compute Shaders) allow parallel injection across many threads, significantly speeding up the process for large grids [S2, unofficial]. CPU implementations are simpler but slower for high resolutions [S1, unofficial].
- Strengths: Simple to implement; allows for dynamic source placement [S2, unofficial].
- Limitations: Random density in [S1, unofficial] may lead to visual noise; fixed radius in [S1, unofficial] may not suit all artistic needs [S1, unofficial].

## Background
The specific implementations cited here are from open-source projects demonstrating real-time volumetric smoke simulation [S1, S2, unofficial].

## Sources
[S1] nevermoe/SmokeSimulation, https://github.com/nevermoe/SmokeSimulation, fetched 2026-10-26 (unofficial)
[S2] furkanunsal42/VolumetricSmoke, https://github.com/furkanunsal42/VolumetricSmoke, fetched 2026-10-26 (unofficial)
