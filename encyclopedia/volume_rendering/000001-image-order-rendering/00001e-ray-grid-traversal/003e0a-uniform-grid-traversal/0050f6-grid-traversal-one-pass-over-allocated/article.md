# Grid Traversal (One-Pass Over-Allocated)
Also called: traverse_grids (over-allocate)
## Overview
A single-pass algorithm to traverse voxel grids along rays, using pre-allocated maximum memory to avoid a second pass [S1, unofficial]. It differs from [Grid Traversal (Two-Pass)](#grid-traversal-two-pass) by allocating buffers large enough for the worst-case scenario upfront, trading memory for a single kernel launch, and from [Computational Space Traversal](10339-computational-space-traversal.md) by operating directly on grid coordinates rather than abstract computational spaces.
## How it works
**Inputs:** Ray origins, ray directions, grid dimensions, `traverse_steps_limit` (maximum steps per ray), and a ray mask.
**Outputs:** Truncated interval and sample buffers containing only valid traversal steps.

**Data Structures:**
1.  `intervals.chunk_cnts`: Tensor of size `[num_rays]` storing the actual number of valid intervals per ray.
2.  `intervals.data`: Pre-allocated buffer of size `[num_rays, traverse_steps_limit * 2]`. The factor of 2 accounts for potential entry/exit pairs or max steps depending on implementation specifics [S1, unofficial].
3.  `samples.chunk_cnts`: Tensor of size `[num_rays]` storing the actual number of valid samples per ray.
4.  `samples.data`: Pre-allocated buffer of size `[num_rays, traverse_steps_limit]` [S1, unofficial].

**Parameters:**
*   `traverse_steps_limit`: Integer > 0. The maximum number of steps allowed per ray [S1, unofficial].

**Algorithm:**
1.  Initialize `intervals.chunk_cnts` and `samples.chunk_cnts` to zero for all rays [S1, unofficial].
2.  Allocate `intervals.data` with shape `[num_rays, traverse_steps_limit * 2]` [S1, unofficial].
3.  Allocate `samples.data` with shape `[num_rays, traverse_steps_limit]` [S1, unofficial].
4.  Launch `traverse_grids_kernel` once with the over-allocated buffers. The kernel performs the DDA traversal, writing valid intervals and samples into the buffers and updating `chunk_cnts` with the actual count of valid entries per ray [S1, unofficial].
5.  Post-process: Truncate `intervals.data` and `samples.data` to the sizes specified in `intervals.chunk_cnts` and `samples.chunk_cnts` respectively, discarding unused pre-allocated space [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_grids_one_pass(origins, viewdirs, grid_dims, traverse_steps_limit, rays_mask):
    num_rays = origins.shape[0]
    
    # Step 1 & 2: Allocate buffers
    # intervals.data shape: [num_rays, traverse_steps_limit * 2]
    # samples.data shape: [num_rays, traverse_steps_limit]
    # chunk_cnts shape: [num_rays] initialized to 0
    intervals_data = torch.zeros((num_rays, traverse_steps_limit * 2), dtype=origins.dtype)
    samples_data = torch.zeros((num_rays, traverse_steps_limit), dtype=origins.dtype)
    intervals_cnts = torch.zeros(num_rays, dtype=torch.int32)
    samples_cnts = torch.zeros(num_rays, dtype=torch.int32)
    
    # Step 4: Single kernel launch
    # The kernel handles the DDA logic, boundary checks, and writes to the pre-allocated buffers
    traverse_grids_kernel(
        origins=origins,
        viewdirs=viewdirs,
        grid_dims=grid_dims,
        intervals_data=intervals_data,
        samples_data=samples_data,
        intervals_cnts=intervals_cnts,
        samples_cnts=samples_cnts,
        max_steps=traverse_steps_limit
    )
    
    # Step 5: Truncate based on actual counts
    # Note: In a real implementation, this might involve gathering or slicing
    final_intervals = []
    final_samples = []
    
    for i in range(num_rays):
        cnt_i = intervals_cnts[i].item()
        final_intervals.append(intervals_data[i, :cnt_i])
        
        cnt_s = samples_cnts[i].item()
        final_samples.append(samples_data[i, :cnt_s])
        
    return final_intervals, final_samples
```

**Edge Cases:**
*   If `traverse_steps_limit` is too small, rays may be truncated prematurely [S1, unofficial].
*   Precision issues near grid boundaries may cause rays to miss voxels or double-count; standard DDA epsilon handling is required [G].
## Subtopics

### Grid Traversal (Two-Pass)
A two-pass algorithm to traverse voxel grids along rays, first counting segments to allocate exact memory, then filling them [S1, unofficial]. It differs from the one-pass variant by using two kernel launches: the first pass counts the number of intervals/samples per ray, and the second pass allocates exact-sized buffers and fills them [S1, unofficial]. Choose this when memory is constrained and the average number of steps per ray is significantly lower than the maximum, avoiding the waste of over-allocation.

## Operating characteristics
**Time Complexity:** $O(N \cdot S)$, where $N$ is the number of rays and $S$ is the average number of steps per ray. The constant factor is lower than two-pass because it avoids the overhead of a second kernel launch and dynamic memory allocation, but it processes potentially empty slots in the buffer.
**Memory Complexity:** $O(N \cdot S_{max})$, where $S_{max}$ is `traverse_steps_limit`. This is the worst-case memory usage. It is significantly higher than the two-pass method if the scene is sparse or rays traverse few voxels.
**Performance:** Measured on standard GPU hardware (implied by PyTorch usage in sources [S1, unofficial]). Performance scales linearly with ray count. Strengths include simplicity of implementation and reduced kernel launch overhead. Limitations include high memory consumption for sparse scenes or large grids. Use when memory is abundant and latency is critical; avoid when memory is tight or scenes are very sparse.
## Background
Originated in the context of neural radiance fields (NeRF) rendering optimizations, specifically within the `taylor-nerf` repository [S1, unofficial]. It is part of a family of grid traversal methods designed to efficiently sample volumetric data for volume rendering.
## Sources
[S1] taylor2nerf/taylor-nerf, https://github.com/taylor2nerf/taylor-nerf, fetched 2026-10-26 (unofficial)
