# Wide DDA Cell Traversal


## Overview
Wide DDA Cell Traversal is an algorithm for traversing a fixed 4x4x4 grid of "wide cells" within an Axis-Aligned Bounding Box (AABB) using a Digital Differential Analyzer (DDA) approach [S1, unofficial]. It generates bounding boxes and t-intervals for each cell to optimize octree traversal by visiting occupied child cells in ray order [S1, unofficial].

## How it works
**Inputs:**
- `bounds_min`, `bounds_max`: AABB defining the 4x4x4 grid region (`vec3`).
- `origin`: Ray origin point (`vec3`) [S1, unofficial].
- `direction`: Ray direction vector (`vec3`) [S1, unofficial].
- `max_s`: Maximum traversal distance parameter (float) [S1, unofficial].
- `base_t`: Base parameter for t-interval calculation (float) [S1, unofficial].
- `reverse`: Boolean flag controlling t-interval sign convention (bool) [S1, unofficial].
- `visitor`: Callback function `bool visitor(cell)` that receives the current cell data [S1, unofficial].

**Outputs:**
- Sequence of `WideDdaCell` structures passed to the visitor, representing occupied cells in traversal order [S1, unofficial].

**Data Structures:**
- `WideDdaCell`:
  - `child_index` (`int`): Packed index of the cell within the 4x4x4 grid. Layout: `cell_x | (cell_y << 2) | (cell_z << 4)` [S1, unofficial].
  - `bounds_min`, `bounds_max` (`vec3`): Local bounding box of the cell [S1, unofficial].
  - `t_near` (`float`): Entry distance along the ray [S1, unofficial].
  - `t_far` (`float`): Exit distance along the ray [S1, unofficial].

**Constants:**
- `kEpsilon`: Small floating-point tolerance for boundary comparisons (exact value not given in the sources) [S1, unofficial].
- `cell_size`: Dimensions of each wide cell, calculated as `(bounds_max - bounds_min) * 0.25` [S1, unofficial].

**Steps:**

1.  **Initialize Cell Indices:**
    Map the ray origin to local cell coordinates $(cell_x, cell_y, cell_z)$ using `local_cell_for_position`, which handles boundary conditions and rounding with `kEpsilon` tolerance [S1, unofficial].
    - Compute local coordinate: $local = (origin - bounds\_min) / cell\_size$ [S1, unofficial].
    - Compute rounded value: $rounded = round(local)$ [S1, unofficial].
    - Check boundary: If $|local - rounded| \le 8.0 \times kEpsilon$, the position is on a boundary [S1, unofficial].
      - Clamp boundary to $[0, 3]$ [S1, unofficial].
      - If $boundary \le 0$, return $0$ [S1, unofficial].
      - If $boundary \ge 4$, return $3$ [S1, unofficial].
      - Otherwise, return $boundary - 1$ if $walk\_direction < -kEpsilon$, else $boundary$ [S1, unofficial].
    - Otherwise, return $clamp(floor(local), 0, 3)$ [S1, unofficial].
    - Note: $walk\_direction$ is the sign of the ray direction component ($-1$ or $1$) [S1, unofficial].

2.  **Setup Axis Parameters:**
    For each axis $i \in \{x, y, z\}$:
    - Determine step direction: $step_i = sign(direction_i)$ [S1, unofficial]. If $|direction_i| \le kEpsilon$, set $step_i = 0$ [S1, unofficial].
    - Calculate delta distance: $delta\_s_i = cell\_size / |direction_i|$ if $|direction_i| > kEpsilon$, else $\infty$ [S1, unofficial].
    - Calculate next intersection distance: $next\_s_i$ [S1, unofficial]. If $step_i = 0$, set $next\_s_i = \infty$ [S1, unofficial]. Otherwise, compute the distance to the next cell boundary along axis $i$ from the current cell index [S1, unofficial].

3.  **Iterate:**
    Loop up to 96 times [S1, unofficial].
    
    a.  **Check Bounds:**
        If $cell_x \notin [0, 4)$ or $cell_y \notin [0, 4)$ or $cell_z \notin [0, 4)$ or $current\_s > max\_s + kEpsilon$, break loop [S1, unofficial].
        
    b.  **Determine Next Cell:**
        Calculate $next\_s = \min(next\_s_x, next\_s_y, next\_s_z, max\_s)$ [S1, unofficial].
        Determine $child\_index = cell\_x \mid (cell\_y \ll 2) \mid (cell\_z \ll 4)$ [S1, unofficial].
        
    c.  **Compute Cell Bounds:**
        $child\_min = bounds\_min + cell\_size \times vec3(cell\_x, cell\_y, cell\_z)$ [S1, unofficial].
        $child\_max = child\_min + cell\_size$ [S1, unofficial].
        
    d.  **Compute t-intervals:**
        If `reverse` is true:
        - $t\_near = base\_t - next\_s$ [S1, unofficial].
        - $t\_far = base\_t - current\_s$ [S1, unofficial].
        Else:
        - $t\_near = base\_t + current\_s$ [S1, unofficial].
        - $t\_far = base\_t + next\_s$ [S1, unofficial].
        
    e.  **Visit Cell:**
        Construct `WideDdaCell` with `child_index`, `child_min`, `child_max`, `t_near`, `t_far` [S1, unofficial].
        Call `visitor(cell)` [S1, unofficial]. If it returns `false`, terminate traversal immediately [S1, unofficial].
        
    f.  **Check Termination:**
        If $next\_s \ge max\_s - kEpsilon$, break loop [S1, unofficial].
        
    g.  **Advance:**
        Update $current\_s = next\_s$ [S1, unofficial].
        Advance the axis with the smallest $next\_s_i$ [S1, unofficial].
        - If $next\_s_x \le next\_s_y + kEpsilon$ and $next\_s_x \le next\_s_z + kEpsilon$, advance x [S1, unofficial].
        - Else if $next\_s_y \le next\_s_z + kEpsilon$, advance y [S1, unofficial].
        - Else, advance z [S1, unofficial].
        Use `advance_axis` to update the corresponding cell index and $next\_s_i$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class WideDdaCell:
    def __init__(self, child_index, bounds_min, bounds_max, t_near, t_far):
        self.child_index = child_index
        self.bounds_min = bounds_min
        self.bounds_max = bounds_max
        self.t_near = t_near
        self.t_far = t_far

def wide_dda_cell_traversal(bounds_min, bounds_max, origin, direction, max_s, base_t, reverse, visitor):
    kEpsilon = 1e-6  # Not given in the sources, standard tolerance [G]
    cell_size = (bounds_max - bounds_min) * 0.25
    
    # Initialize cell indices (scalar per axis as walk_direction is per-axis)
    dir_x_sign = 1.0 if direction.x > 0 else (-1.0 if direction.x < 0 else 0.0)
    dir_y_sign = 1.0 if direction.y > 0 else (-1.0 if direction.y < 0 else 0.0)
    dir_z_sign = 1.0 if direction.z > 0 else (-1.0 if direction.z < 0 else 0.0)
    
    cell_x = local_cell_for_position(origin.x, bounds_min.x, cell_size, dir_x_sign, kEpsilon)
    cell_y = local_cell_for_position(origin.y, bounds_min.y, cell_size, dir_y_sign, kEpsilon)
    cell_z = local_cell_for_position(origin.z, bounds_min.z, cell_size, dir_z_sign, kEpsilon)
    
    # Setup axis parameters
    step_x, next_s_x, delta_s_x = axis_setup(direction.x, cell_size, cell_x, bounds_min.x, origin.x, kEpsilon)
    step_y, next_s_y, delta_s_y = axis_setup(direction.y, cell_size, cell_y, bounds_min.y, origin.y, kEpsilon)
    step_z, next_s_z, delta_s_z = axis_setup(direction.z, cell_size, cell_z, bounds_min.z, origin.z, kEpsilon)
    
    current_s = 0.0
    
    for _ in range(96):
        if not (0 <= cell_x < 4 and 0 <= cell_y < 4 and 0 <= cell_z < 4):
            break
        if current_s > max_s + kEpsilon:
            break
            
        next_s = min(next_s_x, next_s_y, next_s_z, max_s)
        child_index = cell_x | (cell_y << 2) | (cell_z << 4)
        
        child_min = vec3(bounds_min.x + cell_size * cell_x, bounds_min.y + cell_size * cell_y, bounds_min.z + cell_size * cell_z)
        child_max = vec3(child_min.x + cell_size, child_min.y + cell_size, child_min.z + cell_size)
        
        if reverse:
            t_near = base_t - next_s
            t_far = base_t - current_s
        else:
            t_near = base_t + current_s
            t_far = base_t + next_s
            
        cell = WideDdaCell(child_index=child_index, bounds_min=child_min, bounds_max=child_max, t_near=t_near, t_far=t_far)
        if not visitor(cell):
            return
            
        if next_s >= max_s - kEpsilon:
            break
            
        current_s = next_s
        
        if next_s_x <= next_s_y + kEpsilon and next_s_x <= next_s_z + kEpsilon:
            cell_x, next_s_x = advance_axis(step_x, delta_s_x, next_s_x)
        elif next_s_y <= next_s_z + kEpsilon:
            cell_y, next_s_y = advance_axis(step_y, delta_s_y, next_s_y)
        else:
            cell_z, next_s_z = advance_axis(step_z, delta_s_z, next_s_z)

def local_cell_for_position(pos, min_bound, cell_size, walk_direction, kEpsilon):
    local = (pos - min_bound) / cell_size
    rounded = round(local)
    if abs(local - rounded) <= 8.0 * kEpsilon:
        boundary = max(0, min(3, rounded))
        if boundary <= 0:
            return 0
        if boundary >= 4:
            return 3
        if walk_direction < -kEpsilon:
            return boundary - 1
        else:
            return boundary
    else:
        return max(0, min(3, floor(local)))

def axis_setup(dir_val, cell_size, cell_idx, min_bound, origin, kEpsilon):
    if abs(dir_val) <= kEpsilon:
        return 0, float('inf'), float('inf')
    step = 1 if dir_val > 0 else -1
    delta_s = cell_size / abs(dir_val)
    local = (origin - min_bound) / cell_size
    if step > 0:
        next_s = (cell_idx + 1 - local) * cell_size / dir_val
    else:
        next_s = (cell_idx - local) * cell_size / dir_val
    return step, next_s, delta_s

def advance_axis(step, delta_s, current_next_s):
    return current_next_s + step, current_next_s + delta_s
```

## Subtopics

### Wide Node DDA Cell Calculation
Wide Node DDA Cell Calculation is a helper function that maps a continuous position to a discrete cell index (0-3) within a 4x4x4 wide node, handling boundary cases where the position falls exactly on a cell edge [S1, unofficial]. It is used by Wide DDA Cell Traversal to initialize the starting cell indices [S1, unofficial]. It differs from standard floor-based mapping by using an 8*kEpsilon tolerance window and the `walk_direction` parameter to break ties, ensuring consistent DDA stepping [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per wide cell visited, with a maximum of 96 iterations [S1, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space, storing only current cell indices and axis parameters [S1, unofficial].
- **Strengths:** Handles near-zero ray directions by setting step to 0 and distances to infinity, preventing division by zero and infinite loops [S1, unofficial]. Handles boundary cases robustly using `kEpsilon` and `walk_direction` [S1, unofficial].
- **Limitations:** Limited to 4x4x4 grids [S1, unofficial].
- **When to use:** When traversing wide octree nodes or fixed-size blocks where occupancy is high and ray order is important [S1, unofficial].
- **When not to use:** For sparse grids or when the grid size is not a multiple of 4 [S1, unofficial].

## Background
Wide DDA Cell Traversal is part of the `svo` (Sparse Voxel Octree) rendering pipeline, specifically designed for efficient traversal of wide nodes in a GPU-based renderer [S1, unofficial].

## Sources
[S1] hectorpiteau/sparse-voxels, https://github.com/hectorpiteau/sparse-voxels, fetched 2026-10-24 (unofficial)
