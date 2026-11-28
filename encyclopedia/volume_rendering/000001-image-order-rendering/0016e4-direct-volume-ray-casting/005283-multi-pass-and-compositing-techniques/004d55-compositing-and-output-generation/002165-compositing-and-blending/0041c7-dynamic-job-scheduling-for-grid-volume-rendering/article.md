# Dynamic Job-Scheduling for Grid Volume Rendering
Also called: Obstacle-flag based scheduling, Dynamic Job-Scheduling using Obstacle-Flags, Visibility-driven job-scheduling

## Overview
Dynamic Job-Scheduling for Grid Volume Rendering is a distributed rendering technique that determines the processing order of sub-volumes based on their real-time visibility rather than a static sequence [S1]. It solves the bottleneck caused by heterogeneous agent speeds in grid computing, where static sequential scheduling forces fast agents to wait for slow agents to complete prior steps [S1]. By using obstacle-flags to track obscuration, the method allows agents to immediately process newly visible sub-volumes, significantly improving efficiency over sequential approaches [S1].

## How it works
### Inputs and Outputs
**Inputs:**
- Large-scale volume dataset.
- Grid configuration: The volume is divided into equally sized cubic sub-volumes [S1].
- View direction: Used to determine which neighbors obscure the current sub-volume [S1].
- Agent pool: A set of computers with varying computational capabilities [S1].

**Outputs:**
- Final composited image [S1].

### Data Structures
**Sub-Volume Grid:**
- The volume is partitioned into a regular grid of cubic sub-volumes [S1].
- Each sub-volume is identified by its grid coordinates $(x, y, z)$ [S1].

**Obstacle-Flag:**
- A bitmask associated with each sub-volume to manage visibility relationships with neighbors [S1].
- **2D Layout:** 4 bits, representing the four cardinal directions (e.g., right, lower, left, upper) [S1].
- **3D Layout:** 6 bits, representing the six faces of the cube (±x, ±y, ±z) [S1].
- **Bit Meaning:** A bit value of `1` indicates an obscuring sub-volume exists in that direction; `0` indicates no obscuring sub-volume [S1].
- **Obscure Count:** The number of `1` bits in the obstacle-flag [S1].
  - **2D:** Maximum obscure count is 2 (fully obscured) [S1].
  - **3D:** Maximum obscure count is 3 (fully obscured) [S1].
  - **Visibility Levels:**
    - Obscure Count = 0: Not obscured (combinable) [S1].
    - Obscure Count > 0: Partially or fully obscured [S1].

**Processing List:**
- A queue of sub-volumes currently assigned to agents for rendering [S1].

### Algorithm Steps
1.  **Grid Division:** Divide the large-scale volume data into equally sized cubic sub-volumes [S1].
2.  **Flag Initialization:** Generate obstacle-flags for each sub-volume based on the grid structure and view directions [S1].
    - For each sub-volume, check its neighbors in the 6 (or 4) directions relative to the view [S1].
    - If a neighbor exists and is "in front" (closer to the camera) or obscures the line of sight, set the corresponding bit in the obstacle-flag to `1` [S1].
3.  **Initial Job Selection:** Identify all sub-volumes with an obscure count of 0 (zero bits set in their obstacle-flag) [S1].
4.  **Dispatch:** Add these sub-volumes to the processing list and send them to available agent computers for rendering [S1].
5.  **Result Return:** When an agent completes rendering a sub-volume, it returns the result to the central manager [S1].
6.  **Combination:** The central manager combines the returned result into the final image using front-to-back compositing [S1].
    - If the accumulated opacity reaches 1.0, further samples in that ray are ignored [S1].
7.  **List Update:** Remove the completed sub-volume from the processing list [S1].
8.  **Flag Update:** Update the obstacle-flags of the remaining neighboring sub-volumes [S1].
    - For each neighbor of the completed sub-volume, check if the completed sub-volume was the obscuring volume in the direction pointing back to the neighbor [S1].
    - If so, clear the corresponding bit in the neighbor's obstacle-flag (change `1` to `0`) [S1].
9.  **Re-evaluation:** Recalculate the obscure count for the affected neighbors [S1].
10. **New Job Selection:** If a neighbor's obscure count becomes 0, it is now combinable [S1]. Add it to the processing list and send it to an available agent [S1].
11. **Repeat:** Continue steps 5–10 until all sub-volumes are rendered and combined [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_job_scheduling(volume_data, grid_dims, view_dir):
    """
    Implements dynamic job-scheduling for grid volume rendering using obstacle-flags.
    
    Args:
        volume_data: The full 3D volume dataset.
        grid_dims: Tuple (nx, ny, nz) defining the number of sub-volumes in each axis.
        view_dir: 3D vector indicating the camera view direction.
        
    Returns:
        final_image: The composited final image.
    """
    # 1. Initialize Grid and Obstacle-Flags
    sub_volumes = [(x, y, z) for x in range(grid_dims[0]) 
                           for y in range(grid_dims[1]) 
                           for z in range(grid_dims[2])]
    obstacle_flags = {vol: 0 for vol in sub_volumes}
    obscure_counts = {vol: 0 for vol in sub_volumes}
    directions = [(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)]

    def get_neighbors(vol):
        return [(vol[0]+dx, vol[1]+dy, vol[2]+dz) for dx,dy,dz in directions
                if 0 <= vol[0]+dx < grid_dims[0] and 0 <= vol[1]+dy < grid_dims[1] and 0 <= vol[2]+dz < grid_dims[2]]

    def count_bits(flag):
        return bin(flag).count('1')

    # Initialize flags based on view direction
    for vol in sub_volumes:
        flag = 0
        for i, (dx, dy, dz) in enumerate(directions):
            neighbor = (vol[0]+dx, vol[1]+dy, vol[2]+dz)
            if neighbor in sub_volumes:
                # [G] Geometric obscuration check: neighbor must be closer to camera along view_dir
                # and overlap in the projection plane. Exact logic depends on ray-casting.
                flag |= (1 << i)
        obstacle_flags[vol] = flag
        obscure_counts[vol] = count_bits(flag)

    # 2. Initial Job Selection
    processing_list = [vol for vol, count in obscure_counts.items() if count == 0]
    pending_jobs = []
    final_image = None

    # 3. Dispatch Loop
    while processing_list or pending_jobs:
        # Dispatch available jobs to agents
        dispatched = list(processing_list)
        for vol in dispatched:
            pending_jobs.append(vol)
        processing_list.clear()

        # Wait for a job to complete (synchronous simulation)
        completed_vol = pending_jobs.pop(0)

        # 4. Combine Result using front-to-back compositing
        # [G] Standard compositing loop updates C_acc and T for the sub-volume's samples
        # If T <= 1e-6, break early. Composite over background.
        if final_image is None:
            final_image = render_sub_volume(volume_data, completed_vol)
        else:
            sub_result = render_sub_volume(volume_data, completed_vol)
            # [G] Front-to-back: C_acc += T * alpha * c; T *= (1 - alpha)
            combine_result_into_image(final_image, sub_result)

        # 5. Update Flags and Visibility
        neighbors = get_neighbors(completed_vol)
        for neighbor in neighbors:
            dir_idx = directions.index((completed_vol[0]-neighbor[0], completed_vol[1]-neighbor[1], completed_vol[2]-neighbor[2]))
            if obstacle_flags[neighbor] & (1 << dir_idx):
                obstacle_flags[neighbor] &= ~(1 << dir_idx)
                obscure_counts[neighbor] = count_bits(obstacle_flags[neighbor])
                if obscure_counts[neighbor] == 0 and neighbor not in processing_list and neighbor not in pending_jobs:
                    processing_list.append(neighbor)

    return final_image

def render_sub_volume(data, vol):
    # [G] Standard direct volume ray casting or splatting for the sub-volume
    return None

def combine_result_into_image(img, result):
    # [G] Standard front-to-back compositing step
    pass
```

### Parameters and Edge Cases
- **Sub-Volume Size:** Equally sized cubic sub-volumes are used to allow flexible screen placement and easy obstacle-flag implementation [S1].
- **View Direction:** Critical for determining obscuration. The flags are generated based on the current view [S1].
- **Precision:** The method relies on integer bitmasks, avoiding floating-point precision issues in scheduling logic [S1].
- **Memory:** Non-combinable sub-volumes are not stored in memory on the central manager until they become combinable, reducing memory usage compared to storing all results [S1].

## Operating characteristics
- **Time Complexity:** The scheduling overhead is low, as it involves simple bit manipulation and neighbor checks [S1]. The overall rendering time is reduced compared to sequential scheduling due to better utilization of heterogeneous agents [S1].
- **Memory Complexity:** Memory usage is minimized because only combinable (visible) sub-volumes are processed and stored in the final image accumulation. Intermediate results for obscured sub-volumes are not held in memory [S1].
- **Performance:** Experiments on a university campus grid showed significant improvements in efficiency for large-scale volume rendering compared to sequential job-scheduling [S1].
- **Strengths:**
  - Handles heterogeneous agent speeds effectively [S1].
  - Reduces idle time for fast agents [S1].
  - Flexible screen placement due to cubic sub-volumes [S1].
- **Limitations:**
  - Requires a central manager to coordinate and combine results [S1].
  - The visibility update logic depends on the grid structure and view direction [S1].
  - Not given in the sources: Performance impact of frequent flag updates in very large grids.

## Background
- **Origin:** Proposed in a paper by researchers implementing a grid computing system for direct volume rendering [S1].
- **History:** The method was developed to address the inefficiencies of static sequential scheduling in grid environments where agents have varying capabilities and resources [S1].
- **Usage:** Implemented on a university campus grid for large-scale volume rendering [S1].

## Sources
[S1] Volume rendering using grid computing for large-scale volume data, https://doi.org/10.1109/cadcg.2009.5246856, fetched 2026-10-24
