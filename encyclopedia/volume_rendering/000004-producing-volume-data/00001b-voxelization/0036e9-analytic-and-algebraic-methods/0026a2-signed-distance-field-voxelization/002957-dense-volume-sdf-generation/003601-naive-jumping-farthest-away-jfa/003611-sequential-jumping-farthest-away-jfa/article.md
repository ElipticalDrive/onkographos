# Sequential Jumping Farthest Away (JFA)

Also called: Sequential JFA (Jumping Farthest Away), SequentialJFA

## Overview
Sequential Jumping Farthest Away (JFA) is a sequential algorithm to compute the Signed Distance Field (SDF) and surface positions for a voxel grid by identifying boundary voxels and propagating distance information using a multi-scale search [S1, unofficial]. It differs from its parent [Naive Jumping Farthest Away (JFA)](7621-naive-jumping-farthest-away-jfa.md) by explicitly defining the sequential execution order and data layout for CPU implementation, and from [Breadth-First Distance Field Expansion](9703-breadth-first-distance-field-expansion.md) by using a geometric stride-based search rather than a queue-based expansion [G].

## How it works
The algorithm initializes boundary voxels with zero distance and infinite distance for interior voxels, then iteratively refines the SDF by searching neighbors at decreasing strides $k$ [S1, unofficial].

**Inputs and Outputs:**
- Input: A voxel grid `gridV` containing boolean occupancy (`Voxel`) and origin/size metadata [S1, unofficial].
- Output: Two grids of the same dimensions as `gridV`:
  - `sdf`: Float grid storing the signed distance value [S1, unofficial].
  - `positions`: 3D vector grid storing the world position of the closest surface point [S1, unofficial].

**Data Structures:**
- `gridV`: Contains `VoxelsPerSide()` (assumed cubic grid side length $N$), `VoxelSize()`, and origin offsets `OriginX()`, `OriginY()`, `OriginZ()` [S1, unofficial].
- `sdf`: 3D array `float[N][N][N]` [S1, unofficial].
- `positions`: 3D array of 3D vectors `vec3[N][N][N]` [S1, unofficial].
- `sdfApp`, `positionsApp`: Temporary buffers of the same layout, used to store updates for the current stride $k$ to avoid race conditions in sequential updates (though strictly sequential, the source uses a double-buffering pattern per stride) [S1, unofficial].

**Parameters:**
- $k$: The search stride. Starts at $N/2$ and halves until $k=1$ [S1, unofficial].

**Steps:**

1. **Initialization**:
   Iterate over all voxel coordinates $(voxelZ, voxelY, voxelX)$ from $0$ to $N-1$ [S1, unofficial].
   - If `!gridV.Voxel(voxelX, voxelY, voxelZ)` (voxel is empty), skip [S1, unofficial].
   - Calculate world position:
     $$ pos = \begin{bmatrix} gridV.OriginX() + voxelX \cdot gridV.VoxelSize() \\ gridV.OriginY() + voxelY \cdot gridV.VoxelSize() \\ gridV.OriginZ() + voxelZ \cdot gridV.VoxelSize() \end{bmatrix} $$ [S1, unofficial].
   - Check all 26 neighbors $(nx, ny, nz)$ where $nx, ny, nz \in \{-1, 0, 1\}$ excluding $(0,0,0)$ [S1, unofficial].
   - If any neighbor is out of bounds or `!gridV.Voxel(nx, ny, nz)`, the voxel is a boundary [S1, unofficial].
     - Set `sdf(voxelX, voxelY, voxelZ) = 0.0f` [S1, unofficial].
     - Set `positions(voxelX, voxelY, voxelZ) = pos` [S1, unofficial].
   - Else (voxel is interior, all neighbors occupied):
     - Set `sdf(voxelX, voxelY, voxelZ) = INFINITY` [S1, unofficial].

2. **Processing Loop**:
   - Initialize `sdfApp` and `positionsApp` as copies of `sdf` and `positions` [S1, unofficial].
   - Let $N = gridV.VoxelsPerSide()$.
   - Iterate $k$ from $N/2$ down to $1$, halving $k$ each iteration ($k = \lfloor k/2 \rfloor$) [S1, unofficial].

3. **Inner Loop (for each $k$)**:
   - For each voxel $(voxelZ, voxelY, voxelX)$:
     - Calculate `voxelPos` using the same formula as in Initialization [S1, unofficial].
     - Initialize `bestDistance = sdf(voxelX, voxelY, voxelZ)` [S1, unofficial].
     - Initialize `findNewBest = false` [S1, unofficial].
     - Iterate neighbor directions $(x, y, z)$ where $x, y, z \in \{-1, 0, 1\}$ excluding $(0,0,0)$ [S1, unofficial].
     - Calculate neighbor indices:
       $$ nx = voxelX + (x \cdot k) $$
       $$ ny = voxelY + (y \cdot k) $$
       $$ nz = voxelZ + (z \cdot k) $$ [S1, unofficial].
     - If $(nx, ny, nz)$ is out of bounds, continue [S1, unofficial].
     - Get `seed = sdf(nx, ny, nz)` [S1, unofficial].
     - If `fabs(seed) < INFINITY`:
       - Get `seedPos = positions(nx, ny, nz)` [S1, unofficial].
       - Calculate `distance = CalculateDistance(voxelPos, seedPos)` [S1, unofficial].
       - If `distance < fabs(bestDistance)`:
         - Set `findNewBest = true` [S1, unofficial].
         - Set `bestDistance = copysignf(distance, bestDistance)` [S1, unofficial].
         - Set `bestPosition = seedPos` [S1, unofficial].
     - If `findNewBest` is true:
       - Set `sdfApp(voxelX, voxelY, voxelZ) = bestDistance` [S1, unofficial].
       - Set `positionsApp(voxelX, voxelY, voxelZ) = bestPosition` [S1, unofficial].

4. **Update**:
   - After processing all voxels for the current $k$, update:
     - `sdf = sdfApp` [S1, unofficial].
     - `positions = positionsApp` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sequential_jfa(grid_v):
    N = grid_v.voxels_per_side()
    voxel_size = grid_v.voxel_size()
    origin_x = grid_v.origin_x()
    origin_y = grid_v.origin_y()
    origin_z = grid_v.origin_z()
    INFINITY = float('inf')
    
    # Initialize output grids
    sdf = [[[INFINITY for _ in range(N)] for _ in range(N)] for _ in range(N)]
    positions = [[[None for _ in range(N)] for _ in range(N)] for _ in range(N)]
    
    # Step 1: Initialization
    for z in range(N):
        for y in range(N):
            for x in range(N):
                if not grid_v.voxel(x, y, z):
                    continue
                
                pos = [
                    origin_x + x * voxel_size,
                    origin_y + y * voxel_size,
                    origin_z + z * voxel_size
                ]
                
                is_boundary = False
                for dx in [-1, 0, 1]:
                    for dy in [-1, 0, 1]:
                        for dz in [-1, 0, 1]:
                            if dx == 0 and dy == 0 and dz == 0:
                                continue
                            nx, ny, nz = x + dx, y + dy, z + dz
                            if not (0 <= nx < N and 0 <= ny < N and 0 <= nz < N):
                                is_boundary = True
                                break
                            if not grid_v.voxel(nx, ny, nz):
                                is_boundary = True
                                break
                        if is_boundary: break
                    if is_boundary: break
                
                if is_boundary:
                    sdf[z][y][x] = 0.0
                    positions[z][y][x] = pos
                else:
                    sdf[z][y][x] = INFINITY
    
    # Step 2: Processing Loop
    sdf_app = [[row[:] for row in grid] for grid in sdf]
    positions_app = [[row[:] for row in grid] for grid in positions]
    
    k = N // 2
    while k >= 1:
        # Step 3: Inner Loop
        for z in range(N):
            for y in range(N):
                for x in range(N):
                    voxel_pos = [
                        origin_x + x * voxel_size,
                        origin_y + y * voxel_size,
                        origin_z + z * voxel_size
                    ]
                    
                    best_distance = sdf[z][y][x]
                    find_new_best = False
                    best_position = positions[z][y][x]
                    
                    for dx in [-1, 0, 1]:
                        for dy in [-1, 0, 1]:
                            for dz in [-1, 0, 1]:
                                if dx == 0 and dy == 0 and dz == 0:
                                    continue
                                
                                nx = x + dx * k
                                ny = y + dy * k
                                nz = z + dz * k
                                
                                if not (0 <= nx < N and 0 <= ny < N and 0 <= nz < N):
                                    continue
                                
                                seed = sdf[nz][ny][nx]
                                if abs(seed) < INFINITY:
                                    seed_pos = positions[nz][ny][nx]
                                    distance = math.sqrt(
                                        (voxel_pos[0] - seed_pos[0])**2 +
                                        (voxel_pos[1] - seed_pos[1])**2 +
                                        (voxel_pos[2] - seed_pos[2])**2
                                    )
                                    
                                    if distance < abs(best_distance):
                                        find_new_best = True
                                        best_distance = math.copysign(distance, best_distance)
                                        best_position = seed_pos
                    
                    if find_new_best:
                        sdf_app[z][y][x] = best_distance
                        positions_app[z][y][x] = best_position
        
        # Step 4: Update
        sdf = sdf_app
        positions = positions_app
        
        k = k // 2
    
    return sdf, positions
```

**Edge Cases and Pitfalls:**
- **Interior Voxels**: Voxels with no empty neighbors are initialized to `INFINITY`. If the grid is fully occupied, no distance propagation occurs [S1, unofficial].
- **Boundary Conditions**: Neighbors outside the grid bounds are treated as empty, correctly identifying surface voxels [S1, unofficial].
- **Precision**: `copysignf` is used to preserve the sign of the distance while updating the magnitude. Floating-point precision may affect the exact distance value for large grids [G].

## Subtopics

### [Parallel Jumping Farthest Away (JFA)](7860-parallel-jumping-farthest-away-jfa.md)
Parallel Jumping Farthest Away (JFA) is an OpenMP-based parallelization of the Sequential JFA algorithm for computing Signed Distance Fields (SDFs) on voxel grids. It accelerates the distance propagation phase by parallelizing the inner triple loop over voxel coordinates, while keeping the outer iteration over the jump distance scale $k$ sequential. It differs from the Compute Shader variant by running on CPU multi-core architectures rather than GPU, and from the Sequential JFA by utilizing thread-level parallelism for the spatial updates.

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Performance**: Not given in the sources.
- **Strengths**: Not given in the sources.
- **Limitations**: Not given in the sources.
- **When to use**: Not given in the sources.
- **When not to use**: Not given in the sources.

## Background
Sequential Jumping Farthest Away (JFA) is derived from the broader class of Jumping Farthest Away algorithms for SDF generation. It is implemented in the `bigmat18/cuda-mesh-voxelization` repository [S1, unofficial]. The algorithm is part of a family of methods that include Naive JFA, Parallel JFA, and Tiled JFA, each optimizing the core propagation logic for different hardware architectures [G].

## Sources
[S1] bigmat18/cuda-mesh-voxelization, https://github.com/bigmat18/cuda-mesh-voxelization, fetched 2026-10-26 (unofficial)
