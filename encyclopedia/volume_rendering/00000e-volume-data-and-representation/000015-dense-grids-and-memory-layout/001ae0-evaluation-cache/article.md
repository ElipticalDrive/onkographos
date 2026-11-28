# Evaluation Cache
Also called: CachedEval, EvalCache

## Overview
Evaluation Cache is a spatial caching mechanism used in volume rendering and geometric processing to eliminate redundant computations for shared data points, such as voxel corners or isosurface intersection points [G]. It solves the problem of repeatedly evaluating the same grid position or interpolating the same geometric edge across adjacent cells [G]. It differs from [Dual Grid Traversal](1233-dual-grid-traversal.md) by explicitly storing computed values in a hash map keyed by quantized grid coordinates, ensuring that shared corners are evaluated only once regardless of how many adjacent cubes share them [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Query points (e.g., cell corners or ray sample positions) and grid parameters (origin, cell size) [S1, unofficial].
- **Output:** A cache structure containing computed values and their associated metadata, allowing subsequent queries for the same grid position to return cached results instantly [S1, unofficial].

**Data Structures:**
The method relies on two primary structures defined in the source implementation [S1, unofficial]:
1.  **`CachedEval`**: A struct storing the result of an evaluation [S1, unofficial].
    -   `value`: `f64` (64-bit float), the scalar value at the position [S1, unofficial].
    -   `gradient`: `Option<[f64; 3]>`, an optional 3D vector representing the gradient [S1, unofficial]. If `None`, the gradient is not computed or stored [S1, unofficial].
2.  **`EvalCache`**: A container for the cache entries [S1, unofficial].
    -   `entries`: A `HashMap` mapping `GridPos` (quantized integer coordinates) to `CachedEval` [S1, unofficial]. The key ensures that shared corners are evaluated only once [S1, unofficial].

**Quantized Grid Position:**
To serve as a unique key for the cache, continuous world-space coordinates are converted to discrete integer grid positions [S1, unofficial].
1.  **World to Grid:** Given a world-space point $\mathbf{p} = (x, y, z)$, grid coordinates $(i, j, k)$ are computed by subtracting the origin component and dividing by the cell size component [S1, unofficial].
2.  **Corner Indexing:** For cell corners, the integer coordinates of the cell $(c_x, c_y, c_z)$ are combined with corner bits [S1, unofficial]. The quantized grid position is derived by adding corner bits (1 for X, 2 for Y, 4 for Z) to the cell's integer coordinates [S1, unofficial]. This creates a unique integer key for each of the 8 corners of a voxel [S1, unofficial].
3.  **Grid to World:** The inverse conversion reconstructs world-space coordinates by adding the product of the grid position component and cell size to the origin component: $\mathbf{p}_{world} = \mathbf{p}_{grid} \odot \mathbf{s} + \mathbf{o}$ [S1, unofficial].

**Algorithm Steps:**
1.  **Query:** For a given query point $\mathbf{p}$, compute its quantized grid position $P_{key}$ using the corner indexing formula [S1, unofficial].
2.  **Lookup:** Check if $P_{key}$ exists in `EvalCache.entries` [S1, unofficial].
3.  **Hit:** If present, return the stored `CachedEval` (value and optional gradient) [S1, unofficial].
4.  **Miss:** If not present:
    a. Compute the value $V$ and gradient $\nabla V$ at $\mathbf{p}$ using the underlying volume function or interpolation [G].
    b. Create a `CachedEval` struct with $V$ and $\nabla V$ [S1, unofficial].
    c. Insert the entry into `EvalCache.entries` with key $P_{key}$ [S1, unofficial].
    d. Return the new `CachedEval` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from typing import Dict, Tuple, Optional, List

class CachedEval:
    def __init__(self, value: float, gradient: Optional[List[float]]):
        self.value = value
        self.gradient = gradient

class EvalCache:
    def __init__(self):
        self.entries: Dict[Tuple[int, int, int], CachedEval] = {}

    def get_or_compute(
        self,
        point: Tuple[float, float, float],
        origin: Tuple[float, float, float],
        cell_size: Tuple[float, float, float],
        corner_idx: int = 0
    ) -> CachedEval:
        x, y, z = point
        ox, oy, oz = origin
        sx, sy, sz = cell_size

        # Compute integer cell coordinates
        ix = math.floor((x - ox) / sx)
        iy = math.floor((y - oy) / sy)
        iz = math.floor((z - oz) / sz)

        # Compute quantized grid position key: add corner bits (1 for X, 2 for Y, 4 for Z)
        key = (ix + (corner_idx & 1), iy + ((corner_idx >> 1) & 1) * 2, iz + ((corner_idx >> 2) & 1) * 4)

        if key in self.entries:
            return self.entries[key]

        # Sources do not specify the volume evaluation function; implement as needed [G]
        value = 0.0
        gradient = None

        cached_eval = CachedEval(value, gradient)
        self.entries[key] = cached_eval
        return cached_eval
```

**Parameters:**
-   `origin`: Grid origin $(x_0, y_0, z_0)$ [S1, unofficial].
-   `cell_size`: Voxel dimensions $(\Delta x, \Delta y, \Delta z)$ [S1, unofficial].
-   `corner_idx`: Integer 0-7 specifying the corner offset bits [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Precision:** Floating-point to integer conversion must be consistent. Using `floor` ensures that points within the same voxel/cell map to the same integer key [G]. Ensure that points exactly on boundaries are handled consistently with the rest of the pipeline [G].
-   **Memory:** The cache grows with the number of unique query points. For dense sampling, this may consume significant memory [G]. Consider eviction policies if memory is constrained [G].
-   **Degenerate Input:** If `cell_size` is zero, division by zero occurs. Validate inputs [G].

## Subtopics

### [Edge Point Caching](891-edge-point-caching.md)
Edge Point Caching eliminates redundant linear interpolation calculations for isosurface intersection points on cube edges shared between adjacent voxels in the Marching Cubes algorithm. It differs from standard Marching Cubes by maintaining a lookup structure that ensures each geometric edge is interpolated only once, regardless of how many adjacent cubes share it.

### [Volume Access Caching](2125-volume-access-caching.md)
Volume Access Caching is an optimization for direct volume rendering that reduces redundant memory bandwidth by caching the volume value at the end of one ray-marching step for use as the starting value of the next step. It solves the problem of repeatedly fetching overlapping voxel data between consecutive iterations along a single ray, distinguishing itself from spatial caching methods like [Edge Point Caching](891-edge-point-caching.md) by exploiting temporal coherence along the ray [G].

### Radiance Caching with Visibility
Radiance Caching with Visibility extends standard radiance caching by considering visibility during interpolation [S2]. It uses the second-order Hessian error metric to determine whether the interpolation error is acceptable, thereby considering visibility [S2]. This approach produces results with higher quality than those of Jarosz et al. [57], as it accounts for occlusion changes between stored sample points and points to be calculated [S2].

## Operating characteristics
-   **Time Complexity:** Lookup is $O(1)$ on average for hash map operations [G]. Computation is $O(1)$ per unique point [G]. Total time is $O(N)$ where $N$ is the number of unique query points [G].
-   **Memory Complexity:** $O(U)$ where $U$ is the number of unique cached points. Each entry stores a value and optionally a gradient [G].
-   **Strengths:** Eliminates redundant computations for shared points (corners, edges). Improves performance in scenarios with high spatial coherence or shared geometry [G].
-   **Limitations:** Requires additional memory for the cache. Hash map overhead may be significant for very large caches [G]. Not suitable for purely random, non-repeating queries [G].
-   **When to use:** When evaluating the same grid positions multiple times (e.g., Marching Cubes, shared ray samples) [G].
-   **When not to use:** When memory is extremely constrained or queries are entirely unique [G].

## Background
The Evaluation Cache mechanism is described in the source implementation [S1, unofficial]. The technique of using caching to reduce redundant evaluations in volume rendering and radiance transfer has been explored in various forms, such as radiance caching for participating media [S2].

## Sources
[S1] narnia-ai-mason/isomesh, https://github.com/narnia-ai-mason/isomesh, fetched 2026-10-27 (unofficial)
[S2] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
