# Pressure Poisson Solver with Solid Boundary Handling

Also called: Pressure Poisson Solver

## Overview
This method solves the pressure Poisson equation on a 3D grid to enforce incompressibility in fluid simulations, specifically handling solid boundaries by modifying the stencil weights. It differs from standard solvers by replacing neighbor pressures with the center pressure when neighbors are solid, effectively creating a zero-gradient boundary condition at solid interfaces [S1, unofficial].

## How it works
**Inputs:**
- `divergence`: 3D texture/array of divergence values ($d$).
- `pressure`: 3D texture/array of current pressure estimates ($p$).
- `is_solid`: 3D texture/array of boolean flags indicating solid cells.

**Outputs:**
- `pressure_new`: 3D texture/array of updated pressure values.

**Data Structures:**
- The simulation domain is a 3D grid. Each cell $(i, j, k)$ has neighbors at $(i\pm1, j, k)$, $(i, j\pm1, k)$, and $(i, j, k\pm1)$.
- `IsSolidCell(coord)` returns true if the cell at `coord` is solid.

**Steps:**
1. For each cell in the grid, sample the divergence $d_C$ and current pressure $p_C$ at the center.
2. Sample the pressure $p_N$ from each of the six neighbors ($N \in \{L, R, B, T, D, U\}$).
3. If a neighbor cell is solid, replace its pressure value $p_N$ with the center pressure $p_C$. This ignores the solid cell's pressure contribution, enforcing a Neumann boundary condition where the pressure gradient normal to the solid surface is zero [S1, unofficial].
4. Compute the new pressure using the discrete Poisson equation:
   $$ p_{new} = \frac{p_L + p_R + p_B + p_T + p_D + p_U - d_C}{6.0} $$
   where $p_L, p_R, p_B, p_T, p_D, p_U$ are the (possibly modified) neighbor pressures and $d_C$ is the center divergence [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def solve_pressure_poisson_step(divergence, pressure, is_solid, grid_dims):
    """
    Computes one iteration of the pressure Poisson solver with solid boundary handling.
    """
    new_pressure = create_empty_3d_array(grid_dims)
    
    # Define neighbor offsets: Left, Right, Bottom, Top, Down, Up
    neighbors = [
        (-1, 0, 0), (1, 0, 0),
        (0, -1, 0), (0, 1, 0),
        (0, 0, -1), (0, 0, 1)
    ]
    
    for x in range(grid_dims.x):
        for y in range(grid_dims.y):
            for z in range(grid_dims.z):
                center_idx = (x, y, z)
                
                # Get divergence and pressure at center
                d_c = divergence[center_idx]
                p_c = pressure[center_idx]
                
                sum_neighbors = 0.0
                
                for dx, dy, dz in neighbors:
                    nx, ny, nz = x + dx, y + dy, z + dz
                    
                    # Handle boundaries: assume non-solid, non-existent cells have 0 pressure?
                    # The source implies sampling with point clamp. 
                    # If out of bounds, behavior depends on sampler. 
                    # Assuming valid grid indices for core logic.
                    if not (0 <= nx < grid_dims.x and 0 <= ny < grid_dims.y and 0 <= nz < grid_dims.z):
                        # Fallback for out-of-bounds: treat as solid or zero? 
                        # Source uses samPointClamp, which clamps to edge.
                        # For simplicity in implementation, we clamp coordinates.
                        nx = clamp(nx, 0, grid_dims.x - 1)
                        ny = clamp(ny, 0, grid_dims.y - 1)
                        nz = clamp(nz, 0, grid_dims.z - 1)
                        
                    neighbor_idx = (nx, ny, nz)
                    p_n = pressure[neighbor_idx]
                    
                    # Solid boundary handling
                    if is_solid[neighbor_idx]:
                        p_n = p_c
                    
                    sum_neighbors += p_n
                
                new_pressure[center_idx] = (sum_neighbors - d_c) / 6.0
                
    return new_pressure
```

**Parameters:**
- $d_C$: Divergence at the center cell [S1, unofficial].

**Edge Cases:**
- **Solid Cells:** The solver typically skips updating pressure in solid cells or uses them only as boundary conditions. The source implies the kernel runs on fluid cells, checking neighbors for solidity [S1, unofficial].
- **Boundaries:** The source uses `samPointClamp`, which clamps coordinates to the grid edges. This effectively treats the grid boundary as a solid wall with zero velocity/pressure gradient [S1, unofficial].

## Subtopics

### [Pressure Gradient Flow Control](20270-pressure-gradient-flow-control.md)
Manipulates surface flow direction by setting pressure values in surface cells. It allows direct artistic control over topology without altering viscosity, distinct from internal pressure controllers.

## Operating characteristics
- **Complexity:** $O(N)$ per iteration, where $N$ is the number of grid cells.
- **Performance:** Suitable for real-time simulation due to its simple stencil and parallelizability on GPU [S1, unofficial].
- **Limitations:** Single Jacobi-like iteration provides only an approximation; convergence requires multiple iterations or a multigrid solver for high accuracy. The solid boundary handling is approximate (zero-gradient) and may not capture complex solid geometries perfectly without higher resolution.

## Background
This method is derived from real-time 3D fluid simulation techniques, specifically as described in GPU Gems 3 [S1, unofficial]. It is a standard component in grid-based Navier-Stokes solvers for computer graphics.

## Sources
[S1] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
