# Wide DDA Cell Traversal

Also called: Wide Node DDA Traversal, 4x4x4 DDA Traversal

## Overview
Wide DDA Cell Traversal is a specialized ray-grid traversal algorithm designed to iterate through a fixed 4x4x4 grid of "wide cells" (subdivisions of an Axis-Aligned Bounding Box) in front-to-back order [S1, unofficial]. It solves the problem of efficiently visiting child nodes in a hierarchical structure (such as an octree) by generating bounding boxes and t-intervals for each cell, allowing the renderer to skip empty or culled children [S1, unofficial]. It sets itself apart from standard DDA by operating on a coarse, fixed-size block (16 cells) rather than individual voxels, optimizing for cache locality and batch processing in GPU kernels [S1, unofficial]. It builds on [Ray-grid traversal](17-ray-grid-traversal.md).

## How it works
The algorithm traverses a 4x4x4 grid of cells within a parent AABB. It uses a Digital Differential Analyzer (DDA) approach to determine the order in which the ray intersects these wide cells.

### Inputs and Outputs
- **Inputs**:
  - `ray_origin`: The 3D origin of the ray ($\mathbf{o}$).
  - `ray_direction`: The 3D direction vector of the ray ($\mathbf{d}$).
  - `bounds`: The parent AABB defining the region, with `min` and `max` components.
  - `base_t`: The entry distance of the ray into the parent AABB.
  - `max_s`: The maximum traversal distance (exit distance from parent).
  - `reverse`: A boolean flag indicating traversal direction (front-to-back or back-to-front).
- **Outputs**:
  - A sequence of `WideDdaCell` structures, each containing:
    - `child_index`: Packed integer index of the cell (0–63).
    - `bounds`: The AABB of the specific wide cell.
    - `t_near`: Entry distance into the cell.
    - `t_far`: Exit distance from the cell.
  - The traversal terminates early if a visitor callback returns `false`.

### Data Structures
The core data structure yielded by the traversal is `WideDdaCell`:
- `child_index` (`int`): The linear index of the cell in the 4x4x4 grid, calculated as $x + 4y + 16z$.
- `bounds` (`Aabb`): A bounding box defined by `min` (vec3) and `max` (vec3).
- `t_near` (`float`): The distance along the ray where the ray enters the cell.
- `t_far` (`float`): The distance along the ray where the ray exits the cell.

### Parameters
- `cell_size`: The dimensions of each wide cell. Calculated as $(\text{bounds.max} - \text{bounds.min}) \times 0.25$. This implies a 4x4x4 subdivision of the parent AABB.
- `kEpsilon`: A small floating-point tolerance (e.g., $10^{-6}$ or similar) used for boundary comparisons to handle precision issues.

### Steps
1.  **Initialize Cell Indices**:
    Map the ray origin to local cell coordinates $(0, 1, 2, 3)$ within the 4x4x4 grid. This is done using a helper `local_cell_for_position`.
    - Compute local coordinate: $\text{local} = (\mathbf{o} - \text{min\_bound}) / \text{cell\_size}$.
    - Compute rounded value: $\text{rounded} = \text{round}(\text{local})$.
    - If $|\text{local} - \text{rounded}| \le 8 \cdot \text{kEpsilon}$, the position is on a boundary. Clamp the boundary to $[0, 3]$. If $\text{boundary} \le 0$, return 0; if $\text{boundary} \ge 4$, return 3; otherwise, return $\text{boundary} - 1$ if $\text{walk\_direction} < -\text{kEpsilon}$, else $\text{boundary}$.
    - Otherwise, return $\text{clamp}(\lfloor \text{local} \rfloor, 0, 3)$.
    - The `walk_direction` is the sign of the ray direction component.

2.  **Setup Axis Parameters**:
    For each axis $i \in \{x, y, z\}$:
    - Determine `step_i`: $+1$ if $d_i > 0$, $-1$ if $d_i < 0$, $0$ if $|d_i| \le \text{kEpsilon}$.
    - Calculate `delta_s_i` (distance to traverse one cell): $\Delta s_i = \text{cell\_size}_i / |d_i|$. If $|d_i| \le \text{kEpsilon}$, set $\Delta s_i = \infty$.
    - Calculate `next_s_i` (distance to the first boundary):
      - Determine the initial boundary position: $\text{boundary}_i = (\text{current\_cell}_i + \text{step}_i) \times \text{cell\_size}_i + \text{min\_bound}_i$.
      - If $|d_i| \le \text{kEpsilon}$, set $\text{next\_s}_i = \infty$.
      - Otherwise, $\text{next\_s}_i = (\text{boundary}_i - o_i) / d_i$.

3.  **Traversal Loop**:
    Iterate up to 96 times (maximum cells in 4x4x4 grid). Maintain `current_s` (distance traveled so far, initialized to 0) and `current_cell` (indices $x, y, z$).
    - **Check Bounds**: If any `current_cell` component is outside $[0, 4)$ or `current_s` > `max_s` + `kEpsilon`, break.
    - **Determine Next Intersection**:
      - $\text{next\_s} = \min(\text{next\_s}_x, \text{next\_s}_y, \text{next\_s}_z, \text{max\_s})$.
      - Identify the axis $k$ with the minimum `next_s`. If ties occur, use `kEpsilon` tolerance: if $\text{next\_s}_x \le \text{next\_s}_y + \text{kEpsilon}$ and $\text{next\_s}_x \le \text{next\_s}_z + \text{kEpsilon}$, advance $x$; else if $\text{next\_s}_y \le \text{next\_s}_z + \text{kEpsilon}$, advance $y$; else advance $z$.
    - **Construct Cell Info**:
      - $\text{child\_index} = \text{current\_cell}_x + 4 \cdot \text{current\_cell}_y + 16 \cdot \text{current\_cell}_z$.
      - $\text{child\_min} = \text{min\_bound} + \text{cell\_size} \times \text{current\_cell}$.
      - $\text{child\_max} = \text{child\_min} + \text{cell\_size}$.
      - If `reverse` is true: $\text{t\_near} = \text{base\_t} - \text{next\_s}$, $\text{t\_far} = \text{base\_t} - \text{current\_s}$.
      - If `reverse` is false: $\text{t\_near} = \text{base\_t} + \text{current\_s}$, $\text{t\_far} = \text{base\_t} + \text{next\_s}$.
    - **Yield Cell**: Create `WideDdaCell` with `child_index`, `bounds` ($\text{child\_min}, \text{child\_max}$), `t_near`, and `t_far`. Call `visitor(cell)`. If `visitor` returns `false`, terminate.
    - **Update State**:
      - If $\text{next\_s} \ge \text{max\_s} - \text{kEpsilon}$, break.
      - $\text{current\_s} = \text{next\_s}$.
      - Advance `current_cell_k` by `step_k`.
      - Update $\text{next\_s}_k = \text{next\_s}_k + \Delta s_k$.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def wide_dda_cell_traversal(ray_origin, ray_direction, bounds, base_t, max_s, reverse):
    # Constants
    k_epsilon = 1e-6
    cell_size = (bounds.max - bounds.min) * 0.25
    
    # Helper: local cell for position
    def local_cell_for_position(pos, walk_dir):
        local = (pos - bounds.min) / cell_size
        rounded = round(local)
        if abs(local - rounded) <= 8.0 * k_epsilon:
            boundary = rounded
            if boundary <= 0: return 0
            if boundary >= 4: return 3
            if walk_dir < -k_epsilon: return boundary - 1
            return boundary
        return clamp(floor(local), 0, 3)

    # Initialize current cell indices
    current_cell = [
        local_cell_for_position(ray_origin.x, sign(ray_direction.x)),
        local_cell_for_position(ray_origin.y, sign(ray_direction.y)),
        local_cell_for_position(ray_origin.z, sign(ray_direction.z))
    ]
    
    # Setup axis parameters
    step = [sign(ray_direction[i]) if abs(ray_direction[i]) > k_epsilon else 0 for i in range(3)]
    delta_s = [cell_size[i] / abs(ray_direction[i]) if abs(ray_direction[i]) > k_epsilon else float('inf') for i in range(3)]
    
    next_s = [0.0, 0.0, 0.0]
    for i in range(3):
        if step[i] == 0:
            next_s[i] = float('inf')
        else:
            boundary = (current_cell[i] + step[i]) * cell_size[i] + bounds.min[i]
            next_s[i] = (boundary - ray_origin[i]) / ray_direction[i]
            
    current_s = 0.0
    
    # Traversal loop
    for _ in range(96):
        # Check bounds
        if not (0 <= current_cell[0] < 4 and 0 <= current_cell[1] < 4 and 0 <= current_cell[2] < 4):
            break
        if current_s > max_s + k_epsilon:
            break
            
        # Determine next intersection
        next_s_val = min(next_s[0], next_s[1], next_s[2], max_s)
        
        # Determine axis to advance
        if next_s[0] <= next_s[1] + k_epsilon and next_s[0] <= next_s[2] + k_epsilon:
            axis = 0
        elif next_s[1] <= next_s[2] + k_epsilon:
            axis = 1
        else:
            axis = 2
            
        # Construct cell info
        child_index = current_cell[0] + 4 * current_cell[1] + 16 * current_cell[2]
        child_min = bounds.min + cell_size * current_cell
        child_max = child_min + cell_size
        
        if reverse:
            t_near = base_t - next_s_val
            t_far = base_t - current_s
        else:
            t_near = base_t + current_s
            t_far = base_t + next_s_val
            
        cell = {
            'child_index': child_index,
            'bounds': {'min': child_min, 'max': child_max},
            't_near': t_near,
            't_far': t_far
        }
        
        # Call visitor
        if not visitor(cell):
            return
            
        # Update state
        if next_s_val >= max_s - k_epsilon:
            break
            
        current_s = next_s_val
        current_cell[axis] += step[axis]
        next_s[axis] += delta_s[axis]
```

### Edge Cases and Pitfalls
- **Zero Direction Components**: If a ray direction component is near zero ($|d_i| \le \text{kEpsilon}$), the step is set to 0, and both `delta_s` and `next_s` are set to infinity. This prevents division by zero and infinite loops [S1, unofficial].
- **Boundary Precision**: Positions exactly on cell edges are handled with an $8 \cdot \text{kEpsilon}$ tolerance window. The `walk_direction` parameter breaks ties to ensure consistent DDA stepping, preferring the cell in the direction of travel [S1, unofficial].
- **Off-by-One Errors**: The `local_cell_for_position` function clamps return values to $[0, 3]$ to prevent out-of-bounds access [S1, unofficial].

## Subtopics

### Wide Node DDA Cell Calculation
This is a helper function used within Wide DDA Cell Traversal to map a continuous position to a discrete cell index (0–3) within a 4x4x4 wide node. It handles boundary cases where the position falls exactly on a cell edge. It computes the local coordinate, rounds it, and uses a tolerance window ($8 \cdot \text{kEpsilon}$) to detect boundary conditions. If on a boundary, it uses the `walk_direction` (sign of ray direction) to resolve ambiguity, ensuring the ray enters the correct cell. If not on a boundary, it uses `floor` and clamps to $[0, 3]$. This ensures consistent traversal order and prevents missing cells due to floating-point inaccuracies [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(1)$ per wide cell visited, with a maximum of 64 iterations (4x4x4). The traversal is efficient for fixed-size blocks.
- **Memory Complexity**: $O(1)$ auxiliary space, as it only stores a few scalar values per axis.
- **Performance**: Optimized for GPU execution, particularly in sparse voxel octree renderers where wide nodes are used to batch processing. It reduces overhead compared to traversing individual voxels.
- **Strengths**: Efficient for hierarchical structures with fixed-size nodes; handles boundary conditions robustly; suitable for parallel execution.
- **Limitations**: Only applicable to 4x4x4 wide cells; not suitable for arbitrary grid sizes or non-uniform grids without modification.
- **When to Use**: When traversing sparse voxel octrees or similar hierarchical structures where nodes are subdivided into 4x4x4 blocks.
- **When Not to Use**: For uniform grids, large variable-sized cells, or when the traversal order does not need to be strictly front-to-back within the block.

## Background
Wide DDA Cell Traversal is derived from the Digital Differential Analyzer (DDA) algorithm, adapted for hierarchical grid structures. It is specifically implemented in the `sparse-voxels` project by Hector Piteau, which focuses on efficient sparse voxel octree rendering [S1, unofficial]. The method is designed to optimize the traversal of wide nodes in a GPU-based renderer, reducing the overhead of individual cell visits by batching them into 4x4x4 blocks.

## Sources
[S1] hectorpiteau/sparse-voxels, https://github.com/hectorpiteau/sparse-voxels, fetched 2026-10-24 (unofficial)
