# Evaluation Cache
Also called: CachedEval, EvalCache

## Overview
Evaluation Cache is a spatial caching mechanism used in volume rendering and geometric processing to eliminate redundant computations for shared data points, such as voxel corners or isosurface intersection points. It solves the problem of repeatedly evaluating the same grid position or interpolating the same geometric edge across adjacent cells. It differs from [Dual Grid Traversal](1233-dual-grid-traversal.md) by explicitly storing computed values in a hash map keyed by quantized grid coordinates, ensuring that shared corners are evaluated only once regardless of how many adjacent cubes share them [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A set of query points (e.g., cell corners or ray sample positions) and a grid definition (origin, cell size, dimensions).
- **Output:** A cache structure containing computed values (e.g., scalar values, gradients) and their associated metadata, allowing subsequent queries for the same grid position to return cached results instantly.

**Data Structures:**
The method relies on two primary structures defined in the source implementation [S1, unofficial]:

1.  **`CachedEval`**: A struct storing the result of an evaluation.
    -   `value`: `f64` (64-bit float), the scalar value at the position.
    -   `gradient`: `Option<[f64; 3]>`, an optional 3D vector representing the gradient. If `None`, the gradient is not computed or stored.

2.  **`EvalCache`**: A container for the cache entries.
    -   `entries`: A `HashMap` mapping `GridPos` (quantized integer coordinates) to `CachedEval`. The key ensures that shared corners are evaluated only once.

**Quantized Grid Position:**
To serve as a unique key for the cache, continuous world-space coordinates are converted to discrete integer grid positions.
1.  **World to Grid:** Given a world-space point $\mathbf{p} = (x, y, z)$, grid coordinates $(i, j, k)$ are computed as:
    $$ i = \lfloor \frac{x - x_0}{\Delta x} \rfloor, \quad j = \lfloor \frac{y - y_0}{\Delta y} \rfloor, \quad k = \lfloor \frac{z - z_0}{\Delta z} \rfloor $$
    where $(x_0, y_0, z_0)$ is the grid origin and $\Delta x, \Delta y, \Delta z$ are cell sizes [S1, unofficial].
2.  **Corner Indexing:** For cell corners, the integer coordinates of the cell $(c_x, c_y, c_z)$ are combined with corner bits. The quantized grid position is derived by adding corner bits (1 for X, 2 for Y, 4 for Z) to the cell's integer coordinates [S1, unofficial]. This creates a unique integer key for each of the 8 corners of a voxel.

**Algorithm Steps:**
1.  **Query:** For a given query point $\mathbf{p}$, compute its quantized grid position $P_{key}$.
2.  **Lookup:** Check if $P_{key}$ exists in `EvalCache.entries`.
3.  **Hit:** If present, return the stored `CachedEval` (value and optional gradient).
4.  **Miss:** If not present:
    a. Compute the value $V$ and gradient $\nabla V$ at $\mathbf{p}$ using the underlying volume function or interpolation.
    b. Create a `CachedEval` struct with $V$ and $\nabla V$.
    c. Insert the entry into `EvalCache.entries` with key $P_{key}$.
    d. Return the new `CachedEval`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from typing import Dict, Tuple, Optional, List

# Data structures
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
        compute_func: callable
    ) -> CachedEval:
        x, y, z = point
        ox, oy, oz = origin
        sx, sy, sz = cell_size

        # Compute quantized grid position (integer coordinates)
        # Using floor for consistent indexing
        ix = math.floor((x - ox) / sx)
        iy = math.floor((y - oy) / sy)
        iz = math.floor((z - oz) / sz)

        key = (ix, iy, iz)

        if key in self.entries:
            return self.entries[key]

        # Compute value and gradient if not cached
        # compute_func is a user-provided function that evaluates the volume
        value, gradient = compute_func(point)
        
        cached_eval = CachedEval(value, gradient)
        self.entries[key] = cached_eval
        return cached_eval

# Example usage context
def evaluate_volume_with_cache():
    cache = EvalCache()
    origin = (0.0, 0.0, 0.0)
    cell_size = (1.0, 1.0, 1.0)
    
    # Dummy compute function
    def dummy_compute(p):
        return p[0] + p[1] + p[2], [1.0, 1.0, 1.0]

    point = (0.5, 0.5, 0.5)
    result = cache.get_or_compute(point, origin, cell_size, dummy_compute)
    return result
```

**Parameters:**
-   `origin`: Grid origin $(x_0, y_0, z_0)$.
-   `cell_size`: Voxel dimensions $(\Delta x, \Delta y, \Delta z)$.
-   `compute_func`: Function to evaluate the volume field.

**Edge Cases and Pitfalls:**
-   **Precision:** Floating-point to integer conversion must be consistent. Using `floor` ensures that points within the same voxel/cell map to the same integer key. [G] Ensure that points exactly on boundaries are handled consistently with the rest of the pipeline.
-   **Memory:** The cache grows with the number of unique query points. For dense sampling, this may consume significant memory. [G] Consider eviction policies if memory is constrained.
-   **Degenerate Input:** If `cell_size` is zero, division by zero occurs. Validate inputs.

## Subtopics

### [Edge Point Caching](891-edge-point-caching.md)
Edge Point Caching eliminates redundant linear interpolation calculations for isosurface intersection points on cube edges shared between adjacent voxels in the Marching Cubes algorithm. It differs from standard Marching Cubes by maintaining a lookup structure that ensures each geometric edge is interpolated only once, regardless of how many adjacent cubes share it. This is a specific application of the Evaluation Cache principle focused on geometric edges rather than arbitrary grid points.

### [Volume Access Caching](2125-volume-access-caching.md)
Volume Access Caching is a technique used within the Evaluation Cache framework to reduce redundant memory bandwidth consumption during ray-marching volume rendering. It solves the problem of repeatedly fetching the same voxel data when the sampling step for the current iteration overlaps with the end point of the previous iteration. It sets itself apart from sibling methods like [Edge Point Caching](891-edge-point-caching.md) by focusing on temporal coherence between consecutive samples along a single ray, rather than spatial coherence across neighboring pixels or rays.

### Radiance Caching with Visibility
Radiance Caching with Visibility extends standard radiance caching by considering visibility during interpolation. It uses the second-order Hessian error metric to determine whether the interpolation error is acceptable, thereby considering visibility [S2]. This approach produces results with higher quality than those of Jarosz et al. [57], as it accounts for occlusion changes between stored sample points and points to be calculated [S2].

## Operating characteristics
-   **Time Complexity:** Lookup is $O(1)$ on average for hash map operations. Computation is $O(1)$ per unique point. Total time is $O(N)$ where $N$ is the number of unique query points.
-   **Memory Complexity:** $O(U)$ where $U$ is the number of unique cached points. Each entry stores a value and optionally a gradient.
-   **Strengths:** Eliminates redundant computations for shared points (corners, edges). Improves performance in scenarios with high spatial coherence or shared geometry.
-   **Limitations:** Requires additional memory for the cache. Hash map overhead may be significant for very large caches. Not suitable for purely random, non-repeating queries.
-   **When to use:** When evaluating the same grid positions multiple times (e.g., Marching Cubes, shared ray samples).
-   **When not to use:** When memory is extremely constrained or queries are entirely unique.

## Background
The Evaluation Cache concept is implemented in the `isomesh` library, which focuses on manifold dual contouring and isosurface extraction [S1, unofficial]. The technique of using caching to reduce redundant evaluations in volume rendering and radiance transfer has been explored in various forms, such as radiance caching for participating media [S2]. The specific implementation details (structs, quantization) are derived from the `isomesh` source code [S1, unofficial].

## Sources
[S1] narnia-ai-mason/isomesh, https://github.com/narnia-ai-mason/isomesh, fetched 2026-10-27 (unofficial)
[S2] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
