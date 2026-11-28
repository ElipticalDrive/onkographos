# Empty-Space Skipping Grid

Also called: Grid acceleration data structure

## Overview
The Empty-Space Skipping Grid is a 3D regular grid acceleration structure used in conjunction with the Volumetric Depth Image (VDI) to optimize raycasting by skipping empty regions along a ray's path [S1]. It reduces memory accesses by allowing rays to jump over grid cells that contain no supersegments, thereby avoiding unnecessary traversal of empty supersegment lists [S1]. Unlike hierarchical grids such as octrees or SparseLeap, this structure is chosen for its ability to be constructed dynamically during VDI generation using atomic operations, trading off some skipping efficiency for simpler, parallelizable build times [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A Volumetric Depth Image (VDI) consisting of supersegment lists $\mathbb{L}_i$ and supersegments $\mathbb{S}_j^i$; a viewing frustum defined in view space.
*   **Output:** A 3D regular grid where each cell stores an integer count of overlapping supersegments.

**Data Structures**
The grid is a 3D regular grid aligned with the viewing plane.
*   **Cell Layout:** Each cell stores a single integer value representing the count of supersegments $\mathbb{S}$ overlapping with that cell [S1].
*   **Dimensions:**
    *   Along the viewing plane (width and height), each cell spans an equal number of supersegment lists $\mathbb{L}$ [S1].
    *   Along the depth axis (view space), each cell has a constant depth extent [S1].
    *   The depth of a cell in view space is larger than its width or height to account for the anisotropic nature of the VDI, where there are fewer supersegments per list than there are lists [S1].

**Algorithm Steps**
1.  **Grid Construction:** During VDI generation, for each generated supersegment $\mathbb{S}_j^i$, determine the grid cell it overlaps. Perform an atomic add operation on that cell to increment its count [S1].
2.  **Ray Traversal:** When a ray traverses the grid:
    a.  Query the grid cell at the current ray position [S1].
    b.  If the cell value is 0, jump the ray to the other end of that cell (skipping the empty region) [S1].
    c.  If the cell value is non-zero, proceed to traverse the supersegment lists within that depth range [S1].
3.  **Query Cost:** Querying the grid data structure requires one memory access per cell traversed [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Grid construction during VDI generation
# grid is a 3D array initialized to 0
# For each supersegment S_j^i generated:
#     cell_coords = get_grid_cell_coords(S_j^i.position)
#     atomic_add(grid[cell_coords], 1)

# Ray traversal
def traverse_ray_with_grid(ray, grid):
    current_pos = ray.origin
    direction = ray.direction
    
    while current_pos is within bounds:
        # Determine which grid cell the ray is currently in
        cell = get_grid_cell(current_pos, grid.cell_size)
        
        # Check if the cell is empty
        if grid[cell] == 0:
            # Jump to the end of the current cell
            # Calculate the exit point of the cell along the ray
            t_exit = calculate_cell_exit_t(current_pos, direction, cell, grid.cell_size)
            current_pos = ray.origin + direction * t_exit
        else:
            # Cell is not empty; proceed to process supersegments
            # This involves traversing the supersegment lists in this depth range
            process_supersegments_in_cell(ray, cell, grid)
            # Move ray forward past the cell depth
            t_exit = calculate_cell_exit_t(current_pos, direction, cell, grid.cell_size)
            current_pos = ray.origin + direction * t_exit
```

**Parameters**
*   **Grid Cell Depth:** The depth extent of a cell in view space is constant and larger than its width/height [S1].
*   **Grid Cell Width/Height:** Each cell spans an equal number of supersegment lists $\mathbb{L}$ along both dimensions of the viewing plane [S1].

**Edge Cases and Pitfalls**
*   **NDC Distortion:** In Normalized Device Coordinate (NDC) space, grid cells appear larger toward the near plane and smaller toward the far plane due to the constant view-space depth extent [S1]. Implementers must account for this non-uniform scaling when mapping ray positions to grid indices in NDC.
*   **Empty Region Traversal:** Without the grid, a ray must sample each list $\mathbb{L}_i$ at least once to determine if it is empty, leading to excessive memory accesses [S1]. The grid mitigates this but requires correct atomic operations during construction to ensure accuracy.

## Operating characteristics
*   **Construction Time:** Low, as it allows construction during VDI generation via atomic adds [S1]. This is simpler than building hierarchical structures dynamically [S1].
*   **Memory Access:** One memory access per grid cell traversed [S1].
*   **Performance:** Reduces memory accesses in empty regions by allowing rays to jump over cells with zero supersegments [S1]. Performance depends on the number of memory accesses and calculations; high frame rates are achieved near the original view direction, but performance reduces with larger viewpoint deviations [S1].
*   **Comparison:** Chosen over hierarchical grids (octree, SparseLeap) because it allows construction during VDI generation via atomic adds, whereas hierarchical structures are more complex to build dynamically [S1]. Hierarchical grids may provide better empty-space skipping performance but at the cost of build complexity [S1].

## Background
The Empty-Space Skipping Grid was introduced in the context of efficient raycasting of Volumetric Depth Images (VDI) for remote visualization of large volumes at high frame rates [S1]. It addresses the limitation of VDI alone, where rays cannot efficiently query empty regions without sampling each supersegment list [S1]. The structure was selected over hierarchical alternatives like octrees or SparseLeap to enable parallel construction during VDI generation [S1].

## Sources
[S1] Efficient Raycasting of Volumetric Depth Images for Remote Visualization of Large Volumes at High Frame Rates, https://doi.org/10.1109/pacificvis56936.2023.00014, fetched 2026-10-24
