# Multi-Grid Ray Traversal

Also called: Grid Traversal (Multi-Grid), Ray traversal through multiple voxel grids

## Overview
Multi-Grid Ray Traversal traverses rays through multiple binary occupancy grids to generate ray intervals, sample positions, and termination planes [S1, S2, S3, unofficial]. It solves the problem of efficiently sampling hierarchical or overlapping volumetric structures by pre-computing ray-AABB intersections to determine which grids are hit, sorting these entry/exit points, and then performing voxel-level traversal within the valid intervals [S2, S3, unofficial]. Unlike single-grid traversal, this method handles multiple grid levels (mipmaps) and overlapping bounding boxes, where the first grid's occupancy status takes precedence in overlapping regions [S2, S3, unofficial]. It builds on [Multi-level Grid Structures](6354-multi-level-grid-structures.md) by explicitly managing the intersection events across distinct grid instances rather than just hierarchical pointers [S2, S3, unofficial].

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
    *   `continuous`: Boolean flag. If `false`, adjusts `t_last` to align with interval boundaries [S2, unofficial].
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
    *   If `t_sorted`, `t_indices`, or `hits` are `None`, compute ray-AABB intersections to get `t_mins`, `t_maxs`, and `hits` [S3, unofficial].
    *   Concatenate `t_mins` and `t_maxs` along the last dimension and sort them to get `t_sorted` and `t_indices` [S2, S3, unofficial].

2.  **Parallel Execution Setup:**
    *   Parallelize over rays using grid/block indices: `tid = blockIdx.x * blockDim.x + threadIdx.x` [S2, unofficial].
    *   Skip rays where `rays_mask` is `false` or `chunk_cnts[tid] == 0` on non-first passes [S2, unofficial].
    *   Initialize ray spec with origins, directions, near/far planes [S2, unofficial].
    *   Set base indices for intersections: `base_hits = tid * n_grids`, `base_t_sorted = tid * n_grids * 2` [S2, unofficial].

3.  **Interval Processing:**
    *   Loop over intersection pairs `i` from `base_t_sorted` to `base_t_sorted + n_grids * 2 - 1` with step 2 [S2, unofficial].
    *   Determine `is_entering = t_indices[i] < n_grids` and `level = t_indices[i] % n_grids` [S2, unofficial].
    *   Compute interval bounds:
        $$this\_tmin = \max(t\_sorted[i], near\_plane)$$
        $$this\_tmax = \min(t\_sorted[i + 1], far\_plane)$$
    *   Skip if `this_tmin >= this_tmax` to prevent degenerate marching [S2, unofficial].

4.  **Step Size Adjustment (Discontinuous Traversal):**
    *   If `!continuous`, adjust `t_last`:
        *   If `step_size <= 0.0`, set `t_last = this_tmin` [S2, unofficial].
        *   Else, march forward using `dt = _calc_dt(t_last, cone_angle, step_size, 1e10)` until `t_last + dt * 0.5 >= this_tmin` [S2, unofficial].

5.  **Voxel Traversal Setup:**
    *   Setup traversal for the current AABB using `setup_traversal(ray, this_tmin, this_tmax, eps, aabb, resolution, delta, tdist, step_index, current_index, final_index)` with `eps = 1e-6` [S2, unofficial].

6.  **Kernel Execution:**
    *   Call the C++ kernel `traverse_grids` with the prepared inputs, setting `compute_intervals`, `compute_samples`, and `compute_terminate_planes` to `True` [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from typing import List, Tuple, Optional

def ray_aabb_intersect(
    rays_o: List[List[float]], 
    rays_d: List[List[float]], 
    aabbs: List[List[float]]
) -> Tuple[List[List[float]], List[List[float]], List[List[bool]]]:
    """Standard slab method for ray-AABB intersection. Returns t_near, t_far, hit flags."""
    n_rays = len(rays_o)
    n_grids = len(aabbs)
    t_mins = [[0.0] * n_grids for _ in range(n_rays)]
    t_maxs = [[0.0] * n_grids for _ in range(n_rays)]
    hits = [[False] * n_grids for _ in range(n_rays)]
    
    for r in range(n_rays):
        ox, oy, oz = rays_o[r]
        dx, dy, dz = rays_d[r]
        for g in range(n_grids):
            xmin, ymin, zmin, xmax, ymax, zmax = aabbs[g]
            t1 = (xmin - ox) / dx if dx != 0 else float('inf')
            t2 = (xmax - ox) / dx if dx != 0 else float('-inf')
            t3 = (ymin - oy) / dy if dy != 0 else float('inf')
            t4 = (ymax - oy) / dy if dy != 0 else float('-inf')
            t5 = (zmin - oz) / dz if dz != 0 else float('inf')
            t6 = (zmax - oz) / dz if dz != 0 else float('-inf')
            
            t_near = max(min(t1, t2), min(t3, t4), min(t5, t6))
            t_far = min(max(t1, t2), max(t3, t4), max(t5, t6))
            
            if t_near <= t_far and t_far >= 0.0:
                t_mins[r][g] = t_near if t_near > 0.0 else 0.0
                t_maxs[r][g] = t_far
                hits[r][g] = True
            else:
                t_mins[r][g] = float('inf')
                t_maxs[r][g] = float('inf')
                hits[r][g] = False
    return t_mins, t_maxs, hits

def sort_intersections(t_mins: List[List[float]], t_maxs: List[List[float]]) -> Tuple[List[List[float]], List[List[int]]]:
    """Concatenate t_mins and t_maxs, sort, and track original grid indices."""
    n_rays = len(t_mins)
    n_grids = len(t_mins[0])
    t_sorted = [[0.0] * (n_grids * 2) for _ in range(n_rays)]
    t_indices = [[0] * (n_grids * 2) for _ in range(n_rays)]
    
    for r in range(n_rays):
        # Create pairs: (value, grid_id, is_entry)
        events = []
        for g in range(n_grids):
            events.append((t_mins[r][g], g, True))
            events.append((t_maxs[r][g], g, False))
        events.sort(key=lambda x: x[0])
        
        for i, (val, gid, _) in enumerate(events):
            t_sorted[r][i] = val
            t_indices[r][i] = gid
    return t_sorted, t_indices

def calc_dt(t_last: float, cone_angle: float, step_size: float, max_dt: float) -> float:
    """Compute adaptive step size based on cone angle and distance."""
    if cone_angle <= 0.0:
        return step_size
    dt = step_size * (1.0 + cone_angle * t_last)
    return min(dt, max_dt)

def setup_traversal(
    ray_origin: List[float], ray_dir: List[float], 
    t_min: float, t_max: float, eps: float, 
    aabb: List[float], resolution: Tuple[int, int, int], 
    delta: float
) -> dict:
    """Initialize DDA traversal state for a specific grid interval."""
    xmin, ymin, zmin, xmax, ymax, zmax = aabb
    resx, resy, resz = resolution
    # Compute voxel step sizes
    dx = (xmax - xmin) / resx
    dy = (ymax - ymin) / resy
    dz = (zmax - zmin) / resz
    
    # Initial voxel coordinates
    ix = int((ray_origin[0] - xmin) / dx)
    iy = int((ray_origin[1] - ymin) / dy)
    iz = int((ray_origin[2] - zmin) / dz)
    
    # Direction signs
    step_x = 1 if ray_dir[0] > 0 else -1
    step_y = 1 if ray_dir[1] > 0 else -1
    step_z = 1 if ray_dir[2] > 0 else -1
    
    # T-max for crossing planes
    t_max_x = (xmin + (ix + (1 if step_x > 0 else 0)) * dx - ray_origin[0]) / ray_dir[0] if ray_dir[0] != 0 else float('inf')
    t_max_y = (ymin + (iy + (1 if step_y > 0 else 0)) * dy - ray_origin[1]) / ray_dir[1] if ray_dir[1] != 0 else float('inf')
    t_max_z = (zmin + (iz + (1 if step_z > 0 else 0)) * dz - ray_origin[2]) / ray_dir[2] if ray_dir[2] != 0 else float('inf')
    
    return {
        "ix": ix, "iy": iy, "iz": iz,
        "step_x": step_x, "step_y": step_y, "step_z": step_z,
        "t_max_x": t_max_x, "t_max_y": t_max_y, "t_max_z": t_max_z,
        "dx": dx, "dy": dy, "dz": dz,
        "t_min": t_min, "t_max": t_max, "eps": eps
    }

def traverse_grids_kernel(
    rays_o: List[List[float]], rays_d: List[List[float]], binaries: List, aabbs: List[List[float]],
    t_sorted: Optional[List[List[float]]] = None, t_indices: Optional[List[List[int]]] = None, hits: Optional[List[List[bool]]] = None,
    near_planes: Optional[List[float]] = None, far_planes: Optional[List[float]] = None,
    step_size: float = 1e-3, cone_angle: float = 0.0,
    traverse_steps_limit: int = -1, over_allocate: bool = False,
    rays_mask: Optional[List[bool]] = None, first_pass: bool = True, continuous: bool = True
) -> Tuple[List, List, List]:
    """Multi-Grid Ray Traversal Kernel. Self-contained Python implementation."""
    n_rays = len(rays_o)
    n_grids = len(aabbs)
    
    # 1. Initialization
    if rays_mask is None:
        rays_mask = [True] * n_rays
    if near_planes is None:
        near_planes = [0.0] * n_rays
    if far_planes is None:
        far_planes = [float('inf')] * n_rays
    if traverse_steps_limit is None:
        traverse_steps_limit = -1
        
    # Compute intersections if not provided
    if t_sorted is None or t_indices is None or hits is None:
        t_mins, t_maxs, hits = ray_aabb_intersect(rays_o, rays_d, aabbs)
        t_sorted, t_indices = sort_intersections(t_mins, t_maxs)
        
    # Output buffers
    intervals = []
    samples = []
    terminate_planes = [0.0] * n_rays
    
    # 2. Parallel Execution Setup (simulated sequentially for clarity)
    for tid in range(n_rays):
        if not rays_mask[tid]:
            continue
        if not first_pass:
            # chunk_cnts[tid] would be checked here in CUDA
            pass
            
        base_t_sorted = tid * n_grids * 2
        t_last = 0.0
        
        # 3. Interval Processing
        for i in range(base_t_sorted, base_t_sorted + n_grids * 2 - 1, 2):
            level = t_indices[tid][i] % n_grids
            this_tmin = max(t_sorted[tid][i], near_planes[tid])
            this_tmax = min(t_sorted[tid][i + 1], far_planes[tid])
            
            # Skip invalid intervals
            if this_tmin >= this_tmax:
                continue
                
            # 4. Step Size Adjustment
            if not continuous:
                if step_size <= 0.0:
                    t_last = this_tmin
                else:
                    dt = calc_dt(t_last, cone_angle, step_size, 1e10)
                    while t_last + dt * 0.5 < this_tmin:
                        t_last += dt
                        dt = calc_dt(t_last, cone_angle, step_size, 1e10)
                        
            # 5. Voxel Traversal Setup
            state = setup_traversal(
                ray_origin=rays_o[tid], ray_dir=rays_d[tid],
                t_min=this_tmin, t_max=this_tmax, eps=1e-6,
                aabb=aabbs[level], resolution=binaries[level].shape, delta=step_size
            )
            
            # 6. Perform Voxel Traversal (simplified loop)
            step_count = 0
            current_t = this_tmin
            while current_t < this_tmax and (traverse_steps_limit <= 0 or step_count < traverse_steps_limit):
                # Advance DDA to next voxel crossing
                if state["t_max_x"] < state["t_max_y"] and state["t_max_x"] < state["t_max_z"]:
                    current_t = state["t_max_x"]
                    state["ix"] += state["step_x"]
                    state["t_max_x"] += state["dx"]
                elif state["t_max_y"] < state["t_max_z"]:
                    current_t = state["t_max_y"]
                    state["iy"] += state["step_y"]
                    state["t_max_y"] += state["dy"]
                else:
                    current_t = state["t_max_z"]
                    state["iz"] += state["step_z"]
                    state["t_max_z"] += state["dz"]
                    
                # Record sample
                sample_pos = [
                    rays_o[tid][0] + current_t * rays_d[tid][0],
                    rays_o[tid][1] + current_t * rays_d[tid][1],
                    rays_o[tid][2] + current_t * rays_d[tid][2]
                ]
                samples.append(sample_pos)
                intervals.append((this_tmin, this_tmax))
                step_count += 1
                
            terminate_planes[tid] = this_tmax
            
    return intervals, samples, terminate_planes
```

## Subtopics

### [Ray Marching Particle Grid Traversal](5182-ray-marching-particle-grid-traversal.md)
A hybrid technique using a low-resolution grid for culling and a high-resolution grid for precise hit testing in dense particle systems. It skips large empty regions by querying a coarser occupancy grid first.

### [Multi-Grid Ray Traversal with Adaptive Step Size](14341-multi-grid-ray-traversal-with-adaptive-step-size.md)
A CUDA kernel that handles overlapping grids by priority and outputs traversal intervals with adaptive step sizing based on cone angle. It explicitly manages multiple hierarchical grid levels with sorted intersection events.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of rays and the number of grids, $O(N_{rays} \cdot N_{grids})$ for intersection testing, plus $O(N_{samples})$ for voxel traversal within valid intervals [S2, S3, unofficial].
*   **Memory Complexity:** $O(N_{rays} \cdot N_{grids})$ for storing intersection data (`t_sorted`, `t_indices`, `hits`) [S2, S3, unofficial]. Output buffers scale with the number of samples per ray [S2, unofficial].
*   **Measured Performance:** Used in NeRF acceleration libraries like NerfAcc to speed up training on NeRF-Synthetic and Tanks&Temple datasets [S2, unofficial].
*   **Strengths:** Efficiently handles multiple grid levels and overlapping regions [S2, S3, unofficial]. Pre-computing intersections reduces per-ray overhead during traversal [S2, S3, unofficial]. Adaptive step sizing allows for coarser sampling in empty space or distant regions [S3, unofficial].
*   **Limitations:** Not differentiable to any inputs [S3, unofficial]. Overlapping grids require careful ordering as the first grid's occupancy status overwrites the second's [S2, unofficial]. Requires careful handling of step size alignment when entering new intervals [S2, unofficial].
*   **When to use:** When rendering scenes with multiple distinct volumetric grids (e.g., mipmaps, separate objects) or when hierarchical occupancy grids are used for acceleration [S2, S3, unofficial].
*   **When not to use:** When differentiability is required for the traversal step [S3, unofficial]. When grids are not axis-aligned or have complex shapes not representable by AABBs [S2, unofficial].

## Background
Multi-Grid Ray Traversal is a core component of efficient NeRF acceleration libraries, specifically implemented in `nerfacc` [S2, unofficial]. It is used in conjunction with occupancy grid estimators to skip empty space during rendering [S2, unofficial]. The method is also referenced in other NeRF-related projects like `taylor-nerf` [S1, unofficial] and `CNC` [S3, unofficial]. It addresses the computational bottleneck of ray marching through dense volumetric data by leveraging hierarchical structures and pre-computed intersections [S2, unofficial].

## Sources
[S1] taylor2nerf/taylor-nerf, https://github.com/taylor2nerf/taylor-nerf, fetched 2026-10-26 (unofficial)
[S2] nerfstudio-project/nerfacc, https://github.com/nerfstudio-project/nerfacc, fetched 2026-10-26 (unofficial)
[S3] YihangChen-ee/CNC, https://github.com/YihangChen-ee/CNC, fetched 2026-10-27 (unofficial)
