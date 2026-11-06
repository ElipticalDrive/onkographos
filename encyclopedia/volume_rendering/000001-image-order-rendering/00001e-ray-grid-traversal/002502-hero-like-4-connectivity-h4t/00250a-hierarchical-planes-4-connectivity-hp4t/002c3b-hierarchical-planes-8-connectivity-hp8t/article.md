# Hierarchical Planes 8-Connectivity (HP8T)


## Overview
Hierarchical Planes 8-Connectivity (HP8T) is a ray traversal algorithm for hierarchical grids (such as Octrees or Sparse Voxel Octrees) that simplifies the update logic of its predecessor, Hierarchical Planes 4-Connectivity (HP4T)[S1, unofficial]. While HP4T enforces strict 4-connectivity (no diagonal jumps) by clamping the non-hit axis, HP8T removes this clamping to allow 8-connectivity, permitting diagonal jumps between cells[S1, unofficial]. This simplification reduces computational overhead but introduces the requirement to floor fractional grid positions before data access and creates ambiguities in surface normal calculation when using simple comparison variables[S1, unofficial].

## How it works
HP8T builds on [Hierarchical Planes 4-Connectivity (HP4T)](3444-hierarchical-planes-4-connectivity-hp4t.md) by modifying the update step for the axis that is not currently being hit by the ray plane intersection[S1, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   `ray_position`: The starting position of the ray in world space.
    *   `ray_dir`: The normalized direction vector of the ray.
    *   `inv_dir`: The reciprocal of `ray_dir` ($1.0 / ray\_dir$).
    *   `cellsize`: The side length of the uniform grid cells.
    *   `is_dir_pos`: A vector indicating the positive direction of traversal for each axis (typically 1.0 if `ray_dir` > 0, else 0.0).
    *   `is_ray_neg`: A vector indicating the negative direction (typically -1.0 if `ray_dir` < 0, else 0.0). Note: In the source code, this is used to adjust the plane position.
    *   `grid_width`, `grid_height`: The dimensions of the 2D grid slice being traversed.
    *   `grid_position`: The current continuous position of the ray within the grid.
*   **Outputs**:
    *   Updated `grid_position` for each step.
    *   `t`: The distance to the next plane intersection.

### Data Structures
The method operates on standard floating-point vectors. No complex data structures are defined within the traversal loop itself, other than the grid data structure accessed via the floored `grid_position`.

### Steps
1.  **Calculate Planes**: Determine the world coordinates of the next grid planes for the X and Y axes based on the current `grid_position`.
    $$ planes = (\lfloor grid\_position / cellsize \rfloor + is\_dir\_pos) \times cellsize $$
2.  **Calculate Total T**: Compute the distance along the ray to reach these planes.
    $$ total\_t = (planes - ray\_position) \times inv\_dir $$
3.  **Determine Hit Axis**: Compare `total_t.x` and `total_t.y`.
    *   If `total_t.x <= total_t.y`: The ray hits the X plane first.
    *   Else: The ray hits the Y plane first.
4.  **Update Position**:
    *   **If X is hit**:
        *   Update X coordinate to the plane position:
            $$ grid\_position.x = planes.x - is\_ray\_neg.x $$
        *   Update Y coordinate by adding the delta corresponding to the travel time `total_t.x`:
            $$ grid\_position.y += ray\_dir.y \times \max(0.0, total\_t.x - (grid\_position.y - ray\_position.y) \times inv\_dir.y) $$
        *   Set $t = total\_t.x$.
    *   **If Y is hit**:
        *   Update Y coordinate to the plane position:
            $$ grid\_position.y = planes.y - is\_ray\_neg.y $$
        *   Update X coordinate by adding the delta corresponding to the travel time `total_t.y`:
            $$ grid\_position.x += ray\_dir.x \times \max(0.0, total\_t.y - (grid\_position.x - ray\_position.x) \times inv\_dir.x) $$
        *   Set $t = total\_t.y$.
5.  **Access Grid**: Before reading the grid data at the new position, floor the `grid_position` components to obtain integer indices.
    $$ index = \lfloor grid\_position \rfloor $$

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_hp8t(ray_position, ray_dir, inv_dir, cellsize, is_dir_pos, is_ray_neg, grid_width, grid_height, grid_position):
    """
    Traverses a 2D grid using HP8T logic.
    Assumes grid_position is initialized to the ray's starting grid position.
    """
    while 0 <= grid_position.x < grid_width and 0 <= grid_position.y < grid_height:
        # 1. Calculate planes
        # floor is applied element-wise
        planes = ((floor(grid_position / cellsize) + is_dir_pos) * cellsize)
        
        # 2. Calculate total_t
        total_t = (planes - ray_position) * inv_dir
        
        if total_t.x <= total_t.y:
            # Hit X plane
            # 3. Update X
            grid_position.x = planes.x - is_ray_neg.x
            
            # 4. Update Y (simplified delta addition, no clamping)
            # Note: The term (grid_position.y - ray_position.y) * inv_dir.y 
            # represents the current t value for the Y axis relative to the ray origin.
            delta_t_y = total_t.x - ((grid_position.y - ray_position.y) * inv_dir.y)
            grid_position.y += ray_dir.y * max(0.0, delta_t_y)
            
            t = total_t.x
        else:
            # Hit Y plane
            # 5. Update Y
            grid_position.y = planes.y - is_ray_neg.y
            
            # 6. Update X (simplified delta addition, no clamping)
            delta_t_x = total_t.y - ((grid_position.x - ray_position.x) * inv_dir.x)
            grid_position.x += ray_dir.x * max(0.0, delta_t_x)
            
            t = total_t.y
            
        # Access grid data using floored position
        # index_x = floor(grid_position.x)
        # index_y = floor(grid_position.y)
        # read_grid(index_x, index_y)
```

### Parameters
*   `cellsize`: The resolution of the underlying uniform grid.
*   `is_dir_pos`, `is_ray_neg`: Derived from `ray_dir`. `is_dir_pos` is 1.0 where `ray_dir > 0`, else 0.0. `is_ray_neg` is -1.0 where `ray_dir < 0`, else 0.0.

### Edge Cases and Pitfalls
*   **Fractional Positions**: Unlike HP4T, `grid_position` retains fractional values. It **must** be floored before accessing the grid data to avoid reading incorrect cells or causing memory errors[S1, unofficial].
*   **Surface Normal Ambiguity**: HP8T allows diagonal jumps (8-connectivity). If surface normals are calculated using a simple `compare` variable (e.g., checking if the ray entered or exited a cell based on a single scalar value), the normal calculation will be incorrect for diagonal transitions[S1, unofficial]. The `compare` value cannot distinguish between entering a cell from the side versus entering from a corner/diagonal[S1, unofficial]. A post-processing step analyzing surrounding cells is required to determine the correct normal if 8-connectivity is used[S1, unofficial].
*   **Boundary Checks**: The loop condition checks `grid_position` against `grid_width` and `grid_height`. Ensure `grid_position` is updated correctly before the check if the loop structure varies.

## Operating characteristics
*   **Complexity**: Time complexity is proportional to the number of cells visited. Memory complexity is minimal, requiring only a few floating-point variables for state.
*   **Accuracy**: In randomized testing (100,000,000 cases), HP8T achieved a 99.99% chance of correct position and 99.99% chance of correct normal[S1, unofficial]. This is slightly lower than the 100% accuracy of Hero-like 4-Connectivity (H4T)[S1, unofficial].
*   **Strengths**: Simpler calculation for the non-hit axis compared to HP4T, as it avoids the complex clamping logic[S1, unofficial].
*   **Limitations**: 8-connectivity introduces potential errors in surface normal determination if not handled with additional logic[S1, unofficial]. The lack of clamping means the ray can "jump" diagonally, which may be undesirable for algorithms requiring strict 4-connectivity (e.g., certain marching cubes implementations or specific normal calculations).

## Background
HP8T is described in a thesis by a KTH student, "versal algorithm for ray tracing Hur man gör en hierarkisk ..." [S1, unofficial]. It is presented as a variant of HP4T, designed to simplify the traversal logic by removing the clamping step for the non-hit axis. The method was evaluated against HP4T and H4T using randomized and specific test cases.

## Sources
[S1] versal algorithm for ray tracing Hur man gör en hierarkisk ..., https://kth.diva-portal.org/smash/get/diva2:2069409/FULLTEXT01.pdf, fetched 2026-10-25 (unofficial)
