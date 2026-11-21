# Sphere-Clamped Voxel Neighbor Search


## Overview
Sphere-Clamped Voxel Neighbor Search retrieves the voxel cell whose center is closest to a query point, restricted to a spherical search radius and bounded by the grid extents. It solves the problem of finding the nearest occupied material cell within a local neighborhood, which is a prerequisite for operations like [Voxel Occupancy Test](1281-voxel-occupancy-test.md) or physics-based neighbor lookups. It differs from [Point-in-Box Inclusion Test](1400-point-in-box-inclusion-test.md) by enforcing a spherical distance constraint rather than an axis-aligned bounding box, and from [Moore Neighbourhood Sampling](16763-moore-neighbourhood-sampling.md) by returning only the single closest valid cell rather than all 27 neighbors.

## How it works
**Inputs:**
- `from`: Query point in world coordinates ($\mathbb{R}^3$).
- `radius`: Search radius in world units.
- `p`: Grid parameters including `voxel_size_m` (X/Z cell width), `height_step_m` (Y cell height), `base_depth_m` (Y-axis offset), `extent` (X/Z grid dimension), and `height_cells` (Y grid dimension).
- `grid`: Spatial data structure providing `column_center_x(ix)`, `column_center_z(iz)`, `cell_bottom_y(iy)`, and `cell_index(ix, iy, iz)`.
- `mats`: Array of material data indexed by linear cell index.
- `pred`: Predicate function `bool(VoxMat)` determining if a material is valid for selection.

**Outputs:**
- `best`: The world coordinates of the center of the closest valid cell, or `None` if no cell is found within the sphere.

**Data Structures:**
- The grid is stored as a linear array `mats` of material objects.
- 3D voxel coordinates $(ix, iy, iz)$ are mapped to a linear index via `grid.cell_index(ix, iy, iz)`.
- Coordinate conversion functions map integer voxel indices to world coordinates:
  - $cx = \text{grid.column\_center\_x}(ix)$
  - $cz = \text{grid.column\_center\_z}(iz)$
  - $cy = \text{grid.cell\_bottom\_y}(iy) + 0.5 \times p.\text{height\_step\_m}$

**Steps:**
1. Compute the integer bounding box of the search sphere clamped to the grid extents.
   - For X and Z axes:
     $$ ix_0 = \text{clampi}\left(\left\lfloor \frac{\text{from}.x - \text{radius} + 0.5 \times p.\text{voxel\_size\_m}}{p.\text{voxel\_size\_m}} \right\rfloor, 0, p.\text{extent} - 1\right) $$
     $$ ix_1 = \text{clampi}\left(\left\lfloor \frac{\text{from}.x + \text{radius} + 0.5 \times p.\text{voxel\_size\_m}}{p.\text{voxel\_size\_m}} \right\rfloor, 0, p.\text{extent} - 1\right) $$
     $$ iz_0 = \text{clampi}\left(\left\lfloor \frac{\text{from}.z - \text{radius} + 0.5 \times p.\text{voxel\_size\_m}}{p.\text{voxel\_size\_m}} \right\rfloor, 0, p.\text{extent} - 1\right) $$
     $$ iz_1 = \text{clampi}\left(\left\lfloor \frac{\text{from}.z + \text{radius} + 0.5 \times p.\text{voxel\_size\_m}}{p.\text{voxel\_size\_m}} \right\rfloor, 0, p.\text{extent} - 1\right) $$
   - For Y axis:
     $$ iy_0 = \text{clampi}\left(\left\lfloor \frac{\text{from}.y - \text{radius} + p.\text{base\_depth\_m}}{p.\text{height\_step\_m}} \right\rfloor, 0, p.\text{height\_cells} - 1\right) $$
     $$ iy_1 = \text{clampi}\left(\left\lfloor \frac{\text{from}.y + \text{radius} + p.\text{base\_depth\_m}}{p.\text{height\_step\_m}} \right\rfloor, 0, p.\text{height\_cells} - 1\right) $$
   - Note: The term $0.5 \times p.\text{voxel\_size\_m}$ represents half the voxel size, used to align the floor/ceiling of the search range with voxel boundaries. The Y-axis uses `base_depth_m` instead of half-height, implying an offset coordinate system.
2. Initialize `best` to `None`, `best2` to $\infty$, and `found` to `false`.
3. Iterate through all integer coordinates $(ix, iy, iz)$ such that $ix_0 \le ix \le ix_1$, $iy_0 \le iy \le iy_1$, and $iz_0 \le iz \le iz_1$.
4. For each cell, compute the linear index `idx = grid.cell_index(ix, iy, iz)` and retrieve the material `m = mats[idx]`.
5. If `pred(m)` is false, continue to the next cell.
6. Compute the center coordinates of the current cell:
   - $cx = \text{grid.column\_center\_x}(ix)$
   - $cz = \text{grid.column\_center\_z}(iz)$
   - $cy = \text{grid.cell\_bottom\_y}(iy) + 0.5 \times p.\text{height\_step\_m}$
7. Compute the squared Euclidean distance from `from` to the cell center:
   - $dx = cx - \text{from}.x$
   - $dy = cy - \text{from}.y$
   - $dz = cz - \text{from}.z$
   - $d2 = dx^2 + dy^2 + dz^2$
8. If $d2 \le \text{radius}^2$ and (`!found` or $d2 < \text{best2}$):
   - Update `best2 = d2`
   - Update `best = (cx, cy, cz)`
   - Set `found = true`
   - Note: The strict inequality $d2 < \text{best2}$ ensures that if multiple cells are equidistant, the first one encountered in the iteration order (lowest index) is retained.
9. After the loop, if `found` is true, return `best`. Otherwise, return `None`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sphere_clamped_voxel_neighbor_search(from: Vec3, radius: float, p: GridParams, grid: Grid, mats: List[VoxMat], pred: Callable[[VoxMat], bool]) -> Optional[Vec3]:
    r2 = radius * radius
    
    # Compute integer bounds for X and Z
    half_voxel = 0.5 * p.voxel_size_m
    ix0 = clamp(floor((from.x - radius + half_voxel) / p.voxel_size_m), 0, p.extent - 1)
    ix1 = clamp(floor((from.x + radius + half_voxel) / p.voxel_size_m), 0, p.extent - 1)
    iz0 = clamp(floor((from.z - radius + half_voxel) / p.voxel_size_m), 0, p.extent - 1)
    iz1 = clamp(floor((from.z + radius + half_voxel) / p.voxel_size_m), 0, p.extent - 1)
    
    # Compute integer bounds for Y
    iy0 = clamp(floor((from.y - radius + p.base_depth_m) / p.height_step_m), 0, p.height_cells - 1)
    iy1 = clamp(floor((from.y + radius + p.base_depth_m) / p.height_step_m), 0, p.height_cells - 1)
    
    best = None
    best2 = float('inf')
    found = False
    
    for iy in range(iy0, iy1 + 1):
        for ix in range(ix0, ix1 + 1):
            for iz in range(iz0, iz1 + 1):
                idx = grid.cell_index(ix, iy, iz)
                m = mats[idx]
                
                if not pred(m):
                    continue
                
                cx = grid.column_center_x(ix)
                cz = grid.column_center_z(iz)
                cy = grid.cell_bottom_y(iy) + 0.5 * p.height_step_m
                
                dx = cx - from.x
                dy = cy - from.y
                dz = cz - from.z
                d2 = dx*dx + dy*dy + dz*dz
                
                if d2 <= r2 and (not found or d2 < best2):
                    best2 = d2
                    best = Vec3(cx, cy, cz)
                    found = True
                    
    return best if found else None
```

**Parameters:**
- `radius`: Search radius in world units. Typical values depend on the application scale; in the source implementation, it is used directly without normalization.
- `from`: Query point.
- `p.voxel_size_m`: Width of a voxel in X/Z dimensions.
- `p.height_step_m`: Height of a voxel in Y dimension.
- `p.base_depth_m`: Offset for Y-axis voxel indexing. Clamped to $[0.5, 100.0]$ in configuration.
- `p.extent`: Number of voxels in X and Z dimensions.
- `p.height_cells`: Number of voxels in Y dimension. Clamped to $[16, 512]$ in configuration.

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point division and floor operations may cause off-by-one errors at boundaries. The clamping to grid extents prevents out-of-bounds access but may exclude valid neighbors if the query point is very close to the edge.
- **Ties:** The strict inequality $d2 < \text{best2}$ means that if two cells are exactly equidistant, the one with the lower linear index (determined by iteration order) is chosen.
- **Empty Grid:** If no cells satisfy the predicate or are within the radius, the function returns `None`.
- **Degenerate Input:** If `radius` is 0, only the cell containing `from` (if any) is considered. If `voxel_size_m` or `height_step_m` is 0, division by zero occurs.

## Subtopics

### [PBF Displacement Calculation with Grid-Based Neighbor Search](9026-pbf-displacement-calculation-with-grid-based-neighbor-search.md)
PBF Displacement Calculation with Grid-Based Neighbor Search is a particle-based fluid simulation step that calculates particle displacements to enforce incompressibility via density constraints. It operates by using a spatial hash grid to identify neighboring particles within a search radius and applying separation forces to push particles apart if they are too close. Unlike its parent method, Sphere-Clamped Voxel Neighbor Search, which focuses on efficient neighbor retrieval, this method performs the actual physical constraint resolution. It differs from Local Short-Range Repulsion, which optimizes geometric distribution regularity through tangential force projection, by strictly enforcing volumetric incompressibility [15114-local-short-range-repulsion.md].

### [Moore Neighbourhood Sampling](16763-moore-neighbourhood-sampling.md)
Moore Neighbourhood Sampling reads the values of all 27 voxels in a 3x3x3 block centered on a target voxel, returning both the data and a bitmask indicating which neighbors are non-empty. It serves as the foundational data access primitive for local voxel operations, such as the Sphere-Clamped Voxel Neighbor Search. It is distinguished from its sibling [PBF Displacement Calculation with Grid-Based Neighbor Search](9026-pbf-displacement-calculation-with-grid-based-neighbor-search.md) by its fixed, exhaustive 27-neighbor scope rather than a physics-based radius search.

## Operating characteristics
- **Time Complexity:** $O(R^3)$, where $R$ is the number of voxels along one axis of the search sphere. Specifically, it iterates over the bounding box of the sphere, which contains approximately $\frac{4}{3}\pi r^3$ voxels, where $r$ is the radius in voxel units.
- **Memory Complexity:** $O(1)$ auxiliary space, excluding the grid storage.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Performance scales linearly with the volume of the search sphere. For large radii, this becomes expensive compared to sparse data structures.
- **Strengths:** Simple to implement, robust to grid boundaries, and provides a deterministic result for the closest cell.
- **Limitations:** Inefficient for sparse grids as it iterates over empty space. Does not support early termination based on distance if the bounding box is large.
- **When to Use:** When a single closest neighbor within a spherical radius is needed, and the grid is dense or the search radius is small.
- **When Not to Use:** For sparse grids with large search radii, or when multiple neighbors are required.

## Background
The method is derived from the voxel ocean simulation codebase, specifically the neighbor search logic used for fluid or particle interactions. It is part of a larger system that includes voxelization, marching, and rendering configurations. The implementation uses a clamped bounding box approach to efficiently limit the search space to a sphere, avoiding the need for more complex spatial partitioning structures like KD-trees or octrees for small-scale searches.

## Sources
[S1] wlim33/vox-ocean, https://github.com/wlim33/vox-ocean, fetched 2026-10-27 (unofficial)
