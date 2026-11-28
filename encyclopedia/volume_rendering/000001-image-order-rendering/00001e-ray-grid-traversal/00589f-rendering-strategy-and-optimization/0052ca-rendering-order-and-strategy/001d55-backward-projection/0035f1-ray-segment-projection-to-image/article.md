# Ray Segment Projection to Image

Also called: Ray Bounding Box Computation in Image Space, Ray Segment Projection, Ray-Image Frame Intersection with Near/Far Clipping

## Overview
This method computes the bounding segment of a ray (defined by origin and direction) that overlaps with the canonical image frame, returning the parametric $t$-range and the 2D $xy$ coordinates at the entry and exit points [S2, unofficial]. It solves the problem of determining which parts of a 3D ray are visible within the 2D image plane boundaries, which is a prerequisite for subsequent rasterization or sampling in backward projection pipelines [S1, unofficial]. It differs from [GPU Backprojection](5843-gpu-backprojection.md) by explicitly computing geometric bounds rather than relying on hardware rasterization, and from [Cone Beam Forward Projection with Trilinear Interpolation](8247-cone-beam-forward-projection-with-trilinear-interpolation.md) by operating on individual rays rather than volumetric integration [S1, unofficial].

## How it works
**Inputs:**
- `origins`: Tensor of shape `(*batch, 3)`, ray origins in world space.
- `directions`: Tensor of shape `(*batch, 3)`, ray directions in world space.
- `extrinsics`: $4 \times 4$ matrix transforming world space to camera space.
- `near`: Optional scalar or tensor of shape `(*batch,)`, near clipping distance. If `None`, defaults to $t=0$.
- `far`: Optional scalar or tensor of shape `(*batch,)`, far clipping distance. If `None`, defaults to $t=\infty$.

**Outputs:**
- `RaySegmentProjection`: A TypedDict containing:
  - `t_min`: Float tensor of shape `(*batch,)`, minimum parametric distance.
  - `t_max`: Float tensor of shape `(*batch,)`, maximum parametric distance.
  - `xy_min`: Float tensor of shape `(*batch, 2)`, 2D coordinates at $t_{min}$.
  - `xy_max`: Float tensor of shape `(*batch, 2)`, 2D coordinates at $t_{max}$.
  - `overlaps_image`: Bool tensor of shape `(*batch,)`, true if the segment intersects the image frame [S1, unofficial][S2, unofficial].

**Data Structures:**
The output is a dictionary-like structure (TypedDict) where each field is a batched tensor [S1, unofficial][S2, unofficial]. The `xy` fields store the projected coordinates $(x, y)$ in the canonical image plane, computed by intersecting with the frame boundaries $x=0, x=1, y=0, y=1$ [S1, unofficial][S2, unofficial].

**Steps:**
1. Transform ray origins and directions from world space to camera space using the inverse extrinsics matrix [S1, unofficial].
2. Compute intersections with the four image frame boundaries ($x=0, x=1, y=0, y=1$) using `_intersect_image_coordinate` [S1, unofficial][S2, unofficial].
3. Find the minimum and maximum $t$ values among the four frame intersections using `_compare_projections` with reduction "min" and "max" respectively, handling validity masking [S1, unofficial].
4. Handle near clipping: If `near` is provided, compute projection at $t = \text{near}$: $\mathbf{o}' + \text{near} \cdot \mathbf{d}'$. If `near` is `None`, compute projection at $t=0$ (origin). If the origin's depth ($z'$) is near zero (less than a small epsilon $\epsilon$), check if the origin is at the camera position (norm $< \epsilon$). If so, use the direction for projection; otherwise, mark the near projection as invalid [S1, unofficial][S2, unofficial].
5. Handle far clipping: If `far` is provided, compute projection at $t = \text{far}$: $\mathbf{o}' + \text{far} \cdot \mathbf{d}'$. If `far` is `None`, compute projection at $t=\infty$ using the direction vector directly [S1, unofficial][S2, unofficial].
6. Combine the min/max frame intersections and the near/far projections to determine the final bounding box ($t_{min}, t_{max}, xy_{min}, xy_{max}$) and overlap status. Iterate through all 4 combinations of validity for near and far projections (valid/invalid for each). For each combination, create a mask where the corresponding projection (at zero or frame min, at infinity or frame max) is valid. Assign the appropriate $t$ and $xy$ values to the result. Set `overlaps_image` to `True` only if both the min and max projections are valid [S1, unofficial][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def ray_segment_projection_to_image(
    origins: torch.Tensor,
    directions: torch.Tensor,
    extrinsics: torch.Tensor,
    near: torch.Tensor = None,
    far: torch.Tensor = None,
    epsilon: float = 1e-6
) -> dict:
    # Step 1: Transform to camera space
    R_inv = extrinsics[:3, :3].inverse()
    t_inv = extrinsics[:3, 3]
    origins_cam = torch.einsum('ij,bj->bi', R_inv, origins) + t_inv
    directions_cam = torch.einsum('ij,bj->bi', R_inv, directions)
    
    # Step 2 & 3: Compute frame intersections and find min/max t
    boundaries = torch.tensor([0.0, 1.0, 0.0, 1.0], device=origins.device)
    axes = torch.tensor([0, 0, 1, 1], device=origins.device)
    
    o_vals = origins_cam[:, axes]
    d_vals = directions_cam[:, axes]
    
    valid = torch.abs(d_vals) > epsilon
    t_frame = torch.where(valid, (boundaries - o_vals) / d_vals, torch.full_like(o_vals, float('inf')))
    
    t_frame_min = torch.where(valid, t_frame, torch.full_like(t_frame, float('inf'))).min(dim=-1).values
    t_frame_max = torch.where(valid, t_frame, torch.full_like(t_frame, float('-inf'))).max(dim=-1).values
    
    xy_min_frame = origins_cam[:, :2] + t_frame_min.unsqueeze(1) * directions_cam[:, :2]
    xy_max_frame = origins_cam[:, :2] + t_frame_max.unsqueeze(1) * directions_cam[:, :2]
    
    # Step 4: Handle near clipping
    if near is None:
        t_near = torch.zeros_like(origins_cam[:, 0])
        origin_norm = torch.norm(origins_cam, dim=-1)
        is_at_camera = origin_norm < epsilon
        near_valid = ~((origins_cam[:, 2] < epsilon) & ~is_at_camera)
        xy_near = origins_cam[:, :2] + t_near.unsqueeze(1) * directions_cam[:, :2]
        xy_near = torch.where(is_at_camera.unsqueeze(1), directions_cam[:, :2], xy_near)
    else:
        t_near = near
        near_valid = torch.ones_like(t_near, dtype=torch.bool)
        xy_near = origins_cam[:, :2] + t_near.unsqueeze(1) * directions_cam[:, :2]
        
    # Step 5: Handle far clipping
    if far is None:
        t_far = torch.full_like(origins_cam[:, 0], float('inf'))
        far_valid = torch.ones_like(t_far, dtype=torch.bool)
        xy_far = directions_cam[:, :2]
    else:
        t_far = far
        far_valid = torch.ones_like(t_far, dtype=torch.bool)
        xy_far = origins_cam[:, :2] + t_far.unsqueeze(1) * directions_cam[:, :2]
        
    # Step 6: Combine and assign bounds
    t_min = torch.full_like(t_near, float('inf'))
    t_max = torch.full_like(t_near, float('-inf'))
    xy_min = torch.zeros_like(xy_near)
    xy_max = torch.zeros_like(xy_near)
    overlaps_image = torch.zeros_like(t_near, dtype=torch.bool)
    
    # Combination 1: near valid, far valid
    mask1 = near_valid & far_valid
    if mask1.any():
        t_min[mask1] = torch.minimum(t_near[mask1], t_frame_min[mask1])
        t_max[mask1] = torch.maximum(t_far[mask1], t_frame_max[mask1])
        xy_min[mask1] = torch.where(t_near[mask1] < t_frame_min[mask1], xy_near[mask1], xy_min_frame[mask1])
        xy_max[mask1] = torch.where(t_far[mask1] > t_frame_max[mask1], xy_far[mask1], xy_max_frame[mask1])
        overlaps_image[mask1] = t_min[mask1] < t_max[mask1]
        
    # Combination 2: near valid, far invalid
    mask2 = near_valid & ~far_valid
    if mask2.any():
        t_min[mask2] = t_near[mask2]
        t_max[mask2] = t_frame_max[mask2]
        xy_min[mask2] = xy_near[mask2]
        xy_max[mask2] = xy_max_frame[mask2]
        overlaps_image[mask2] = t_min[mask2] < t_max[mask2]
        
    # Combination 3: near invalid, far valid
    mask3 = ~near_valid & far_valid
    if mask3.any():
        t_min[mask3] = t_frame_min[mask3]
        t_max[mask3] = t_far[mask3]
        xy_min[mask3] = xy_min_frame[mask3]
        xy_max[mask3] = xy_far[mask3]
        overlaps_image[mask3] = t_min[mask3] < t_max[mask3]
        
    # Combination 4: near invalid, far invalid
    mask4 = ~near_valid & ~far_valid
    if mask4.any():
        t_min[mask4] = t_frame_min[mask4]
        t_max[mask4] = t_frame_max[mask4]
        xy_min[mask4] = xy_min_frame[mask4]
        xy_max[mask4] = xy_max_frame[mask4]
        overlaps_image[mask4] = t_min[mask4] < t_max[mask4]
        
    return {
        "t_min": t_min,
        "t_max": t_max,
        "xy_min": xy_min,
        "xy_max": xy_max,
        "overlaps_image": overlaps_image
    }
```

**Edge Cases and Pitfalls:**
- **Degenerate Camera Origins:** If the ray origin is at the camera position (norm $< \epsilon$), the direction is used for projection to avoid invalid depth [S1, unofficial][S2, unofficial].
- **Zero Depth:** If the origin's depth is near zero but not at the camera, the projection is marked as invalid [S1, unofficial][S2, unofficial].

## Subtopics

### [Pixel-Weighted Line Integration](8503-pixel-weighted-line-integration.md)
A forward projection technique that computes a sinogram by approximating line integrals over a 2D image using pixel weights. It determines which pixels a line passes through and their intersection lengths using a Bresenham-like rasterization approach. It differs from matrix-based forward projection by computing the transformation manually to avoid constructing and inverting a large matrix.

### [Ray-to-Image Projection with Near/Far Plane Handling](15852-ray-to-image-projection-with-near-far-plane-handling.md)
This method projects 3D camera rays into 2D image space to determine their intersection with the image frame boundaries, computing bounding segments $(t_{min}, t_{max})$ and pixel coordinates $(xy_{min}, xy_{max})$ along with a validity flag. It specifically handles near/far clipping planes and degenerate camera origins. It differs from [Pixel-Weighted Line Integration](8503-pixel-weighted-line-integration.md) by explicitly computing the geometric bounding box of the ray segment within the image plane rather than integrating weights directly, providing the necessary bounds for subsequent rasterization or sampling.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of rays, as it involves a constant number of arithmetic operations per ray.
- **Memory Complexity:** $O(N)$ for storing the input and output tensors.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of rays. Suitable for GPU parallelization.
- **Strengths:** Explicit geometric computation, handles near/far clipping, robust to degenerate cases.
- **Limitations:** Requires explicit handling of validity masks, may be slower than hardware rasterization for very large numbers of rays if not optimized.
- **When to Use:** When precise geometric bounds are needed for sampling or rasterization, or when hardware rasterization is not available.
- **When Not to Use:** When hardware rasterization is available and sufficient precision is not required.

## Background
This method is part of the backward projection pipeline in volume rendering, specifically for rendering order and strategy.

## Sources
[S1] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
[S2] pulangk97/VoLoDSplat, https://github.com/pulangk97/VoLoDSplat, fetched 2026-10-27 (unofficial)
