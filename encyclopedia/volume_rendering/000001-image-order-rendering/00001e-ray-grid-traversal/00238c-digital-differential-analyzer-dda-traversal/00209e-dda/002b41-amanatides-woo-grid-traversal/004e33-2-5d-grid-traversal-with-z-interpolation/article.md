# 2.5D Grid Traversal with Z-interpolation

Also called: 2.5D DDA with Z-check, 2.5D Grid Walk

## Overview
A method to traverse a 2D grid where each cell has a min/max height range, by extending a 2D DDA algorithm to calculate the Z-coordinate at each cell entry/exit to check for overlap with the cell's height range [S1, unofficial]. It solves the problem of efficiently culling cells in a 2.5D terrain or height-field representation during ray traversal, avoiding the cell-skipping errors of Bresenham's algorithm when the ray has a significant Z-component [S1, unofficial]. It builds on [Amanatides–Woo grid traversal](5003-amanatides-woo-grid-traversal.md) by adding Z-interpolation logic to the standard 2D grid walk.

## How it works
**Inputs:**
- Ray origin $(start.x, start.y, start.z)$ and end $(end.x, end.y, end.z)$ [S1, unofficial].
- Grid resolution $m\_gridresolution$ [S1, unofficial].
- Step direction $(step.x, step.y)$, where each component is $+1$ or $-1$ indicating the traversal direction in that axis [S1, unofficial].
- Cell height bounds: for each grid cell $(i, j)$, a minimum height $min\_z_{i,j}$ and maximum height $max\_z_{i,j}$ [S1, unofficial].

**Outputs:**
- Sequence of grid cells $(i, j)$ intersected by the ray segment that overlap with their respective height ranges [S1, unofficial].

**Data Structures:**
- `deltaZ`: A 2D vector storing the change in Z per unit distance in X and Y respectively [S1, unofficial].
- `currentOffset`: A 2D vector storing the accumulated distance along X and Y from the ray origin to the current grid boundary being crossed [S1, unofficial].

**Formulas:**
1.  **Z-offset per unit distance:**
    $$fzoffset = end.z - start.z$$
    $$deltaZ.x = \frac{fzoffset}{|end.x - start.x|}$$
    $$deltaZ.y = \frac{fzoffset}{|end.y - start.y|}$$
    [S1, unofficial]

2.  **Initial Z-offsets at starting grid boundaries:**
    $$currentOffset.x = \begin{cases} (start.x) \cdot m\_gridresolution - start.x & \text{if } step.x < 0 \\ (start.x + 1) \cdot m\_gridresolution - start.x & \text{if } step.x > 0 \end{cases}$$
    $$currentOffset.y = \begin{cases} (start.y) \cdot m\_gridresolution - start.y & \text{if } step.y < 0 \\ (start.y + 1) \cdot m\_gridresolution - start.y & \text{if } step.y > 0 \end{cases}$$
    [S1, unofficial]
    *Note: The source uses `(step.x > 0 ? start.x : start.x + 1)`. This implies that if stepping in the negative direction, the first boundary is at `start.x` (the current integer coordinate), and if stepping in the positive direction, the first boundary is at `start.x + 1` (the next integer coordinate). This assumes `start.x` is not exactly on a grid line, or handles the boundary case by treating the current cell as the starting point.*

3.  **Z-value at cell entry/exit:**
    When stepping in the X axis:
    $$z = currentOffset.x \cdot deltaZ.x + start.z$$
    [S1, unofficial]
    When stepping in the Y axis:
    $$z = currentOffset.y \cdot deltaZ.y + start.z$$
    [S1, unofficial]

**Algorithm Steps:**
1.  Calculate `fzoffset`, `deltaZ`, and `currentOffset` using the formulas above [S1, unofficial].
2.  Initialize the current grid cell $(cx, cy)$ to the cell containing $(start.x, start.y)$ [S1, unofficial].
3.  Check if the ray segment overlaps with the height range of the current cell $(cx, cy)$. If not, terminate or skip [S1, unofficial].
4.  Enter the traversal loop, which continues until the ray exits the grid or the end point is reached:
    a.  Determine which axis boundary is next (X or Y) based on the standard Amanatides–Woo logic (comparing `currentDistance.x` and `currentDistance.y`) [S1, unofficial].
    b.  If stepping in the X axis:
        i.  Increment `currentOffset.x` by `m_gridresolution` [S1, unofficial].
        ii. Calculate $z$ using $z = currentOffset.x \cdot deltaZ.x + start.z$ [S1, unofficial].
        iii. Update the current cell index $cx$ by adding $step.x$ [S1, unofficial].
    c.  If stepping in the Y axis:
        i.  Increment `currentOffset.y` by `m_gridresolution` [S1, unofficial].
        ii. Calculate $z$ using $z = currentOffset.y \cdot deltaZ.y + start.z$ [S1, unofficial].
        iii. Update the current cell index $cy$ by adding $step.y$ [S1, unofficial].
    d.  Check if the new cell $(cx, cy)$ overlaps with its height range using the calculated $z$ value (representing the Z-coordinate at the boundary crossed) [S1, unofficial].
    e.  If the cell overlaps, process it (e.g., add to list, sample volume) [S1, unofficial].
    f.  If the ray has passed the end point, terminate [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_25d_grid(start, end, grid_resolution, step_x, step_y, get_cell_heights):
    """
    Traverses a 2.5D grid, checking Z-overlap for each cell.
    start: (x, y, z) tuple
    end: (x, y, z) tuple
    grid_resolution: float, size of one grid cell
    step_x: int, +1 or -1
    step_y: int, +1 or -1
    get_cell_heights: function that returns (min_z, max_z) for a given (cx, cy)
    """
    # 1. Calculate deltas and initial offsets
    fzoffset = end[2] - start[2]
    
    # Handle potential division by zero if ray is parallel to an axis
    dx = abs(end[0] - start[0])
    dy = abs(end[1] - start[1])
    
    if dx == 0:
        delta_z_x = 0.0
    else:
        delta_z_x = fzoffset / dx
        
    if dy == 0:
        delta_z_y = 0.0
    else:
        delta_z_y = fzoffset / dy
        
    # Initial offsets at the first grid boundary
    if step_x > 0:
        current_offset_x = (start[0] + 1) * grid_resolution - start[0]
    else:
        current_offset_x = start[0] * grid_resolution - start[0]
        
    if step_y > 0:
        current_offset_y = (start[1] + 1) * grid_resolution - start[1]
    else:
        current_offset_y = start[1] * grid_resolution - start[1]
        
    # Initialize current cell
    cx = int(start[0])
    cy = int(start[1])
    
    # Check initial cell
    min_z, max_z = get_cell_heights(cx, cy)
    # Check if the ray segment overlaps with the cell's height range
    # This is a simplified check; a robust implementation would check the 
    # full segment against the cell's Z-range
    if not (min_z <= start[2] <= max_z or min_z <= end[2] <= max_z):
        # Optional: Check if the interpolated Z at boundaries overlaps
        pass
        
    cells = []
    
    # Standard Amanatides-Woo distance variables (assumed to be maintained)
    # current_distance_x, current_distance_y, t_max_x, t_max_y
    # These are part of the parent algorithm [Amanatides–Woo grid traversal](5003-amanatides-woo-grid-traversal.md)
    
    while not ray_exited_grid(cx, cy, grid_resolution) and not ray_passed_end(start, end, cx, cy):
        # Determine which axis to step
        if current_distance_x < current_distance_y:
            # Step in X
            current_offset_x += grid_resolution
            z_at_boundary = current_offset_x * delta_z_x + start[2]
            cx += step_x
            # Update Amanatides-Woo variables for X
            current_distance_x += t_max_x
            t_max_x += t_delta_x
        else:
            # Step in Y
            current_offset_y += grid_resolution
            z_at_boundary = current_offset_y * delta_z_y + start[2]
            cy += step_y
            # Update Amanatides-Woo variables for Y
            current_distance_y += t_max_y
            t_max_y += t_delta_y
            
        # Check overlap with cell height range
        min_z, max_z = get_cell_heights(cx, cy)
        
        # Check if the ray segment within this cell overlaps with [min_z, max_z]
        # A simple check is to see if the start Z of the cell (or interpolated Z) is within range
        # More robust: check if the segment from previous Z to current Z intersects [min_z, max_z]
        if min_z <= z_at_boundary <= max_z:
            cells.append((cx, cy))
            
    return cells
```

**Edge Cases and Pitfalls:**
- **Division by Zero:** If the ray is parallel to the X or Y axis (i.e., $end.x = start.x$ or $end.y = start.y$), the corresponding `deltaZ` component involves division by zero. The source does not specify handling, but logically, if $dx=0$, $deltaZ.x$ should be $0$ or handled separately as the Z does not change relative to X movement [S1, unofficial].
- **Precision Errors:** Accumulating `currentOffset` and `currentDistance` can lead to floating-point precision errors over long traversals. The source mentions that an error in Z reaching a critical level indicates being outside the min/max, suggesting a tolerance check might be needed [S1, unofficial].
- **Boundary Conditions:** The initialization of `currentOffset` assumes the starting point is not exactly on a grid line. If it is, the behavior might be ambiguous. The source uses `start.x` or `start.x + 1` based on step direction, which handles the immediate next boundary [S1, unofficial].
- **Bresenham Inadequacy:** Bresenham's algorithm is insufficient because it only finds one cell in the non-primary axis for each step of the primary axis, skipping cells that are intersected by the line (e.g., cell (3,4) in the example sequence (2,3), (3,3), (3,4), (4,4)) [S1, unofficial].

## Subtopics

### [Amanatides–Woo grid traversal (fixed-point Q24.8 variant)](5006-amanatides-woo-grid-traversal-fixed-point-q24-8-variant.md)
Uses fixed-point arithmetic for grid traversal to avoid floating-point precision issues. Choose when high precision is required and floating-point operations are expensive.

### [Compiled Per-Ray Triangle Grid Walk](5198-compiled-per-ray-triangle-grid-walk.md)
Traverses a grid of triangles, compiling a list of triangles to test. Choose when dealing with unstructured triangle meshes rather than regular grids.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of grid cells intersected by the ray, same as the underlying Amanatides–Woo traversal [S1, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space, excluding the output list of cells [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Strengths:** Efficiently culls cells based on Z-height, avoiding unnecessary volume sampling or ray-triangle intersection tests for cells outside the height range [S1, unofficial].
- **Limitations:** Requires pre-computed or accessible cell height bounds. Precision issues may arise with large grids or long rays [S1, unofficial].
- **When to Use:** When traversing a 2D grid with associated Z-heights (e.g., terrain, height fields) and needing to quickly discard cells that do not overlap with the ray's Z-range [S1, unofficial].
- **When Not to Use:** When the grid is purely 2D without Z-heights, or when the ray is axis-aligned and parallel to X or Y (requires special handling) [S1, unofficial].

## Background
This method was proposed in a Stack Overflow answer by Dean Harris in July 2012, addressing a user's need to traverse a 2.5D grid with min/max height ranges [S1, unofficial]. It extends the 2D DDA algorithm to include Z-interpolation, allowing for Z-overlap checks at each cell boundary [S1, unofficial].

## Sources
[S1] Traverse a 2.5D grid, https://stackoverflow.com/questions/11694886, fetched 2026-10-25 (unofficial)
