# Ray-Bounding Box Intersection and Sampling Preparation

Also called: Ray Sampling with AABB Clipping and Cyclic Wrapping, Ray-Bounding Box Intersection and Sampling

## Overview
This method computes the intersection interval $[t_{min}, t_{max}]$ of camera rays with a scene bounding box (AABB), clamps these intervals to near/far clipping planes, and generates a set of query points along each ray for volume rendering [S1, unofficial][S2, unofficial][S5, unofficial]. It solves the problem of determining valid spatial extents for ray marching and preparing structured sample coordinates (including ray IDs and step indices) for downstream neural radiance field evaluation [S1, unofficial]. It differs from its sibling [Ray-Cube Intersection](2466-ray-cube-intersection.md) by supporting arbitrary axis-aligned bounding boxes rather than just cubes, and from [Branchless Ray/AABB Intersection with NaN Handling](3297-branchless-ray-aabb-intersection-with-nan-handling.md) by explicitly handling the generation of sample points and metadata (ray/step IDs) rather than just returning boolean intersection results [S1, unofficial][S5, unofficial].

## How it works
**Inputs:**
- `rays_o`: Ray origins, shape $(N, 3)$, float32.
- `rays_d`: Ray directions, shape $(N, 3)$, float32.
- `xyz_min`: Bounding box minimum corner, shape $(3)$, float32.
- `xyz_max`: Bounding box maximum corner, shape $(3)$, float32.
- `near`: Near clipping plane distance, float32 scalar.
- `far`: Far clipping plane distance, float32 scalar.
- `stepsize`: Number of voxels per sample step, float32 scalar.
- `voxel_size`: Physical size of a single voxel, float32 scalar.
- `bbox_cycle_xy`: Boolean flag enabling cyclic wrapping in X/Y axes [S1, unofficial].
- `randomize`: Boolean flag enabling uniform jitter within sample bins [S1, unofficial].

**Outputs:**
- `rays_pts`: Sampled query points, shape $(total\_len, 3)$, float32.
- `mask_outbbox`: Boolean mask for points outside the bounding box, shape $(total\_len)$.
- `ray_id`: Ray index for each point, shape $(total\_len)$, int32.
- `step_id`: Step index within each ray, shape $(total\_len)$, int32.
- `N_steps`: Number of samples per ray, shape $(N)$, int32.
- `t_min`: Clamped entry distance per ray, shape $(N)$, float32.
- `t_max`: Clamped exit distance per ray, shape $(N)$, float32.

**Data Structures:**
- Tensors are typically contiguous in memory. `rays_o` and `rays_d` are reshaped to $(-1, 3)$ if necessary [S5, unofficial].
- `ray_id` is constructed via cumulative sum of a marker array to assign ray indices to the flattened point list [S5, unofficial].

**Parameters:**
- `stepsize`: Typical values depend on scene scale; often 1-4 voxels [S1, unofficial][S5, unofficial].
- `near`: Typically $>0$, e.g., 0.2 or 1.0 [S2, unofficial][S5, unofficial].
- `far`: Typically large, e.g., $10^9$ or scene diameter [S5, unofficial].
- `voxel_size`: Determined by grid resolution and bounding box dimensions [S5, unofficial].

**Steps:**

1. **Preprocessing:**
   - Ensure `rays_o` and `rays_d` are contiguous and reshaped to $(-1, 3)$ [S5, unofficial].
   - Replace any zero components in `rays_d` with $10^{-6}$ to prevent division by zero during intersection calculation [S2, unofficial][S5, unofficial]. Let the modified direction be `vec`.

2. **Ray-AABB Intersection:**
   - Compute intermediate rates for each axis $i \in \{0, 1, 2\}$:
     $$ ax_i = \frac{xyz_{max}[i] - rays_o[i]}{vec[i]} $$
     $$ bx_i = \frac{xyz_{min}[i] - rays_o[i]}{vec[i]} $$
   - Compute unclamped $t_{min}$ and $t_{max}$ for each ray:
     $$ t_{min}^{raw} = \max(\min(ax_0, bx_0), \min(ax_1, bx_1), \min(ax_2, bx_2)) $$
     $$ t_{max}^{raw} = \min(\max(ax_0, bx_0), \max(ax_1, bx_1), \max(ax_2, bx_2)) $$
   - Clamp to near/far planes:
     $$ t_{min} = \max(\min(t_{min}^{raw}, far), near) $$
     $$ t_{max} = \max(\min(t_{max}^{raw}, far), near) $$
   - Note: If `bbox_cycle_xy` is enabled, $t_{min}$ is computed using only the Z-axis intersection rates [S1, unofficial].

3. **Sample Count Calculation:**
   - Compute step distance: $stepdist = stepsize \times voxel\_size$ [S5, unofficial].
   - Compute ray direction norm: $r_{norm} = \sqrt{rays_d[0]^2 + rays_d[1]^2 + rays_d[2]^2}$ [S2, unofficial].
   - Calculate number of samples per ray:
     $$ n_{samples} = \max\left(\left\lceil \frac{(t_{max} - t_{min}) \times r_{norm}}{stepdist} \right\rceil, 1\right) $$
   - Store $n_{samples}$ as $N_{steps}$ [S2, unofficial][S5, unofficial].

4. **Point Generation:**
   - Compute total number of points: $total\_len = \sum N_{steps}$ [S5, unofficial].
   - Compute ray start position: $rays_{start} = rays_o + rays_d \times t_{min}$ [S2, unofficial].
   - Compute normalized direction: $rays_{dir} = rays_d / r_{norm}$ [S2, unofficial].
   - Generate base step distances: $dist_k = stepdist \times k$ for $k = 0, \dots, n_{samples}-1$ [S5, unofficial].
   - If `randomize` is enabled, add uniform jitter:
     $$ jitter = \text{rand}() \times \frac{t_{max} - t_{min}}{n_{samples}} $$
     $$ dist_k = dist_k + jitter $$
   - Compute query points:
     $$ p = rays_{start} + rays_{dir} \times dist_k $$
   - If `bbox_cycle_xy` is enabled, wrap X and Y coordinates of $p$ around the AABB bounds [S1, unofficial].

5. **Masking and Metadata:**
   - Compute `mask_outbbox`: True if any coordinate of $p$ is outside $[xyz_{min}, xyz_{max}]$. If `bbox_cycle_xy` is enabled, check only Z-axis bounds [S1, unofficial][S5, unofficial].
   - Construct `ray_id`: Initialize array of zeros length $total\_len$. Set `ray_id[N_steps_cumsum[i-1]] = 1` for $i > 0$. Perform cumulative sum [S5, unofficial].
   - Construct `step_id`: For index $idx$, let $rid = ray_id[idx]$. Then $step\_id[idx] = idx - (N_{steps\_cumsum}[rid-1] \text{ if } rid \neq 0 \text{ else } 0)$ [S5, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def ray_bbox_intersection_and_sampling(
    rays_o: torch.Tensor,
    rays_d: torch.Tensor,
    xyz_min: torch.Tensor,
    xyz_max: torch.Tensor,
    near: float,
    far: float,
    stepsize: float,
    voxel_size: float,
    bbox_cycle_xy: bool = False,
    randomize: bool = False
) -> dict:
    """
    Compute ray-AABB intersection and generate sample points.
    
    Args:
        rays_o: (N, 3) ray origins
        rays_d: (N, 3) ray directions
        xyz_min: (3,) bounding box min
        xyz_max: (3,) bounding box max
        near: near clipping plane
        far: far clipping plane
        stepsize: voxels per step
        voxel_size: physical voxel size
        bbox_cycle_xy: enable cyclic wrapping in X/Y
        randomize: enable jitter
        
    Returns:
        Dictionary with rays_pts, mask_outbbox, ray_id, step_id, N_steps, t_min, t_max
    """
    # 1. Preprocessing
    N = rays_o.shape[0]
    rays_o = rays_o.contiguous().reshape(-1, 3)
    rays_d = rays_d.contiguous().reshape(-1, 3)
    
    # Replace zero direction components with 1e-6
    vec = torch.where(rays_d == 0, torch.full_like(rays_d, 1e-6), rays_d)
    
    # 2. Ray-AABB Intersection
    # ax = (xyz_max - rays_o) / vec
    # bx = (xyz_min - rays_o) / vec
    ax = (xyz_max.unsqueeze(0) - rays_o) / vec.unsqueeze(0)
    bx = (xyz_min.unsqueeze(0) - rays_o) / vec.unsqueeze(0)
    
    if bbox_cycle_xy:
        # Use only Z-axis for t_min calculation
        t_min_raw = torch.min(ax[:, 2], bx[:, 2])
    else:
        # Standard slab method
        t_min_raw = torch.max(torch.min(ax, bx), dim=1).values
        
    # t_max uses all axes
    t_max_raw = torch.min(torch.max(ax, bx), dim=1).values
    
    # Clamp to near/far
    t_min = torch.clamp(t_min_raw, min=near, max=far)
    t_max = torch.clamp(t_max_raw, min=near, max=far)
    
    # 3. Sample Count Calculation
    r_norm = torch.norm(rays_d, dim=1, keepdim=True)
    stepdist = stepsize * voxel_size
    
    # Avoid division by zero if r_norm is 0 (shouldn't happen with valid rays)
    r_norm_safe = torch.where(r_norm == 0, torch.full_like(r_norm, 1e-6), r_norm)
    
    n_samples_float = (t_max - t_min) * r_norm_safe.squeeze() / stepdist
    N_steps = torch.clamp(torch.ceil(n_samples_float).long(), min=1)
    
    # 4. Point Generation
    total_len = N_steps.sum().item()
    if total_len == 0:
        # Handle edge case where no samples are generated
        return {
            'rays_pts': torch.empty(0, 3),
            'mask_outbbox': torch.empty(0, dtype=torch.bool),
            'ray_id': torch.empty(0, dtype=torch.long),
            'step_id': torch.empty(0, dtype=torch.long),
            'N_steps': N_steps,
            't_min': t_min,
            't_max': t_max
        }
        
    rays_start = rays_o + rays_d * t_min.unsqueeze(1)
    rays_dir = rays_d / r_norm_safe
    
    # Generate step indices
    steps = torch.arange(0, N_steps.max().item(), device=rays_o.device)
    # Create mask for valid steps per ray
    valid_mask = steps.unsqueeze(0) < N_steps.unsqueeze(1)
    
    # Compute distances
    dists = steps.float() * stepdist
    
    if randomize:
        bin_size = (t_max - t_min) / N_steps.float().unsqueeze(1)
        jitter = torch.rand_like(dists) * bin_size
        dists = dists + jitter
        
    # Apply valid mask
    dists = dists * valid_mask.float()
    
    # Compute points
    # rays_start: (N, 3), rays_dir: (N, 3), dists: (N, max_steps)
    # Expand for broadcasting
    rays_start_exp = rays_start.unsqueeze(1) # (N, 1, 3)
    rays_dir_exp = rays_dir.unsqueeze(1)    # (N, 1, 3)
    dists_exp = dists.unsqueeze(2)          # (N, max_steps, 1)
    
    rays_pts_unmasked = rays_start_exp + rays_dir_exp * dists_exp # (N, max_steps, 3)
    
    # Flatten
    rays_pts = rays_pts_unmasked[valid_mask] # (total_len, 3)
    
    # 5. Masking and Metadata
    # Check bounds
    mask_x = (rays_pts[:, 0] < xyz_min[0]) | (rays_pts[:, 0] > xyz_max[0])
    mask_y = (rays_pts[:, 1] < xyz_min[1]) | (rays_pts[:, 1] > xyz_max[1])
    mask_z = (rays_pts[:, 2] < xyz_min[2]) | (rays_pts[:, 2] > xyz_max[2])
    
    if bbox_cycle_xy:
        mask_outbbox = mask_z
    else:
        mask_outbbox = mask_x | mask_y | mask_z
        
    # Cyclic wrapping
    if bbox_cycle_xy:
        range_x = xyz_max[0] - xyz_min[0]
        range_y = xyz_max[1] - xyz_min[1]
        rays_pts[:, 0] = xyz_min[0] + (rays_pts[:, 0] - xyz_min[0]) % range_x
        rays_pts[:, 1] = xyz_min[1] + (rays_pts[:, 1] - xyz_min[1]) % range_y
        
    # Ray IDs
    N_steps_cumsum = torch.cat([torch.zeros(1, dtype=torch.long, device=N_steps.device), N_steps.cumsum(0)[:-1]])
    ray_id = torch.zeros(total_len, dtype=torch.long, device=N_steps.device)
    # Mark start of each ray segment
    start_indices = N_steps_cumsum[1:] # Skip first as it's 0
    ray_id[start_indices] = 1
    ray_id = ray_id.cumsum(0)
    
    # Step IDs
    step_id = torch.arange(total_len, dtype=torch.long, device=N_steps.device)
    # Subtract offset for each ray
    offsets = N_steps_cumsum[ray_id]
    step_id = step_id - offsets
    
    return {
        'rays_pts': rays_pts,
        'mask_outbbox': mask_outbbox,
        'ray_id': ray_id,
        'step_id': step_id,
        'N_steps': N_steps,
        't_min': t_min,
        't_max': t_max
    }
```

## Subtopics

### NDC Bounding Box Computation
Computes the bounding box in Normalized Device Coordinates (NDC) space from a set of rays by calculating the minimum and maximum of ray origins and endpoints, then stacking them [S6, unofficial]. This is used when rendering in NDC space to define the volume extent for sampling [S6, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot S)$ where $N$ is the number of rays and $S$ is the average number of samples per ray. The intersection calculation is $O(N)$, and point generation is $O(N \cdot S)$.
- **Memory Complexity:** $O(N \cdot S)$ to store the generated points and metadata.
- **Performance:** Efficiently implemented in CUDA for batched processing, often splitting rays into batches (e.g., 10,000) to manage memory [S5, unofficial].
- **Strengths:** Handles arbitrary bounding boxes, supports cyclic boundary conditions for periodic scenes, and provides structured metadata (ray/step IDs) for efficient downstream processing [S1, unofficial][S5, unofficial].
- **Limitations:** Requires careful handling of zero-direction components to avoid division by zero [S5, unofficial]. The fixed step size may lead to over-sampling in sparse regions or under-sampling in dense regions if not combined with importance sampling [S1, unofficial].
- **When to use:** When rendering neural radiance fields with a defined bounding box, especially for unbounded scenes or those requiring periodic boundary conditions [S1, unofficial][S5, unofficial].
- **When not to use:** For unbounded scenes without a contracted space or bounding box, or when adaptive sampling is required [S1, unofficial].

## Background
This method is commonly used in neural radiance field (NeRF) implementations, particularly those dealing with unbounded scenes or specific bounding box constraints [S1, unofficial][S2, unofficial][S5, unofficial]. It builds on the standard slab method for ray-AABB intersection [Parent] and extends it to prepare samples for volume rendering [S1, unofficial]. Variants are found in projects like DirectVoxGO, UnboundedNeRFPytorch, and ref-dvgo [S1, unofficial][S2, unofficial][S5, unofficial].

## Sources
[S1] sjtuytc/UnboundedNeRFPytorch, https://github.com/sjtuytc/UnboundedNeRFPytorch, fetched 2026-10-26 (unofficial)
[S2] sunset1995/DirectVoxGO, https://github.com/sunset1995/DirectVoxGO, fetched 2026-10-26 (unofficial)
[S3] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
[S4] InanXR/FLASH, https://github.com/InanXR/FLASH, fetched 2026-10-27 (unofficial)
[S5] gkouros/ref-dvgo, https://github.com/gkouros/ref-dvgo, fetched 2026-10-28 (unofficial)
[S6] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
