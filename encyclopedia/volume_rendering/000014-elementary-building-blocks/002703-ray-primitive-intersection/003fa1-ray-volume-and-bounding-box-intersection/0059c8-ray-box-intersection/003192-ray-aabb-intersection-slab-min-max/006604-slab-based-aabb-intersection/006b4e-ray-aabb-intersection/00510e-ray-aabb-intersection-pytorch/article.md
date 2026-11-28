# Ray-AABB Intersection (PyTorch)

Also called: AABBBoxCollider, Axis-Aligned Bounding Box Collider

## Overview
A pure PyTorch implementation of ray-axis-aligned bounding box (AABB) intersection that computes near and far clipping distances for rays against a scene box [S2, unofficial]. It is functionally identical to the CUDA-accelerated version but executes slower [S1, unofficial]. It serves as a training-compatible intersection primitive in neural rendering pipelines where GPU kernel compilation is unavailable or dynamic shapes require eager execution [S2, unofficial].

## How it works
**Inputs and outputs:**
- `rays_o`: Ray origins, shape `[N, 3]` or `[H, W, 3]` [S3, unofficial].
- `rays_d`: Ray directions, shape `[N, 3]` or `[H, W, 3]` [S3, unofficial].
- `aabb_min`: AABB minimum bounds, shape `[3]` or `[N, 3]` [S1, unofficial].
- `aabb_max`: AABB maximum bounds, shape `[3]` or `[N, 3]` [S1, unofficial].
- `near_plane`: Scalar float, minimum clamp for the near distance (default 0.0 during inference) [S2, unofficial].
- **Output:** `nears` and `fars` tensors of shape `[N]` or `[H, W]` representing the valid sampling intervals [S3, unofficial].

**Data structures:**
The AABB is typically stored as two 3D vectors or a single 6D vector `[min_x, min_y, min_z, max_x, max_y, max_z]` [S1, unofficial]. In the broadcasting implementation, `aabb_min` and `aabb_max` are expanded to match the ray batch dimensions [S1, unofficial].

**Steps:**
1. **Reciprocal Direction:** Compute the reciprocal of the ray direction components to replace division with multiplication [S2, unofficial]. Add a small epsilon ($10^{-6}$) to avoid division by zero for rays parallel to axes [S2, unofficial].
   $$ dir\_frac = \frac{1.0}{rays\_d + 10^{-6}} $$
2. **Plane Intersections (Broadcasting Variant):** For each axis $i \in \{x, y, z\}$, compute the intersection distances $t_1$ (with min plane) and $t_2$ (with max plane) [S1, unofficial]:
   $$ t_1 = (aabb\_min - rays\_o) \cdot dir\_frac $$
   $$ t_2 = (aabb\_max - rays\_o) \cdot dir\_frac $$
3. **Min/Max Aggregation:** For each axis, determine the entry ($t_{min}^{(i)} = \min(t_1^{(i)}, t_2^{(i)})$) and exit ($t_{max}^{(i)} = \max(t_1^{(i)}, t_2^{(i)})$) distances [S1, unofficial][S2, unofficial].
4. **Global Interval:** Compute the global near ($t_{near}$) and far ($t_{far}$) bounds by taking the maximum of all entry distances and the minimum of all exit distances [S1, unofficial]:
   $$ t_{near} = \max_i(t_{min}^{(i)}) $$
   $$ t_{far} = \min_i(t_{max}^{(i)}) $$
5. **Hit Detection:** A ray hits the box if $t_{far} > t_{near}$ and $t_{far} > 0$ [S1, unofficial].
6. **Clamping:** Clamp $t_{near}$ to `near_plane` [S2, unofficial].
7. **Invalid Interval Handling:** If the ray misses, set $t_{near}$ and $t_{far}$ to a miss value (e.g., infinity or a specific flag) [S1, unofficial]. Ensure $t_{far} \ge t_{near} + 10^{-6}$ to prevent invalid zero-length intervals due to numerical precision [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_aabb_intersect_pytorch(rays_o, rays_d, aabb_min, aabb_max, near_plane=0.0):
    # Assumes torch is available in the environment
    # 1. Reciprocal direction with epsilon guard
    dir_frac = 1.0 / (rays_d + 1e-6)
    
    # 2. Compute t1 (min plane) and t2 (max plane) for each axis
    # Broadcasting handles batch dimensions automatically
    t1 = (aabb_min - rays_o) * dir_frac
    t2 = (aabb_max - rays_o) * dir_frac
    
    # 3. Determine min and max per axis
    t_min_axis = torch.minimum(t1, t2)
    t_max_axis = torch.maximum(t1, t2)
    
    # 4. Aggregate across axes (dim=-1)
    t_near = torch.max(t_min_axis, dim=-1).values
    t_far = torch.min(t_max_axis, dim=-1).values
    
    # 5. Hit detection
    hits = (t_far > t_near) & (t_far > 0)
    
    # 6. Clamp near to near_plane
    t_near = torch.clamp(t_near, min=near_plane)
    
    # 7. Handle misses and numerical stability
    miss_value = float('inf')
    t_near = torch.where(hits, t_near, miss_value)
    t_far = torch.where(hits, t_far, miss_value)
    
    # Ensure far >= near + epsilon to avoid invalid intervals
    t_far = torch.maximum(t_far, t_near + 1e-6)
    
    return t_near, t_far
```

**Parameters:**
- `near_plane`: Minimum distance for the near clip. Defaults to 0.0 during inference, but may be set higher during training to avoid sampling too close to the camera origin [S2, unofficial].

**Edge cases and pitfalls:**
- **Zero Direction:** Rays with zero direction components are handled by adding $10^{-6}$ to the denominator [S2, unofficial]. This approximates the ray as slightly offset, which may cause slight inaccuracies for rays exactly parallel to axes [S2, unofficial].
- **Misses:** Rays that do not intersect the box must be explicitly handled to prevent invalid sampling intervals [S1, unofficial]. The implementation sets $t_{near}$ and $t_{far}$ to a miss value (often infinity) and ensures $t_{far} \ge t_{near}$ [S1, unofficial][S2, unofficial].
- **Numerical Precision:** The check $t_{far} \ge t_{near} + 10^{-6}$ prevents degenerate intervals caused by floating-point errors when the ray grazes the box [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of rays [G]. The operations are element-wise and reduce across 3 dimensions [G].
- **Memory Complexity:** $O(N)$ for storing intermediate tensors ($t_1, t_2, t_{near}, t_{far}$) [G].
- **Performance:** Slower than the CUDA counterpart [S1, unofficial]. Suitable for training on CPU or when dynamic shapes prevent CUDA kernel compilation [S2, unofficial].
- **Strengths:** Pure Python/PyTorch, no external dependencies, supports autograd, handles dynamic batch sizes [G].
- **Limitations:** Slower than CUDA; not suitable for real-time inference on large datasets without batching hacks [G].

## Background
The implementation is part of the Nerfstudio project [S2, unofficial] and is also used in Taylor-NeRF [S1, unofficial]. It provides a software fallback for ray-AABB intersection, enabling neural rendering pipelines to run on hardware or configurations where CUDA kernels are not available or efficient [S1, unofficial]. It is part of the elementary building blocks for volume rendering, specifically within the ray-primitive intersection hierarchy [G].

## Sources
[S1] taylor2nerf/taylor-nerf, https://github.com/taylor2nerf/taylor-nerf, fetched 2026-10-26 (unofficial)
[S2] nerfstudio-project/nerfstudio, https://github.com/nerfstudio-project/nerfstudio, fetched 2026-10-26 (unofficial)
[S3] ashawkey/stable-dreamfusion, https://github.com/ashawkey/stable-dreamfusion, fetched 2026-10-26 (unofficial)
