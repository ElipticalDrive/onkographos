# Smoke Source Injection

Also called: GenerateSmoke, SmokeSourceVolumetric

## Overview
Smoke Source Injection initializes a volumetric smoke source at a fixed grid location by setting density and velocity values within a defined spatial region [S1, S2, unofficial]. It solves the problem of introducing initial mass and momentum into the Navier-Stokes solver domain [S2, unofficial]. It differs from [Smoke Advection](4639-smoke-advection.md) by creating new data rather than moving existing data, and from the [General Fluid Simulation Update Loop](13472-general-fluid-simulation-update-loop.md) by being a boundary/initial condition step rather than a time-integration step.

## How it works
**Inputs:**
- Grid resolution $RES$ (assumed uniform or defined by dimensions) [S1, unofficial].
- Source center coordinates $(centerY, centerZ)$ [S1, unofficial].
- Source radius threshold, typically $RES/10$ [S1, unofficial].
- Density amount $dens$ [S1, S2, unofficial].
- Velocity vector $\mathbf{v}_{init}$ [S1, S2, unofficial].
- Simulation time step $dt$ (implicit in Unity context) [S2, unofficial].

**Outputs:**
- Updated density field $\rho$ [S1, S2, unofficial].
- Updated velocity field $\mathbf{u}$ [S1, S2, unofficial].

**Data Structures:**
- 3D arrays for density and velocity components (e.g., `_density`, `_velX`, `_velY`, `_velZ`) indexed by grid coordinates $(i, j, k)$ [S1, unofficial].
- In Unity implementations, these are often `RenderTexture` or `ComputeBuffer` objects accessed via GPU kernels [S2, unofficial].

**Steps:**
1.  Identify the grid cells belonging to the source region. The source is typically cylindrical or spherical, centered at $(centerY, centerZ)$ in the Y-Z plane, extending along the X-axis or defined by a specific slice [S1, unofficial].
2.  For each grid cell $(i, j, k)$ within the source bounds:
    a. Calculate the distance from the center in the relevant plane. For the 2D slice example in [S1, unofficial], this is $d = \sqrt{(i - centerY)^2 + (j - centerZ)^2}$.
    b. If $d < RES/10$, inject the source values [S1, unofficial].
3.  Set the density value:
    - In [S1, unofficial], $dens$ is generated randomly: $dens = \frac{\text{rand()} \% 1000}{1000.0}$.
    - In [S2, unofficial], $dens$ is a user-defined parameter `smoke_amount` [S2, unofficial].
4.  Set the velocity value:
    - In [S1, unofficial], a constant X-velocity is applied: $u_x = 2.0$.
    - In [S2, unofficial], a user-defined vector `smoke_initial_velocity` is applied [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inject_smoke_source(grid, center_y, center_z, radius_threshold, density_amount, velocity_vector):
    """
    Injects smoke density and velocity into the grid.
    
    Args:
        grid: Object containing density and velocity arrays (vel_x, vel_y, vel_z).
        center_y: Y-coordinate of the source center.
        center_z: Z-coordinate of the source center.
        radius_threshold: Radius of the source region (e.g., RES/10).
        density_amount: Scalar value for density injection.
        velocity_vector: Vector3 (vx, vy, vz) for velocity injection.
    """
    # Determine grid dimensions if not provided
    res_y = grid.height
    res_z = grid.depth
    
    # Iterate over the relevant grid cells
    # Note: Implementation details vary between CPU (S1) and GPU (S2)
    for i in range(res_y):
        for j in range(res_z):
            # Calculate 2D distance from center in Y-Z plane
            dist = math.sqrt((i - center_y)**2 + (j - center_z)**2)
            
            if dist < radius_threshold:
                # Inject density
                # [S1, unofficial] uses random density, [S2, unofficial] uses fixed amount. 
                # We use the parameter passed for generality.
                grid.density[i, j] = density_amount
                
                # Inject velocity
                # [S1, unofficial] sets only X-velocity. [S2, unofficial] sets full vector.
                grid.vel_x[i, j] = velocity_vector.x
                grid.vel_y[i, j] = velocity_vector.y
                grid.vel_z[i, j] = velocity_vector.z
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
- **Time Complexity:** $O(R^2)$ for a 2D slice injection, or $O(R^3)$ for a 3D volume, where $R$ is the grid resolution. The number of cells checked is proportional to the source volume [S1, S2, unofficial].
- **Memory Complexity:** $O(1)$ additional memory, as it modifies the grid in-place [S1, S2, unofficial].
- **Performance:** GPU implementations (e.g., Unity Compute Shaders) allow parallel injection across many threads, significantly speeding up the process for large grids [S2, unofficial]. CPU implementations are simpler but slower for high resolutions [S1, unofficial].
- **Strengths:** Simple to implement; allows for dynamic source placement [S2, unofficial].
- **Limitations:** Random density in [S1, unofficial] may lead to visual noise; fixed radius in [S1, unofficial] may not suit all artistic needs [S1, unofficial].

## Background
Smoke source injection is a fundamental component of fluid simulation, particularly in computer graphics for creating smoke effects. It originated from early grid-based fluid simulation techniques, such as those described by Jos Stam [S2, unofficial]. The specific implementations cited here are from open-source projects demonstrating real-time volumetric smoke simulation [S1, S2, unofficial].

## Sources
[S1] nevermoe/SmokeSimulation, https://github.com/nevermoe/SmokeSimulation, fetched 2026-10-26 (unofficial)
[S2] furkanunsal42/VolumetricSmoke, https://github.com/furkanunsal42/VolumetricSmoke, fetched 2026-10-26 (unofficial)
