# Obstacle-Flag


## Overview
The Obstacle-Flag method is a dynamic job-scheduling mechanism used in distributed volume rendering to determine when a sub-volume is combinable (ready to be rendered and merged into the final image) [S1]. It solves the inefficiency of sequential job-scheduling, where agents remain idle waiting for rigid, pre-computed visibility orders, by tracking real-time obscuration relationships between neighboring sub-volumes [S1]. Unlike static visibility culling, it updates its state as rendering progresses, allowing agents to immediately pick up the next available sub-volume as soon as its obscuring neighbors are completed [S1].

## How it works
The method maintains a bitmask for each sub-volume in the grid, representing the visibility status relative to its immediate neighbors [S1].

**Inputs and Outputs**
*   **Input:** A grid of sub-volumes, the initial geometry, and the initial view direction [S1].
*   **Output:** A pool of sub-volumes with an obscure count of zero, ready for assignment to rendering agents [S1].

**Data Structures**
Each sub-volume $V$ holds an `obstacle_flag`, which is an integer bitmask [S1]. The size of the flag depends on the dimensionality of the grid:
*   **2D Grid:** A 4-bit integer. Each bit corresponds to one of the four cardinal directions (e.g., Up, Down, Left, Right) [S1]. A bit is set to 1 if a neighboring sub-volume in that direction currently obscures $V$ from the viewpoint [S1].
*   **3D Grid:** A 6-bit integer. Each bit corresponds to one of the six axial directions (±X, ±Y, ±Z) [S1]. A bit is set to 1 if a neighboring sub-volume in that direction currently obscures $V$ [S1].

The `obscure_count` is defined as the population count (number of set bits) of the `obstacle_flag` [S1].
*   In 2D, the maximum obscure count is 2 [S1].
*   In 3D, the maximum obscure count is 3 [S1].

**Algorithm Steps**
1.  **Initialization:** For every sub-volume $V$ in the grid, calculate the initial `obstacle_flag` based on the geometry and the initial view direction [S1]. A flag of all zeroes means no obscuring sub-volumes exist [S1].
2.  **Selection:** Select the next sub-volume to send to an agent from the pool of sub-volumes whose obscure count is zero [S1].
3.  **Dynamic Update:** When a sub-volume is rendered and combined, update the obstacle-flags of its neighbors [S1]. If a neighbor was obscured by the rendered sub-volume, decrement its obscure count [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_obstacle_flags(grid, view_direction):
    """
    grid: dict mapping (x,y,z) -> SubVolume
    view_direction: normalized vector (dx, dy, dz) pointing from camera to scene
    """
    for pos, subvol in grid.items():
        subvol.obstacle_flag = 0
        subvol.obscure_count = 0
        offsets = [(0, 1), (0, -1), (1, 0), (-1, 0)] if subvol.dim == 2 else \
                  [(1, 0, 0), (-1, 0, 0), (0, 1, 0), (0, -1, 0), (0, 0, 1), (0, 0, -1)]
        for offset in offsets:
            neighbor_pos = tuple(p + o for p, o in zip(pos, offset))
            if neighbor_pos in grid:
                neighbor = grid[neighbor_pos]
                vec = tuple(n - p for n, p in zip(neighbor_pos, pos))
                dot_prod = sum(v * d for v, d in zip(vec, view_direction))
                if dot_prod > 0:
                    bit_idx = direction_to_bit(offset, subvol.dim)
                    subvol.obstacle_flag |= (1 << bit_idx)
                    subvol.obscure_count += 1

def direction_to_bit(offset, dim):
    if dim == 2:
        mapping = {(0, 1): 0, (0, -1): 1, (1, 0): 2, (-1, 0): 3}
        return mapping[offset]
    else:
        mapping = {(1, 0, 0): 0, (-1, 0, 0): 1, (0, 1, 0): 2, (0, -1, 0): 3, (0, 0, 1): 4, (0, 0, -1): 5}
        return mapping[offset]

def process_rendered_volume(grid, subvol_done):
    """
    Called when subvol_done is finished rendering and combined.
    Updates neighbors' obstacle flags.
    """
    subvol_done.rendered = True
    offsets = [(0, 1), (0, -1), (1, 0), (-1, 0)] if subvol_done.dim == 2 else \
              [(1, 0, 0), (-1, 0, 0), (0, 1, 0), (0, -1, 0), (0, 0, 1), (0, 0, -1)]
    for offset in offsets:
        neighbor_pos = tuple(p + o for p, o in zip(subvol_done.pos, offset))
        if neighbor_pos in grid:
            neighbor = grid[neighbor_pos]
            if neighbor.rendered:
                continue
            vec = tuple(d - n for d, n in zip(subvol_done.pos, neighbor_pos))
            bit_idx = direction_to_bit(vec, subvol_done.dim)
            if (neighbor.obstacle_flag >> bit_idx) & 1:
                neighbor.obstacle_flag &= ~(1 << bit_idx)
                neighbor.obscure_count -= 1
                if neighbor.obscure_count == 0:
                    add_to_ready_queue(neighbor)

def add_to_ready_queue(subvol):
    # Standard queue/priority queue insertion [G]
    pass
```

**Parameters**
*   **Obscure Count:** The number of set bits (1s) in the obstacle-flag. Represents the number of obscuring neighboring sub-volumes [S1].

**Edge Cases and Pitfalls**
*   **Sequential Scheduling Failure:** Sequential job-scheduling fails because it uses initial visibility values [S1]. If a sub-volume is obscured by multiple neighbors, it cannot be sent until all potential obscuring neighbors are processed, even if some are already done, causing idle agents [S1]. The dynamic update method alleviates this by clearing bits as neighbors complete [S1].
*   **View Direction Dependence:** Obscuration relationships depend on the view direction [S1]. Flags are typically computed once per view change [G].
*   **Boundary Conditions:** Sub-volumes at the grid boundary have fewer neighbors. The flag bits for non-existent neighbors should be initialized to 0 and never set [G].

## Operating characteristics
*   **Time Complexity:** Initialization is $O(N)$ where $N$ is the number of sub-volumes, as each sub-volume checks its immediate neighbors [G]. Dynamic updates are $O(1)$ per rendered sub-volume, as only the immediate neighbors' flags need updating [G].
*   **Memory Complexity:** $O(N)$, storing one integer (4 or 6 bits) per sub-volume [G].
*   **Strengths:** Maximizes parallelism by allowing agents to start rendering sub-volumes as soon as their specific obscuring constraints are met, rather than waiting for a global sequential order [S1]. Reduces idle time in grid computing environments [S1].
*   **Limitations:** Requires dynamic management of job queues [G]. Performance is dependent on the granularity of the sub-volumes; too fine granularity increases overhead, too coarse reduces parallelism [G].
*   **When to Use:** Distributed or grid-based volume rendering where sub-volumes are processed independently and combined into a final image [S1].
*   **When Not to Use:** Single-threaded rendering or when the view direction changes frequently during rendering [G].

## Background
The Obstacle-Flag method was proposed in the context of grid computing for large-scale volume rendering [S1]. It addresses the limitations of sequential job-scheduling, which fails to utilize computing resources efficiently due to rigid dependency chains [S1]. By dynamically updating visibility parameters, it enables a more flexible and efficient distribution of rendering tasks [S1].

## Sources
[S1] Volume rendering using grid computing for large-scale volume data, https://doi.org/10.1109/cadcg.2009.5246856, fetched 2026-10-24
