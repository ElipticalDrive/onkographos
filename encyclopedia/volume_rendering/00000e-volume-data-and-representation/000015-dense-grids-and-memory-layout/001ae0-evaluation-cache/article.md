# Evaluation Cache

Also called: CachedEval, EvalCache

## Overview
Evaluation Cache is a data structure optimization that eliminates redundant geometric evaluations by caching results for shared grid positions, primarily in manifold dual contouring and isosurface extraction. It solves the problem of computing the same vertex properties (such as position and gradient) multiple times for corners shared by adjacent voxels. It sets itself apart from siblings like [Tile-based Volume Ray Marching](1678-tile-based-volume-ray-marching.md) by focusing on spatial coherence at geometry boundaries rather than temporal coherence along a ray.

## How it works
The method caches evaluation results keyed by quantized grid positions to ensure that each geometric edge or corner is processed only once, regardless of how many adjacent cubes share it.

**Inputs and Outputs:**
- Input: A set of voxel cells with corner masks indicating inside/outside status.
- Output: A cache of unique evaluation results (`EvalCache`) and computed vertices for manifold dual contouring.

**Data Structures:**
The core structure is `EvalCache`, which maps quantized grid positions to cached evaluation results.

1.  **`CachedEval` struct**:
    -   `value`: `f64` (scalar field value at the position).
    -   `gradient`: `Option<[f64; 3]>` (3D vector gradient, or `None` if not computed).
2.  **`EvalCache` struct**:
    -   `entries`: `HashMap<GridPos, CachedEval>`.
    -   `GridPos`: A key representing a quantized grid position.

**Steps:**

1.  **Quantize Grid Position**: For a given cell corner, compute a unique integer key (`GridPos`) that identifies the corner's location in the global grid.
    -   Let `cell_coords` be the integer coordinates of the cell $(x, y, z)$.
    -   Let `corner_idx` be the bit index of the corner within the cell (0–7).
    -   Compute `quantized_pos` by adding corner-specific bits to the cell coordinates:
        $$ \text{quantized\_pos} = (x, y, z) + (\text{corner\_idx} \& 1, (\text{corner\_idx} \gg 1) \& 1, (\text{corner\_idx} \gg 2) \& 1) $$
        *Note: The source [S1, unofficial] describes adding corner_idx bits (1 for X, 2 for Y, 4 for Z) to the cell's integer coordinates. This effectively maps each of the 8 corners of a cell to a unique global grid vertex.*
2.  **Check Cache**: Look up `quantized_pos` in `EvalCache.entries`.
    -   If present, retrieve `CachedEval`.
    -   If absent, proceed to step 3.
3.  **Evaluate**: Compute the field value and gradient at the world-space coordinate corresponding to `quantized_pos`.
    -   Convert `quantized_pos` to world space:
        $$ \text{world\_pos} = \text{origin} + \text{quantized\_pos} \times \text{cell\_size} $$
    -   Compute `value` and `gradient` at `world_pos`.
4.  **Store**: Insert `(quantized_pos, CachedEval{value, gradient})` into `EvalCache.entries`.
5.  **Return**: Return the cached `CachedEval`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from typing import Optional, Tuple, Dict
import math

# Assuming these are defined elsewhere or standard
# GridPos: Tuple[int, int, int]
# Vec3: Tuple[float, float, float]

class CachedEval:
    def __init__(self, value: float, gradient: Optional[Tuple[float, float, float]]):
        self.value = value
        self.gradient = gradient

class EvalCache:
    def __init__(self):
        self.entries: Dict[GridPos, CachedEval] = {}

    def get_or_compute(self, cell_coords: Tuple[int, int, int], corner_idx: int, 
                       origin: Vec3, cell_size: float, 
                       evaluate_field: callable) -> CachedEval:
        """
        cell_coords: (x, y, z) of the current cell
        corner_idx: 0-7, bit index of the corner
        origin: World space origin of the grid
        cell_size: Size of one voxel
        evaluate_field: Function that takes world_pos and returns (value, gradient)
        """
        # Step 1: Quantize grid position
        # Add corner_idx bits to cell coordinates
        x = cell_coords[0] + (corner_idx & 1)
        y = cell_coords[1] + ((corner_idx >> 1) & 1)
        z = cell_coords[2] + ((corner_idx >> 2) & 1)
        quantized_pos = (x, y, z)

        # Step 2: Check cache
        if quantized_pos in self.entries:
            return self.entries[quantized_pos]

        # Step 3: Evaluate
        # Convert to world space
        world_x = origin[0] + x * cell_size
        world_y = origin[1] + y * cell_size
        world_z = origin[2] + z * cell_size
        world_pos = (world_x, world_y, world_z)

        # Compute value and gradient
        value, gradient = evaluate_field(world_pos)
        
        # Step 4: Store
        cached_eval = CachedEval(value, gradient)
        self.entries[quantized_pos] = cached_eval

        # Step 5: Return
        return cached_eval
```

**Edge Cases and Pitfalls:**
-   **Precision**: Floating-point comparisons for `world_pos` should be avoided; use the integer `quantized_pos` as the key to ensure exact matching.
-   **Boundaries**: Corners on the grid boundary must be handled consistently to avoid duplicate evaluations or missing vertices.
-   **Degenerate Input**: If `corner_mask` is 0 or 0xFF, no inside corners exist, and no evaluation is needed for that cell's corners in the context of manifold dual contouring [S1, unofficial].

## Subtopics

### [Edge Point Caching](891-edge-point-caching.md)
Edge Point Caching eliminates redundant linear interpolation calculations for isosurface intersection points on cube edges shared between adjacent voxels in the Marching Cubes algorithm. It differs from standard Marching Cubes by maintaining a lookup structure that ensures each geometric edge is interpolated only once, regardless of how many adjacent cubes share it.

### [Volume Access Caching](2125-volume-access-caching.md)
Volume Access Caching is a technique used within Evaluation Cache frameworks to reduce redundant texture lookups during ray marching in direct volume rendering. It solves the problem of repeatedly fetching the same voxel data for adjacent integration steps by caching the result of the previous iteration's back sample. It sets itself apart from siblings like [Edge Point Caching](891-edge-point-caching.md) by focusing on temporal coherence along the ray rather than spatial coherence at geometry boundaries.

### Radiance Caching with Visibility
Radiance Caching with Visibility uses the second-order Hessian error metric to determine whether the interpolation error is acceptable, thereby considering visibility [S2]. This approach addresses the issue where stored sample points are sparse, and points to be calculated may be differently occluded compared to stored points, resulting in different illumination [S2]. By considering visibility, it produces results with higher quality than those of Jarosz et al. [57] [S2].

## Operating characteristics
-   **Time Complexity**: $O(1)$ average case for cache lookups and inserts using a hash map. Total time is proportional to the number of unique grid corners evaluated.
-   **Memory Complexity**: $O(V)$, where $V$ is the number of unique vertices (grid corners) in the domain. Each entry stores a `f64` value and an optional `Vec3` gradient.
-   **Strengths**: Eliminates redundant computations for shared geometric features; improves performance in manifold dual contouring and isosurface extraction.
-   **Limitations**: Requires additional memory for the cache; hash map overhead may be significant for very large grids if not optimized.
-   **When to Use**: When extracting isosurfaces from dense grids where many adjacent cells share corners or edges.
-   **When Not to Use**: When memory is extremely constrained, or when evaluations are inherently unique per cell (e.g., some forms of direct volume rendering without shared geometry).

## Background
Evaluation Cache originates from implementations of Manifold Dual Contouring, where it is used to ensure manifoldness by processing shared corners only once [S1, unofficial]. The concept of caching evaluations to improve performance is common in computer graphics, particularly in geometry processing and volume rendering. The specific struct layout (`CachedEval` with `value` and `gradient`) is documented in the `isomesh` repository [S1, unofficial]. The extension to radiance caching with visibility considerations is discussed in surveys on rendering homogeneous participating media [S2].

## Sources
[S1] narnia-ai-mason/isomesh, https://github.com/narnia-ai-mason/isomesh, fetched 2026-10-27 (unofficial)
[S2] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
