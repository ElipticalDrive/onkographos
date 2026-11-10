# Wide DDA Cell Traversal

Also called: Wide Node DDA Cell Traversal, 4x4x4 DDA Traversal

## Overview
Wide DDA Cell Traversal is an algorithm for traversing a fixed 4x4x4 grid of "wide cells" within an Axis-Aligned Bounding Box (AABB) using a Digital Differential Analyzer (DDA) approach [S1, unofficial]. It generates bounding boxes and t-intervals for each cell to optimize octree traversal by visiting only occupied child cells in ray order [S1, unofficial]. It builds on [Digital Differential Analyzer (DDA) Traversal](3061-digital-differential-analyzer-dda-traversal.md) by providing the specific visitor callback mechanism and mask-based filtering required for wide octree nodes [S1, unofficial].

## How it works
**Inputs:**
- `bounds`: The AABB defining the 4x4x4 grid region (`min`, `max` as `vec3`).
- `origin`: Ray origin point (`vec3`).
- `direction`: Ray direction vector (`vec3`).
- `max_s`: Maximum traversal distance parameter.
- `visitor`: A callback function `bool visitor(cell)` that receives the current cell data. If it returns `false`, traversal terminates.

**Outputs:**
- Sequence of `WideDdaCell` structures passed to the visitor, representing occupied cells in traversal order.

**Data Structures:**
- `WideDdaCell`:
  - `child_index` (`int`): Packed index of the cell within the 4x4x4 grid. Layout: `cell_x | (cell_y << 2) | (cell_z << 4)`.
  - `bounds` (`Aabb`): Local bounding box of the cell. `min` and `max` are `vec3`.
  - `t_near` (`float`): Entry distance along the ray.
  - `t_far` (`float`): Exit distance along the ray.

**Constants:**
- `kEpsilon`: Small floating-point tolerance (e.g., $10^{-6}$ or hardware-specific epsilon) for boundary comparisons.
- `cell_size`: Dimensions of each wide cell, calculated as `(bounds.max - bounds.min) * 0.25f`. This implies a uniform 4x4x4 subdivision of the parent AABB.

**Steps:**

1.  **Initialize Cell Indices:**
    Map the ray origin to local cell coordinates $(cell_x, cell_y, cell_z)$ using `local_cell_for_position`. This function handles boundary conditions and rounding with `kEpsilon` tolerance.
    - Compute local coordinate: $local = (origin - bounds.min) / cell\_size$.
    - Compute rounded value: $rounded = round(local)$.
    - Check boundary: If $|local - rounded| \le 8.0 \times kEpsilon$, the position is on a boundary.
      - Clamp boundary to $[0, 3]$.
      - If $boundary \le 0$, return $0$.
      - If $boundary \ge 4$, return $3$.
      - Otherwise, return $boundary - 1$ if $walk\_direction < -kEpsilon$, else $boundary$.
    - Otherwise, return $clamp(floor(local), 0, 3)$.
    - Note: $walk\_direction$ is the sign of the ray direction component ($-1$ or $1$).

2.  **Setup Axis Parameters:**
    For each axis $i \in \{x, y, z\}$:
    - Determine step direction: $step_i = sign(direction_i)$. If $|direction_i| \le kEpsilon$, set $step_i = 0$.
    - Calculate delta distance: $delta\_s_i = cell\_size / |direction_i|$ if $|direction_i| > kEpsilon$, else $\infty$.
    - Calculate next intersection distance: $next\_s_i$. If $step_i = 0$, set $next\_s_i = \infty$. Otherwise, compute the distance to the next cell boundary along axis $i$ from the current cell index.

3.  **Iterate:**
    Loop up to 96 times (since $4 \times 4 \times 4 = 64$, but the loop bound is explicitly 96 in the source, potentially to handle edge cases or overshoot).
    
    a.  **Check Bounds:**
        If $cell_x \notin [0, 4)$ or $cell_y \notin [0, 4)$ or $cell_z \notin [0, 4)$ or $current\_s > max\_s + kEpsilon$, break loop.
        
    b.  **Determine Next Cell:**
        Calculate $next\_s = \min(next\_s_x, next\_s_y, next\_s_z, max\_s)$.
        Determine $child\_index = cell\_x | (cell\_y \ll 2) | (cell\_z \ll 4)$.
        
    c.  **Compute Cell Bounds:**
        $child\_min = bounds.min + cell\_size \times vec3(cell\_x, cell\_y, cell\_z)$.
        $child\_bounds = \{child\_min, child\_min + cell\_size\}$.
        
    d.  **Compute t-intervals:**
        If `reverse` is true:
        - $t\_near = base\_t - next\_s$
        - $t\_far = base\_t - current\_s$
        Else:
        - $t\_near = base\_t + current\_s$
        - $t\_far = base\_t + next\_s$
        
    e.  **Visit Cell:**
        Construct `WideDdaCell` with `child_index`, `child_bounds`, `t_near`, `t_far`.
        Call `visitor(cell)`. If it returns `false`, terminate traversal immediately.
        
    f.  **Check Termination:**
        If $next\_s \ge max\_s - kEpsilon$, break loop.
        
    g.  **Advance:**
        Update $current\_s = next\_s$.
        Advance the axis with the smallest $next\_s_i$.
        - If $next\_s_x \le next\_s_y + kEpsilon$ and $next\_s_x \le next\_s_z + kEpsilon$, advance x.
        - Else if $next\_s_y \le next\_s_z + kEpsilon$, advance y.
        - Else, advance z.
        Use `advance_axis` to update the corresponding cell index and $next\_s_i$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def wide_dda_cell_traversal(bounds, origin, direction, max_s, base_t, reverse, visitor):
    cell_size = (bounds.max - bounds.min) * 0.25
    
    # Initialize cell indices
    cell_x, cell_y, cell_z = local_cell_for_position(origin, bounds.min, cell_size, direction)
    
    # Setup axis parameters
    step_x, next_s_x, delta_s_x = axis_setup(direction.x, cell_size, cell_x, bounds.min.x)
    step_y, next_s_y, delta_s_y = axis_setup(direction.y, cell_size, cell_y, bounds.min.y)
    step_z, next_s_z, delta_s_z = axis_setup(direction.z, cell_size, cell_z, bounds.min.z)
    
    current_s = 0.0
    
    for _ in range(96):
        # Check bounds
        if not (0 <= cell_x < 4 and 0 <= cell_y < 4 and 0 <= cell_z < 4):
            break
        if current_s > max_s + kEpsilon:
            break
            
        # Determine next cell
        next_s = min(next_s_x, next_s_y, next_s_z, max_s)
        child_index = cell_x | (cell_y << 2) | (cell_z << 4)
        
        # Compute cell bounds
        child_min = bounds.min + cell_size * vec3(cell_x, cell_y, cell_z)
        child_bounds = Aabb(min=child_min, max=child_min + cell_size)
        
        # Compute t-intervals
        if reverse:
            t_near = base_t - next_s
            t_far = base_t - current_s
        else:
            t_near = base_t + current_s
            t_far = base_t + next_s
            
        # Visit cell
        cell = WideDdaCell(child_index=child_index, bounds=child_bounds, t_near=t_near, t_far=t_far)
        if not visitor(cell):
            return
            
        # Check termination
        if next_s >= max_s - kEpsilon:
            break
            
        # Advance
        current_s = next_s
        
        if next_s_x <= next_s_y + kEpsilon and next_s_x <= next_s_z + kEpsilon:
            cell_x, next_s_x = advance_axis(step_x, delta_s_x, next_s_x)
        elif next_s_y <= next_s_z + kEpsilon:
            cell_y, next_s_y = advance_axis(step_y, delta_s_y, next_s_y)
        else:
            cell_z, next_s_z = advance_axis(step_z, delta_s_z, next_s_z)

def local_cell_for_position(position, min_bound, cell_size, walk_direction):
    local = (position - min_bound) / cell_size
    rounded = round(local)
    
    # Check boundary
    if abs(local - rounded) <= 8.0 * kEpsilon:
        boundary = clamp(rounded, 0, 3)
        if boundary <= 0:
            return 0
        if boundary >= 4:
            return 3
        if walk_direction < -kEpsilon:
            return boundary - 1
        else:
            return boundary
    else:
        return clamp(floor(local), 0, 3)

def axis_setup(dir_val, cell_size, cell_idx, min_bound):
    if abs(dir_val) <= kEpsilon:
        return 0, float('inf'), float('inf')
    
    step = 1 if dir_val > 0 else -1
    delta_s = cell_size / abs(dir_val)
    
    # Calculate next_s based on current cell index and direction
    # This is a simplified representation; actual implementation depends on specific DDA logic
    # For a standard DDA, next_s is the distance to the next grid line
    # Assuming current_s is 0 relative to cell start for initialization
    # next_s_x = (cell_idx + 1 - local_x) * cell_size / dir_val if dir > 0 else ...
    # The source notes do not provide the exact formula for next_s initialization, 
    # so we assume a standard DDA setup where next_s is computed from origin to first boundary.
    # [G] Standard DDA: next_s = (boundary - origin) / direction
    # Here, we assume axis_setup handles the initial distance calculation.
    next_s = 0.0 # Placeholder for actual initial distance calculation
    return step, next_s, delta_s

def advance_axis(step, delta_s, current_next_s):
    # Advance cell index
    # This function updates the cell index and next_s for the specific axis
    # The source notes do not provide the exact implementation of advance_axis
    # [G] Standard DDA: next_s += delta_s
    # Cell index is incremented/decremented by step
    return 0, current_next_s + delta_s # Placeholder
```

## Subtopics

### Wide Node DDA Cell Calculation
Wide Node DDA Cell Calculation is a helper function that maps a continuous position to a discrete cell index (0-3) within a 4x4x4 wide node, handling boundary cases where the position falls exactly on a cell edge [S1, unofficial]. It is used by Wide DDA Cell Traversal to initialize the starting cell indices [S1, unofficial]. It differs from standard floor-based mapping by using an 8*kEpsilon tolerance window and the `walk_direction` parameter to break ties, ensuring consistent DDA stepping [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per wide cell visited, with a maximum of 96 iterations. The traversal is efficient for dense regions within the 4x4x4 block.
- **Memory Complexity:** $O(1)$ auxiliary space, storing only current cell indices and axis parameters.
- **Performance:** Optimized for GPU execution, as seen in the CUDA implementation [S1, unofficial]. The fixed 4x4x4 grid size allows for predictable memory access patterns and reduced branching.
- **Strengths:** Handles boundary precision issues robustly using `kEpsilon` and `walk_direction`. Avoids division by zero for near-zero ray directions.
- **Limitations:** Limited to 4x4x4 grids. Not suitable for arbitrary grid sizes without modification.
- **When to use:** When traversing wide octree nodes or fixed-size blocks where occupancy is high and ray order is important.
- **When not to use:** For sparse grids or when the grid size is not a multiple of 4.

## Background
Wide DDA Cell Traversal is part of the `svo` (Sparse Voxel Octree) rendering pipeline, specifically designed for efficient traversal of wide nodes in a GPU-based renderer [S1, unofficial]. It is implemented in CUDA and integrates with the broader octree traversal system [S1, unofficial].

## Sources
[S1] hectorpiteau/sparse-voxels, https://github.com/hectorpiteau/sparse-voxels, fetched 2026-10-24 (unofficial)
