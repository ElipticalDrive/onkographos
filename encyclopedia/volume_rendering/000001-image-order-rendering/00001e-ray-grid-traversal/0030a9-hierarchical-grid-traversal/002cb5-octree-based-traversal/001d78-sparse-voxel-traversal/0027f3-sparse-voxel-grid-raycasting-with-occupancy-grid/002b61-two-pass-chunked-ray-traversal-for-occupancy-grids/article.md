# Two-pass chunked ray traversal for occupancy grids

Also called: Chunked ray traversal with occupancy grid, Two-pass occupancy grid traversal

## Overview
This method accelerates sparse voxel grid raycasting by decoupling the counting of occupied traversal steps from the actual sample generation. It solves the problem of variable-length sample buffers per ray in a batched CUDA kernel by first computing the exact number of occupied intervals (chunks) each ray intersects, allocating a contiguous buffer, and then filling that buffer with precise sample intervals. It builds on [Sparse Voxel Grid Raycasting with Occupancy Grid](4152-sparse-voxel-grid-raycasting-with-occupancy-grid.md) by replacing fixed-step iteration with a dynamic, occupancy-aware traversal that minimizes memory waste and coalesces memory accesses.

## How it works
The method consists of two sequential CUDA kernel launches. The first pass counts the number of occupied voxel intervals (chunks) each ray traverses to determine buffer sizes. The second pass writes the actual sample data into pre-allocated tensors using the computed offsets.

### Inputs and Outputs
**Inputs:**
- `grids`: `int64` array of pointers to `nanovdb::MaskGrid**` (device memory).
- `rays_o`: `float` array of shape `[n_rays, 3]`, ray origins.
- `rays_d`: `float` array of shape `[n_rays, 3]`, ray directions.
- `resolution`: `int32` array of shape `[3]`, grid resolution per axis.
- `aabbs`: `float` array of shape `[n_grids, 6]`, axis-aligned bounding boxes for each grid.
- `hits`: `bool` array of shape `[n_rays, n_grids]`, pre-computed ray-grid intersection flags.
- `t_sorted`: `float` array of shape `[n_rays, n_grids*2]`, sorted entry/exit distances per grid.
- `t_indices`: `int64` array of shape `[n_rays, n_grids*2]`, indices into `t_sorted`.
- `near_planes`: `float` array of shape `[n_rays]`.
- `far_planes`: `float` array of shape `[n_rays]`.

**Outputs (allocated before launch):**
- `chunk_counts`: `int64` array of shape `[n_rays]`, number of occupied chunks per ray.
- `chunk_indices`: `int64` array of shape `[n_rays]`, starting index in the sample buffer for each ray.
- `ray_indices`: `int64` array of shape `[n_samples]`, maps each sample to its originating ray.
- `t_left`: `float` array of shape `[n_samples]`, entry distance of each sample interval.
- `t_right`: `float` array of shape `[n_samples]`, exit distance of each sample interval.

### Data Structures and Layout
The core data structure is the occupancy grid (`nanovdb::MaskGrid`), accessed via pointers in `grids`. The traversal relies on sorted entry/exit distances (`t_sorted`) and their corresponding grid indices (`t_indices`) to efficiently skip empty regions. The output buffers are flattened 1D arrays where samples from all rays are concatenated. `chunk_indices` acts as a prefix sum (exclusive) of `chunk_counts` to determine write offsets.

### Parameters
- `step_size`: `float`, fixed step length for traversal within occupied regions. Not given in the sources.
- `cone_angle`: `float`, cone angle for frustum traversal. Not given in the sources.
- `threads`: `int`, number of threads per block. Typical value: 256 [S1, unofficial].
- `n`: `int`, update frequency for the grid. Default: 16 [S1, unofficial].

### Numbered Steps
1. **First Pass (Counting):** Launch `traverse_kernel` with `first_pass=true`. Each thread processes one ray. It iterates through the sorted `t_sorted` entries for that ray, checking `hits` and occupancy. It counts the number of occupied intervals (chunks) and stores the result in `chunk_counts[ray_id]`. Output pointers for sample data are ignored or set to `nullptr` to avoid writes.
2. **Prefix Sum:** Compute the exclusive prefix sum of `chunk_counts` to determine `chunk_indices`. The total number of samples `n_samples` is the sum of all `chunk_counts`.
3. **Second Pass (Filling):** Launch `traverse_kernel` with `first_pass=false`. Each thread processes one ray. It iterates through the same sorted intervals, but this time writes the `t_left`, `t_right`, and `ray_id` into the pre-allocated `t_left`, `t_right`, and `ray_indices` buffers at the offset specified by `chunk_indices[ray_id]`. The offset is incremented after each write.

### Pseudocode
```python
# Kernel logic for traverse_kernel
def traverse_kernel(rays_o, rays_d, resolution, aabbs, hits, t_sorted, t_indices, 
                    near_planes, far_planes, chunk_counts, chunk_indices, 
                    ray_indices, t_left, t_right, first_pass):
    ray_id = get_thread_id()
    if ray_id >= n_rays: return

    # Determine traversal range based on hits and sorted distances
    # This part depends on the specific nanovdb traversal logic
    # We assume t_sorted and t_indices are pre-processed for this ray
    
    count = 0
    offset = 0
    if not first_pass:
        offset = chunk_indices[ray_id]

    # Iterate through relevant grid intervals for this ray
    # The exact iteration logic over t_sorted/t_indices is implementation-specific
    # but generally involves checking occupancy and accumulating intervals
    
    for i in range(num_intervals_for_ray):
        # Check if this interval is occupied
        if is_occupied(t_sorted[ray_id, i], t_sorted[ray_id, i+1], grids):
            t_l = t_sorted[ray_id, i]
            t_r = t_sorted[ray_id, i+1]
            
            # Clamp to near/far planes if necessary
            t_l = max(t_l, near_planes[ray_id])
            t_r = min(t_r, far_planes[ray_id])
            
            if t_l < t_r:
                if first_pass:
                    count += 1
                else:
                    # Write sample interval
                    t_left[offset] = t_l
                    t_right[offset] = t_r
                    ray_indices[offset] = ray_id
                    offset += 1

    if first_pass:
        chunk_counts[ray_id] = count
```

### Edge Cases and Pitfalls
- **Pre-allocation:** The second pass requires pre-allocated output tensors for `t_left`, `t_right`, and `ray_indices`. Passing `nullptr` for these in the first pass avoids overwriting or invalid memory access [S1, unofficial].
- **Precision:** Floating-point precision in `t_sorted` and `t_left`/`t_right` can cause gaps or overlaps if not handled carefully. Ensure consistent clamping to `near_planes` and `far_planes`.
- **Empty Rays:** Rays with no hits should result in `chunk_counts[ray_id] = 0` and no writes in the second pass.
- **Degenerate Input:** If `n_rays` is 0 or `n_grids` is 0, the kernel should handle this gracefully without launching.

## Operating characteristics
- **Time Complexity:** $O(N_{rays} \cdot N_{grids})$ for the first pass (counting) and $O(N_{samples})$ for the second pass (filling), where $N_{samples}$ is the total number of occupied intervals. The prefix sum is $O(N_{rays})$.
- **Memory Complexity:** $O(N_{rays} \cdot N_{grids})$ for `t_sorted` and `t_indices`, plus $O(N_{samples})$ for the output buffers. This is more memory-efficient than fixed-step methods if occupancy is sparse.
- **Performance:** Measured performance depends on hardware and scene density. The two-pass approach adds overhead for the counting pass but reduces memory bandwidth usage in the second pass by avoiding redundant writes. It is most beneficial when occupancy is highly variable across rays.
- **Strengths:** Exact buffer allocation, reduced memory waste, coalesced memory accesses in the second pass.
- **Limitations:** Requires two kernel launches and a prefix sum operation. Not suitable for real-time applications with strict latency constraints where the overhead of the first pass is prohibitive.

## Background
This method is derived from implementations in the `faster-occgrid` repository, which adapts techniques from neural radiance field (NeRF) rendering to sparse occupancy grids. It addresses the inefficiency of fixed-step ray marching in sparse environments by dynamically allocating memory based on actual occupancy.

## Sources
[S1] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
