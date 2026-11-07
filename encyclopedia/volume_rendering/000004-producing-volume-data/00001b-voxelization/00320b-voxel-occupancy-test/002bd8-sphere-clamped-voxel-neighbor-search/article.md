# Sphere-Clamped Voxel Neighbor Search


## Overview
Sphere-Clamped Voxel Neighbor Search finds the closest occupied voxel cell to a query point within a spherical radius, constrained to the grid boundaries. It differs from [Point-in-Box Inclusion Test](1400-point-in-box-inclusion-test.md) by returning the nearest neighbor rather than a binary occupancy result, and from [3D Filled Circle Kernel Generation](1520-3d-filled-circle-kernel-generation.md) by performing a runtime search for a specific query point rather than precomputing a static kernel. It serves as the spatial lookup primitive for [PBF Displacement Calculation with Grid-Based Neighbor Search](9026-pbf-displacement-calculation-with-grid-based-neighbor-search.md).

## How it works
**Inputs:**
- `from`: Query point coordinates $(x, y, z)$ in world space.
- `radius`: Search radius in world units.
- `voxel_size_m`: Horizontal voxel width in world units.
- `height_step_m`: Vertical voxel height in world units.
- `base_depth_m`: Vertical offset of the grid base in world units.
- `extent`: Number of voxels in the X and Z dimensions.
- `height_cells`: Number of voxels in the Y dimension.
- `mats`: Array of voxel materials, indexed by linear cell index.
- `pred`: Predicate function returning true if a material is occupied/valid.

**Outputs:**
- `best`: The center coordinates $(cx, cy, cz)$ of the closest valid voxel cell within the sphere.
- `found`: Boolean indicating if any valid cell was found.

**Data Structures:**
- The grid is a 3D array of materials `mats`.
- Linear indexing is computed via `grid.cell_index(ix, iy, iz)`. The exact mapping (e.g., row-major vs column-major) is implementation-specific but must be consistent with `grid.column_center_x`, `grid.column_center_z`, and `grid.cell_bottom_y`.

**Steps:**
1. **Compute Integer Bounds:** Calculate the bounding box of the search sphere in voxel coordinates, clamped to the grid extents.
   - For X and Z axes:
     $$ ix_0 = \text{clamp}\left(\left\lfloor \frac{from.x - radius + 0.5 \cdot \text{voxel\_size\_m}}{\text{voxel\_size\_m}} \right\rfloor, 0, \text{extent} - 1\right) $$
     $$ ix_1 = \text{clamp}\left(\left\lfloor \frac{from.x + radius + 0.5 \cdot \text{voxel\_size\_m}}{\text{voxel\_size\_m}} \right\rfloor, 0, \text{extent} - 1\right) $$
     $$ iz_0 = \text{clamp}\left(\left\lfloor \frac{from.z - radius + 0.5 \cdot \text{voxel\_size\_m}}{\text{voxel\_size\_m}} \right\rfloor, 0, \text{extent} - 1\right) $$
     $$ iz_1 = \text{clamp}\left(\left\lfloor \frac{from.z + radius + 0.5 \cdot \text{voxel\_size\_m}}{\text{voxel\_size\_m}} \right\rfloor, 0, \text{extent} - 1\right) $$
   - For Y axis:
     $$ iy_0 = \text{clamp}\left(\left\lfloor \frac{from.y - radius + \text{base\_depth\_m}}{\text{height\_step\_m}} \right\rfloor, 0, \text{height\_cells} - 1\right) $$
     $$ iy_1 = \text{clamp}\left(\left\lfloor \frac{from.y + radius + \text{base\_depth\_m}}{\text{height\_step\_m}} \right\rfloor, 0, \text{height\_cells} - 1\right) $$
   - Note: The term `0.5 * voxel_size_m` represents the half-voxel offset to align the sphere center with voxel centers. The term `base_depth_m` shifts the Y coordinate to the grid's local vertical origin.

2. **Iterate Bounding Box:** Loop through all integer coordinates $(ix, iy, iz)$ such that $ix_0 \le ix \le ix_1$, $iy_0 \le iy \le iy_1$, and $iz_0 \le iz \le iz_1$.

3. **Check Occupancy:** For each cell, compute the linear index and retrieve the material. If `pred(material)` is false, skip to the next cell.

4. **Compute Cell Center:** Calculate the world coordinates of the current cell center:
   - $cx = \text{grid.column\_center\_x}(ix)$
   - $cz = \text{grid.column\_center\_z}(iz)$
   - $cy = \text{grid.cell\_bottom\_y}(iy) + 0.5 \cdot \text{height\_step\_m}$

5. **Distance Check:** Compute the squared Euclidean distance from `from` to the cell center:
   $$ dx = cx - from.x $$
   $$ dy = cy - from.y $$
   $$ dz = cz - from.z $$
   $$ d^2 = dx^2 + dy^2 + dz^2 $$

6. **Update Best:** If $d^2 \le radius^2$ and (`!found` or $d^2 < best^2$), update `best` to $(cx, cy, cz)$ and `best^2` to $d^2$. Set `found = true`.
   - Ties in distance are resolved by the iteration order (ascending indices), effectively preferring the cell with the lowest linear index.

7. **Return Result:** If `found` is true, return `best`. Otherwise, return `std::nullopt` (or equivalent null indicator).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sphere_clamped_voxel_neighbor_search(from, radius, voxel_size_m, height_step_m, base_depth_m, extent, height_cells, mats, pred, grid):
    r2 = radius * radius
    best = None
    best2 = float('inf')
    found = False

    # Step 1: Compute integer bounds
    ix0 = clamp(floor((from.x - radius + 0.5 * voxel_size_m) / voxel_size_m), 0, extent - 1)
    ix1 = clamp(floor((from.x + radius + 0.5 * voxel_size_m) / voxel_size_m), 0, extent - 1)
    iz0 = clamp(floor((from.z - radius + 0.5 * voxel_size_m) / voxel_size_m), 0, extent - 1)
    iz1 = clamp(floor((from.z + radius + 0.5 * voxel_size_m) / voxel_size_m), 0, extent - 1)
    iy0 = clamp(floor((from.y - radius + base_depth_m) / height_step_m), 0, height_cells - 1)
    iy1 = clamp(floor((from.y + radius + base_depth_m) / height_step_m), 0, height_cells - 1)

    # Step 2: Iterate bounding box
    for iy in range(iy0, iy1 + 1):
        for ix in range(ix0, ix1 + 1):
            for iz in range(iz0, iz1 + 1):
                # Step 3: Check occupancy
                idx = grid.cell_index(ix, iy, iz)
                mat = mats[idx]
                if not pred(mat):
                    continue

                # Step 4: Compute cell center
                cx = grid.column_center_x(ix)
                cz = grid.column_center_z(iz)
                cy = grid.cell_bottom_y(iy) + 0.5 * height_step_m

                # Step 5: Distance check
                dx = cx - from.x
                dy = cy - from.y
                dz = cz - from.z
                d2 = dx * dx + dy * dy + dz * dz

                # Step 6: Update best
                if d2 <= r2 and (not found or d2 < best2):
                    best2 = d2
                    best = (cx, cy, cz)
                    found = True

    # Step 7: Return result
    if found:
        return best
    else:
        return None
```

**Parameters:**
- `radius`: Search radius in world units. Typical values depend on the application scale.
- `from`: Query point.
- `voxel_size_m`: Horizontal voxel size.
- `height_step_m`: Vertical voxel size.
- `base_depth_m`: Vertical offset of the grid base. Clamped to $[0.5, 100.0]$ in some implementations [S1, unofficial].
- `extent`: Grid width/depth in voxels.
- `height_cells`: Grid height in voxels. Clamped to $[16, 512]$ in some implementations [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point errors in distance calculations may cause boundary cases where $d^2$ is extremely close to $r^2$. The condition $d^2 \le r^2$ includes the boundary.
- **Boundaries:** The clamping ensures no out-of-bounds array access. If the query point is far outside the grid, the bounds may be empty or fully clamped, resulting in no search or a search of the nearest corner.
- **Degenerate Input:** If `radius` is 0, the search checks only the voxel containing `from` (if valid). If `voxel_size_m` or `height_step_m` is 0, division by zero occurs.

## Subtopics

### [PBF Displacement Calculation with Grid-Based Neighbor Search](9026-pbf-displacement-calculation-with-grid-based-neighbor-search.md)
Uses this method to find the closest occupied voxel for density constraint enforcement in Particle-Based Fluid simulation. It differs by applying the neighbor search within a physics simulation loop to calculate displacements, whereas this method is a general-purpose spatial query.

## Operating characteristics
- **Time Complexity:** $O(R^3)$, where $R$ is the number of voxels in the search radius along one axis. Specifically, it iterates over the bounding box of the sphere, which contains $\approx \frac{4}{3}\pi R^3$ cells.
- **Memory Complexity:** $O(1)$ auxiliary space, excluding the grid storage.
- **Performance:** Performance depends on the density of valid cells and the radius. In sparse grids, the predicate check is fast, but the iteration over the bounding box remains. In dense grids, more cells are processed.
- **Strengths:** Simple to implement, robust boundary handling, deterministic tie-breaking.
- **Limitations:** Inefficient for very large radii or sparse grids where most cells in the bounding box are invalid. Does not use spatial acceleration structures like hash grids or octrees, which are used in [PBF Displacement Calculation with Grid-Based Neighbor Search](9026-pbf-displacement-calculation-with-grid-based-neighbor-search.md) for better performance in fluid simulations.

## Background
This method is derived from the voxel ocean simulation codebase [S1, unofficial]. It is used as a fundamental building block for spatial queries in voxel-based environments, particularly for fluid simulation and entity interaction.

## Sources
[S1] wlim33/vox-ocean, https://github.com/wlim33/vox-ocean, fetched 2026-10-27 (unofficial)
