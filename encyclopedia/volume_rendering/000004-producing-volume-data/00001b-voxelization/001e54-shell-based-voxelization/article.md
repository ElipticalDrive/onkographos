# Shell-based Voxelization

Also called: AABB Expansion Voxelization, Mark Boundary Voxels

## Overview
Shell-based Voxelization identifies and marks all voxels in a grid that lie within a specified shell radius of any primitive (triangle or polygon) in a mesh [S1, unofficial]. It serves to determine the spatial support for subsequent voxelization operations, effectively creating a boundary mask around the geometry [S1, unofficial]. Unlike methods that fill the interior or rely on rasterization, this method focuses exclusively on the immediate neighborhood of the surface primitives [S1, unofficial].

## How it works
**Inputs:**
- `vertices`: Array of vertex positions, layout `Float[N]` per vertex, size `num_vertices * N`.
- `simplices`: Array of vertex indices, layout `uint32_t[N]` per primitive, size `num_simplices * N`.
- `num_simplices`: Number of primitives.
- `grid`: Grid extent `Extent<N>`, storing dimensions `width`, `height`, `depth` (for N=3) or `width`, `height` (for N=2).
- `voxel_size`: Size of a single voxel, layout `Vector<Float, N>`.
- `shell_radius`: Distance from the primitive to expand the bounding box.
- `mask`: A bitset to store the resulting voxel indices.

**Outputs:**
- `mask`: Bitset with bits set for all voxels within the shell radius of any primitive.

**Data Structures:**
- `Extent<N>`: Struct with `uint32_t extents[N]`. Index 0 is width, 1 is height, 2 is depth.
- `AABB<Float, N>`: Axis-Aligned Bounding Box with `min` and `max` vectors of type `Vector<Float, N>`.
- `Bitset`: A dynamic array of bits supporting `set(index)` operations.

**Formulas:**
- Grid coordinate conversion: $grid\_coord = \frac{x - (-1)}{voxel\_size} + 0$, where $x$ is the world coordinate and the grid origin is at -1 [S1, unofficial].
- Voxel size calculation (for a grid spanning [-1, 1] with extent $E$): $voxel\_size = \frac{2}{E}$ [S1, unofficial].

**Steps:**
1. Iterate over each primitive index $i$ from $0$ to $num\_simplices - 1$.
2. Construct an AABB for the current primitive:
   - Initialize `aabb.min` to positive infinity and `aabb.max` to negative infinity.
   - For each vertex index $v$ in `simplices[i]`:
     - Get vertex position $P = vertices[v]$.
     - Update `aabb.min = min(aabb.min, P)` and `aabb.max = max(aabb.max, P)`.
3. Expand the AABB by the shell radius:
   - `aabb.min -= shell_radius` (subtract `shell_radius` from all dimensions).
   - `aabb.max += shell_radius` (add `shell_radius` to all dimensions).
4. Convert expanded AABB boundaries to grid coordinates:
   - $min\_grid = point\_to\_grid\_coord(aabb.min, voxel\_size)$.
   - $max\_grid = point\_to\_grid\_coord(aabb.max, voxel\_size)$.
5. Compute integer grid bounds:
   - $min\_coord[d] = \lfloor min\_grid[d] \rfloor$ for each dimension $d$.
   - $max\_coord[d] = \lfloor max\_grid[d] \rfloor$ for each dimension $d$.
6. Iterate over all voxel indices within the bounding box $[min\_coord, max\_coord]$:
   - Clamp iteration bounds to grid extent $[0, grid.extent[d])$.
   - For each voxel $(x, y, z)$ (or $(x, y)$ for N=2):
     - Compute linear index $L = coords\_to\_linear\_index(x, y, z, grid)$.
     - Set bit $L$ in `mask`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mark_boundary_voxels(vertices, simplices, num_simplices, grid, voxel_size, shell_radius):
    """
    Marks all voxels within shell_radius of any primitive.
    vertices: list of [x, y, z] floats
    simplices: list of [v0, v1, v2] uint32 indices
    grid: dict with 'width', 'height', 'depth'
    voxel_size: float
    shell_radius: float
    Returns: set of linear voxel indices
    """
    mask = set()
    N = 3 # Assuming 3D for this implementation
    
    for i in range(num_simplices):
        # 1. Construct AABB
        min_coord = [float('inf')] * N
        max_coord = [float('-inf')] * N
        
        simplex_indices = simplices[i]
        for v_idx in simplex_indices:
            v = vertices[v_idx]
            for d in range(N):
                if v[d] < min_coord[d]:
                    min_coord[d] = v[d]
                if v[d] > max_coord[d]:
                    max_coord[d] = v[d]
        
        # 2. Expand AABB
        for d in range(N):
            min_coord[d] -= shell_radius
            max_coord[d] += shell_radius
            
        # 3. Convert to grid coordinates
        # Formula: grid_coord = (x - (-1)) / voxel_size
        min_grid = [(min_coord[d] + 1) / voxel_size for d in range(N)]
        max_grid = [(max_coord[d] + 1) / voxel_size for d in range(N)]
        
        # 4. Floor to integer bounds
        min_int = [int(m) for m in min_grid] # floor
        max_int = [int(m) for m in max_grid] # floor
        
        # 5. Iterate and mark
        for x in range(max(0, min_int[0]), min(grid['width'], max_int[0] + 1)):
            for y in range(max(0, min_int[1]), min(grid['height'], max_int[1] + 1)):
                for z in range(max(0, min_int[2]), min(grid['depth'], max_int[2] + 1)):
                    # Linear index calculation (row-major)
                    idx = x + y * grid['width'] + z * grid['width'] * grid['height']
                    mask.add(idx)
                    
    return mask
```

**Parameters:**
- `shell_radius`: Determines the thickness of the boundary shell. Typical values depend on the desired anti-aliasing or gradient support, but are not explicitly defined in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
- The implementation uses hardcoded loops for N=2 and N=3, limiting scalability to higher dimensions [S1, unofficial].
- Precision issues may arise if `voxel_size` is very small or `shell_radius` is very large relative to the grid extent.
- Boundary clamping is essential to prevent out-of-bounds access [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(P \cdot V_{shell})$, where $P$ is the number of primitives and $V_{shell}$ is the number of voxels in the expanded AABB. In the worst case, if the shell covers the entire grid, it is $O(P \cdot V_{total})$.
- **Memory Complexity:** $O(V_{total})$ for the bitset mask.
- **Strengths:** Simple to implement; efficient for marking boundary voxels without computing interior occupancy.
- **Limitations:** Does not distinguish between inside and outside; may mark voxels that are not actually close to the surface if the shell radius is too large.
- **When to use:** When a boundary mask is needed for subsequent operations like differentiable rendering or marching cubes.
- **When not to use:** When interior occupancy or signed distance values are required.

## Background
Shell-based Voxelization is part of the Differentiable Voxelization framework [S1, unofficial]. It is used to identify the spatial support for voxelization operations, enabling gradients to flow through the voxelization process [S1, unofficial]. The method is implemented in the `dvx` namespace of the Differentiable Voxelization of Surface Representations repository [S1, unofficial].

## Sources
[S1] Differentiable Voxelization of Surface Representations, https://github.com/mworchel/differentiable-voxelization, fetched 2026-10-29 (unofficial)
