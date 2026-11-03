# Divergence Grid Computation
Also called: Divergence Calculation with Obstacle Velocity Boundary Conditions, Divergence Computation, Finite Difference Divergence

## Overview
Computes the scalar divergence field $\nabla \cdot \mathbf{v}$ for a 3D vector velocity grid using central finite differences [S1]. It quantifies local volume expansion or compression in fluid simulations, serving as a prerequisite for pressure projection steps [G]. It is distinguished from sibling methods like [Gradient-based Diffuse Lighting with Alpha Blending](1257-gradient-based-diffuse-lighting-with-alpha-blending.md) by its focus on vector field calculus rather than scalar shading, and from [Sobel Operator](1416-sobel-operator.md) by its specific application to fluid dynamics boundary conditions rather than general edge detection [S2].

## How it works
**Inputs:**
- A 3D vector grid $\mathbf{v}$ containing velocity components $(v_x, v_y, v_z)$ at voxel centers [S1].
- A grid tree structure defining active voxels and their coordinates [S1].
- Obstacle velocity data for boundary cells [S2].

**Outputs:**
- A scalar grid containing the divergence value for each active voxel [S1].
- Background value of zero for inactive voxels [S1].

**Data Structures:**
- **Grid Tree:** Stores active voxel coordinates [S1].
- **Velocity Field:** 3-component float vectors per active voxel [S1].
- **Divergence Field:** 1-component float scalar per active voxel [S1].

**Parameters:**
- `half`: Scalar multiplier $0.5$, representing $\frac{1}{2 \cdot \Delta x}$ in index space, assuming unit voxel spacing or normalized coordinates [S1].

**Steps:**
1. Initialize a gradient stencil with the grid tree [S1].
2. Iterate over all active voxels $(c, \dots)$ in the grid tree [S1].
3. For each active voxel, sample velocity vectors from its six immediate neighbors: Left, Right, Bottom, Top, Down, and Up [S2].
4. **Boundary Handling:** If a neighboring cell is a boundary (solid/obstacle) cell, replace the sampled velocity with the obstacle velocity at that specific location [S2].
5. Compute the divergence using central differences:
   $$ \text{div} = 0.5 \times ((v_R.x - v_L.x) + (v_T.y - v_B.y) + (v_U.z - v_D.z)) $$
   Where $v_R, v_L$ are velocities at Right and Left neighbors, $v_T, v_B$ at Top and Bottom, $v_U, v_D$ at Up and Down, and $x, y, z$ denote the respective velocity components [S2].
6. Store the computed divergence in a dictionary keyed by the current coordinate [S1].
7. Build the result grid using the computed data, a background value of zero, and the original grid's voxel size [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_divergence_grid(grid_tree, velocity_field, obstacle_velocity_field):
    """
    Computes divergence for a 3D velocity grid with obstacle boundary conditions.
    """
    divergence_grid = {}
    # Neighbor offsets: Left, Right, Bottom, Top, Down, Up
    # Mapping to (dx, dy, dz) assuming standard Cartesian indexing
    neighbors = {
        'L': (1, 0, 0), 'R': (-1, 0, 0),
        'B': (0, 1, 0), 'T': (0, -1, 0),
        'D': (0, 0, 1), 'U': (0, 0, -1)
    }
    
    for coord in grid_tree.active_voxels():
        x, y, z = coord
        v_L = get_velocity(velocity_field, obstacle_velocity_field, x + 1, y, z)
        v_R = get_velocity(velocity_field, obstacle_velocity_field, x - 1, y, z)
        v_B = get_velocity(velocity_field, obstacle_velocity_field, x, y + 1, z)
        v_T = get_velocity(velocity_field, obstacle_velocity_field, x, y - 1, z)
        v_D = get_velocity(velocity_field, obstacle_velocity_field, x, y, z + 1)
        v_U = get_velocity(velocity_field, obstacle_velocity_field, x, y, z - 1)
        
        div = 0.5 * ((v_R[0] - v_L[0]) + (v_T[1] - v_B[1]) + (v_U[2] - v_D[2]))
        divergence_grid[coord] = div
        
    return divergence_grid

def get_velocity(velocity_field, obstacle_velocity_field, x, y, z):
    coord = (x, y, z)
    # Check if neighbor is a boundary cell [G]
    # Sources do not specify the exact boundary detection predicate, 
    # so a direct lookup in the obstacle field is used as a standard implementation guard.
    if coord in obstacle_velocity_field:
        return obstacle_velocity_field[coord]
    return velocity_field.get(coord, (0.0, 0.0, 0.0))
```

**Edge Cases and Pitfalls:**
- **Boundary Cells:** Failure to replace neighbor velocities with obstacle velocities leads to incorrect divergence near solid objects, violating the no-penetration condition [S2].
- **Precision:** Using floating-point arithmetic for divergence accumulation may introduce small errors; standard float32 is typically sufficient for real-time simulation [G].
- **Grid Boundaries:** Voxels on the edge of the simulation domain may lack neighbors. The sources imply handling via obstacle velocity or zero-padding, but explicit edge handling logic is not detailed in the provided notes [S1, S2].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of active voxels. Each voxel requires 6 neighbor lookups and constant arithmetic operations [G].
- **Memory Complexity:** $O(N)$ for storing the resulting divergence grid [G].
- **Implementation:** The method is implemented in HLSL pixel shaders for real-time simulation [S2].
- **Strengths:** Simple, efficient, and directly applicable to fluid simulation pressure projection [G].
- **Limitations:** First-order accuracy in space (central difference is second-order accurate for the derivative, but the stencil is minimal). Sensitive to grid alignment and boundary condition implementation [G].

## Background
The method originates from real-time fluid simulation techniques, specifically those described in "Real-Time Simulation and Rendering of 3D Fluids" [S2]. It is a standard component in grid-based fluid solvers to compute the divergence field required for the Helmholtz-Hodge decomposition [G].

## Sources
[S1] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S2] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27
