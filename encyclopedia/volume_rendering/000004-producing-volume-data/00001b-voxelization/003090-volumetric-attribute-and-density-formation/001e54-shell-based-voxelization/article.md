# Shell-based Voxelization

Also called: AABB Expansion Voxelization, Mark Boundary Voxels

## Overview
Shell-based Voxelization identifies and marks all voxels in a grid that lie within a specified shell radius of any primitive (triangle or polygon) in a mesh [S1, unofficial]. It serves to determine the spatial support for subsequent voxelization operations [S1, unofficial].

## How it works
**Inputs:**
- `vertices`: Array of vertex positions, layout `Float[N]` per vertex, size `num_vertices * N` [S1, unofficial].
- `simplices`: Array of vertex indices, layout `uint32_t[N]` per primitive, size `num_simplices * N` [S1, unofficial].
- `num_simplices`: Number of primitives [S1, unofficial].
- `grid`: Grid extent `Extent<N>`, storing dimensions `width`, `height`, `depth` (for N=3) or `width`, `height` (for N=2) [S1, unofficial].
- `voxel_size`: Size of a single voxel, layout `Vector<Float, N>` [S1, unofficial].
- `shell_radius`: Distance from the primitive to expand the bounding box [S1, unofficial].
- `mask`: A bitset to store the resulting voxel indices [S1, unofficial].

**Outputs:**
- `mask`: Bitset with bits set for all voxels within the shell radius of any primitive [S1, unofficial].

**Data Structures:**
- `Extent<N>`: Struct with `uint32_t extents[N]`. Index 0 is width, 1 is height, 2 is depth [S1, unofficial].
- `AABB<Float, N>`: Axis-Aligned Bounding Box with `min` and `max` vectors of type `Vector<Float, N>` [S1, unofficial].
- `Bitset`: A dynamic array of bits supporting `set(index)` operations [S1, unofficial].

**Formulas:**
- Grid coordinate conversion: $grid\_coord = \frac{x - (-1)}{voxel\_size} + 0$, where $x$ is the world coordinate and the grid origin is at -1 [S1, unofficial].
- Voxel size calculation (for a grid spanning [-1, 1] with extent $E$): $voxel\_size = \frac{2}{E}$ [S1, unofficial].

**Steps:**
1. Iterate over each primitive index $i$ from $0$ to $num\_simplices - 1$ [S1, unofficial].
2. Construct an AABB for the current primitive by iterating over its $N$ vertices and extending the AABB to include each vertex position [S1, unofficial].
3. Expand the AABB by subtracting `shell_radius` from the minimum coordinates and adding `shell_radius` to the maximum coordinates in all $N$ dimensions [S1, unofficial].
4. Convert the expanded AABB boundaries to grid coordinates using the conversion formula, then floor the results to get integer `min_coord` and `max_coord` arrays [S1, unofficial].
5. Iterate over the voxel grid indices within the bounding box range `[min_coord, max_coord]`, clamping to the grid boundaries `[0, grid.extent[d])` for each dimension $d$ [S1, unofficial].
6. Compute the linear index for each voxel and set the corresponding bit in the mask [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def mark_boundary_voxels(vertices, simplices, num_simplices, grid_width, grid_height, grid_depth, voxel_size, shell_radius):
    """
    Marks all voxels within shell_radius of any primitive.
    vertices: list of [x, y, z] floats
    simplices: list of [v0, v1, v2] uint32 indices
    grid_width, grid_height, grid_depth: integer extents
    voxel_size: float
    shell_radius: float
    Returns: set of linear voxel indices
    """
    mask = set()
    N = 3
    grid_extents = [grid_width, grid_height, grid_depth]

    for i in range(num_simplices):
        # 1. Construct AABB
        aabb_min = [float('inf')] * N
        aabb_max = [float('-inf')] * N
        simplex_indices = simplices[i]
        for v_idx in simplex_indices:
            v = vertices[v_idx]
            for d in range(N):
                if v[d] < aabb_min[d]:
                    aabb_min[d] = v[d]
                if v[d] > aabb_max[d]:
                    aabb_max[d] = v[d]

        # 2. Expand AABB
        for d in range(N):
            aabb_min[d] -= shell_radius
            aabb_max[d] += shell_radius

        # 3. Convert to grid coordinates & floor
        min_int = [math.floor((aabb_min[d] + 1) / voxel_size) for d in range(N)]
        max_int = [math.floor((aabb_max[d] + 1) / voxel_size) for d in range(N)]

        # 4. Iterate and mark
        for x in range(max(0, min_int[0]), min(grid_extents[0], max_int[0] + 1)):
            for y in range(max(0, min_int[1]), min(grid_extents[1], max_int[1] + 1)):
                for z in range(max(0, min_int[2]), min(grid_extents[2], max_int[2] + 1)):
                    idx = x + y * grid_extents[0] + z * grid_extents[0] * grid_extents[1]
                    mask.add(idx)
    return mask
```

**Parameters:**
- `shell_radius`: Determines the thickness of the boundary shell. Typical values are not given in the sources.

**Edge Cases and Pitfalls:**
- The implementation uses hardcoded loops for N=2 and N=3, limiting scalability to higher dimensions [S1, unofficial].
- Clamping iteration bounds to `[0, grid.extent[d])` prevents out-of-bounds access [S1, unofficial].
- Precision issues may arise if `voxel_size` is very small or `shell_radius` is very large relative to the grid extent [G].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.
- **When to use:** Not given in the sources.
- **When not to use:** Not given in the sources.

## Background
Shell-based Voxelization originates from the Differentiable Voxelization of Surface Representations framework [S1, unofficial]. It is implemented in the `dvx` namespace of the corresponding repository [S1, unofficial].

## Sources
[S1] Differentiable Voxelization of Surface Representations, https://github.com/mworchel/differentiable-voxelization, fetched 2026-10-29 (unofficial)
