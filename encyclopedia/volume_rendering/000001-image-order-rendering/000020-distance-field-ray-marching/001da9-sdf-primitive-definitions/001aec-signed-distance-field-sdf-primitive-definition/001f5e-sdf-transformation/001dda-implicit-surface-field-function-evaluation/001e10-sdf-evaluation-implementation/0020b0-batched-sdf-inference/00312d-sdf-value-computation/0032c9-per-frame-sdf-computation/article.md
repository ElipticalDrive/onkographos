# Per-Frame SDF Computation


## Overview
Computes the signed distance function (SDF) and RGB color for a set of 3D points projected into a single camera view by comparing projected depth against a sampled depth map [S1, unofficial]. It serves as the per-view evaluation step in multi-view fusion pipelines, distinct from batched inference by operating on a single viewpoint's projection at a time [S1, unofficial].

## How it works
**Inputs:**
- `points`: Tensor of shape $(N, 3)$ containing 3D point coordinates in world space.
- `full_proj_transform`: $4 \times 4$ projection matrix mapping world coordinates to clip space.
- `depth_map`: Tensor of shape $(H, W)$ containing the depth buffer for the current view.
- `rgb_map`: Tensor of shape $(H, W, 3)$ containing the color buffer for the current view.
- `inv_contraction`: Optional tensor or flag indicating if points are in contracted space (used in Unbounded TSDF Fusion variant).

**Outputs:**
- `sdf`: Tensor of shape $(N, 1)$ containing signed distance values.
- `rgb`: Tensor of shape $(N, 3)$ containing interpolated RGB colors.
- `mask`: Boolean tensor of shape $(N, 1)$ indicating valid pixels.

**Data Structures:**
- Points are stored as floating-point tensors.
- Depth and RGB maps are sampled using bilinear interpolation with border padding.

**Steps:**
1. **Homogenization**: Append a column of ones to `points` to create homogeneous coordinates of shape $(N, 4)$.
2. **Projection**: Multiply homogeneous points by `full_proj_transform` to obtain clip space coordinates $z_{clip}$.
3. **Perspective Division**: Extract $z_{clip}$ and compute pixel coordinates $(u, v)$ by dividing $x_{clip}$ and $y_{clip}$ by $z_{clip}$:
   $$ u = \frac{x_{clip}}{z_{clip}}, \quad v = \frac{y_{clip}}{z_{clip}} $$
4. **Masking**: Create a projection mask where $-1 \le u \le 1$, $-1 \le v \le 1$, and $z_{clip} > 0$.
5. **Sampling**:
   - Sample `depth_map` at $(u, v)$ using bilinear `grid_sample` with border padding. Reshape to $(N, 1)$.
   - Sample `rgb_map` at $(u, v)$ using bilinear `grid_sample` with border padding. Reshape to $(N, 3)$.
6. **SDF Calculation**: Compute SDF as the difference between the sampled depth $d_{sampled}$ and the projected z-coordinate $z_{clip}$:
   $$ \text{SDF} = d_{sampled} - z_{clip} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sdf_perframe(points, full_proj_transform, depth_map, rgb_map):
    # points: (N, 3)
    # full_proj_transform: (4, 4)
    # depth_map: (H, W)
    # rgb_map: (H, W, 3)

    # 1. Homogenize
    ones = torch.ones(points.shape[0], 1, device=points.device)
    points_homo = torch.cat([points, ones], dim=1)  # (N, 4)

    # 2. Project to clip space
    clip_coords = torch.matmul(points_homo, full_proj_transform.T)  # (N, 4)
    x_clip = clip_coords[:, 0]
    y_clip = clip_coords[:, 1]
    z_clip = clip_coords[:, 2]

    # 3. Perspective division
    # Avoid division by zero
    z_clip_safe = torch.clamp(z_clip, min=1e-5)
    u = x_clip / z_clip_safe
    v = y_clip / z_clip_safe

    # 4. Create projection mask
    # Normalize u, v to [-1, 1] range for grid_sample if not already, 
    # but grid_sample expects normalized coordinates. 
    # Assuming u, v are already in normalized device coordinates (-1 to 1).
    mask = (u >= -1) & (u <= 1) & (v >= -1) & (v <= 1) & (z_clip > 0)

    # 5. Sample depth and RGB
    # Reshape to (1, H, W) and (1, H, W, 3) for grid_sample if needed, 
    # or use appropriate tensor shapes. 
    # grid_sample expects input (N, C, H, W) or (N, C, H, W, D).
    # Here we assume single view, so N=1 for the map.
    depth_map_4d = depth_map.unsqueeze(0).unsqueeze(0)  # (1, 1, H, W)
    rgb_map_4d = rgb_map.unsqueeze(0)  # (1, H, W, 3) -> needs adjustment for grid_sample
    
    # grid_sample expects coordinates in (N, H_out, W_out, 2)
    # We have N points. We need to reshape u, v to (1, N, 1, 2)
    coords = torch.stack([u, v], dim=-1).unsqueeze(0).unsqueeze(2)  # (1, N, 1, 2)
    
    # Sample depth
    sampled_depth = torch.nn.functional.grid_sample(
        depth_map_4d, coords, mode='bilinear', padding_mode='border', align_corners=False
    ).squeeze()  # (1, 1, N, 1) -> (N, 1)
    
    # Sample RGB
    # rgb_map needs to be (1, 3, H, W) for grid_sample
    rgb_map_4d_ch = rgb_map.permute(2, 0, 1).unsqueeze(0)  # (1, 3, H, W)
    sampled_rgb = torch.nn.functional.grid_sample(
        rgb_map_4d_ch, coords, mode='bilinear', padding_mode='border', align_corners=False
    ).squeeze()  # (1, 3, N, 1) -> (N, 3, 1) -> (N, 3)

    # 6. Compute SDF
    sdf = sampled_depth - z_clip.unsqueeze(1)  # (N, 1)

    return sdf, sampled_rgb, mask
```

**Edge Cases:**
- Points with $z_{clip} \le 0$ are masked out.
- Points outside the image bounds are masked out.
- Precision issues may arise if $z_{clip}$ is very small; clamping is applied.

## Subtopics

### Unbounded TSDF Fusion
Fuses SDF values from multiple frames in contracted space with adaptive truncation, then uncontracts the space [S1, unofficial]. It differs from standard per-frame computation by integrating multi-view data and handling unbounded scenes via contraction mapping [S1, unofficial]. Choose this when fusing multiple views for a complete reconstruction [S1, unofficial].

**Steps:**
1. If `inv_contraction` is provided, identify points outside the unit sphere ($||samples|| > 1$).
2. Set base `sdf_trunc` to $5 \times \text{voxel\_size}$. For points outside the unit sphere, scale truncation by $1/(2-||x||)$. If no `inv_contraction`, set `sdf_trunc` to constant $5 \times \text{voxel\_size}$.
3. Initialize `tsdfs` to -1 and `rgbs` to 0. Initialize `weights` to 1.
4. Iterate through `viewpoint_stack`. Call `compute_sdf_perframe` for each view.
5. Flatten SDF. Update mask to include points where SDF > `-sdf_trunc`.
6. Normalize SDF by `sdf_trunc`, clamp to $[-1, 1]$. Apply mask.
7. Update TSDF and RGB using weighted average: $new = (old \times w + val) / (w + 1)$.
8. Update weights: $w_{new} = w_{old} + 1$.
9. If `inv_contraction` is provided, uncontract the samples before returning.

## Operating characteristics
- **Time Complexity**: Linear in the number of points $N$ and number of views $V$ for fusion.
- **Memory Complexity**: Linear in $N$ for storing SDF and RGB values.
- **Strengths**: Efficient per-view evaluation; supports multi-view fusion.
- **Limitations**: Requires depth and RGB maps; sensitive to projection accuracy.
- **When to use**: Multi-view reconstruction pipelines.
- **When not to use**: Single-view rendering without depth buffer.

## Background
Originated in the context of 2D Gaussian Splatting for novel view synthesis, specifically in the `2d-gaussian-splatting` repository by Inria's GRAPHDECO research group [S1, unofficial]. It is used to fuse multi-view depth information into a consistent SDF representation.

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
