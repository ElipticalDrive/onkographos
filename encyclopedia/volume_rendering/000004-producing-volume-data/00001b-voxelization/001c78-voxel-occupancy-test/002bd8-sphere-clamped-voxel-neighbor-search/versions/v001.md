# Sphere-Clamped Voxel Neighbor Search


## Overview
Sphere-Clamped Voxel Neighbor Search finds the closest voxel cell to a query point within a specified radius, constrained to the grid boundaries. It solves the problem of locating the nearest occupied or valid material cell in a voxelized environment for operations like ray casting or neighbor lookups. It differs from [Point-in-Box Inclusion Test](1400-point-in-box-inclusion-test.md) by using a spherical distance metric and from [Voxel Filled Check](1705-voxel-filled-check.md) by returning the specific closest cell rather than a boolean occupancy status.

## How it works
**Inputs:**
- `from`: Query point in world coordinates (`glm::vec3` or equivalent).
- `radius`: Search radius in world units.
- `p`: Configuration struct containing:
  - `voxel_size_m`: World size of a single voxel in X/Z.
  - `height_step_m`: World size of a single voxel in Y.
  - `base_depth_m`: Vertical offset for the Y-axis grid origin.
  - `extent`: Grid width/depth in voxels (X and Z dimensions).
  - `height_cells`: Grid height in voxels (Y dimension).
- `grid`: Voxel grid object providing coordinate mapping functions.
- `mats`: Array of material data indexed by linear cell index.
- `pred`: Predicate function `bool(VoxMat)` to filter valid materials.

**Outputs:**
- `best`: The world coordinates of the center of the closest valid cell, or `std::nullopt` if no valid cell is found within the sphere.

**Data Structures:**
- The grid uses a linear indexing scheme via `grid.cell_index(ix, iy, iz)` to map 3D coordinates to a 1D array index for `mats`.
- Coordinate accessors:
  - `grid.column_center_x(ix)`: Returns the world X coordinate of the center of column `ix`.
  - `grid.column_center_z(iz)`: Returns the world Z coordinate of the center of column `iz`.
  - `grid.cell_bottom_y(iy)`: Returns the world Y coordinate of the bottom face of cell `iy`.

**Algorithm Steps:**

1. **Compute Integer Bounds:**
   Calculate the bounding box of the search sphere in voxel coordinates, clamped to the grid extents.
   
   For X and Z axes:
   $$ ix_0 = \text{clampi}\left(\left\lfloor \frac{\text{from}.x - \text{radius} + 0.5 \cdot \text{voxel\_size\_m}}{\text{voxel\_size\_m}} \right\rfloor, 0, \text{extent} - 1\right) $$
   $$ ix_1 = \text{clampi}\left(\left\lfloor \frac{\text{from}.x + \text{radius} + 0.5 \cdot \text{voxel\_size\_m}}{\text{voxel\_size\_m}} \right\rfloor, 0, \text{extent} - 1\right) $$
   $$ iz_0 = \text{clampi}\left(\left\lfloor \frac{\text{from}.z - \text{radius} + 0.5 \cdot \text{voxel\_size\_m}}{\text{voxel\_size\_m}} \right\rfloor, 0, \text{extent} - 1\right) $$
   $$ iz_1 = \text{clampi}\left(\left\lfloor \frac{\text{from}.z + \text{radius} + 0.5 \cdot \text{voxel\_size\_m}}{\text{voxel\_size\_m}} \right\rfloor, 0, \text{extent} - 1\right) $$

   For Y axis:
   $$ iy_0 = \text{clampi}\left(\left\lfloor \frac{\text{from}.y - \text{radius} + \text{base\_depth\_m}}{\text{height\_step\_m}} \right\rfloor, 0, \text{height\_cells} - 1\right) $$
   $$ iy_1 = \text{clampi}\left(\left\lfloor \frac{\text{from}.y + \text{radius} + \text{base\_depth\_m}}{\text{height\_step\_m}} \right\rfloor, 0, \text{height\_cells} - 1\right) $$

   Where $\text{clampi}(v, min, max)$ clamps value $v$ to $[min, max]$.

2. **Initialize Search State:**
   Set `found = false`, `best2 = infinity`, `best = null`.

3. **Iterate and Check:**
   Loop through all integer coordinates $(ix, iy, iz)$ such that $ix_0 \le ix \le ix_1$, $iy_0 \le iy \le iy_1$, and $iz_0 \le iz \le iz_1$.
   
   a. **Material Check:**
      Compute linear index: $idx = \text{grid.cell\_index}(ix, iy, iz)$.
      If $\neg \text{pred}(\text{mats}[idx])$, continue to next cell.
   
   b. **Center Calculation:**
      $$ cx = \text{grid.column\_center\_x}(ix) $$
      $$ cz = \text{grid.column\_center\_z}(iz) $$
      $$ cy = \text{grid.cell\_bottom\_y}(iy) + 0.5 \cdot \text{height\_step\_m} $$
   
   c. **Distance Check:**
      $$ dx = cx - \text{from}.x $$
      $$ dy = cy - \text{from}.y $$
      $$ dz = cz - \text{from}.z $$
      $$ d2 = dx^2 + dy^2 + dz^2 $$
      
      If $d2 \le \text{radius}^2$ and ($\neg \text{found}$ or $d2 < \text{best2}$):
      $$ \text{best2} = d2 $$
      $$ \text{best} = (cx, cy, cz) $$
      $$ \text{found} = \text{true} $$

4. **Return Result:**
   If `found` is true, return `best`. Otherwise, return `std::nullopt`.

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point errors in distance comparison may affect tie-breaking. The implementation uses strict inequality ($<$) for updates, ensuring that ties resolve to the cell with the lowest index due to the ascending iteration order of the loops.
- **Boundaries:** The clamping ensures no out-of-bounds access, but cells near the boundary may be partially outside the sphere. The algorithm correctly handles this by checking the center distance.
- **Empty Grid:** If no cells satisfy the predicate, the function returns `std::nullopt`.

## Operating characteristics
- **Time Complexity:** $O(R^3)$ where $R$ is the number of voxels along the radius dimension, bounded by the grid dimensions.
- **Memory Complexity:** $O(1)$ auxiliary space.
- **Strengths:** Simple to implement; robust boundary handling; deterministic tie-breaking.
- **Limitations:** Brute-force search within the bounding box; may be slow for large radii or high-resolution grids.
- **When to Use:** Small search radii; need for exact closest cell within a sphere; sparse grids where predicate filtering reduces work.
- **When Not to Use:** Large radii; performance-critical paths requiring spatial acceleration structures (e.g., BVH, Octree).

## Background
Originated in the Vox-Ocean project [S1, unofficial], a voxel-based ocean simulation and rendering system. The method is used for efficient neighbor lookup in voxelized environments, particularly for surface generation and interaction queries.

## Sources
[S1] wlim33/vox-ocean, https://github.com/wlim33/vox-ocean, fetched 2026-10-27 (unofficial)
