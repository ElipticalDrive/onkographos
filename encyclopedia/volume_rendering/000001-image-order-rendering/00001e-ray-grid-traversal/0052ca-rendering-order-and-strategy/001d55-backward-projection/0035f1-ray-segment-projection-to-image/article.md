# Ray Segment Projection to Image
Also called: Ray Bounding Box Computation in Image Space, Ray-Image Frame Intersection with Near/Far Clipping

## Overview
This method computes the bounding box of a ray segment within the normalized image plane $[0, 1] \times [0, 1]$ for backward projection. It determines the valid range of ray parameters ($t_{min}, t_{max}$) and the corresponding 2D image coordinates ($xy_{min}, xy_{max}$) where the ray intersects the image frame. It sets itself apart from sibling methods by explicitly handling near/far clipping boundaries and validity masking to ensure only rays overlapping the image frame contribute to the projection, avoiding unnecessary computation for rays missing the frame entirely.

## How it works
**Inputs:**
- Ray origins $O$ and directions $D$ in world space.
- Inverse extrinsics matrix $M_{inv}$ to transform world space to camera space.
- Optional near clipping distance $t_{near}$ and far clipping distance $t_{far}$. If `None`, defaults to $t=0$ (origin) and $t=\infty$ respectively.

**Outputs:**
- `RaySegmentProjection` data structure containing:
  - `t_min`: Minimum ray parameter.
  - `t_max`: Maximum ray parameter.
  - `xy_min`: Minimum 2D image coordinates $(x, y)$.
  - `xy_max`: Maximum 2D image coordinates $(x, y)$.
  - `overlaps_image`: Boolean flag indicating if the ray segment intersects the image frame.

**Data Structures:**
The output is a typed dictionary (or equivalent struct) with fields:
- `t_min`: Float tensor of shape `[*batch]`.
- `t_max`: Float tensor of shape `[*batch]`.
- `xy_min`: Float tensor of shape `[*batch, 2]`.
- `xy_max`: Float tensor of shape `[*batch, 2]`.
- `overlaps_image`: Bool tensor of shape `[*batch]`.

**Steps:**
1. **Transform to Camera Space:**
   Transform ray origins $O$ and directions $D$ from world space to camera space using the inverse extrinsics matrix $M_{inv}$.
   $$ O_c = M_{inv} \cdot O $$
   $$ D_c = M_{inv} \cdot D $$
   Note: Directions are transformed as vectors (rotation only, no translation).

2. **Compute Frame Intersections:**
   For each ray, compute intersections with the four image frame boundaries: $x=0, x=1, y=0, y=1$. This is done using a helper function `_intersect_image_coordinate` which solves for $t$ such that the ray's projected coordinate equals the boundary value.
   Let $t_{x0}, t_{x1}, t_{y0}, t_{y1}$ be the $t$ values for these intersections.

3. **Determine Near/Far Projections:**
   - If $t_{near}$ is `None`, set $t_{near} = 0$. If the origin depth is near zero, use the direction vector to determine validity. Mark as invalid if depth is near zero but not at the camera.
   - If $t_{far}$ is `None`, set $t_{far} = \infty$. Compute projection at $t=\infty$ using the direction vector directly.

4. **Find Min/Max $t$ Values:**
   Use a reduction operation with `_compare_projections` to find the minimum and maximum $t$ values among the four frame intersections ($t_{x0}, t_{x1}, t_{y0}, t_{y1}$). This handles validity masking by ignoring invalid intersections.
   $$ t_{frame\_min} = \min(t_{x0}, t_{x1}, t_{y0}, t_{y1}) $$
   $$ t_{frame\_max} = \max(t_{x0}, t_{x1}, t_{y0}, t_{y1}) $$

5. **Combine Bounds:**
   Combine the frame intersections with the near/far projections to determine the final bounding box. Iterate through all 4 combinations of validity for near and far projections to assign the correct bounds.
   $$ t_{min} = \max(t_{near}, t_{frame\_min}) $$
   $$ t_{max} = \min(t_{far}, t_{frame\_max}) $$

6. **Compute Image Coordinates:**
   Calculate the 2D image coordinates at $t_{min}$ and $t_{max}$ using the ray equation in camera space.
   $$ P(t) = O_c + t \cdot D_c $$
   $$ xy_{min} = (P_x(t_{min}), P_y(t_{min})) $$
   $$ xy_{max} = (P_x(t_{max}), P_y(t_{max})) $$

7. **Determine Overlap:**
   The ray overlaps the image if $t_{min} < t_{max}$ and the resulting $xy$ coordinates are within $[0, 1]$. Set `overlaps_image` to `True` if valid, `False` otherwise.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_segment_projection_to_image(
    origins: Tensor, 
    directions: Tensor, 
    inv_extrinsics: Tensor, 
    t_near: Optional[float] = None, 
    t_far: Optional[float] = None
) -> RaySegmentProjection:
    # Step 1: Transform to camera space
    # origins and directions are [N, 3]
    # inv_extrinsics is [4, 4]
    origins_c = transform_to_camera_space(origins, inv_extrinsics)
    directions_c = transform_direction_to_camera_space(directions, inv_extrinsics)
    
    # Step 2: Compute frame intersections
    # _intersect_image_coordinate returns t values for x=0, x=1, y=0, y=1
    t_x0 = _intersect_image_coordinate(origins_c, directions_c, axis='x', value=0.0)
    t_x1 = _intersect_image_coordinate(origins_c, directions_c, axis='x', value=1.0)
    t_y0 = _intersect_image_coordinate(origins_c, directions_c, axis='y', value=0.0)
    t_y1 = _intersect_image_coordinate(origins_c, directions_c, axis='y', value=1.0)
    
    # Step 3: Handle near/far defaults
    if t_near is None:
        t_near = 0.0
        # Check for degenerate case where origin depth is near zero
        # Mark as invalid if depth is near zero but not at camera [G]
        near_valid = origins_c[:, 2] > 1e-6
    else:
        near_valid = torch.ones_like(origins_c[:, 0], dtype=torch.bool)
        
    if t_far is None:
        t_far = float('inf')
        far_valid = torch.ones_like(origins_c[:, 0], dtype=torch.bool)
    else:
        far_valid = torch.ones_like(origins_c[:, 0], dtype=torch.bool)
        
    # Step 4: Find min/max t among frame intersections
    # _compare_projections with reduction "min" and "max"
    t_frame_min = torch.min(torch.stack([t_x0, t_x1, t_y0, t_y1], dim=-1), dim=-1).values
    t_frame_max = torch.max(torch.stack([t_x0, t_x1, t_y0, t_y1], dim=-1), dim=-1).values
    
    # Handle invalid intersections by setting to inf or -inf
    # This is a simplification; actual implementation may use masking
    t_frame_min = torch.where(torch.isfinite(t_frame_min), t_frame_min, float('inf'))
    t_frame_max = torch.where(torch.isfinite(t_frame_max), t_frame_max, float('-inf'))
    
    # Step 5: Combine bounds
    t_min = torch.max(t_near * near_valid.float(), t_frame_min)
    t_max = torch.min(t_far * far_valid.float(), t_frame_max)
    
    # Step 6: Compute image coordinates
    # P(t) = O_c + t * D_c
    # xy coordinates are x and y components
    t_min_expanded = t_min.unsqueeze(-1).expand(-1, 3)
    t_max_expanded = t_max.unsqueeze(-1).expand(-1, 3)
    
    p_min = origins_c + t_min_expanded * directions_c
    p_max = origins_c + t_max_expanded * directions_c
    
    xy_min = p_min[:, :2]
    xy_max = p_max[:, :2]
    
    # Step 7: Determine overlap
    # Ray overlaps if t_min < t_max and coordinates are within [0, 1]
    overlaps = (t_min < t_max) & (xy_min[:, 0] >= 0) & (xy_min[:, 0] <= 1) & \
               (xy_min[:, 1] >= 0) & (xy_min[:, 1] <= 1) & \
               (xy_max[:, 0] >= 0) & (xy_max[:, 0] <= 1) & \
               (xy_max[:, 1] >= 0) & (xy_max[:, 1] <= 1)
               
    return RaySegmentProjection(
        t_min=t_min,
        t_max=t_max,
        xy_min=xy_min,
        xy_max=xy_max,
        overlaps_image=overlaps
    )
```

**Parameters:**
- `t_near`: Default `0.0`.
- `t_far`: Default `inf`.

**Edge Cases:**
- **Near Zero Depth:** If the origin depth is near zero, the ray direction is used to determine validity. If depth is near zero but not at the camera, mark as invalid [S1, unofficial].
- **Invalid Intersections:** Intersections with frame boundaries that are not valid (e.g., behind the camera) are handled by validity masking.

## Subtopics

### [Pixel-Weighted Line Integration](8503-pixel-weighted-line-integration.md)
A forward projection technique that computes a sinogram by approximating line integrals over a 2D image using pixel weights, iterating over a set of angles and parallel rays.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of rays, as it involves constant-time operations per ray.
- **Memory Complexity:** $O(N)$ for storing the output bounding boxes.
- **Strengths:** Efficiently culls rays that do not intersect the image frame, reducing unnecessary computation in subsequent projection steps.
- **Limitations:** Requires accurate transformation to camera space and handling of edge cases like near-zero depth.
- **When to Use:** When performing backward projection in volume rendering where rays need to be clipped to the image frame.
- **When Not to Use:** When rays are guaranteed to intersect the image frame or when a simpler projection method suffices.

## Background
This method is part of the VolSplat library, which focuses on efficient volume rendering techniques. It is used in the context of backward projection to ensure that only relevant rays contribute to the final image.

## Sources
[S1] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
