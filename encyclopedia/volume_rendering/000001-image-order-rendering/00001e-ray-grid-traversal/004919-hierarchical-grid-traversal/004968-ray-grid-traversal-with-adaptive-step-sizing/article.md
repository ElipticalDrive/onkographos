# Ray Grid Traversal with Adaptive Step Sizing

Also called: CNC Traversal, Ray Traversal through Multiple Voxel Grids

## Overview
This method traverses one or more axis-aligned voxel grids to generate sample points and interval boundaries for volume rendering, specifically designed for GPU execution [S1, unofficial]. It differs from standard DDA by supporting multiple overlapping grids with priority-based occupancy masking and employing cone-angle-based adaptive step sizing to vary sample density with distance from the camera [S1, unofficial]. It is primarily used in the `nerfacc` library for accelerating NeRF and 3D Gaussian Splatting rendering pipelines [S1, unofficial].

## How it works
The method operates on a batch of rays, processing them in parallel. It relies on precomputed sorted intersection distances between rays and grid boundaries to efficiently skip empty space and manage multiple grids.

### Inputs and Outputs
**Inputs:**
*   `rays_o`: Ray origins, shape `[n_rays, 3]` [S1, unofficial].
*   `rays_d`: Ray directions, shape `[n_rays, 3]` [S1, unofficial].
*   `rays_mask`: Optional boolean mask, shape `[n_rays]`, to skip specific rays [S1, unofficial].
*   `binaries`: Voxel occupancy grids, shape `[n_grids, resx, resy, resz]`, boolean type [S1, unofficial].
*   `aabbs`: Axis-aligned bounding boxes for each grid, shape `[n_grids, 6]` (min/max x, y, z), float type [S1, unofficial].
*   `t_sorted`: Sorted intersection distances for all grids, shape `[n_rays, n_grids * 2]` [S1, unofficial].
*   `t_indices`: Grid IDs corresponding to each intersection in `t_sorted`, shape `[n_rays, n_grids * 2]` [S1, unofficial].
*   `hits`: Boolean flags indicating if a ray hits a grid, shape `[n_rays, n_grids]` [S1, unofficial].
*   `near_planes`: Near clipping planes for each ray, shape `[n_rays]` [S1, unofficial].
*   `far_planes`: Far clipping planes for each ray, shape `[n_rays]` [S1, unofficial].
*   `step_size`: Base step length for ray marching. Default `1e-3`. If `<= 0.0`, the kernel records steps through each voxel cell instead of marching [S1, unofficial].
*   `cone_angle`: Controls linear increase of step size with distance. Must be `>= 0.0`. A value of `0.0` means uniform marching with `step_size` [S1, unofficial].
*   `traverse_steps_limit`: Maximum number of traversal steps/samples per ray. Default `None` (unlimited). If `<= 0`, no limit is applied [S1, unofficial].
*   `over_allocate`: Boolean flag. If `true`, memory is pre-allocated based on limits for single-pass traversal. If `false`, a two-pass approach (count then fill) is used [S1, unofficial].

**Outputs:**
*   `intervals`: PackedRaySegmentsSpec containing interval values, ray indices, `is_left`, `is_right` flags, and `chunk_cnts` [S1, unofficial].
*   `samples`: PackedRaySegmentsSpec containing sample values, ray indices, `is_valid` flags, and `chunk_cnts` [S1, unofficial].
*   `terminate_planes`: Optional float array, shape `[n_rays]`, storing the final traversal distance `t_last` [S1, unofficial].

### Data Structures
The algorithm uses packed segment specifications for outputs to handle variable-length outputs per ray efficiently on GPU.
*   `PackedRaySegmentsSpec`: Contains `vals` (float array of samples/interval bounds), `ray_indices` (int array mapping to ray ID), `is_left`/`is_right`/`is_valid` (boolean flags), and `chunk_cnts` (int array storing the count of elements per ray) [S1, unofficial].

### Parameters
*   `step_size`: Default `1e-3` [S1, unofficial].
*   `cone_angle`: Default `0.0` (uniform) [S1, unofficial].
*   `traverse_steps_limit`: Default `None` [S1, unofficial].
*   `dt_max`: Hard upper bound for step size, typically `1e10f` [S1, unofficial].

### Algorithm Steps
The traversal is performed in a CUDA kernel parallelized over rays (`tid = blockIdx.x * blockDim.x + threadIdx.x`) [S1, unofficial].

1.  **Initialization:**
    *   If `rays_mask` is provided and `rays_mask[tid]` is false, terminate the thread [S1, unofficial].
    *   If not the first pass and `intervals.chunk_cnts[tid] == 0`, terminate [S1, unofficial].
    *   Initialize `t_last = near_planes[tid]`, `continuous = false`, `n_intervals = 0`, `n_samples = 0` [S1, unofficial].
    *   Determine the start index in `t_sorted` based on the number of grids [S1, unofficial].

2.  **Loop over Intersections:**
    *   Iterate through the sorted intersection distances `t_sorted` for the current ray. The loop runs from `base_t_sorted` to `base_t_sorted + n_grids * 2 - 1` [S1, unofficial].
    *   For each pair of intersections (entry/exit) associated with a grid:
        *   Calculate valid interval bounds clamped to near/far planes:
            $$
            \text{this\_tmin} = \max(\text{t\_sorted}[i], \text{near\_planes}[tid])
            $$
            $$
            \text{this\_tmax} = \min(\text{t\_sorted}[i + 1], \text{far\_planes}[tid])
            $$
        *   If `this_tmin >= this_tmax`, skip this interval (invalid/zero-length) [S1, unofficial].

3.  **Grid Traversal and Step Sizing:**
    *   Setup AABB traversal for the current grid using `setup_traversal` to compute voxel step `delta`, distance to next boundary `tdist`, and indices [S1, unofficial].
    *   **Empty Cell Handling:** If the current voxel `binaries[cell_id]` is false:
        *   Set `continuous = false` [S1, unofficial].
        *   If `step_size <= 0.0`, set `t_last = t_traverse` (the end of the empty segment) [S1, unofficial].
        *   If `step_size > 0.0`, march `t_last` forward using adaptive step size until `t_last + dt * 0.5 >= t_traverse` [S1, unofficial].
    *   **Occupied Cell Handling:** If `binaries[cell_id]` is true:
        *   Calculate `t_next` (the end of the current voxel traversal) [S1, unofficial].
        *   If `step_size <= 0.0`, set `t_next = t_traverse` [S1, unofficial].
        *   If `step_size > 0.0`:
            *   Calculate adaptive step size `dt` (see Formula).
            *   Check if `t_last + dt * 0.5 >= t_traverse`. If so, break the inner loop (end of voxel) [S1, unofficial].
            *   Otherwise, set `t_next = t_last + dt` and record the sample [S1, unofficial].
            *   Update `t_last = t_next` [S1, unofficial].

4.  **Adaptive Step Size Formula:**
    The step size $dt$ is calculated based on the current distance $t$ along the ray:
    $$
    dt = \text{clamp}(t \cdot \text{cone\_angle}, dt_{min}, dt_{max})
    $$
    Where:
    *   $t$ is the current distance along the ray (`t_last`) [S1, unofficial].
    *   $dt_{min}$ is the `step_size` parameter [S1, unofficial].
    *   $dt_{max}$ is `1e10f` [S1, unofficial].

5.  **Output Writing:**
    *   If `intervals.chunk_cnts` is not null and not the first pass:
        *   If `!continuous`: Write left interval (`t_last`) with `is_left=true`, `is_right=true`. Increment `n_intervals` twice [S1, unofficial].
        *   If `continuous`: Write right interval (`t_next`) with `is_left` set on previous index, `is_right=true`. Increment `n_intervals` once [S1, unofficial].
    *   Advance traversal to the next voxel using `single_traversal` [S1, unofficial]. If it returns false, break the loop [S1, unofficial].

6.  **Finalization:**
    *   Set `terminate_planes[tid] = t_last` if the output array is provided [S1, unofficial].
    *   Set `intervals.chunk_cnts[tid] = n_intervals` and `samples.chunk_cnts[tid] = n_samples` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_grid_traversal_adaptive(
    rays_o, rays_d, rays_mask, binaries, aabbs, 
    t_sorted, t_indices, hits, near_planes, far_planes,
    step_size, cone_angle, traverse_steps_limit, over_allocate
):
    """
    Simulates the GPU kernel logic for Ray Grid Traversal with Adaptive Step Sizing.
    This is a sequential simulation for clarity; the actual implementation is parallelized over rays.
    """
    n_rays = len(rays_o)
    n_grids = len(binaries)
    
    # Output containers
    intervals_vals = []
    intervals_ray_indices = []
    intervals_is_left = []
    intervals_is_right = []
    intervals_chunk_cnts = [0] * n_rays
    
    samples_vals = []
    samples_ray_indices = []
    samples_is_valid = []
    samples_chunk_cnts = [0] * n_rays
    
    terminate_planes = [0.0] * n_rays

    for tid in range(n_rays):
        # 1. Skip masked rays
        if rays_mask is not None and not rays_mask[tid]:
            continue
            
        # 2. Initialize state
        t_last = near_planes[tid]
        continuous = False
        n_intervals = 0
        n_samples = 0
        
        # Base index in sorted intersections for this ray
        # Assuming t_sorted is structured per ray
        base_idx = tid * n_grids * 2
        
        # 3. Loop over intersections
        # t_sorted has shape [n_rays, n_grids * 2]
        # We iterate through pairs of intersections for each grid
        for g in range(n_grids):
            idx_start = base_idx + g * 2
            t_min_raw = t_sorted[idx_start]
            t_max_raw = t_sorted[idx_start + 1]
            
            # Clamp to near/far planes
            this_tmin = max(t_min_raw, near_planes[tid])
            this_tmax = min(t_max_raw, far_planes[tid])
            
            # Skip invalid intervals
            if this_tmin >= this_tmax:
                continue
                
            # Determine if this grid is hit
            if not hits[tid, g]:
                continue
                
            # Setup AABB traversal for this grid
            # In the actual kernel, this computes delta, tdist, etc.
            # Here we simulate stepping through voxels in this interval
            # Note: The actual kernel uses a specific AABB traversal setup
            # For this pseudocode, we assume we are inside the grid bounds
            # and need to march from this_tmin to this_tmax
            
            # Get grid resolution and AABB
            res = binaries[g].shape
            aabb = aabbs[g] # [min_x, min_y, min_z, max_x, max_y, max_z]
            
            # Simple simulation of voxel stepping within this_tmin to this_tmax
            # The actual kernel uses 'setup_traversal' and 'single_traversal'
            # which handle the DDA logic internally.
            
            current_t = this_tmin
            
            # While loop to march through voxels in this interval
            # The kernel handles this via the intersection pairs, 
            # but internally it steps voxel by voxel.
            # For the purpose of the adaptive step logic:
            
            while current_t < this_tmax:
                # Determine current voxel index (simplified)
                # In reality, this is done via the traversal state
                # voxel_idx = floor((current_t * direction + origin - aabb_min) / voxel_size)
                
                # Check occupancy
                # Note: The kernel checks 'binaries[cell_id]'
                # We simulate checking if the cell is occupied
                # In a real implementation, we'd need the full DDA state
                # Here we assume we are in an occupied cell for sample generation
                # or empty for skipping.
                
                # For the sake of the adaptive step logic described in [S1, unofficial]:
                # If empty: continuous = False, t_last = this_tmax (end of segment)
                # If occupied: march with adaptive steps
                
                # Simulating occupied cell logic
                if step_size <= 0.0:
                    # Fixed step to boundary
                    t_next = this_tmax
                else:
                    # Adaptive step
                    dt = clamp(t_last * cone_angle, step_size, 1e10)
                    
                    # Check if next step exceeds interval end
                    if t_last + dt * 0.5 >= this_tmax:
                        t_next = this_tmax
                    else:
                        t_next = t_last + dt
                        # Record sample
                        sample_val = (t_next + t_last) * 0.5
                        samples_vals.append(sample_val)
                        samples_ray_indices.append(tid)
                        samples_is_valid.append(True)
                        n_samples += 1
                        t_last = t_next
                
                # If we reached the end of the interval
                if t_next >= this_tmax:
                    current_t = this_tmax
                    # Handle interval output
                    if not continuous:
                        # Write left interval
                        intervals_vals.append(t_last)
                        intervals_ray_indices.append(tid)
                        intervals_is_left.append(True)
                        intervals_is_right.append(True)
                        n_intervals += 2 # Left and Right
                    else:
                        # Write right interval
                        intervals_vals.append(t_next)
                        intervals_ray_indices.append(tid)
                        intervals_is_left.append(False) # Left was written previously
                        intervals_is_right.append(True)
                        n_intervals += 1
                    
                    continuous = True
                    break
                else:
                    # Continue to next voxel
                    # In real kernel, 'single_traversal' advances the index
                    # Here we just advance t_last
                    pass

        # Finalize counts
        intervals_chunk_cnts[tid] = n_intervals
        samples_chunk_cnts[tid] = n_samples
        terminate_planes[tid] = t_last

    return {
        'intervals': {
            'vals': intervals_vals,
            'ray_indices': intervals_ray_indices,
            'is_left': intervals_is_left,
            'is_right': intervals_is_right,
            'chunk_cnts': intervals_chunk_cnts
        },
        'samples': {
            'vals': samples_vals,
            'ray_indices': samples_ray_indices,
            'is_valid': samples_is_valid,
            'chunk_cnts': samples_chunk_cnts
        },
        'terminate_planes': terminate_planes
    }

def clamp(val, min_val, max_val):
    if val < min_val: return min_val
    if val > max_val: return max_val
    return val
```

### Edge Cases and Pitfalls
*   **Overlapping Grids:** The order of grids matters. The grid with the lower index (front) has higher priority when examining occupancy status. If grids overlap, the first grid's occupancy status overwrites the second grid's [S1, unofficial].
*   **Invalid Intervals:** If the clamped near bound is greater than or equal to the clamped far bound (e.g., due to `near_plane > far_plane` or grid boundaries), the interval is skipped to prevent zero-length or negative-length marching segments [S1, unofficial].
*   **Zero Step Size:** If `step_size <= 0.0`, the kernel records steps through each voxel cell instead of marching, effectively disabling adaptive sampling in favor of voxel-boundary traversal [S1, unofficial].
*   **Memory Allocation:** If `over_allocate` is false, a two-pass approach is required: the first pass counts the number of intervals/samples, and the second pass fills the data [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of rays and the number of voxels traversed per ray. The adaptive step size reduces the number of samples for rays far from the origin, improving performance for large scenes [S1, unofficial].
*   **Memory Complexity:** Depends on the output size. If `over_allocate` is true, memory is pre-allocated based on `traverse_steps_limit`, which can be wasteful if the limit is not tight. If false, a two-pass approach is used, requiring temporary storage for counts [S1, unofficial].
*   **Performance:** Optimized for GPU execution using CUDA. It is designed for high-throughput ray marching in NeRF and 3D Gaussian Splatting [S1, unofficial].
*   **Strengths:** Supports multiple grids, adaptive step sizing, and efficient empty space skipping.
*   **Limitations:** Requires precomputed sorted intersections. Overlapping grids require careful ordering.

## Background
This method is implemented in the `nerfacc` library, a general NeRF acceleration toolbox [S1, unofficial]. It is used in the context of CNC (Neural Compression of NeRFs) and other NeRF rendering pipelines [S1, unofficial]. The specific kernel is named `traverse_grids` or `traverse_grids_kernel` [S1, unofficial].

## Sources
[S1] YihangChen-ee/CNC, https://github.com/YihangChen-ee/CNC, fetched 2026-10-27 (unofficial)
