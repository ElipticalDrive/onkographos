# Multi-Grid Ray Traversal

Also called: Ray traversal through multiple voxel grids

## Overview
Multi-Grid Ray Traversal traverses rays through multiple binary occupancy grids to generate ray intervals and samples, enabling efficient rendering in scenes with complex or disjoint geometry [S1, unofficial]. It sets itself apart from single-grid methods by pre-computing ray-AABB intersections to determine which grids are hit, sorting the entry/exit points, and then performing voxel-level traversal only within the valid intervals defined by those intersections [S1, unofficial]. This approach avoids unnecessary traversal through empty space or non-existent geometry in multi-grid setups.

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `rays_o`: `[n_rays, 3]` float array of ray origins.
    *   `rays_d`: `[n_rays, 3]` float array of normalized ray directions.
    *   `binaries`: `[n_grids, resx, resy, resz]` boolean array representing voxel occupancy. All grids share the same resolution [S1, unofficial].
    *   `aabbs`: `[n_grids, 6]` float array storing axis-aligned bounding boxes per grid (min/max x, y, z) [S1, unofficial].
    *   `t_sorted`: `[n_rays, n_grids * 2]` float array of pre-computed, sorted ray-grid intersection distances [S1, unofficial].
    *   `t_indices`: `[n_rays, n_grids * 2]` int64 array mapping each intersection to its grid ID (0-indexed) [S1, unofficial].
    *   `hits`: `[n_rays, n_grids]` boolean array indicating whether each ray hits each grid [S1, unofficial].
    *   `near_plane`: Optional float array `[n_rays]`. Defaults to 0.0 [S1, unofficial].
    *   `far_plane`: Optional float array `[n_rays]`. Defaults to infinity [S1, unofficial].
    *   `step_size`: Optional float for the ray traversal step size. Default is $10^{-3}$ [S1, unofficial].
    *   `cone_angle`: Optional float for linearly-increased step size. $0.0$ means constant step size. Default is $0.0$ [S1, unofficial].
    *   `traverse_steps_limit`: Optional int for the maximum number of samples per ray. Default is $-1$ (unlimited) [S1, unofficial].
    *   `rays_mask`: Optional BoolTensor `[n_rays]` to skip specific rays. Default is all ones [S1, unofficial].
    *   `first_pass`: Boolean flag. If false, skips rays with `chunk_cnts[tid] == 0` to avoid redundant processing [S1, unofficial].
    *   `eps`: Epsilon for traversal setup. Default is $10^{-6}$ [S1, unofficial].
*   **Outputs:**
    *   `intervals`: `RayIntervals` object containing traversal intervals [S1, unofficial].
    *   `samples`: `RaySamples` object containing sample positions within each interval [S1, unofficial].
    *   `terminated_t`: Tensor `[n_rays]` containing terminated t values for each ray [S1, unofficial].

**Data Structures:**
The method relies on pre-computed intersection data (`t_sorted`, `t_indices`, `hits`) to define the coarse-grained intervals. Within these intervals, standard DDA traversal is performed on the binary grids. The output structures (`RayIntervals`, `RaySamples`) support chunked memory layouts via `chunk_cnts` and `chunk_starts` to handle variable numbers of samples per ray [S1, unofficial].

**Steps:**
1.  **Initialization:**
    *   Initialize `rays_mask` to all ones if not provided [S1, unofficial].
    *   Initialize `near_planes` to zeros and `far_planes` to infinity if not provided [S1, unofficial].
    *   Initialize `traverse_steps_limit` to $-1$ if not provided [S1, unofficial].
    *   Parallelize over rays using grid/block indices: `tid = blockIdx.x * blockDim.x + threadIdx.x` [S1, unofficial].
    *   Skip rays where `rays_mask` is false or `chunk_cnts[tid] == 0` on non-first passes [S1, unofficial].
    *   Set base indices for intersections: `base_hits = tid * n_grids`, `base_t_sorted = tid * n_grids * 2` [S1, unofficial].

2.  **Interval Extraction:**
    *   Loop over intersections `i` from `base_t_sorted` to `base_t_sorted + n_grids * 2 - 1` [S1, unofficial].
    *   Determine `is_entering = t_indices[i] < n_grids` and `level = t_indices[i] % n_grids` [S1, unofficial].
    *   Compute interval bounds:
        $$
        \text{this\_tmin} = \max(t_{\text{sorted}}[i], \text{near\_plane})
        $$
        $$
        \text{this\_tmax} = \min(t_{\text{sorted}}[i + 1], \text{far\_plane})
        $$
    *   Skip if `this_tmin >= this_tmax` (invalid/degenerate interval) [S1, unofficial].

3.  **Discontinuity Handling:**
    *   If `!continuous`, adjust `t_last` (the last processed sample position):
        *   If `step_size <= 0.0`, set `t_last = this_tmin` [S1, unofficial].
        *   Else, march forward using `dt = _calc_dt(t_last, cone_angle, step_size, 1e10f)` until `t_last + dt * 0.5f >= this_tmin` [S1, unofficial].
    *   This ensures samples are aligned with the start of the new grid interval [S1, unofficial].

4.  **Voxel Traversal Setup:**
    *   Setup traversal for the current AABB using `setup_traversal(ray, this_tmin, this_tmax, eps, aabb, resolution, delta, tdist, step_index, current_index, final_index)` [S1, unofficial].
    *   This step initializes the DDA state (step directions, deltas, max times) for the specific grid and interval [S1, unofficial].

5.  **Voxel Traversal Loop:**
    *   Perform standard DDA traversal within the interval `[this_tmin, this_tmax]` on the `binaries` grid [S1, unofficial].
    *   Record samples at `t_last + dt * 0.5f` (or similar midpoint) depending on `step_size` and `cone_angle` [S1, unofficial].
    *   Stop if `traverse_steps_limit` is reached [S1, unofficial].

6.  **Output Assembly:**
    *   Store intervals and samples in the `PackedRaySegmentsSpec` structures [S1, unofficial].
    *   Return `RayIntervals`, `RaySamples`, and `terminated_t` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_grids_kernel(rays_o, rays_d, binaries, aabbs, t_sorted, t_indices, hits,
                          near_plane, far_plane, step_size, cone_angle, traverse_steps_limit,
                          rays_mask, first_pass, eps):
    tid = blockIdx.x * blockDim.x + threadIdx.x
    if tid >= n_rays:
        return

    if not rays_mask[tid]:
        return

    if not first_pass and chunk_cnts[tid] == 0:
        return

    base_t_sorted = tid * n_grids * 2
    n_intervals = 0
    
    # Extract valid intervals from pre-sorted intersections
    for i in range(n_grids * 2):
        idx = base_t_sorted + i
        t_val = t_sorted[idx]
        grid_id = t_indices[idx] % n_grids
        
        # Determine if this is an entry or exit point
        # Note: t_indices maps to grid IDs. Even indices are entries, odd are exits?
        # Based on source: is_entering = t_indices[i] < n_grids
        # This implies t_indices stores grid IDs, and we pair them up.
        # The source logic suggests t_sorted contains pairs of (entry, exit) for each hit grid.
        
        # Simplified interval extraction based on source description:
        # t_sorted has 2*n_grids entries. We iterate and pair them.
        # However, the source says "Loop over intersections i... Determine is_entering...".
        # This implies we process the sorted list sequentially.
        
        if i % 2 == 0:
            this_tmin = max(t_val, near_plane[tid])
        else:
            this_tmax = min(t_val, far_plane[tid])
            
            if this_tmin >= this_tmax:
                continue
                
            # Handle discontinuity
            if not continuous:
                if step_size <= 0.0:
                    t_last = this_tmin
                else:
                    while t_last + calc_dt(t_last, cone_angle, step_size, 1e10) * 0.5 < this_tmin:
                        t_last += calc_dt(t_last, cone_angle, step_size, 1e10)
            
            # Setup DDA for this grid and interval
            # setup_traversal(...) initializes step_index, current_index, final_index, tdist, delta, etc.
            setup_traversal(rays_o[tid], rays_d[tid], this_tmin, this_tmax, eps, 
                            aabbs[grid_id], resolution, step_size, cone_angle)
            
            # Perform Voxel Traversal
            while current_index < final_index and (traverse_steps_limit == -1 or step_count < traverse_steps_limit):
                # Advance DDA
                advance_dda()
                
                # Sample
                sample_t = t_last + calc_dt(t_last, cone_angle, step_size, 1e10) * 0.5
                sample_pos = rays_o[tid] + sample_t * rays_d[tid]
                
                # Check occupancy
                if binaries[grid_id][voxel_index]:
                    store_sample(sample_pos, sample_t)
                
                t_last = sample_t
                step_count += 1
            
            n_intervals += 1

    return intervals, samples, terminated_t
```

## Subtopics

### [Ray Marching Particle Grid Traversal](5182-ray-marching-particle-grid-traversal.md)
A hybrid ray marching technique that traverses a voxel grid to find particle hits, using a low-resolution grid for culling and a high-resolution grid for precise hit testing. It sets itself apart from standard ray marching by introducing a two-tier grid hierarchy to optimize the search for sparse particle distributions, avoiding expensive hit tests in empty regions.

## Operating characteristics
*   **Time Complexity:** Depends on the number of grids hit and the number of voxels traversed within those grids. Pre-computing intersections reduces the overhead of checking every grid for every ray.
*   **Memory Complexity:** Requires storage for `t_sorted`, `t_indices`, `hits`, and the binary grids themselves. The output structures (`RayIntervals`, `RaySamples`) can grow large if many samples are generated.
*   **Performance:** Efficient for scenes with multiple disjoint objects or complex geometry represented by multiple grids. Avoids traversal through empty space.
*   **Strengths:** Handles multiple grids seamlessly; pre-computed intersections reduce per-ray overhead; supports variable step sizes and cone angles.
*   **Limitations:** Requires pre-computation of intersections; overlapping grids require careful ordering (first grid's occupancy overwrites second) [S1, unofficial]; discontinuous traversal requires alignment logic.
*   **When to Use:** Scenes with multiple disjoint objects, dynamic scenes with multiple grids, or when using occupancy grids for NeRF acceleration.
*   **When Not to Use:** Single uniform grid scenes where simpler DDA suffices; scenes with highly overlapping grids where ordering issues arise.

## Background
Multi-Grid Ray Traversal is part of the NerfAcc library, developed by Ruilong Li, Hang Gao, Matthew Tancik, and Angjoo Kanazawa [S1, unofficial]. It was introduced in the paper "NerfAcc: Efficient Sampling Accelerates NeRFs" (arXiv:2305.04966, 2023) [S1, unofficial]. The method is used in various NeRF acceleration pipelines, including TensoRF, T-NeRF, K-Planes, and TiNeuVox, often in conjunction with occupancy grid estimators [S1, unofficial]. It is integrated into frameworks like nerfstudio, sdfstudio, and threestudio [S1, unofficial].

## Sources
[S1] nerfstudio-project/nerfacc, https://github.com/nerfstudio-project/nerfacc, fetched 2026-10-26 (unofficial)
