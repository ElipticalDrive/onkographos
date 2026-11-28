# Ray Filtering

Also called: Filtering Rays, Ray Culling, Ray Filtering Outside Bounding Box, Ray Filtering using Alpha Mask, TensorBase Ray Filtering

## Overview
Ray filtering reduces computational load by discarding rays that do not intersect with the scene's geometry or alpha mask before expensive volume rendering steps [S3, unofficial]. It distinguishes itself from sibling methods by operating as a pre-processing culling stage on a batch of rays, often integrating alpha-mask density checks alongside axis-aligned bounding box (AABB) intersection tests [S1, S2, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: A batch of rays defined by origins $\mathbf{o} \in \mathbb{R}^{N \times 3}$ and directions $\mathbf{d} \in \mathbb{R}^{N \times 3}$; an AABB defined by minimum corner $\mathbf{min} \in \mathbb{R}^3$ and maximum corner $\mathbf{max} \in \mathbb{R}^3$; an optional alpha mask volume for density-based culling; a boolean flag `bbox_only` [S1, S2, S3, unofficial].
- **Outputs**: A boolean mask `valid_mask` of size $N$ indicating which rays intersect the scene; optionally, filtered ray origins and directions [S2, unofficial].

**Data Structures**
- **AABB**: Two 3D vectors, `aabb[0]` (min) and `aabb[1]` (max) [S1, S2, unofficial].
- **Alpha Mask**: A 3D tensor sampled at query points, returning scalar alpha values [S2, unofficial].

**Steps**
1. Split the total number of rays $N$ into chunks of size `chunk` for batched processing [S1, unofficial].
2. For each chunk, extract ray origins and directions [S1, unofficial].
3. Compute the intersection parameter range $[t_{min}, t_{max}]$ with the AABB using the slab method:
   $$ \text{rate}_a = \frac{\mathbf{aabb}[1] - \mathbf{o}}{\mathbf{d}} $$
   $$ \text{rate}_b = \frac{\mathbf{aabb}[0] - \mathbf{o}}{\mathbf{d}} $$
   $$ t_{min} = \max(\min(\text{rate}_a), \min(\text{rate}_b)) $$
   $$ t_{max} = \min(\max(\text{rate}_a), \max(\text{rate}_b)) $$
   A ray intersects the box if $t_{max} > t_{min}$ [S1, S2, unofficial].
4. Determine the initial mask: `mask_inbbox = (t_max > t_min)` [S2, unofficial].
5. If `bbox_only` is False, sample points along the rays within the bounding box interval and check if any sample has an alpha value $> 0$ in the alpha mask. Update `mask_inbbox` to true only if the ray intersects the box AND passes through a non-zero alpha region [S1, S2, unofficial].
6. If `bbox_only` is True, `mask_inbbox` remains based solely on AABB intersection [S3, unofficial].
7. Reinitialize the ray sampler with the filtered ray count to ensure subsequent rendering steps only process valid rays [S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def filter_rays(rays_o, rays_d, aabb_min, aabb_max, alpha_mask=None, bbox_only=False, N_samples=64):
    """
    Filter rays based on AABB intersection and optionally alpha mask density.
    Args:
        rays_o: (N, 3) ray origins
        rays_d: (N, 3) ray directions
        aabb_min: (3,) minimum corner of AABB
        aabb_max: (3,) maximum corner of AABB
        alpha_mask: Object with method sample_xyz(xyz) returning alpha values
        bbox_only: Boolean, if True skip alpha mask check
        N_samples: Number of points sampled along each ray
    Returns:
        valid_mask: (N,) boolean mask of valid rays
    """
    N = len(rays_o)
    valid_mask = [False] * N
    
    for i in range(N):
        o = rays_o[i]
        d = rays_d[i]
        
        # Avoid division by zero for degenerate rays [G]
        d_safe = [1e-10 if abs(dx) < 1e-10 else dx for dx in d]
        
        # Compute rate arrays
        rate_a = [(aabb_max[j] - o[j]) / d_safe[j] for j in range(3)]
        rate_b = [(aabb_min[j] - o[j]) / d_safe[j] for j in range(3)]
        
        # Compute t_min and t_max
        t_min = max(min(rate_a), min(rate_b))
        t_max = min(max(rate_a), max(rate_b))
        
        # Initial bbox mask
        if t_max > t_min:
            if bbox_only:
                valid_mask[i] = True
            else:
                # Clamp start_t to 0 if origin is inside the box [G]
                start_t = max(t_min, 0.0)
                end_t = t_max
                
                if end_t > start_t:
                    # Sample points along the ray
                    t_vals = [start_t + k * (end_t - start_t) / (N_samples - 1) for k in range(N_samples)]
                    xyz_sampled = [[o[j] + d[j] * t for j in range(3)] for t in t_vals]
                    
                    # Check alpha mask
                    alpha_values = [alpha_mask.sample_xyz(pt) for pt in xyz_sampled]
                    if any(a > 0 for a in alpha_values):
                        valid_mask[i] = True
                    else:
                        valid_mask[i] = False
                else:
                    valid_mask[i] = False
        else:
            valid_mask[i] = False
            
    return valid_mask
```

**Parameters**
- `bbox_only`: Boolean. If `True`, only AABB intersection is checked. If `False`, alpha mask density is also checked [S1, S2, S3, unofficial].
- `N_samples`: Number of points sampled along each ray for alpha mask checking. Not given in the sources.

**Edge Cases and Pitfalls**
- **Degenerate Rays**: Rays with zero direction components cause division by zero in `rate_a` and `rate_b`. Handle by clamping zero direction components to a small epsilon before division [G].
- **Origin Inside Box**: If the ray origin is inside the AABB, $t_{min}$ may be negative. Ensure sampling starts at $\max(0, t_{min})$ to avoid sampling behind the camera [G].
- **Precision**: Floating-point errors in $t_{max} > t_{min}$ comparisons can lead to incorrect culling. Use a small epsilon if necessary [G].

## Subtopics

### Ray-box intersection
Standard AABB intersection test. Used as the primary culling step. [Ray-box intersection](776-ray-box-intersection.md)

### Ray-Volume Intersection (Face Distance Method)
Alternative to AABB for non-box volumes. Not used in this method. [Ray-Volume Intersection (Face Distance Method)](2381-ray-volume-intersection-face-distance-method.md)

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Performance**: Reduces computation for rays that do not pass through the scene [S3, unofficial].
- **Strengths**: Not given in the sources.
- **Limitations**: Not given in the sources.

## Background
Ray filtering is used in TensorBase [S2, unofficial] and Collaborative NeRF [S3, unofficial] to cull rays outside the bounding box or alpha mask. The method is typically applied during training iterations when the alpha mask is updated [S2, unofficial].

## Sources
[S1] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
[S2] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S3] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
