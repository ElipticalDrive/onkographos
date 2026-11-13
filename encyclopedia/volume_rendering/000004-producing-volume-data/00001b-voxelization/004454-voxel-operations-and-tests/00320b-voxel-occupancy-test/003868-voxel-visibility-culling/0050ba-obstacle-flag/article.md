# Obstacle-Flag

Also called: Dynamic Visibility Flag

## Overview
The Obstacle-Flag method is a dynamic job-scheduling mechanism used in distributed volume rendering to determine when a sub-volume is "combinable" (i.e., ready to be rendered and merged into the final image). It solves the inefficiency of sequential job-scheduling, where agents remain idle waiting for rigid, pre-computed visibility orders, by tracking real-time obscuration relationships between neighboring sub-volumes. Unlike static visibility culling, it updates its state as rendering progresses, allowing agents to immediately pick up the next available sub-volume as soon as its obscuring neighbors are completed.

## How it works
The method maintains a bitmask for each sub-volume in the grid, representing the visibility status relative to its immediate neighbors.

**Inputs and Outputs**
*   **Input:** A grid of sub-volumes, the current view direction, and the rendering state (which sub-volumes have been combined).
*   **Output:** A set of sub-volumes with an obscure count of zero, ready for assignment to rendering agents.

**Data Structures**
Each sub-volume $V$ holds an `obstacle_flag`, which is an integer bitmask. The size of the flag depends on the dimensionality of the grid:
*   **2D Grid:** A 4-bit integer. Each bit corresponds to one of the four cardinal directions (e.g., Up, Down, Left, Right). A bit is set to 1 if a neighboring sub-volume in that direction currently obscures $V$ from the viewpoint.
*   **3D Grid:** A 6-bit integer. Each bit corresponds to one of the six axial directions (±X, ±Y, ±Z). A bit is set to 1 if a neighboring sub-volume in that direction currently obscures $V$.

The `obscure_count` is defined as the population count (number of set bits) of the `obstacle_flag`.
*   In 2D, the maximum obscure count is 2.
*   In 3D, the maximum obscure count is 3.

**Algorithm Steps**

1.  **Initialization:**
    For every sub-volume $V$ in the grid, calculate the initial `obstacle_flag` based on the geometry and the initial view direction.
    *   For each neighbor $N$ of $V$ in the immediate adjacency directions:
        *   If $N$ lies between $V$ and the camera (i.e., $N$ obscures $V$), set the corresponding bit in $V$'s `obstacle_flag` to 1.
        *   Otherwise, set the bit to 0.
    *   Calculate the initial `obscure_count` for $V$ as the number of 1s in its flag.
    *   [G] Initialize a ready queue $Q$ containing all sub-volumes where `obscure_count` == 0.

2.  **Job Assignment:**
    While there are sub-volumes remaining to be rendered:
    *   Select a sub-volume $V_{next}$ from $Q$.
    *   Assign $V_{next}$ to an available rendering agent.

3.  **Dynamic Update (Post-Render):**
    When a rendering agent completes the rendering and combination of sub-volume $V_{done}$:
    *   Identify all neighbors $N$ of $V_{done}$.
    *   For each neighbor $N$:
        *   Determine if $V_{done}$ was obscuring $N$. This is true if the bit in $N$'s `obstacle_flag` corresponding to the direction of $V_{done}$ was set to 1.
        *   If $V_{done}$ was obscuring $N$:
            *   Decrement $N$'s `obscure_count` by 1.
            *   Clear the corresponding bit in $N$'s `obstacle_flag` to 0.
            *   If $N$'s `obscure_count` becomes 0, add $N$ to the ready queue $Q$.

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class SubVolume:
    def __init__(self, id, position, dimensions):
        self.id = id
        self.position = position
        self.dimensions = dimensions
        # 2D: 4 bits, 3D: 6 bits. Stored as integer.
        self.obstacle_flag = 0 
        self.obscure_count = 0
        self.rendered = False

def initialize_obstacle_flags(grid, view_direction):
    """
    grid: dict mapping (x,y,z) -> SubVolume
    view_direction: normalized vector (dx, dy, dz)
    """
    for pos, subvol in grid.items():
        # Determine neighbors based on grid topology
        neighbors = get_neighbors(pos, grid)
        
        for neighbor_pos, neighbor_vol in neighbors:
            # Check if neighbor obscures subvol
            # Vector from subvol to neighbor
            vec_to_neighbor = neighbor_pos - pos
            
            # If the dot product of view_direction and vec_to_neighbor is positive,
            # the neighbor is "in front" of the subvol relative to the camera?
            # Actually, standard volume rendering: Camera -> Scene.
            # If Neighbor is between Camera and Subvol, Neighbor obscures Subvol.
            # Let's assume view_direction points FROM camera TO scene.
            # If vec_to_neighbor is aligned with view_direction, Neighbor is in front.
            
            if dot(view_direction, vec_to_neighbor) > 0:
                # Neighbor is in front, so it obscures Subvol
                # Set the bit corresponding to the direction of the neighbor
                direction_bit = get_direction_bit(vec_to_neighbor)
                subvol.obstacle_flag |= (1 << direction_bit)
                subvol.obscure_count += 1

def get_direction_bit(vec):
    """
    Returns the bit index for the direction.
    2D: Up(0), Down(1), Left(2), Right(3)
    3D: +X(0), -X(1), +Y(2), -Y(3), +Z(4), -Z(5)
    """
    # Implementation depends on specific grid coordinate system
    # Placeholder for logic mapping vector to bit index
    pass

def process_rendered_volume(grid, subvol_done):
    """
    Called when subvol_done is finished rendering and combined.
    Updates neighbors' obstacle flags.
    """
    subvol_done.rendered = True
    neighbors = get_neighbors(subvol_done.position, grid)
    
    for neighbor_pos, neighbor_vol in neighbors:
        if neighbor_vol.rendered:
            continue
            
        # Check if subvol_done was obscuring neighbor_vol
        # Vector from neighbor to subvol_done
        vec_to_done = subvol_done.position - neighbor_pos
        
        # If subvol_done is in front of neighbor_vol, it was obscuring it
        # Note: We must use the SAME view_direction logic as initialization
        # However, the flag was set based on initial view. 
        # The paper implies static obscuration relationships for the flag bits,
        # but dynamic clearing.
        
        # Re-evaluate if subvol_done is in the obscuring direction for neighbor_vol
        # The bit in neighbor_vol's flag corresponding to subvol_done's direction
        # should be cleared.
        
        direction_bit = get_direction_bit(vec_to_done)
        
        # Check if that bit was set
        if (neighbor_vol.obstacle_flag >> direction_bit) & 1:
            # Clear the bit
            neighbor_vol.obstacle_flag &= ~(1 << direction_bit)
            # Decrement count
            neighbor_vol.obscure_count -= 1
            
            if neighbor_vol.obscure_count == 0:
                add_to_ready_queue(neighbor_vol)

def get_neighbors(pos, grid):
    # Returns adjacent sub-volumes in the grid
    pass

def add_to_ready_queue(subvol):
    # Add to priority queue or list of available jobs
    pass
```

**Parameters**
*   **Obscure Count:** The integer value representing the number of set bits in the `obstacle_flag`. A count of 0 indicates the sub-volume is combinable.

**Edge Cases and Pitfalls**
*   **Sequential Scheduling Failure:** If the system relies on initial visibility values only (static scheduling), sub-volumes obscured by multiple neighbors may remain unassigned until *all* potential obscuring neighbors are processed, even if some are already done. This causes agent idle time [S1]. The Obstacle-Flag method mitigates this by dynamically clearing bits as neighbors complete.
*   **View Direction Changes:** The obscuration relationships are view-dependent. If the view direction changes significantly, the initial flags may become invalid. The method assumes a fixed view direction for the duration of the scheduling phase or requires re-initialization [S1].
*   **Boundary Conditions:** Sub-volumes at the grid boundary have fewer neighbors. The flag bits for non-existent neighbors should be initialized to 0 and never set.

## Operating characteristics
*   **Time Complexity:** Initialization is $O(N)$ where $N$ is the number of sub-volumes, as each sub-volume checks its immediate neighbors. Dynamic updates are $O(1)$ per rendered sub-volume, as only the immediate neighbors' flags need updating.
*   **Memory Complexity:** $O(N)$, storing one integer (4 or 6 bits) per sub-volume.
*   **Strengths:** Maximizes parallelism by allowing agents to start rendering sub-volumes as soon as their specific obscuring constraints are met, rather than waiting for a global sequential order. Reduces idle time in grid computing environments [S1].
*   **Limitations:** Requires dynamic management of job queues. Performance is dependent on the granularity of the sub-volumes; too fine granularity increases overhead, too coarse reduces parallelism.
*   **When to Use:** Distributed or grid-based volume rendering where sub-volumes are processed independently and combined into a final image.
*   **When Not to Use:** Single-threaded rendering or when the view direction changes frequently during rendering.

## Background
The Obstacle-Flag method was proposed in the context of grid computing for large-scale volume rendering. It addresses the limitations of sequential job-scheduling, which fails to utilize computing resources efficiently due to rigid dependency chains. By dynamically updating visibility parameters, it enables a more flexible and efficient distribution of rendering tasks.

## Sources
[S1] Volume rendering using grid computing for large-scale volume data, https://doi.org/10.1109/cadcg.2009.5246856, fetched 2026-10-24
