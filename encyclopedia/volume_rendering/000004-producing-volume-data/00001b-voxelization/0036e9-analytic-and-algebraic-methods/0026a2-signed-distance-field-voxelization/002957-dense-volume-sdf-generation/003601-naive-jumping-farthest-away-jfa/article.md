# Naive Jumping Farthest Away (JFA)

Also called: InizializationNaive, Naive Jumping Farthest Away (JFA) Initialization, Naive Jumping Farthest Away (JFA) Processing, NaiveJFA, ProcessingNaive

## Overview
Naive Jumping Farthest Away (JFA) is a GPU-based algorithm for computing the Signed Distance Field (SDF) and the position of the closest surface point for a dense 3D voxel grid [S1, unofficial]. It solves the problem of efficiently propagating distance information from known surface voxels to empty space using a multi-pass approach where the search radius $K$ is halved in each iteration [S1, unofficial]. It differs from its sibling, Cube SDF Grid Construction, by using a logarithmic halving stride strategy rather than a fixed local neighborhood expansion, and from its child, Sequential JFA, by executing the propagation loop in parallel on the GPU rather than sequentially on the CPU [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A dense 3D voxel grid representing a mesh or solid [S1, unofficial]. The grid is defined by its side length $N$ (assumed cubic, $N \times N \times N$), voxel size, and origin [S1, unofficial]. Each voxel is marked as occupied or empty [S1, unofficial].
*   **Outputs:** Two 3D arrays of size $N \times N \times N$:
    1.  `SDF`: Stores the signed distance value for each voxel [S1, unofficial].
    2.  `Positions`: Stores the 3D physical coordinates of the closest surface point for each voxel [S1, unofficial].

**Data Structures:**
The grid is accessed via a view object (e.g., `DeviceVoxelsGrid` or `VoxelsGrid`) which provides methods to access voxel occupancy (`Voxel(x,y,z)` returns boolean), grid dimensions (`VoxelsPerSide()`), voxel size (`VoxelSize()`), and origin (`OriginX()`, `OriginY()`, `OriginZ()`) [S1, unofficial]. The SDF and Position arrays are stored in device memory (e.g., `CudaPtr<T>`) for GPU access [S1, unofficial].

**Parameters:**
*   $N$: The number of voxels per side of the grid [S1, unofficial].
*   $K$: The current search radius in voxel units [S1, unofficial]. It starts at $N/2$ and is halved in each iteration until it reaches 1 [S1, unofficial].

**Algorithm Steps:**
The algorithm consists of two phases: Initialization and Processing Loop [S1, unofficial].

1.  **Initialization (`InizializationNaive`):**
    *   Each thread corresponds to one voxel $(x, y, z)$ [S1, unofficial].
    *   Compute the linear thread index from $(x, y, z)$ or vice versa [S1, unofficial].
    *   Check if the voxel is occupied [S1, unofficial]. If not, terminate the thread [S1, unofficial].
    *   Compute the physical position `pos` of the current voxel:
        $$ \text{pos} = (\text{OriginX} + x \cdot \text{VoxelSize}, \text{OriginY} + y \cdot \text{VoxelSize}, \text{OriginZ} + z \cdot \text{VoxelSize}) $$ [S1, unofficial].
    *   Identify if the voxel is a **boundary voxel** [S1, unofficial]. A boundary voxel is an occupied voxel that has at least one empty or out-of-bounds neighbor in its 26-neighborhood [S1, unofficial].
    *   Iterate through the 26 neighbors (offsets $dx, dy, dz \in \{-1, 0, 1\}$, excluding $0,0,0$) [S1, unofficial].
    *   Calculate neighbor coordinates $(nx, ny, nz) = (x+dx, y+dy, z+dz)$ [S1, unofficial].
    *   If any neighbor is out of bounds or empty, set a flag `found` to true [S1, unofficial].
    *   If `found` is true, set `SDF[x,y,z] = 0.0f` and `Positions[x,y,z] = pos` [S1, unofficial].
    *   Otherwise, set `SDF[x,y,z] = INFINITY` [S1, unofficial].

2.  **Processing Loop (`ProcessingNaive`):**
    *   Initialize $K = N / 2$ [S1, unofficial].
    *   While $K \ge 1$:
        *   Launch the `ProcessingNaive` kernel in parallel for all voxels [S1, unofficial].
        *   Inside the kernel for voxel $(x, y, z)$:
            *   If the voxel is not occupied, terminate [S1, unofficial].
            *   Initialize `bestDistance = SDF[x,y,z]` and `bestPosition` (undefined or current pos) [S1, unofficial].
            *   Initialize `findNewBest = false` [S1, unofficial].
            *   Compute physical position `voxelPos` of $(x, y, z)$ [S1, unofficial].
            *   Iterate through the $K$-neighborhood offsets $(dx, dy, dz) \in \{-1, 0, 1\}^3 \setminus \{(0,0,0)\}$ [S1, unofficial].
            *   Calculate neighbor grid coordinates:
                $$ nx = x + dx \cdot K $$
                $$ ny = y + dy \cdot K $$
                $$ nz = z + dz \cdot K $$ [S1, unofficial].
            *   **Boundary Check:** If $nx < 0$ or $nx \ge N$ or $ny < 0$ or $ny \ge N$ or $nz < 0$ or $nz \ge N$, continue to next neighbor [S1, unofficial].
            *   **Neighbor Evaluation:**
                *   Get `seed = SDF[nx, ny, nz]` [S1, unofficial].
                *   If `fabs(seed) < INFINITY`: Get `seedPos = Positions[nx, ny, nz]` [S1, unofficial].
                *   Calculate Euclidean distance:
                    $$ \text{distance} = \sqrt{(voxelPos.x - seedPos.x)^2 + (voxelPos.y - seedPos.y)^2 + (voxelPos.z - seedPos.z)^2} $$ [S1, unofficial].
                *   If `distance < fabs(bestDistance)`: Set `findNewBest = true` [S1, unofficial]. Update `bestDistance = copysignf(distance, bestDistance)` (preserving sign of current best distance) [S1, unofficial]. Update `bestPosition = seedPos` [S1, unofficial].
            *   **Update Output:** If `findNewBest` is true, write `SDF[x,y,z] = bestDistance` and `Positions[x,y,z] = bestPosition` [S1, unofficial].
        *   Update $K = K / 2$ (integer division) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def initialize_naive(grid, sdf_array, pos_array):
    """
    Initializes SDF and Position arrays for Naive JFA.
    grid: Object providing Voxel(x,y,z), VoxelsPerSide(), VoxelSize(), OriginX/Y/Z()
    sdf_array: 3D array of floats, initialized to 0
    pos_array: 3D array of Positions, initialized to 0
    """
    N = grid.VoxelsPerSide()
    
    # Parallel execution assumed over all voxels
    for z in range(N):
        for y in range(N):
            for x in range(N):
                if not grid.Voxel(x, y, z):
                    continue # Skip empty voxels [S1, unofficial]
                
                # Compute physical position [S1, unofficial]
                pos_x = grid.OriginX() + x * grid.VoxelSize()
                pos_y = grid.OriginY() + y * grid.VoxelSize()
                pos_z = grid.OriginZ() + z * grid.VoxelSize()
                current_pos = (pos_x, pos_y, pos_z)
                
                # Check 26-neighborhood for boundary condition [S1, unofficial]
                is_boundary = False
                for dx in [-1, 0, 1]:
                    for dy in [-1, 0, 1]:
                        for dz in [-1, 0, 1]:
                            if dx == 0 and dy == 0 and dz == 0:
                                continue
                            
                            nx, ny, nz = x + dx, y + dy, z + dz
                            
                            # Check bounds [S1, unofficial]
                            if nx < 0 or nx >= N or ny < 0 or ny >= N or nz < 0 or nz >= N:
                                is_boundary = True
                                break
                            if not grid.Voxel(nx, ny, nz):
                                is_boundary = True
                                break
                        if is_boundary: break
                    if is_boundary: break
                
                if is_boundary:
                    sdf_array[x, y, z] = 0.0
                    pos_array[x, y, z] = current_pos
                else:
                    sdf_array[x, y, z] = float('inf')
                    pos_array[x, y, z] = current_pos 

def process_naive_iteration(grid, sdf_array, pos_array, K):
    """
    Performs one iteration of Naive JFA with jump distance K.
    """
    N = grid.VoxelsPerSide()
    
    # Parallel execution assumed over all voxels
    for z in range(N):
        for y in range(N):
            for x in range(N):
                if not grid.Voxel(x, y, z):
                    continue
                
                # Current best [S1, unofficial]
                best_dist = sdf_array[x, y, z]
                best_pos = pos_array[x, y, z]
                find_new_best = False
                
                # Physical position of current voxel [S1, unofficial]
                curr_x = grid.OriginX() + x * grid.VoxelSize()
                curr_y = grid.OriginY() + y * grid.VoxelSize()
                curr_z = grid.OriginZ() + z * grid.VoxelSize()
                voxel_pos = (curr_x, curr_y, curr_z)
                
                # Search K-neighborhood [S1, unofficial]
                for dx in [-1, 0, 1]:
                    for dy in [-1, 0, 1]:
                        for dz in [-1, 0, 1]:
                            if dx == 0 and dy == 0 and dz == 0:
                                continue
                            
                            nx = x + dx * K
                            ny = y + dy * K
                            nz = z + dz * K
                            
                            # Boundary check [S1, unofficial]
                            if nx < 0 or nx >= N or ny < 0 or ny >= N or nz < 0 or nz >= N:
                                continue
                            
                            seed_dist = sdf_array[nx, ny, nz]
                            
                            # Skip if seed is infinity [S1, unofficial]
                            if abs(seed_dist) == float('inf'):
                                continue
                            
                            seed_pos = pos_array[nx, ny, nz]
                            
                            # Calculate Euclidean distance [S1, unofficial]
                            dist = math.sqrt(
                                (voxel_pos[0] - seed_pos[0])**2 +
                                (voxel_pos[1] - seed_pos[1])**2 +
                                (voxel_pos[2] - seed_pos[2])**2
                            )
                            
                            # Update if better [S1, unofficial]
                            if dist < abs(best_dist):
                                find_new_best = True
                                # Preserve sign of best_dist [S1, unofficial]
                                sign = 1.0 if best_dist >= 0 else -1.0
                                best_dist = sign * dist
                                best_pos = seed_pos
                
                if find_new_best:
                    sdf_array[x, y, z] = best_dist
                    pos_array[x, y, z] = best_pos

def naive_jfa(grid, sdf_array, pos_array):
    """
    Main Naive JFA algorithm.
    """
    initialize_naive(grid, sdf_array, pos_array)
    
    N = grid.VoxelsPerSide()
    K = N // 2
    
    while K >= 1:
        process_naive_iteration(grid, sdf_array, pos_array, K)
        K = K // 2
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Neighbor indices $nx, ny, nz$ must be strictly checked against $0$ and $N$ to avoid out-of-bounds memory access [S1, unofficial].
*   **Infinity Handling:** Seeds with `fabs(seed) == INFINITY` must be ignored to prevent invalid distance calculations [S1, unofficial].
*   **Precision:** Euclidean distance calculations should use sufficient floating-point precision [S1, unofficial]. The sign of the distance is preserved from the current best, which assumes the sign is consistent or correctly initialized at boundaries [S1, unofficial].

## Subtopics

### [Sequential Jumping Farthest Away (JFA)](7637-sequential-jumping-farthest-away-jfa.md)
Sequential JFA is a CPU-based algorithm for computing the SDF and closest surface point position. It differs from Naive JFA by executing the propagation loop sequentially on the CPU rather than in parallel on the GPU. It is chosen when GPU resources are unavailable or when debugging sequential logic.

### [Breadth-First Distance Field Expansion](9703-breadth-first-distance-field-expansion.md)
Breadth-First Distance Field Expansion computes the SDF by propagating distance values from known surface voxels to empty space using a queue-based breadth-first search. It differs from Naive JFA by processing neighbors in strict distance order (Manhattan distance) rather than using a halving radius strategy, and utilizes a global queue rather than sequential passes. It is chosen when exact distance metrics are required without the approximation errors inherent in JFA.

### Jump Flood Algorithm (JFA) for UV SDF Generation
Generates metric SDFs on a 2D UV texture by iteratively propagating nearest seed distances, achieving $O(\text{texels} \log \text{texels})$ complexity [S2, unofficial]. It differs from 3D Naive JFA by operating on 2D UV textures and using ping-pong buffers for propagation [S2, unofficial]. It is chosen for large-scale UV-based rendering tasks where 3D voxelization is too expensive [S2, unofficial].
*   **Definition:** Generates metric SDFs on a 2D UV texture by iteratively propagating nearest seed distances [S2, unofficial].
*   **Steps:**
    1.  **Stage 1 (Seed):** Rasterize primitives into UV seed buffer using Bresenham line rasterization [S2, unofficial].
    2.  **Stage 2 (JFA):** Iterative distance propagation with ping-pong buffers. Jump steps decrease: $\text{texture\_size}/2, \text{texture\_size}/4, \dots, 1$ [S2, unofficial].
    3.  **Stage 3 (Finalize):** Convert seeds to metric SDF using torus surface metric. Subtract radius for final SDF [S2, unofficial].
    4.  **Pass B (Render):** Ray march inverted torus and sample ink texture [S2, unofficial].
*   **Formula:** Complexity: $O(\text{texels} \times \log(\text{max\_texture\_dimension}))$ [S2, unofficial].
*   **Parameters:** `texture_size` determines the initial jump step and subsequent halving sequence [S2, unofficial].
*   **Layout:** Seed buffer format: R32I (integer distance storage) [S2, unofficial].
*   **Pitfalls:** Previous UV-GPU version crashed at $N=10,000$; JFA implementation achieves infinite speedup by avoiding per-primitive ray marching [S2, unofficial].
*   **Usage:** Enabled via `--uv` CLI flag, recommended for large datasets [S2, unofficial].

### Brixelizer SDF Construction
A method for building a global SDF for triangle geometry in real-time, supporting dynamic geometry, for use in global illumination [S3, unofficial]. It constructs local distance fields called 'Bricks' around the surface of geometry, builds a bottom-up AABB tree of the bricks, and uses JFA to populate samples not directly intersecting the surface [S3, unofficial]. JFA sets all surface-intersecting samples to the minimum representable value and extrapolates distance values for the remaining samples [S3, unofficial]. It offers no benefit when evaluating analytical SDFs because the intersection test product is the distance value itself, making the extrapolation redundant [S3, unofficial]. It is used by AMD Brixelizer (Kramer, 2023) [S3, unofficial].

### Closest Point Propagation
An algorithm to compute a signed distance volume by propagating closest-point information from an initialized zero set to the rest of the volume using a heuristic that neighbors share similar closest points [S4]. It initializes all distance samples to `maxfloat`, places zero set samples into a FIFO queue, and iteratively updates 26-neighbors if the closest point distance improves [S4]. Each distance sample is updated 1.1 times on average in experiments, running in $O(n)$ time on tested surfaces because a simple queue provides constant time access [S4]. Unlike priority-queue methods that visit each sample once with $O(n \log n)$ time, this FIFO method may update a sample multiple times but runs faster on tested surfaces [S4].

## Operating characteristics
*   **Time Complexity:** The algorithm performs $\log_2(N)$ iterations [S1, unofficial]. In each iteration, every voxel processes 26 neighbors [S1, unofficial]. Thus, the complexity is $O(N^3 \log N)$ for a grid of size $N \times N \times N$ [S1, unofficial].
*   **Memory Complexity:** $O(N^3)$ for storing the SDF and Position arrays [S1, unofficial].
*   **Measured Performance:** Specific measured performance numbers for Naive JFA are not given in the sources [S1, unofficial]. However, benchmarks compare it against Sequential and Tiled JFA implementations, indicating it is a baseline GPU implementation [S1, unofficial].
*   **Scaling:** Scales linearly with the number of voxels per iteration and logarithmically with the number of iterations [S1, unofficial]. Parallel execution on GPU allows it to handle large grids efficiently compared to sequential CPU versions [S1, unofficial].
*   **Strengths:** Simple to implement on GPU; parallelizable; faster than sequential CPU implementations for large grids [S1, unofficial].
*   **Limitations:** Approximate distance field (JFA is an approximation algorithm); requires multiple passes; boundary conditions must be handled carefully [S1, unofficial].
*   **When to use:** When a fast, approximate SDF is needed on a GPU and exact distance metrics are not critical [S1, unofficial].
*   **When not to use:** When exact distance metrics are required (use Breadth-First Distance Field Expansion); when CPU-only resources are available (use Sequential JFA) [S1, unofficial].

## Background
Naive Jumping Farthest Away (JFA) is based on the Jump Flooding Algorithm (JFA), originally introduced for generating Voronoi diagrams and distance transforms on GPUs [S1, unofficial]. The specific implementation described here is part of a CUDA-based mesh voxelization library that computes SDFs for voxelized meshes [S1, unofficial]. It is used in applications requiring fast SDF generation for volume rendering, CSG operations, and visualization [S1, unofficial]. The algorithm builds on the parent method, Dense Volume SDF Generation, which provides the framework for storing and accessing dense 3D SDF data [S1, unofficial].

## Sources
[S1] bigmat18/cuda-mesh-voxelization, https://github.com/bigmat18/cuda-mesh-voxelization, fetched 2026-10-26 (unofficial)
[S2] Add OpenGL GPU ray marching renderers for Intel HD Graphics 630 with Jump Flood Algorithm optimization, https://github.com/rhythmatician/phyllotaxis-torus/pull/1, fetched 2026-10-26 (unofficial)
[S3] PDF Real-time Rendering and Construction of Signed Distance Fields, https://jambuttenshaw.github.io/devportfolio/Dissertation.pdf, fetched 2026-10-28 (unofficial)
[S4] Multiresolution Distance Volumes for Progressive Surface Compression, https://digital.library.unt.edu/ark:/67531/metadc1409942/?q=volume data compression rendering, fetched 2026-10-28
