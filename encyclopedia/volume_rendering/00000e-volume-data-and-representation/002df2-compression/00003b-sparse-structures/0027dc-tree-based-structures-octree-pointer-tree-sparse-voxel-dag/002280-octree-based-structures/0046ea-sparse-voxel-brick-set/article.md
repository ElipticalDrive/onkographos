# Sparse Voxel Brick Set

Also called: Brick-based SDF, Dreams-style SDF, Sparse Brick Set

## Overview
The Sparse Voxel Brick Set is a sparse volume representation that groups Signed Distance Field (SDF) data into fixed-size cubic blocks called "bricks," each associated with a raytracing Axis-Aligned Bounding Box (AABB) [S1, unofficial]. It balances memory efficiency and real-time constructibility, specifically optimizing for GPU cache coherency and parallel construction [S1, unofficial]. Unlike Sparse Voxel Octrees (SVOs) which vary resolution, or Grid Sphere Tracing which stores every voxel, the Brick Set uses a uniform brick granularity to balance memory usage and traversal speed, making it particularly effective in complex scenes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of analytical SDF primitives (edits) defining geometry [S1, unofficial].
*   **Output:** A collection of bricks, where each brick contains discrete SDF samples and a bounding box [S1, unofficial].

**Data Structures:**
Each **Brick** consists of:
1.  **AABB:** A 3D bounding box enclosing the brick's spatial extent [S1, unofficial].
2.  **Voxel Array:** A fixed-size array of $8^3 = 512$ SDF samples [S1, unofficial].
    *   **Inner Core:** The central $6^3 = 216$ voxels contain the primary distance field data used for sphere tracing [S1, unofficial].
    *   **Padding Shell:** The outer shell ($8^3 - 6^3 = 296$ voxels) consists of a one-voxel thick duplicate of the inner core's boundary [S1, unofficial]. This padding ensures that linear texture filtering performed by hardware samplers does not cross brick boundaries, which would cause incorrect interpolation [S1, unofficial].

**Parameters:**
*   **Brick Size:** $8 \times 8 \times 8$ voxels [S1, unofficial].
*   **Inner Usable Samples:** $6 \times 6 \times 6$ [S1, unofficial].
*   **Padding:** 1 voxel on all sides [S1, unofficial].

**Algorithm Steps:**
1.  **Construction:** Analytical SDF primitives are evaluated into a global coordinate space [S1, unofficial].
2.  **Brick Assignment:** The 3D domain is partitioned into $8^3$ voxel bricks. Each brick is assigned to a GPU thread group (e.g., 512 threads for an $8^3$ brick) to compute the SDF values for its voxels in parallel [S1, unofficial].
3.  **Padding Application:** For each brick, the inner $6^3$ values are computed. The outer shell is filled by duplicating the nearest inner values to create the 1-voxel neighborhood [S1, unofficial].
4.  **Bounding Box Calculation:** Compute the AABB for each brick based on its world-space position and brick dimensions [S1, unofficial].
5.  **Storage:** Store the bricks in a linear array or texture buffer [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
INF = float('inf')

def compute_brick(brick_origin, primitives):
    """
    Computes the SDF values for a single 8x8x8 brick.
    brick_origin: (x, y, z) world coordinates of the brick's min corner.
    primitives: List of objects with a distance(pos) method.
    Returns: Array of 512 floats (8x8x8).
    """
    brick_size = 8
    inner_start = 1
    inner_end = 6
    samples = [0.0] * (brick_size ** 3)
    
    # Compute inner core and initial values
    for z in range(brick_size):
        for y in range(brick_size):
            for x in range(brick_size):
                voxel_pos = (brick_origin[0] + x + 0.5, 
                             brick_origin[1] + y + 0.5, 
                             brick_origin[2] + z + 0.5)
                min_dist = INF
                for prim in primitives:
                    d = prim.distance(voxel_pos)
                    if d < min_dist:
                        min_dist = d
                samples[z * brick_size * brick_size + y * brick_size + x] = min_dist

    # Apply padding: duplicate the inner 6x6x6 boundary to the outer shell
    padded_samples = [0.0] * (brick_size ** 3)
    
    for z in range(brick_size):
        for y in range(brick_size):
            for x in range(brick_size):
                ix = max(inner_start, min(x, inner_end))
                iy = max(inner_start, min(y, inner_end))
                iz = max(inner_start, min(z, inner_end))
                
                inner_idx = iz * brick_size * brick_size + iy * brick_size + ix
                padded_samples[z * brick_size * brick_size + y * brick_size + x] = samples[inner_idx]
                
    return padded_samples
```

**Edge Cases and Pitfalls:**
*   **Linear Filtering Artifacts:** If the padding is not applied, hardware texture sampling will interpolate across brick boundaries, mixing distance values from adjacent bricks and causing incorrect interpolation [S1, unofficial].
*   **Memory Overhead:** 58% of the brick's data is dedicated to adjacency/padding data. This is a significant memory cost compared to raw storage but is necessary for hardware-accelerated interpolation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Construction is $O(N)$ where $N$ is the number of voxels in the active volume, parallelized efficiently on GPUs (one thread group per brick) [S1, unofficial].
*   **Memory Complexity:** High density per brick. Each brick stores 512 samples, but only 216 are unique. The memory overhead is 58% padding [S1, unofficial].
*   **Performance:** In complex scenes, Brick Sets outperform SVOs and Grid Tracing due to improved cache coherency from compact data storage [S1, unofficial]. In memory-constrained scenarios, it uses the least memory among the compared sparse methods [S1, unofficial].
*   **Strengths:** Fast construction, good cache coherency, suitable for real-time user-generated content (e.g., Dreams) [S1, unofficial].
*   **Limitations:** High memory overhead per brick due to padding; less flexible resolution than SVOs [S1, unofficial].

## Background
The Sparse Voxel Brick Set was popularized by the game *Dreams* (Evans, 2015) for user-generated content sculpting [S1, unofficial]. It was analyzed in the context of real-time SDF rendering and construction, comparing it against Sparse Voxel Octrees and Grid Sphere Tracing [S1, unofficial]. The approach draws inspiration from Gigavoxels (Crassin, 2009) regarding out-of-core rendering and brick-based LOD systems [S1, unofficial].

## Sources
[S1] Real-time Rendering and Construction of Signed Distance Fields, https://jambuttenshaw.github.io/devportfolio/Dissertation.pdf, fetched 2026-10-28 (unofficial)
