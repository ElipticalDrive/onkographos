# GPU-based Eulerian Fluid Solver

Also called: Dense Eulerian Fluid Solver, Eulerian Fluid Simulation, Ignis Solver

## Overview
A GPU-based Eulerian fluid solver that simulates and renders volumetrics at VR resolutions and refresh rates using operator splitting to approximately solve the Navier-Stokes equations [S1]. It sets itself apart by utilizing half-precision 3D textures for state storage and a specific geometric multigrid configuration to achieve interactive frame rates on consumer hardware, building on [Stable Fluids (GPU Implementation)](7906-stable-fluids-gpu-implementation.md).

## How it works
**Inputs and Outputs:**
*   **Inputs:** Voxel position, timestep ($\Delta t$), and scene-specific parameters (e.g., buoyancy coefficients).
*   **Outputs:** Updated velocity, density, and temperature fields.

**Data Structures:**
State is tracked at each grid cell center. The fields are:
1.  **Velocity:** 3-component vector.
2.  **Density:** Scalar.
3.  **Temperature:** Scalar.

These are stored in 3D textures using 16-bit float formats (half-precision) [S1].

**Numbered Steps:**
1.  **Injection:** A custom injection kernel receives voxel position and timestep. It adds density and temperature to the simulation and applies arbitrary forces, such as buoyancy [S1].
2.  **Advection:** Semi-Lagrangian advection is performed using a MacCormack backtrace [S1].
    *   Velocity is sampled using linear interpolation during the backtrace.
    *   All advected properties (velocity, density, temperature) are resampled using cubic interpolation [S1].
    *   Vorticity confinement force is optionally applied [S1].
3.  **Projection:** Pressure projection is solved using geometric multigrid with a single V-cycle [S1].
    *   Smoothing uses Jacobi iteration.
    *   The V-cycle consists of 1 pre-smooth pass and 2 post-smooth passes [S1].
    *   Transfer operators are full-weight (prolongation) and trilinear (restriction) [S1].
    *   On the 3rd, coarsest grid, red-black Gauss-Seidel with successive over-relaxation (SOR) is used for the solve [S1].

**Parameters:**
*   **Voxel scale:** 1.5 cm [S1].
*   **Multigrid levels:** 3 levels (implied by "3rd, coarsest grid") [S1].
*   **Smoothing passes:** 1 pre-smooth, 2 post-smooth [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def step_simulation(grid, dt):
    # 1. Injection
    # Custom kernel adds density, temperature, and forces (e.g., buoyancy)
    # based on voxel position and dt.
    apply_injection(grid, dt)

    # 2. Advection
    # Semi-Lagrangian advection with MacCormack backtrace
    advect_velocity_mac_cormack(grid, dt)
    advect_scalars_cubic(grid, dt)
    
    # Optional vorticity confinement
    apply_vorticity_confinement(grid, dt)

    # 3. Projection
    # Geometric multigrid V-cycle
    solve_pressure_multigrid(grid)

def advect_velocity_mac_cormack(grid, dt):
    # Backtrace step: sample velocity using linear interpolation
    # Forward step: resample using cubic interpolation
    # (Implementation details of MacCormack specific to this solver)
    pass

def solve_pressure_multigrid(grid):
    # Single V-cycle
    # Pre-smoothing: 1 Jacobi pass
    jacobi_smooth(grid, iterations=1)
    
    # Coarsen using trilinear transfer
    coarse_grid = restrict(grid)
    
    # Solve on coarsest grid (3rd level) using red-black Gauss-Seidel with SOR
    solve_coarsest_grid_red_black_sor(coarse_grid)
    
    # Prolongate using full-weight transfer
    grid.residual += prolongate(coarse_grid)
    
    # Post-smoothing: 2 Jacobi passes
    jacobi_smooth(grid, iterations=2)
```

**Edge Cases and Pitfalls:**
*   **Precision:** Half-precision storage is used; the source notes negligible visual impact but acknowledges bandwidth constraints [S1].
*   **Resolution:** Performance is limited by low simulation resolution necessary for VR on current hardware [S1].

## Subtopics

### Free-Slip Boundary Condition Enforcement
Handles boundary conditions for fluid velocity. [Free-Slip Boundary Condition Enforcement](7908-free-slip-boundary-condition-enforcement.md)

### Projection Method
The mathematical framework for enforcing incompressibility. [Projection Method](8336-projection-method.md)

### Operator Splitting for Fluid Dynamics
The technique of separating advection, diffusion, and pressure steps. [Operator Splitting for Fluid Dynamics](12465-operator-splitting-for-fluid-dynamics.md)

## Operating characteristics
**Time and Memory Complexity:**
*   Memory: 3D textures with half-precision floats for velocity, density, and temperature [S1].
*   Time: Linear with respect to grid size per stage, dominated by advection and multigrid projection [S1].

**Measured Performance:**
Tested on NVIDIA RTX 3080 with a 1.5 cm voxel scale [S1]:
*   **Smoke Rings (232 x 232 x 160):** Total 9.63 ms.
*   **Dust Devil (160 x 256 x 160):** Total 9.65 ms.
*   **Wall of Fire (200 x 200 x 200):** Total 9.73 ms.
All scenarios meet the 11.11 ms budget for 90 Hz VR [S1].

**Strengths and Limitations:**
*   **Strengths:** Capable of simulating and rendering at VR resolutions and refresh rates [S1]. Half-precision storage doubles performance on bandwidth-constrained portions [S1].
*   **Limitations:** Limited by low simulation resolution necessary to run in VR on current hardware [S1].

**When to use:**
Interactive VR volumetric simulation where frame rate is critical.

**When not to use:**
High-fidelity offline rendering requiring higher resolution or thin flame combustion (not currently supported) [S1].

## Background
Originated by Charlie Shenton in 2023 with the "Ignis" solver, presented at SIGGRAPH Asia 2023 Posters [S1]. It builds on Stam's operator splitting [S1] and Selle's MacCormack advection [S1].

## Sources
[S1] Ignis: Eulerian Fluid Simulation and Rendering at VR Frame Rates, https://doi.org/10.1145/3610542.3626120, fetched 2026-10-30
