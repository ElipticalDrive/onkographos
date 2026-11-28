# Mask Cache Lookup with Ray Bounding Box Check

Also called: DVGO Hit Coarse Geometry, DVGO Hit Coarse Geometry Check

## Overview
This method checks whether rays intersect with the currently solved coarse geometry mask by combining ray sampling with a 3D boolean grid lookup [S1, unofficial]. It identifies valid sampling points before subsequent volume rendering computations.

## How it works
**Inputs:**
- `rays_o`: Ray origins, shape $[N, 3]$.
- `rays_d`: Ray directions, shape $[N, 3]$.
- `xyz_min`: Minimum corner of the bounding box, shape $[3]$.
- `xyz_max`: Maximum corner of the bounding box, shape $[3]$.
- `near`: Near clipping plane distance.
- `far`: Far clipping plane distance.
- `stepsize`: Base step size for sampling.
- `voxel_size`: Physical size of a single voxel in the grid.
- `mask_cache`: A 3D boolean grid (or tensor) storing occupancy information.

**Outputs:**
- `hit`: A boolean tensor of shape $[N]$, where `hit[i]` is `True` if ray `i` intersects the coarse geometry within the bounding box [S1, unofficial].

**Data Structures:**
- `mask_cache`: A 3D boolean grid. The implementation assumes this supports indexing with world coordinates converted to grid indices [G].
- `ray_pts`: Tensor of sampled points on rays, shape $[N, K, 3]$, where $K$ is the number of samples per ray.
- `mask_outbbox`: Boolean tensor of shape $[N, K]$, `True` if the point is outside the bounding box.
- `ray_id`: Tensor mapping each sampled point back to its source ray index, shape $[N, K]$.

**Parameters:**
- `far`: Set to $10^9$ in the source implementation [S1, unofficial].

**Steps:**
1. Reshape `rays_o` and `rays_d` to shape $[-1, 3]$ and ensure they are contiguous in memory [S1, unofficial].
2. Set `far` to $10^9$ [S1, unofficial].
3. Calculate the effective step distance: $stepdist = stepsize \times voxel\_size$ [S1, unofficial].
4. Sample points along each ray using the bounding box limits and step distance. This yields `ray_pts`, `mask_outbbox`, and `ray_id` [S1, unofficial].
5. Identify points inside the bounding box: $mask\_inbbox = \neg mask\_outbbox$ [S1, unofficial].
6. Initialize a boolean tensor `hit` of zeros with length equal to the number of rays $N$ [S1, unofficial].
7. For points inside the bounding box, query the `mask_cache` using their world coordinates. The coordinates are converted to grid indices internally by the `mask_cache` lookup function [G].
8. Mark rays as hit if the corresponding query points are inside the bounding box and pass the mask cache check:
   $$ hit[ray\_id[mask\_inbbox][mask\_cache(ray\_pts[mask\_inbbox])]] = 1 $$
   This operation sets `hit[i] = 1` if any sample point on ray `i` is inside the bounding box and marked as occupied in the coarse geometry mask [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hit_coarse_geo(rays_o, rays_d, xyz_min, xyz_max, near, far, stepsize, voxel_size, mask_cache):
    # Reshape to [-1, 3] and ensure contiguous layout
    rays_o = rays_o.reshape(-1, 3)
    rays_d = rays_d.reshape(-1, 3)
    
    # Set far distance
    far_val = 1e9
    
    # Calculate step distance
    stepdist = stepsize * voxel_size
    
    # [G] Standard ray-box sampling: computes points along rays within [xyz_min, xyz_max] at stepdist intervals.
    # Returns ray_pts [N, K, 3], mask_outbbox [N, K], ray_id [N, K]
    # ray_pts, mask_outbbox, ray_id = sample_pts_on_rays(rays_o, rays_d, xyz_min, xyz_max, near, far_val, stepdist)
    
    # Identify points inside the bounding box
    mask_inbbox = ~mask_outbbox
    
    # Initialize hit tensor of zeros
    hit = zeros(len(rays_o))
    
    # Query mask cache for points inside bounding box
    # mask_cache converts world coordinates to grid indices and returns occupancy boolean tensor
    occupied = mask_cache(ray_pts[mask_inbbox])
    
    # Mark rays as hit if corresponding points are occupied
    hit[ray_id[mask_inbbox][occupied]] = 1
    
    return hit
```

**Edge Cases and Pitfalls:**
- **Precision:** Ensure `rays_o` and `rays_d` are contiguous to avoid CUDA kernel errors [S1, unofficial].
- **Bounding Box:** The `far` distance is set to $10^9$ in the source [S1, unofficial].
- **Empty Rays:** If no points are inside the bounding box or no points are occupied, the `hit` tensor remains all zeros [G].
- **Multiple Hits:** If multiple points on a single ray are occupied, the ray is marked as hit only once due to tensor indexing semantics [G].

## Operating characteristics
Time and memory complexity, measured performance, scaling, strengths, limitations, and usage conditions are Not given in the sources.

## Background
This method originates from the provided codebase implementation [S1, unofficial].

## Sources
[S1] sjtuytc/UnboundedNeRFPytorch, https://github.com/sjtuytc/UnboundedNeRFPytorch, fetched 2026-10-26 (unofficial)
