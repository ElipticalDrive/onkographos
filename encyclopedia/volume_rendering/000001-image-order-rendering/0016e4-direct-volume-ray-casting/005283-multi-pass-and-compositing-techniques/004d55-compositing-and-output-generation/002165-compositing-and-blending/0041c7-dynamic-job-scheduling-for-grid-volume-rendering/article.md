# Dynamic Job-Scheduling for Grid Volume Rendering
Also called: Obstacle-flag based scheduling, Dynamic Job-Scheduling using Obstacle-Flags, Visibility-driven job-scheduling

## Overview
This method is a distributed rendering strategy for grid-based volume rendering that dynamically determines the processing order of sub-volumes based on their visibility, using obstacle-flags to avoid the bottlenecks of static sequential scheduling [S1]. Unlike sequential methods that force agents to wait for prior results to be combined before sending the next job, this approach continuously monitors obscuring volumes and dispatches jobs as soon as they become combinable (obscure count 0), even if prior jobs are still being processed by other agents [S1]. It solves the inefficiency caused by heterogeneous agent speeds and varying resource availability in grid computing environments [S1].

## How it works
### Inputs and Outputs
**Inputs:**
- Large-scale volume data $V$.
- Camera view direction $\vec{v}$ and screen position.
- Grid of $N$ agent computers with varying computational capabilities.

**Outputs:**
- Final composited image.

### Data Structures
**Sub-volume Grid:**
The volume data is divided into equally sized cubic sub-volumes [S1]. This grid structure allows flexible screen placement and efficient obstacle-flag implementation [S1].

**Obstacle-Flag:**
Each sub-volume is assigned an obstacle-flag, which is a bitmask indicating which neighboring sub-volumes obscure the current sub-volume from the view point [S1].
- **2D Case:** The flag has 4 bits, corresponding to the four cardinal directions (e.g., right, lower, left, upper) [S1].
- **3D Case:** The flag has 6 bits, corresponding to the six faces of the cubic sub-volume (e.g., $\pm x, \pm y, \pm z$) [S1].
- **Bit Value:** A bit value of $1$ indicates an obscuring sub-volume exists in that direction; a value of $0$ indicates no obscuring sub-volume [S1].

**Obscure Count:**
The obscure count is the number of set bits ($1$s) in the obstacle-flag [S1]. It determines the visibility level:
- $0$: Not occluded (combinable).
- $1$: Partially occluded.
- $2$ (2D) / $3$ (3D): Fully or heavily occluded [S1].

### Steps
1.  **Grid Division:** Divide the large-scale volume data into equally sized cubic sub-volumes [S1].
2.  **Flag Generation:** Generate obstacle-flags for each sub-volume based on the grid structure and the current view direction [S1]. This determines the initial obscure count for every sub-volume.
3.  **Initialization:** Identify all sub-volumes with an obscure count of $0$ (zero bits set in their obstacle-flag). Add these sub-volumes to the processing list [S1].
4.  **Dispatch:** Send sub-volumes from the processing list to agent computers for rendering, prioritizing those with the lowest obscure count (specifically $0$) [S1].
5.  **Return and Combine:** When an agent returns a rendered result for a sub-volume, the central manager combines this result into the final image using front-to-back compositing [S1].
    -   The combination follows the standard optical model: $C_{final} \leftarrow C_{final} + c_i \cdot \alpha_i \cdot T$ and $T \leftarrow T \cdot (1 - \alpha_i)$, where $T$ is the accumulated transmittance [S1, S12, S50].
    -   If the accumulated opacity reaches $1.0$, sections further away are ignored [S1].
6.  **Update:** Remove the combined sub-volume from the processing list [S1].
7.  **Re-evaluate Visibility:** Update the obstacle-flags of the remaining neighboring sub-volumes to reflect the removal of the combined sub-volume [S1].
    -   Specifically, if a neighbor was obscured by the combined sub-volume, its obscure count is decremented [S1].
8.  **Dispatch New Jobs:** Determine new visibility levels for neighbors. If a neighbor's obscure count becomes $0$, it becomes combinable and is added to the processing list for dispatch [S1].
9.  **Repeat:** Steps 4–8 repeat until all sub-volumes are rendered and combined [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_job_scheduling(volume_data, view_direction, agents):
    # 1. Divide volume into cubic sub-volumes
    sub_volumes = divide_into_cubic_grid(volume_data)
    
    # 2. Generate initial obstacle-flags based on grid and view
    for sv in sub_volumes:
        sv.obstacle_flag = generate_obstacle_flag(sv, view_direction)
        sv.obscure_count = count_set_bits(sv.obstacle_flag)
        
    # Initialize processing list with non-obscured sub-volumes
    processing_list = [sv for sv in sub_volumes if sv.obscure_count == 0]
    
    # Initialize final image accumulator
    final_image = initialize_image()
    
    while processing_list:
        # 4. Dispatch jobs to agents
        # Select sub-volumes from processing_list and send to available agents
        dispatched_jobs = dispatch_jobs(processing_list, agents)
        
        # Wait for results (simplified synchronization)
        results = wait_for_results(dispatched_jobs)
        
        for result in results:
            sv = result.sub_volume
            
            # 5. Combine result into final image
            combine_front_to_back(final_image, sv.rendered_data)
            
            # 6. Remove from processing list
            processing_list.remove(sv)
            
            # 7. Update neighbors' obstacle-flags
            neighbors = get_neighbors(sv)
            for neighbor in neighbors:
                if neighbor in processing_list:
                    # Check if the combined sub-volume was an obstacle for the neighbor
                    if is_obstacle(neighbor, sv, view_direction):
                        # Decrement obscure count
                        neighbor.obscure_count -= 1
                        # Clear the corresponding bit in the flag
                        clear_bit(neighbor.obstacle_flag, direction(sv, neighbor))
                        
                        # 8. If obscure count becomes 0, add to processing list
                        if neighbor.obscure_count == 0:
                            processing_list.append(neighbor)
                            
    return final_image

def generate_obstacle_flag(sub_volume, view_direction):
    """
    Returns a bitmask where bit i is 1 if a neighbor exists in direction i
    and obscures the sub_volume from the view_direction.
    """
    flag = 0
    directions = get_directions(sub_volume) # e.g., +/- x, +/- y, +/- z
    for i, direction in enumerate(directions):
        neighbor = get_neighbor(sub_volume, direction)
        if neighbor is not None:
            # Check if neighbor is "in front" of sub_volume relative to view
            # and thus obscures it.
            if is_in_front(neighbor, sub_volume, view_direction):
                flag |= (1 << i)
    return flag

def is_obstacle(neighbor, combined_sv, view_direction):
    """
    Returns True if combined_sv was obscuring neighbor.
    """
    # Determine direction from neighbor to combined_sv
    direction = get_direction(neighbor, combined_sv)
    # Check if that direction bit was set in neighbor's flag before update
    # This implies combined_sv was an obstacle.
    return direction_bit_was_set(neighbor.obstacle_flag, direction)
```

### Parameters and Edge Cases
-   **Grid Size:** The sources specify a regular grid of 100 sub-volumes in the simulation [S1]. The method relies on equal-sized cubic sub-volumes to simplify neighbor identification [S1].
-   **View Direction:** The obstacle-flags are view-dependent. If the camera moves, flags must be regenerated [S1].
-   **Precision:** The method assumes binary visibility (obscured or not) based on grid topology, not ray-casting precision. This is a simplification for scheduling efficiency [S1].
-   **Memory:** The method avoids storing results of non-combinable sub-volumes in memory on the central manager, reducing memory pressure [S1].

## Operating characteristics
-   **Time Complexity:** The scheduling overhead is low, as it involves simple bitmask operations and neighbor lookups [S1]. The total rendering time is reduced compared to sequential scheduling because agents work in parallel without idle waiting periods [S1].
-   **Memory Complexity:** Linear with respect to the number of sub-volumes, as each sub-volume stores a small fixed-size flag (4 or 6 bits) [S1].
-   **Performance:** In a simulation with 12 agents (4 low-spec, 4 middle-spec, 4 high-spec) and 100 sub-volumes, the method showed significant efficiency improvements over sequential scheduling [S1]. The improvement is attributed to the ability to dispatch jobs to faster agents immediately when they become available, rather than waiting for a strict sequence [S1].
-   **Strengths:** Handles heterogeneous agent speeds well; reduces idle time; scalable to large grids.
-   **Limitations:** Requires a grid structure; visibility is determined by grid topology rather than actual ray-casting occlusion, which may lead to sub-optimal scheduling in complex scenes with internal transparency [S1]. The method assumes front-to-back compositing [S1].

## Background
-   **Origin:** Proposed in 2009 by researchers utilizing grid computing for large-scale volume rendering [S1].
-   **Context:** Direct volume rendering (ray casting and splatting) computational cost increases proportionally to the cube of the data size [S1]. Grid computing distributes this load, but static scheduling fails to utilize heterogeneous resources efficiently [S1].
-   **Development:** The method was implemented on a university campus grid and evaluated against sequential job-scheduling [S1].

## Sources
[S1] Volume rendering using grid computing for large-scale volume data, https://doi.org/10.1109/cadcg.2009.5246856, fetched 2026-10-24
