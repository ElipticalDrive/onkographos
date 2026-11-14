# Divergence Grid Computation
Also called: Divergence Calculation with Obstacle Velocity Boundary Conditions, Divergence Computation, Finite Difference Divergence

## Overview
Computes the divergence $\nabla \cdot \mathbf{F} = \frac{\partial F_x}{\partial x} + \frac{\partial F_y}{\partial y} + \frac{\partial F_z}{\partial z}$ of a vector field $\mathbf{F}$ defined on a 3D grid using central differences [S1, S2, unofficial]. It is distinguished from sibling methods like [Laplacian Grid Computation](3160-laplacian-grid-computation.md) by operating on a vector field rather than a scalar field, and from [Mean Curvature Grid Computation](3126-mean-curvature-grid-computation.md) by computing the trace of the Jacobian rather than the Hessian. It is a core step in fluid simulation pressure projection to enforce incompressibility [S2, unofficial].

## How it works
**Inputs:**
- A 3D grid of vector values $\mathbf{v}$ (velocity) with dimensions $W \times H \times D$.
- A boundary mask or obstacle velocity field to handle solid cells [S2, unofficial].
- Voxel spacing $\Delta x, \Delta y, \Delta z$. In index space, $\Delta x = \Delta y = \Delta z = 1$, so the scaling factor is $0.5$ (i.e., $1/(2 \cdot 1)$) [S1, unofficial].

**Outputs:**
- A 3D grid of scalar values representing the divergence at each active voxel.
- Inactive/background voxels are set to zero [S1, unofficial].

**Data Structures:**
- The input velocity field is typically stored as a texture or array of `float4` (or `float3` with padding) where `.x`, `.y`, `.z` correspond to velocity components [S2, unofficial].
- A stencil object or direct coordinate indexing is used to access neighbors [S1, unofficial].

**Steps:**
1. Initialize a stencil or prepare to iterate over all active coordinates $c = (x, y, z)$ in the grid [S1, unofficial].
2. For each active voxel $c$:
    a. Identify the six immediate neighbors: Left ($L$), Right ($R$), Bottom ($B$), Top ($T$), Down ($D$), Up ($U$) [S2, unofficial].
    b. Sample the velocity vectors $\mathbf{v}_L, \mathbf{v}_R, \mathbf{v}_B, \mathbf{v}_T, \mathbf{v}_D, \mathbf{v}_U$ from these neighbor locations [S1, S2, unofficial].
    c. **Boundary Handling:** If a neighbor location corresponds to a boundary/solid cell, replace its sampled velocity with the pre-defined obstacle velocity at that location [S1, S2, unofficial].
    d. Compute the divergence using the central difference formula:
       $$ \text{div} = 0.5 \times \left( (v_{R,x} - v_{L,x}) + (v_{T,y} - v_{B,y}) + (v_{U,z} - v_{D,z}) \right) $$
       where $v_{R,x}$ is the x-component of the velocity at the Right neighbor, etc. [S1, S2, unofficial].
    e. Store the result in the output grid at coordinate $c$ [S1, unofficial].
3. Return the constructed divergence grid [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_divergence_grid(velocity_grid, obstacle_velocity_grid, is_boundary_func, grid_dims):
    """
    Computes divergence of a 3D velocity grid with obstacle boundary conditions.
    
    Args:
        velocity_grid: 3D array of shape (W, H, D), each element is a tuple/list (vx, vy, vz).
        obstacle_velocity_grid: 3D array of shape (W, H, D), each element is (vx, vy, vz).
        is_boundary_func: Function taking (x, y, z) returning True if cell is solid.
        grid_dims: Tuple (W, H, D).
        
    Returns:
        3D array of shape (W, H, D) containing scalar divergence values.
    """
    W, H, D = grid_dims
    divergence_grid = [[0.0 for _ in range(D)] for _ in range(H)]
    
    # Directions: Left, Right, Bottom, Top, Down, Up
    # Indices: (x-1, x+1), (y-1, y+1), (z-1, z+1)
    # Note: Assuming standard array indexing where x is first dim, y second, z third.
    # Neighbors relative to (x, y, z):
    # L: (x-1, y, z), R: (x+1, y, z)
    # B: (x, y-1, z), T: (x, y+1, z)
    # D: (x, y, z-1), U: (x, y, z+1)
    
    for z in range(D):
        for y in range(H):
            for x in range(W):
                # Sample neighbors
                neighbors = []
                # Left
                if is_boundary_func(x-1, y, z):
                    neighbors.append(obstacle_velocity_grid[x-1][y][z])
                else:
                    neighbors.append(velocity_grid[x-1][y][z])
                # Right
                if is_boundary_func(x+1, y, z):
                    neighbors.append(obstacle_velocity_grid[x+1][y][z])
                else:
                    neighbors.append(velocity_grid[x+1][y][z])
                # Bottom
                if is_boundary_func(x, y-1, z):
                    neighbors.append(obstacle_velocity_grid[x][y-1][z])
                else:
                    neighbors.append(velocity_grid[x][y-1][z])
                # Top
                if is_boundary_func(x, y+1, z):
                    neighbors.append(obstacle_velocity_grid[x][y+1][z])
                else:
                    neighbors.append(velocity_grid[x][y+1][z])
                # Down
                if is_boundary_func(x, y, z-1):
                    neighbors.append(obstacle_velocity_grid[x][y][z-1])
                else:
                    neighbors.append(velocity_grid[x][y][z-1])
                # Up
                if is_boundary_func(x, y, z+1):
                    neighbors.append(obstacle_velocity_grid[x][y][z+1])
                else:
                    neighbors.append(velocity_grid[x][y][z+1])
                
                # Unpack neighbors: [L, R, B, T, D, U]
                vL, vR, vB, vT, vD, vU = neighbors
                
                # Compute divergence
                # div = 0.5 * ((vR.x - vL.x) + (vT.y - vB.y) + (vU.z - vD.z))
                dx = vR[0] - vL[0]
                dy = vT[1] - vB[1]
                dz = vU[2] - vD[2]
                
                div = 0.5 * (dx + dy + dz)
                
                divergence_grid[y][x][z] = div
                
    return divergence_grid
```

**Parameters:**
- `half` (0.5): The scalar multiplier representing $1/(2 \cdot \text{voxel\_size})$ in index space [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Access:** Neighbors may lie outside the grid bounds. The sources imply that boundary checks (`IsBoundaryCell`) handle solid obstacles, but for the grid edge itself, standard clamping or zero-padding is assumed unless specified otherwise [G]. If a neighbor is outside the grid and not a defined obstacle, it should be treated as a boundary with zero velocity or handled via the grid's boundary condition policy [G].
- **Precision:** Using `float` (single precision) is standard for real-time simulation [S2, unofficial].
- **Inactive Voxels:** The output grid must explicitly set inactive/background voxels to zero [S1, unofficial].

## Subtopics

### Projection Step with Free-Slip Boundary Conditions
A variant used in fluid simulation to project velocity onto a divergence-free field while respecting free-slip boundaries [S2, unofficial].
- **What it is:** Computes the pressure gradient $\nabla p$ and updates velocity $\mathbf{v}_{new} = \mathbf{v}_{old} - \nabla p$, then enforces free-slip conditions by blending obstacle velocities [S2, unofficial].
- **What sets it apart:** It handles solid cells by copying the center pressure to neighbors and using obstacle velocities to mask out invalid pressure gradients, ensuring no flow penetrates or slips incorrectly at boundaries [S2, unofficial].
- **When to choose:** Use when simulating incompressible fluids with solid obstacles where free-slip (tangential flow allowed, normal flow zero) is required [S2, unofficial].
- **Link:** [N/A - Variant of Divergence Grid Computation]

**Steps for Projection Variant:**
1. If the current center cell is a boundary cell, return the obstacle velocity at that cell immediately [S2, unofficial].
2. Sample pressure values $p_C, p_L, p_R, p_B, p_T, p_D, p_U$ from the center and six neighbors [S2, unofficial].
3. Sample obstacle velocities $\mathbf{v}_L, \mathbf{v}_R, \dots$ from the six neighboring cells [S2, unofficial].
4. Initialize obstacle velocity accumulator $\mathbf{v}_{obst} = (0,0,0)$ and velocity mask $\mathbf{v}_{mask} = (1,1,1)$ [S2, unofficial].
5. For each neighbor:
    a. If the neighbor is a boundary cell:
        i. Set the neighbor's pressure to the center pressure ($p_{neighbor} = p_C$) [S2, unofficial].
        ii. Add the neighbor's obstacle velocity component to $\mathbf{v}_{obst}$ (e.g., if Left is boundary, $\mathbf{v}_{obst}.x += \mathbf{v}_L.x$) [S2, unofficial].
        iii. Set the corresponding mask component to 0 (e.g., $\mathbf{v}_{mask}.x = 0$) [S2, unofficial].
6. Compute the pressure gradient: $\nabla p = 0.5 \times (p_R - p_L, p_T - p_B, p_U - p_D)$ [S2, unofficial].
7. Sample the old velocity $\mathbf{v}_{old}$ [S2, unofficial].
8. Compute intermediate velocity: $\mathbf{v}_{new} = \mathbf{v}_{old} - \nabla p$ [S2, unofficial].
9. Enforce free-slip: $\mathbf{v}_{final} = (\mathbf{v}_{new} \odot \mathbf{v}_{mask}) + \mathbf{v}_{obst}$ [S2, unofficial].
10. Return $\mathbf{v}_{final}$ [S2, unofficial].

**Formula:**
$$ \nabla p = 0.5 \times (p_R - p_L, p_T - p_B, p_U - p_D) $$
$$ \mathbf{v}_{new} = \mathbf{v}_{old} - \nabla p $$
$$ \mathbf{v}_{final} = (\mathbf{v}_{new} \odot \mathbf{v}_{mask}) + \mathbf{v}_{obst} $$

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of active voxels, as each voxel requires a constant number of memory accesses and arithmetic operations [S1, S2, unofficial].
- **Memory Complexity:** $O(N)$ for the input velocity grid and output divergence grid [S1, S2, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Linear with grid resolution [S1, S2, unofficial].
- **Strengths:** Simple, efficient, and robust for incompressible fluid simulation [S2, unofficial].
- **Limitations:** First-order accuracy in space (central difference is second-order accurate for the derivative, but the stencil is small) [G]. Requires careful boundary handling for complex geometries [S2, unofficial].
- **When to use:** Real-time 3D fluid simulation, pressure projection steps [S2, unofficial].
- **When not:** When higher-order accuracy is required or for non-fluid scalar fields where other derivatives are needed [G].

## Background
The method originates from real-time 3D fluid simulation techniques, specifically those involving grid-based Navier-Stokes solvers with obstacle handling [S2, unofficial]. It is a standard finite difference approximation for divergence in computational fluid dynamics (CFD) [S1, S2, unofficial]. The specific implementation with obstacle velocity boundary conditions is detailed in GPU Gems 3 [S2, unofficial].

## Sources
[S1] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S2] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
