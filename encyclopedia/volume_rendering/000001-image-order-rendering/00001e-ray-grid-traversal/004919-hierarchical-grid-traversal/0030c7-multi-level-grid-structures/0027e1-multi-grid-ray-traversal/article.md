# Multi-Grid Ray Traversal

Also called: Grid Traversal (Multi-Grid), Ray traversal through multiple voxel grids

## Overview
Multi-Grid Ray Traversal traverses rays through multiple binary occupancy grids to generate ray intervals, sample positions, and termination planes [S1, S2, S3, unofficial]. It solves the problem of efficiently sampling hierarchical or overlapping volumetric structures by pre-computing ray-AABB intersections to determine which grids are hit, sorting these entry/exit points, and then performing voxel-level traversal within the valid intervals [S2, S3, unofficial]. Unlike single-grid traversal, this method handles multiple grid levels (mipmaps) and overlapping bounding boxes, where the first grid's occupancy status takes precedence in overlapping regions [S2, S3, unofficial]. It builds on [Multi-level Grid Structures](6354-multi-level-grid-structures.md) by explicitly managing the intersection events across distinct grid instances rather than just hierarchical pointers.

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `rays_o`: Ray origins, shape `[n_rays, 3]` [S2, S3, unofficial].
    *   `rays_d`: Ray directions, shape `[n_rays, 3]` [S2, S3, unofficial].
    *   `binaries`: Binary occupancy grids, shape `[n_grids, resx, resy, resz]` [S2, S3, unofficial]. All grids share the same resolution [S2, S3, unofficial].
    *   `aabbs`: Axis-aligned bounding boxes for the grids, shape `[n_grids, 6]` storing `{xmin, ymin, zmin, xmax, ymax, zmax}` [S1, S3, unofficial].
    *   `t_sorted` (optional): Pre-computed sorted intersection times, shape `[n_rays, n_grids * 2]` [S2, S3, unofficial].
    *   `t_indices` (optional): Pre-computed sorted t indices, shape `[n_rays, n_grids * 2]` mapping intersections to grid IDs [S2, S3, unofficial].
    *   `hits` (optional): Pre-computed hit flags, shape `[n_rays, n_grids]` [S2, S3, unofficial].
    *   `near_planes` (optional): Per-ray near plane distances, shape `[n_rays]` [S3, unofficial].
    *   `far_planes` (optional): Per-ray far plane distances, shape `[n_rays]` [S3, unofficial].
    *   `step_size`: Step size for ray traversal. Default `1e-3` [S2, S3, unofficial].
    *   `cone_angle`: Cone angle for linearly-increased step size. `0.0` means constant step size. Default `0.0` [S1, S2, S3, unofficial].
    *   `traverse_steps_limit`: Maximum number of samples per ray. Default `-1` (unlimited) [S2, S3, unofficial].
    *   `over_allocate`: Whether to over-allocate memory for outputs. If `True`, `traverse_steps_limit` must be set > 0 [S1, S2, S3, unofficial].
    *   `rays_mask` (optional): Mask for valid rays, shape `[n_rays]` [S2, S3, unofficial].
    *   `first_pass`: Boolean flag. If `false`, skips rays with `chunk_cnts[tid] == 0` [S2, unofficial].
*   **Outputs:**
    *   `intervals`: Ray segment intervals [S2, S3, unofficial].
    *   `samples`: Sample positions within each interval [S2, S3, unofficial].
    *   `terminate_planes`: Termination t values for each ray, shape `[n_rays]` [S2, S3, unofficial].

**Data Structures:**
*   **PackedRaySegmentsSpec:** Stores traversal intervals and sample positions per ray, supporting chunked memory layouts via `chunk_cnts` and `chunk_starts` [S2, unofficial].
*   **RayIntervals:** Object containing traversal intervals [S2, unofficial].
*   **RaySamples:** Object containing samples within each interval [S2, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   If `rays_mask` is `None`, initialize it to all-ones boolean tensor with shape `[n_rays]` [S3, unofficial].
    *   If `near_planes` is `None`, initialize it to zeros with shape `[n_rays]` [S3, unofficial].
    *   If `far_planes` is `None`, initialize it to infinity with shape `[n_rays]` [S3, unofficial].
    *   If `traverse_steps_limit` is `None`, set it to `-1` [S3, unofficial].
    *   If `t_sorted`, `t_indices`, or `hits` are `None`, compute ray-AABB intersections using `ray_aabb_intersect(rays_o, rays_d, aabbs)` to get `t_mins`, `t_maxs`, and `hits` [S3, unofficial].
    *   Concatenate `t_mins` and `t_maxs` along the last dimension and sort them to get `t_sorted` and `t_indices` [S2, S3, unofficial].

2.  **Parallel Execution Setup:**
    *   Parallelize over rays using grid/block indices: `tid = blockIdx.x * blockDim.x + threadIdx.x` [S2, unofficial].
    *   Skip rays where `rays_mask` is `false` or `chunk_cnts[tid] == 0` on non-first passes [S2, unofficial].
    *   Initialize ray spec with origins, directions, near/far planes [S2, unofficial].
    *   Set base indices for intersections: `base_hits = tid * n_grids`, `base_t_sorted = tid * n_grids * 2` [S2, unofficial].

3.  **Interval Processing:**
    *   Loop over intersections `i` from `base_t_sorted` to `base_t_sorted + n_grids * 2 - 1` [S2, unofficial].
    *   Determine `is_entering = t_indices[i] < n_grids` and `level = t_indices[i] % n_grids` [S2, unofficial].
    *   Compute interval bounds:
        $$this\_tmin = \max(t\_sorted[i], near\_plane)$$
        $$this\_tmax = \min(t\_sorted[i + 1], far\_plane)$$
    *   Skip if `this_tmin >= this_tmax` to prevent degenerate marching [S2, unofficial].

4.  **Step Size Adjustment (Discontinuous Traversal):**
    *   If `!continuous`, adjust `t_last`:
        *   If `step_size <= 0.0f`, set `t_last = this_tmin` [S2, unofficial].
        *   Else, march forward using `dt = _calc_dt(t_last, cone_angle, step_size, 1e10f)` until `t_last + dt * 0.5f >= this_tmin` [S2, unofficial].

5.  **Voxel Traversal Setup:**
    *   Setup traversal for the current AABB using `setup_traversal(ray, this_tmin, this_tmax, eps, aabb, resolution, delta, tdist, step_index, current_index, final_index)` with `eps = 1e-6f` [S2, unofficial].

6.  **Kernel Execution:**
    *   Call the C++ kernel `traverse_grids` with the prepared inputs, setting `compute_intervals`, `compute_samples`, and `compute_terminate_planes` to `True` [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_grids_kernel(rays_o, rays_d, binaries, aabbs, 
                          t_sorted=None, t_indices=None, hits=None,
                          near_planes=None, far_planes=None,
                          step_size=1e-3, cone_angle=0.0,
                          traverse_steps_limit=-1, over_allocate=False,
                          rays_mask=None, first_pass=True):
    """
    Multi-Grid Ray Traversal Kernel.
    """
    n_rays = rays_o.shape[0]
    n_grids = binaries.shape[0]
    
    # 1. Initialization
    if rays_mask is None:
        rays_mask = torch.ones(n_rays, dtype=torch.bool)
    if near_planes is None:
        near_planes = torch.zeros(n_rays)
    if far_planes is None:
        far_planes = torch.full((n_rays,), float('inf'))
    if traverse_steps_limit is None:
        traverse_steps_limit = -1
        
    # Compute intersections if not provided
    if t_sorted is None or t_indices is None or hits is None:
        t_mins, t_maxs, hits = ray_aabb_intersect(rays_o, rays_d, aabbs)
        # Concatenate and sort
        t_all = torch.cat([t_mins, t_maxs], dim=-1) # Shape: [n_rays, n_grids * 2]
        # Note: t_indices must track which grid each t belongs to
        # This is a simplified representation; actual implementation requires
        # sorting while keeping track of original grid indices and entry/exit flags
        t_sorted, t_indices = sort_with_indices(t_all)
        
    # 2. Parallel Execution Setup
    tid = blockIdx.x * blockDim.x + threadIdx.x
    if tid >= n_rays:
        return
        
    if not rays_mask[tid]:
        return
        
    if not first_pass:
        if chunk_cnts[tid] == 0:
            return
            
    # Base indices
    base_hits = tid * n_grids
    base_t_sorted = tid * n_grids * 2
    
    # 3. Interval Processing
    # Iterate through sorted intersection points
    # Note: In a real kernel, this loop is unrolled or handled via warp-level primitives
    for i in range(base_t_sorted, base_t_sorted + n_grids * 2 - 1):
        # Determine if entering or exiting
        # t_indices[i] contains the grid ID. 
        # Convention: even indices in sorted list are entries, odd are exits?
        # Or t_indices explicitly flags entry/exit. 
        # Based on notes: is_entering = t_indices[i] < n_grids
        is_entering = t_indices[i] < n_grids
        level = t_indices[i] % n_grids
        
        # Compute interval bounds
        this_tmin = max(t_sorted[i], near_planes[tid])
        this_tmax = min(t_sorted[i + 1], far_planes[tid])
        
        # Skip invalid intervals
        if this_tmin >= this_tmax:
            continue
            
        # 4. Step Size Adjustment
        # Assuming t_last is maintained per ray in shared memory or registers
        if not continuous:
            if step_size <= 0.0:
                t_last = this_tmin
            else:
                # March forward to align with this_tmin
                dt = calc_dt(t_last, cone_angle, step_size, 1e10)
                while t_last + dt * 0.5 < this_tmin:
                    t_last += dt
                    dt = calc_dt(t_last, cone_angle, step_size, 1e10)
                    
        # 5. Voxel Traversal Setup
        # Setup traversal for the current AABB
        # This involves initializing DDA variables for the specific grid 'level'
        setup_traversal(
            ray_origin=rays_o[tid],
            ray_dir=rays_d[tid],
            t_min=this_tmin,
            t_max=this_tmax,
            eps=1e-6,
            aabb=aabbs[level],
            resolution=binaries[level].shape,
            delta=step_size,
            # ... other traversal state variables
        )
        
        # 6. Perform Voxel Traversal
        # Iterate through voxels within [this_tmin, this_tmax]
        # Record samples and intervals
        # Respect traverse_steps_limit
        
    # 7. Output Results
    # Write to intervals, samples, terminate_planes buffers
```

## Subtopics

### [Ray Marching Particle Grid Traversal](5182-ray-marching-particle-grid-traversal.md)
A hybrid technique using a low-resolution grid for culling and a high-resolution grid for precise hit testing in dense particle systems. It skips large empty regions by querying a coarser occupancy grid first.

### [Multi-Grid Ray Traversal with Adaptive Step Size](14341-multi-grid-ray-traversal-with-adaptive-step-size.md)
A CUDA kernel that handles overlapping grids by priority and outputs traversal intervals with adaptive step sizing based on cone angle. It explicitly manages multiple hierarchical grid levels with sorted intersection events.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of rays and the number of grids, $O(N_{rays} \cdot N_{grids})$ for intersection testing, plus $O(N_{samples})$ for voxel traversal within valid intervals [S2, S3, unofficial].
*   **Memory Complexity:** $O(N_{rays} \cdot N_{grids})$ for storing intersection data (`t_sorted`, `t_indices`, `hits`) [S2, S3, unofficial]. Output buffers scale with the number of samples per ray.
*   **Measured Performance:** Used in NeRF acceleration libraries like NerfAcc to speed up training on NeRF-Synthetic and Tanks&Temple datasets [S2, unofficial].
*   **Strengths:** Efficiently handles multiple grid levels and overlapping regions. Pre-computing intersections reduces per-ray overhead during traversal. Adaptive step sizing allows for coarser sampling in empty space or distant regions.
*   **Limitations:** Not differentiable to any inputs [S3, unofficial]. Overlapping grids require careful ordering as the first grid's occupancy status overwrites the second's [S2, unofficial]. Requires careful handling of step size alignment when entering new intervals [S2, unofficial].
*   **When to use:** When rendering scenes with multiple distinct volumetric grids (e.g., mipmaps, separate objects) or when hierarchical occupancy grids are used for acceleration.
*   **When not to use:** When differentiability is required for the traversal step [S3, unofficial]. When grids are not axis-aligned or have complex shapes not representable by AABBs.

## Background
Multi-Grid Ray Traversal is a core component of efficient NeRF acceleration libraries, specifically implemented in `nerfacc` [S2, unofficial]. It is used in conjunction with occupancy grid estimators to skip empty space during rendering [S2, unofficial]. The method is also referenced in other NeRF-related projects like `taylor-nerf` [S1, unofficial] and `CNC` [S3, unofficial]. It addresses the computational bottleneck of ray marching through dense volumetric data by leveraging hierarchical structures and pre-computed intersections.

## Sources
[S1] taylor-nerf, https://github.com/taylor2nerf/taylor-nerf, fetched 2026-10-26 (unofficial)
[S2] nerfacc, https://github.com/nerfstudio-project/nerfacc, fetched 2026-10-26 (unofficial)
[S3] CNC, https://github.com/YihangChen-ee/CNC, fetched 2026-10-27 (unofficial)
