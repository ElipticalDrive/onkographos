# Ray-Bounding Box Intersection and Sampling Preparation

Also called: Ray Sampling with AABB Clipping and Cyclic Wrapping, Ray-Bounding Box Intersection and Sampling

## Overview
This method computes the intersection interval of camera rays with a scene bounding box, determines the number of sample points per ray based on a fixed step distance, and generates query points along the rays for volume rendering [S1, unofficial][S5, unofficial]. It bridges the gap between ray geometry and voxel grid lookups by returning sampled points, their associated ray and step IDs, and a mask for points outside the bounding box [S1, unofficial]. It differs from its parent, [Ray-AABB Intersection (Slab/Min-Max)](6548-ray-aabb-intersection-slab-min-max.md), by adding the discretization of the intersection interval into discrete sample points and the generation of metadata (ray/step IDs) required for batched rendering [S5, unofficial]. It differs from [Ray Point Sampling](2382-ray-point-sampling.md) by handling the initial bounding box clipping and interval calculation before point generation [S1, unofficial].

## How it works
**Inputs:**
- `rays_o`: Ray origins, shape $(N, 3)$, float32.
- `rays_d`: Ray directions, shape $(N, 3)$, float32.
- `xyz_min`: Bounding box minimum corner, shape $(3)$, float32.
- `xyz_max`: Bounding box maximum corner, shape $(3)$, float32.
- `near`: Near clipping plane distance, float32 scalar.
- `far`: Far clipping plane distance, float32 scalar.
- `stepsize`: Number of voxels per sample step, float32 scalar.
- `voxel_size`: Physical size of a voxel, float32 scalar.
- `bbox_cycle_xy`: Boolean flag for periodic boundary conditions in X/Y (optional) [S3, unofficial].

**Outputs:**
- `rays_pts`: Sampled points, shape $(total\_len, 3)$, float32.
- `mask_outbbox`: Boolean mask for points outside the bounding box, shape $(total\_len)$.
- `ray_id`: Ray index for each point, shape $(total\_len)$, int32.
- `step_id`: Step index within each ray, shape $(total\_len)$, int32.
- `N_steps`: Number of samples per ray, shape $(N)$, int32.
- `t_min`: Entry distance for each ray, shape $(N)$, float32.
- `t_max`: Exit distance for each ray, shape $(N)$, float32.

**Data Structures:**
- Tensors are typically contiguous in memory. `rays_o` and `rays_d` are reshaped to $(-1, 3)$ if necessary [S5, unofficial].
- `ray_id` and `step_id` are computed using cumulative sums to map flat point indices back to ray/step coordinates [S5, unofficial].

**Steps:**

1.  **Preprocessing:**
    - Ensure `rays_o` and `rays_d` are contiguous [S1, unofficial].
    - Replace any zero components in `rays_d` with $10^{-6}$ to avoid division by zero [S2, unofficial][S5, unofficial]. Let the modified direction be `vec`.
    - Compute the norm of the direction vector for each ray: $r_{norm} = \sqrt{vec_x^2 + vec_y^2 + vec_z^2}$ [S2, unofficial][S5, unofficial].

2.  **Ray-AABB Intersection:**
    - Compute intermediate intersection rates for the max and min bounds of the bounding box along each axis [S5, unofficial]:
      $$
      ax = \frac{xyz_{max}[0] - rays_o[0]}{vec_x}, \quad bx = \frac{xyz_{min}[0] - rays_o[0]}{vec_x}
      $$
      $$
      ay = \frac{xyz_{max}[1] - rays_o[1]}{vec_y}, \quad by = \frac{xyz_{min}[1] - rays_o[1]}{vec_y}
      $$
      $$
      az = \frac{xyz_{max}[2] - rays_o[2]}{vec_z}, \quad bz = \frac{xyz_{min}[2] - rays_o[2]}{vec_z}
      $$
    - Calculate the entry ($t_{min}$) and exit ($t_{max}$) distances [S2, unofficial][S5, unofficial]:
      $$
      t_{min} = \max(\min(ax, bx), \min(ay, by), \min(az, bz))
      $$
      $$
      t_{max} = \min(\max(ax, bx), \max(ay, by), \max(az, bz))
      $$
    - Clamp the interval to the near and far planes [S2, unofficial][S5, unofficial]:
      $$
      t_{min} = \max(\min(t_{min}, far), near)
      $$
      $$
      t_{max} = \max(\min(t_{max}, far), near)
      $$
    - If `bbox_cycle_xy` is enabled, compute $t_{min}$ using only the Z-axis intersection rates [S3, unofficial].

3.  **Sample Count Calculation:**
    - Compute the physical step distance: $stepdist = stepsize \times voxel\_size$ [S1, unofficial][S5, unofficial].
    - Calculate the number of samples per ray [S2, unofficial][S5, unofficial]:
      $$
      n_{samples} = \max\left(\left\lceil \frac{(t_{max} - t_{min}) \times r_{norm}}{stepdist} \right\rceil, 1\right)
      $$
    - Ensure at least one sample per ray [S5, unofficial].

4.  **Point Generation:**
    - Compute the total number of points: $total\_len = \sum n_{samples}$ [S5, unofficial].
    - Compute the cumulative sum of $n_{samples}$: $N_{steps\_cumsum}$ [S5, unofficial].
    - Initialize `ray_id` array of size $total\_len$ with zeros. Set `ray_id[N_steps_cumsum[i-1]] = 1` for $i > 0$. Perform an in-place cumulative sum to assign the correct ray index to each point [S5, unofficial].
    - Compute `step_id` for each point index $idx$:
      $$
      rid = ray_id[idx]
      $$
      $$
      step\_id[idx] = idx - (N_{steps\_cumsum}[rid-1] \text{ if } rid \neq 0 \text{ else } 0)
      $$
    - Compute the ray start position and normalized direction [S2, unofficial][S5, unofficial]:
      $$
      rays_{start} = rays_o + rays_d \times t_{min}
      $$
      $$
      rays_{dir} = \frac{rays_d}{r_{norm}}
      $$
    - Generate query points [S5, unofficial]:
      $$
      dist = stepdist \times step\_id[idx]
      $$
      $$
      p = rays_{start}[i_{ray}] + rays_{dir}[i_{ray}] \times dist
      $$
    - If randomization is enabled, add uniform jitter within each bin: $jitter = \text{torch.rand\_like}(t_{samples}) \times bin\_size$, where $bin\_size = (t_{max} - t_{min}) / num\_samples$ [S4, unofficial].

5.  **Masking and Wrapping:**
    - Check if points are outside the bounding box [S5, unofficial]:
      $$
      mask_{outbbox}[idx] = (xyz_{min}[0] > p[0]) \lor (xyz_{min}[1] > p[1]) \lor (xyz_{min}[2] > p[2]) \lor (xyz_{max}[0] < p[0]) \lor (xyz_{max}[1] < p[1]) \lor (xyz_{max}[2] < p[2])
      $$
    - If `bbox_cycle_xy` is enabled, wrap X and Y coordinates of `rays_pts` around the AABB bounds to simulate periodic boundary conditions [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def ray_bbox_intersection_and_sampling(rays_o, rays_d, xyz_min, xyz_max, near, far, stepsize, voxel_size, bbox_cycle_xy=False):
    """
    Compute ray-bounding box intersection and sample points.
    
    Args:
        rays_o: (N, 3) Ray origins.
        rays_d: (N, 3) Ray directions.
        xyz_min: (3,) Bounding box minimum.
        xyz_max: (3,) Bounding box maximum.
        near: float Near clipping plane.
        far: float Far clipping plane.
        stepsize: float Number of voxels per sample step.
        voxel_size: float Physical size of a voxel.
        bbox_cycle_xy: bool Enable cyclic wrapping in X/Y.
        
    Returns:
        rays_pts: (total_len, 3) Sampled points.
        mask_outbbox: (total_len,) Boolean mask for out-of-bounds points.
        ray_id: (total_len,) Ray indices.
        step_id: (total_len,) Step indices.
        N_steps: (N,) Number of samples per ray.
        t_min: (N,) Entry distances.
        t_max: (N,) Exit distances.
    """
    # 1. Preprocessing
    # Ensure contiguous
    rays_o = rays_o.contiguous()
    rays_d = rays_d.contiguous()
    
    # Replace zero direction components with 1e-6
    vec = torch.where(rays_d == 0, torch.tensor(1e-6, device=rays_d.device), rays_d)
    
    # Compute direction norm
    r_norm = torch.norm(vec, dim=-1, keepdim=True) # (N, 1)
    
    # 2. Ray-AABB Intersection
    # Compute rates
    ax = (xyz_max[0] - rays_o[:, 0]) / vec[:, 0]
    bx = (xyz_min[0] - rays_o[:, 0]) / vec[:, 0]
    ay = (xyz_max[1] - rays_o[:, 1]) / vec[:, 1]
    by = (xyz_min[1] - rays_o[:, 1]) / vec[:, 1]
    az = (xyz_max[2] - rays_o[:, 2]) / vec[:, 2]
    bz = (xyz_min[2] - rays_o[:, 2]) / vec[:, 2]
    
    if bbox_cycle_xy:
        # Use only Z-axis for t_min if cyclic XY is enabled
        t_min = torch.maximum(torch.minimum(az, bz), torch.tensor(near, device=rays_o.device))
        t_min = torch.minimum(t_min, torch.tensor(far, device=rays_o.device))
        # t_max calculation might still need full bbox or specific logic not fully detailed in sources for cyclic case
        # Assuming standard t_max for now, but sources imply t_min change
        t_max = torch.minimum(torch.maximum(ax, bx), torch.tensor(far, device=rays_o.device))
        t_max = torch.maximum(t_max, torch.tensor(near, device=rays_o.device))
        # Note: Source [S3, unofficial] mentions t_min change, but doesn't explicitly detail t_max change for cyclic. 
        # Standard implementation usually keeps t_max full or handles cyclic differently.
        # For completeness, we stick to the explicit t_min instruction.
        # Re-reading [S3, unofficial]: "compute t_min using only the Z-axis... Otherwise, compute t_min by calculating intersection rates for all axes"
        # It does not specify t_max change. We will use standard t_max.
        t_max = torch.minimum(torch.maximum(torch.maximum(torch.maximum(ax, bx), ay), az), torch.tensor(far, device=rays_o.device))
        t_max = torch.maximum(t_max, torch.tensor(near, device=rays_o.device))
    else:
        # Standard Slab method
        t_min = torch.maximum(torch.maximum(torch.maximum(torch.minimum(ax, bx), torch.minimum(ay, by)), torch.minimum(az, bz)), torch.tensor(near, device=rays_o.device))
        t_min = torch.minimum(t_min, torch.tensor(far, device=rays_o.device))
        
        t_max = torch.minimum(torch.minimum(torch.minimum(torch.maximum(ax, bx), torch.maximum(ay, by)), torch.maximum(az, bz)), torch.tensor(far, device=rays_o.device))
        t_max = torch.maximum(t_max, torch.tensor(near, device=rays_o.device))
        
    # Clamp t_min and t_max to [near, far] again to be safe (sources do this explicitly)
    t_min = torch.clamp(t_min, min=near, max=far)
    t_max = torch.clamp(t_max, min=near, max=far)
    
    # 3. Sample Count Calculation
    stepdist = stepsize * voxel_size
    # Interval length in physical space
    interval_len = (t_max - t_min) * r_norm.squeeze(-1) # (N,)
    n_samples = torch.ceil(interval_len / stepdist)
    N_steps = torch.maximum(n_samples, torch.tensor(1.0, device=rays_o.device)).int()
    
    # 4. Point Generation
    total_len = N_steps.sum().item()
    N_steps_cumsum = torch.cat([torch.tensor([0], device=rays_o.device), torch.cumsum(N_steps[:-1], dim=0)]) # (N,)
    
    # Initialize ray_id
    ray_id = torch.zeros(total_len, dtype=torch.int32, device=rays_o.device)
    # Mark starts
    for i in range(1, len(N_steps)):
        ray_id[N_steps_cumsum[i]] = 1
    ray_id = torch.cumsum(ray_id, dim=0)
    
    # Compute step_id
    # step_id[idx] = idx - (N_steps_cumsum[ray_id[idx]-1] if ray_id[idx]!=0 else 0)
    # This requires gathering from N_steps_cumsum based on ray_id
    # Handle ray_id=0 case (first ray)
    indices = torch.arange(total_len, device=rays_o.device)
    rid = ray_id
    # Get previous cumulative sum
    # If rid > 0, prev_cumsum = N_steps_cumsum[rid-1]
    # If rid == 0, prev_cumsum = 0
    prev_cumsum = torch.where(rid > 0, N_steps_cumsum[rid - 1], torch.tensor(0, device=rays_o.device))
    step_id = indices - prev_cumsum
    
    # Compute ray start and direction
    rays_start = rays_o + rays_d * t_min.unsqueeze(1) # (N, 3)
    rays_dir = vec / r_norm # (N, 3)
    
    # Generate points
    # dist = stepdist * step_id
    dist = stepdist * step_id.float().unsqueeze(1) # (total_len, 1)
    
    # Gather rays_start and rays_dir based on ray_id
    pts = rays_start[ray_id] + rays_dir[ray_id] * dist # (total_len, 3)
    
    # 5. Masking
    # Check bounds
    mask_outbbox = (pts[:, 0] < xyz_min[0]) | (pts[:, 1] < xyz_min[1]) | (pts[:, 2] < xyz_min[2]) | \
                   (pts[:, 0] > xyz_max[0]) | (pts[:, 1] > xyz_max[1]) | (pts[:, 2] > xyz_max[2])
    
    if bbox_cycle_xy:
        # Only check Z bounds for mask if cyclic XY is enabled?
        # Source [S3, unofficial]: "if bbox_cycle_xy is enabled, check only Z-axis bounds"
        mask_outbbox = (pts[:, 2] < xyz_min[2]) | (pts[:, 2] > xyz_max[2])
        
        # Wrap X and Y
        # Wrap around AABB bounds
        # Period = xyz_max[0] - xyz_min[0] for X, etc.
        # p_x = xyz_min[0] + (p_x - xyz_min[0]) % period_x
        # Handle negative modulo correctly if needed, but typically points are within range or wrapped positively
        period_x = xyz_max[0] - xyz_min[0]
        period_y = xyz_max[1] - xyz_min[1]
        
        pts[:, 0] = xyz_min[0] + torch.fmod(pts[:, 0] - xyz_min[0], period_x)
        pts[:, 1] = xyz_min[1] + torch.fmod(pts[:, 1] - xyz_min[1], period_y)
        
    return pts, mask_outbbox, ray_id, step_id, N_steps, t_min, t_max
```

## Subtopics

### NDC Bounding Box Computation
Computes the bounding box in Normalized Device Coordinates (NDC) space from a set of rays [S6, unofficial]. It computes the minimum and maximum of ray origins and endpoints (near and far planes) and returns the stacked min/max bounds: `torch.stack((torch.minimum(near_min, far_min), torch.maximum(near_max, far_max)))` [S6, unofficial]. This is used when rendering in NDC space where the bounding box is axis-aligned in the normalized cube [-1, 1] or similar [S6, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of rays and $S$ is the average number of samples per ray. The intersection calculation is $O(N)$, and point generation is $O(N \cdot S)$.
- **Memory Complexity:** $O(N \cdot S)$ to store the output points, masks, and IDs.
- **Performance:** The method is highly parallelizable and is typically implemented using CUDA kernels for GPU acceleration [S5, unofficial]. Batching rays (e.g., in chunks of 10,000) helps manage memory usage on GPUs [S5, unofficial].
- **Strengths:** Efficiently handles large numbers of rays and provides precise control over sampling density via `stepsize`. The use of cumulative sums for ID assignment is cache-friendly.
- **Limitations:** Fixed step size may lead to over-sampling in sparse regions or under-sampling in dense regions if not adaptive. Division by zero handling requires care.
- **When to use:** When rendering volume data with a known bounding box and fixed sampling density.
- **When not to use:** When adaptive sampling is required, or when the scene has no clear bounding box (use [Clipping Distance Computation via Volume Corners](2622-clipping-distance-computation-via-volume-corners.md) or other methods).

## Background
This method is a core component of modern differentiable volume rendering pipelines, particularly those based on Direct Volume Rendering (DVR) and Neural Radiance Fields (NeRF) [S2, unofficial][S5, unofficial]. It was popularized by implementations such as DirectVoxGO [S2, unofficial] and ref-dvgo [S5, unofficial], which require efficient ray-box intersection and sampling for training neural implicit representations. The approach of using slab intersection with fixed step sizes is a standard technique in computer graphics for ray marching [S5, unofficial].

## Sources
[S1] sjtuytc/UnboundedNeRFPytorch, https://github.com/sjtuytc/UnboundedNeRFPytorch, fetched 2026-10-26 (unofficial)
[S2] sunset1995/DirectVoxGO, https://github.com/sunset1995/DirectVoxGO, fetched 2026-10-26 (unofficial)
[S3] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
[S4] InanXR/FLASH, https://github.com/InanXR/FLASH, fetched 2026-10-27 (unofficial)
[S5] gkouros/ref-dvgo, https://github.com/gkouros/ref-dvgo, fetched 2026-10-28 (unofficial)
[S6] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
