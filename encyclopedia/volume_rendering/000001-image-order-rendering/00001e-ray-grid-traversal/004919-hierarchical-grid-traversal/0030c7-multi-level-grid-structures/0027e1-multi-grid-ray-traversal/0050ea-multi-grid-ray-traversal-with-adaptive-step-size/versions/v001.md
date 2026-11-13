# Multi-Grid Ray Traversal with Adaptive Step Size


## Overview
This method is a CUDA kernel that traverses rays through one or more axis-aligned bounding box (AABB) voxel grids, handling overlapping grids by priority and outputting traversal intervals and sample points with adaptive step sizing based on cone angle [S1, unofficial]. It differs from [Ray Marching Particle Grid Traversal](5182-ray-marching-particle-grid-traversal.md) by explicitly managing multiple hierarchical grid levels with sorted intersection events, ensuring that the first grid's occupancy status takes precedence in overlapping regions, whereas particle grids typically rely on density accumulation without explicit grid-level priority [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   `rays_o`: Ray origins, shape `[n_rays, 3]` [S1, unofficial].
*   `rays_d`: Ray directions, shape `[n_rays, 3]` [S1, unofficial].
*   `rays_mask`: Optional boolean mask, shape `[n_rays]`, to skip rays [S1, unofficial].
*   `binaries`: Voxel occupancy data, shape `[n_grids, resx, resy, resz]` [S1, unofficial].
*   `aabbs`: AABB bounds for each grid, shape `[n_grids, 6]` (min_x, min_y, min_z, max_x, max_y, max_z) [S1, unofficial].
*   `hits`: Pre-computed hit flags, shape `[n_rays, n_grids]` [S1, unofficial].
*   `t_sorted`: Sorted intersection distances, shape `[n_rays, n_grids * 2]` [S1, unofficial].
*   `t_indices`: Indices indicating which grid and whether the intersection is entering or leaving, shape `[n_rays, n_grids * 2]` [S1, unofficial].
*   `near_planes`: Near plane distances, shape `[n_rays]` [S1, unofficial].
*   `far_planes`: Far plane distances, shape `[n_rays]` [S1, unofficial].
*   `traverse_steps_limit`: Maximum number of samples per ray (optional) [S1, unofficial].
*   `step_size`: Fixed step size. If `<= 0.0`, variable step mode is used [S1, unofficial].
*   `cone_angle`: Factor for adaptive step size increase. Must be `>= 0.0`. Value `0.0` means uniform marching [S1, unofficial].
*   `resolution`: Grid resolution `[resx, resy, resz]` [S1, unofficial].
*   `delta`: Voxel size [S1, unofficial].
*   `eps`: Epsilon for float comparisons [S1, unofficial].

**Outputs:**
*   `intervals`: `PackedRaySegmentsSpec` structure containing interval endpoints [S1, unofficial].
*   `samples`: `PackedRaySegmentsSpec` structure containing sample points [S1, unofficial].
*   `terminate_planes`: Termination planes per ray, shape `[n_rays]` [S1, unofficial].

**Data Structures:**
*   `PackedRaySegmentsSpec`:
    *   `vals`: Float array of interval endpoints or sample positions [S1, unofficial].
    *   `ray_indices`: Integer array mapping entries to ray IDs [S1, unofficial].
    *   `is_left`: Boolean array marking left endpoints of intervals [S1, unofficial].
    *   `is_right`: Boolean array marking right endpoints of intervals [S1, unofficial].
    *   `chunk_cnts`: Integer array, optional, count of segments per ray [S1, unofficial].
    *   `chunk_starts`: Integer array, optional, start index per ray [S1, unofficial].

### Parameters
*   `MAX_GRID_LEVELS`: Constant defining the maximum number of grid levels supported. Value is `8` [S1, unofficial].
*   `dt_min`: Minimum step size [S1, unofficial].
*   `dt_max`: Maximum step size. Default value in code is `1e10f` [S1, unofficial].

### Algorithm Steps
1.  **Initialization**: Launch a CUDA kernel with a grid-stride loop over rays:
    ```python
    for tid in range(blockIdx.x * blockDim.x + threadIdx.x, n_rays, blockDim.x * gridDim.x):
    ```
2.  **Masking**: If `rays_mask` is provided and `rays_mask[tid]` is false, skip the ray [S1, unofficial].
3.  **Optimization Check**: If not the first pass, check `intervals.chunk_cnts[tid] == 0` or `samples.chunk_cnts[tid] == 0`. If true, skip the ray [S1, unofficial].
4.  **Base Indices**: Calculate base indices for intersection data:
    ```python
    base_hits = tid * n_grids
    base_t_sorted = tid * n_grids * 2
    ```
5.  **Intersection Loop**: Iterate through sorted intersections:
    ```python
    for i in range(base_t_sorted, base_t_sorted + n_grids * 2 - 1):
    ```
6.  **Intersection Type**: Determine if entering or leaving:
    ```python
    is_entering = t_indices[i] < n_grids
    level = t_indices[i] % n_grids
    ```
7.  **Hit Check**: If `!hits[base_hits + level]`, continue to next intersection [S1, unofficial].
8.  **Interval Bounds**: Calculate valid interval bounds:
    ```python
    this_tmin = max(t_sorted[i], near_planes[tid])
    this_tmax = min(t_sorted[i + 1], far_planes[tid])
    ```
    If `this_tmin >= this_tmax`, skip this interval [S1, unofficial].
9.  **Ray Advancement**: Advance ray to `this_tmin`.
    *   If `step_size <= 0.0` (variable step), set `t_last = this_tmin`.
    *   If `step_size > 0.0` (fixed step), march in steps using `_calc_dt` until `t_last + dt * 0.5f >= this_tmin` [S1, unofficial].
10. **Traversal Setup**: Call `setup_traversal` with ray, `this_tmin`, `this_tmax`, `eps`, `aabb`, `resolution`, `delta`, `tdist`, `step_index`, `current_index`, `final_index` [S1, unofficial].
11. **Cell Traversal Loop**: While `traverse_steps_limit <= 0` or `n_samples < traverse_steps_limit`:
    a. Calculate distance to next cell boundary:
    ```python
    t_traverse = min(tdist.x, min(tdist.y, tdist.z))
    t_traverse = min(t_traverse, this_tmax)
    ```
    b. Calculate cell ID:
    ```python
    cell_id = (current_index.x * resolution.y * resolution.z + 
               current_index.y * resolution.z + 
               current_index.z + 
               level * resolution.x * resolution.y * resolution.z)
    ```
    c. **Empty Cell**: If `!binaries[cell_id]`:
    *   Advance `t_last` to `t_traverse` (variable step) or march using `_calc_dt` (fixed step).
    *   Set `continuous = false` [S1, unofficial].
    d. **Occupied Cell**:
    *   Loop to generate samples until `t_next >= t_traverse`.
    *   Calculate `t_next` using variable step (`t_traverse`) or fixed step (`t_last + dt`) [S1, unofficial].
    e. **Write Intervals**:
    *   If not continuous, write left endpoint at `t_last` and right at `t_next`.
    *   If continuous, write right endpoint at `t_next` and mark previous as left.
    *   Increment `n_intervals` [S1, unofficial].
    f. **Write Samples**:
    *   Write sample point to `samples.vals` at midpoint: `(t_next + t_last) * 0.5f`.
    *   Mark as valid.
    *   Increment `n_samples` [S1, unofficial].

### Formulas
*   **Step Size Calculation**:
    $$ dt = \text{clamp}(t \cdot \text{cone\_angle}, dt_{min}, dt_{max}) $$
    Where $t$ is the current position along the ray [S1, unofficial].

### Edge Cases and Pitfalls
*   **Overlapping Grids**: The code assumes grid order matters. The first grid's occupancy status overwrites the second if they overlap. This is implemented by iterating through sorted intersections and checking `hits` and `is_entering` logic to prioritize earlier grids [S1, unofficial].
*   **Invalid Intervals**: Intervals where `this_tmin >= this_tmax` (e.g., 0.0 to 0.0) are skipped to prevent degenerate rendering [S1, unofficial].
*   **Precision**: Use of `eps` in `setup_traversal` is critical for avoiding self-intersection artifacts in ray-grid traversal [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of rays and the number of intersections per ray. The inner loop complexity depends on the number of cells traversed within each AABB intersection.
*   **Memory Complexity**: Linear with respect to the number of rays and the number of output intervals/samples. The input data structures scale with `n_grids` and `resolution`.
*   **Performance**: Optimized for CUDA execution with grid-stride loops and early ray termination. Performance is heavily dependent on the sparsity of the `binaries` grid and the efficiency of the intersection sorting.
*   **Strengths**: Handles multiple grid levels and overlapping grids explicitly. Adaptive step size allows for efficient traversal in sparse regions.
*   **Limitations**: Requires pre-computed intersections (`t_sorted`, `t_indices`). Performance may degrade with highly dense grids or many overlapping grids.
*   **When to Use**: Suitable for neural radiance field rendering and volume rendering where multiple hierarchical grids are used and overlapping regions need explicit priority handling.
*   **When Not to Use**: Not suitable for single-grid, uniform sampling scenarios where simpler ray marching is sufficient.

## Background
The method originates from the Taylor-NeRF project, an unofficial implementation of neural radiance fields using Taylor series expansions for implicit functions [S1, unofficial]. It is designed to work with the tiny-cuda-nn library for efficient CUDA kernel execution [S1, unofficial]. The method is part of a broader effort to improve the efficiency and accuracy of neural rendering by leveraging hierarchical grid structures and adaptive sampling.

## Sources
[S1] taylor2nerf/taylor-nerf, https://github.com/taylor2nerf/taylor-nerf, fetched 2026-10-26 (unofficial)
