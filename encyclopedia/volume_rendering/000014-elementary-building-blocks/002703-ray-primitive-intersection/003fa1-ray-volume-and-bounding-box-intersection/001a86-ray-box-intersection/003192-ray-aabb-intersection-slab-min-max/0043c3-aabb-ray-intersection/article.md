# AABB Ray Intersection

Also called: Batched AABB Ray Intersection, GPU Kernel AABB Intersection, Chunked Ray-Box Intersection

## Overview
This method performs parallel ray-axis-aligned bounding box (AABB) intersection tests for volume rendering, specifically designed to handle large-scale ray-voxel queries on GPUs without exceeding memory limits. It differs from standard single-ray implementations by batching rays into chunks and processing them via a CUDA kernel that outputs intersection depths ($t_{min}, t_{max}$) and hit status for multiple rays and AABBs simultaneously [S1, S2, unofficial]. It is chosen when the number of rays and voxels is too large for a single contiguous memory allocation, requiring explicit chunking to stay within hardware constraints (e.g., <2GB VRAM) [S1, unofficial].

## How it works
**Inputs:**
- `ray_start`: Tensor of shape $(N, 3)$, ray origins.
- `ray_dir`: Tensor of shape $(N, 3)$, ray directions (normalized).
- `points`: Tensor of shape $(M, 3)$, centers of $M$ AABBs.
- `voxelsize`: Scalar, side length of each cubic AABB.
- `near`, `far`: Scalars, clipping planes for the ray segment.
- `n_max`: Integer, maximum number of AABBs to check per ray (optional, context-dependent).

**Outputs:**
- `inds`: Tensor of shape $(N, K)$, indices of intersected AABBs.
- `min_depth`: Tensor of shape $(N, K)$, $t_{min}$ for each hit.
- `max_depth`: Tensor of shape $(N, K)$, $t_{max}$ for each hit.

**Data Structures:**
The core intersection logic uses a slab-based approach. For an AABB centered at $c$ with size $s$, the bounds are $[c_i - s/2, c_i + s/2]$ for axis $i \in \{x, y, z\}$. The ray is defined as $O + tD$. The intersection interval $[t_{min}, t_{max}]$ is computed by intersecting the ray with the slab defined by each axis pair and taking the intersection of these intervals.

**Parameters:**
- Batch size limit $G$: Typically capped at 2048 or calculated dynamically to fit memory [S1, unofficial].
- Memory limit: 2 GB (used to derive $G$) [S1, unofficial].

**Steps:**
1. **Batching Calculation:**
   Calculate the maximum batch size $G$ to prevent out-of-memory errors:
   $$ G = \min\left(2048, \left\lfloor \frac{2 \times 10^9}{\text{points.numel()}} \right\rfloor \right) $$
   where `points.numel()` is the total number of elements in the points tensor [S1, unofficial].

2. **Chunking:**
   Let $N$ be the number of rays and $S$ be the ray batch size (often 1 or the number of rays per frame). Calculate the number of chunks $K$:
   $$ K = \left\lceil \frac{N}{G} \right\rceil $$
   Calculate the padded size $H = K \times G$.

3. **Padding:**
   If $H > N$, pad `ray_start` and `ray_dir` by repeating the first $H - N$ rays to reach size $H$ [S1, unofficial].

4. **Reshaping:**
   Reshape `ray_start` and `ray_dir` to shape $(S \times G, K, 3)$ to prepare for batched processing. Expand `points` to match the batch dimension $S \times G$ if necessary [S1, unofficial].

5. **Kernel Execution:**
   Call the CUDA extension `_ext.aabb_intersect` with the reshaped rays, points, `voxelsize`, and `n_max`. The kernel processes rays in parallel [S1, S2, unofficial].

6. **Post-processing:**
   Reshape outputs `inds`, `min_depth`, `max_depth` to $(S, H, -1)$. If padding was applied, truncate the results to the original $N$ rays. Mark outputs as non-differentiable [S1, unofficial].

**Pseudocode (Python-like):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def batched_aabb_intersect(ray_start, ray_dir, points, voxelsize, near=0.0, far=1e10):
    """
    Intersects rays against AABBs centered at points.
    ray_start: (N, 3)
    ray_dir: (N, 3)
    points: (M, 3)
    voxelsize: float
    """
    N = ray_start.shape[0]
    # Step 1: Calculate batch size G
    # Avoid division by zero if points is empty
    if points.numel() == 0:
        G = 2048
    else:
        G = min(2048, int(2 * 10**9 / points.numel()))
    
    # Step 2: Calculate chunks
    K = int(np.ceil(N / G))
    H = K * G
    
    # Step 3: Pad rays if necessary
    if H > N:
        pad_size = H - N
        # Repeat the first ray to fill padding
        repeat_indices = torch.cat([torch.arange(N), torch.zeros(pad_size, dtype=torch.long)])
        # Actually, we need to repeat the *first* ray for the padding? 
        # Source says "repeating the first H-N rays". This phrasing is ambiguous.
        # Usually, padding repeats the last or first. Let's assume repeating the first ray for simplicity,
        # or replicating the batch. The source says "repeating the first H-N rays".
        # If H-N > N, this is impossible. Assuming H-N <= N.
        # Let's interpret as: append copies of ray 0 to ray_start/_dir.
        ray_start = torch.cat([ray_start, ray_start[0:1].expand(pad_size, -1)], dim=0)
        ray_dir = torch.cat([ray_dir, ray_dir[0:1].expand(pad_size, -1)], dim=0)
        
    # Step 4: Reshape
    # Assuming S=1 for simplicity, or S is a batch dimension.
    # Source: Reshape to (S * G, K, 3). Let S=1.
    ray_start = ray_start.view(H, 1, 3).permute(1, 0, 2) # (1, H, 3) -> (1, K, G, 3) ?
    # Source says (S * G, K, 3). Let's stick to the source's reshape logic.
    # If S=1, shape is (G, K, 3).
    ray_start = ray_start.view(G, K, 3)
    ray_dir = ray_dir.view(G, K, 3)
    
    # Expand points to match batch dimension G
    # points is (M, 3). We need to broadcast or expand.
    # The kernel likely handles broadcasting or expects points to be expanded.
    # Let's assume the kernel takes points (M, 3) and rays (G, K, 3).
    
    # Step 5: Call CUDA kernel
    # inds, min_depth, max_depth = _ext.aabb_intersect(ray_start, ray_dir, points, voxelsize, n_max)
    # For the sake of the article, we describe the logic inside the kernel.
    
    # Kernel Logic (Parallel over tid):
    # n_rays = G * K
    # n_aabbs = M
    # for tid in range(0, n_rays * n_aabbs, blockDim.x * gridDim.x):
    #     ray_id = tid // n_aabbs
    #     aabb_id = tid % n_aabbs
    #     o = ray_start[ray_id // K, ray_id % K, :] # Flatten index
    #     d = ray_dir[ray_id // K, ray_id % K, :]
    #     center = points[aabb_id]
    #     half_size = voxelsize / 2.0
    #     min_bound = center - half_size
    #     max_bound = center + half_size
    #     
    #     # Slab intersection
    #     t1 = (min_bound - o) / d
    #     t2 = (max_bound - o) / d
    #     t_min_axis = torch.min(t1, t2)
    #     t_max_axis = torch.max(t1, t2)
    #     
    #     # Intersection of slabs
    #     t_min = torch.max(torch.max(t_min_axis))
    #     t_max = torch.min(torch.min(t_max_axis))
    #     
    #     hit = (t_min <= t_max) and (t_max >= near) and (t_min <= far)
    #     
    #     if hit:
    #         t_mins[tid] = t_min
    #         t_maxs[tid] = t_max
    #     else:
    #         t_mins[tid] = miss_value
    #         t_maxs[tid] = miss_value
    #     hits[tid] = hit
    
    # Step 6: Reshape outputs
    # inds, min_depth, max_depth = ... # From kernel
    # min_depth = min_depth.view(1, H, -1)
    # max_depth = max_depth.view(1, H, -1)
    
    # Step 7: Truncate
    # min_depth = min_depth[:, :N, :]
    # max_depth = max_depth[:, :N, :]
    
    return min_depth, max_depth
```

**Edge Cases and Pitfalls:**
- **Out-of-Memory:** Large numbers of points can cause memory overflow if not batched. The batching hack $G$ is critical [S1, unofficial].
- **Precision:** Division by zero in ray direction components must be handled (typically by checking for zero and using large epsilon or swapping axes) [G].
- **Degenerate Input:** Empty point sets or zero-sized voxels require guards [G].

## Operating characteristics
- **Time Complexity:** $O(N \cdot M)$ for $N$ rays and $M$ AABBs, but parallelized on GPU. The batching adds a factor of $K = \lceil N/G \rceil$.
- **Memory Complexity:** $O(G \cdot M)$ for the active batch, where $G$ is limited by memory constraints (e.g., 2GB) [S1, unofficial].
- **Performance:** Highly efficient on GPUs due to parallelization over rays and AABBs. Performance scales with the number of CUDA cores and memory bandwidth [S2, unofficial].
- **Strengths:** Handles large-scale ray-voxel queries; prevents OOM errors; parallelizable.
- **Limitations:** Requires CUDA extension; batching introduces overhead; not differentiable by default [S1, unofficial].

## Background
This method is derived from implementations in neural volume rendering frameworks, specifically the unofficial implementation for NeRF and NSVF (Neural Scene Volumes) [S1, unofficial]. It addresses the scalability issues of ray-voxel intersection in large scenes by introducing a batching mechanism [S1, unofficial]. The parallel kernel structure is common in GPU-based ray tracing libraries [S2, unofficial].

## Sources
[S1] facebookresearch/NSVF, https://github.com/facebookresearch/NSVF, fetched 2026-10-27 (unofficial)
[S2] YihangChen-ee/CNC, https://github.com/YihangChen-ee/CNC, fetched 2026-10-27 (unofficial)
