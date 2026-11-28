# Per-Frame SDF Computation


## Overview
Per-Frame SDF Computation calculates the signed distance function (SDF) and RGB color for a set of 3D points by projecting them into a single camera view and comparing their projected depth against a reference depth map [S1, unofficial]. It differs from batched inference by operating on a per-view basis, allowing for incremental fusion or single-view analysis, and serves as the core evaluation step within multi-view fusion pipelines like Unbounded TSDF Fusion [S1, unofficial].

## How it works
**Inputs:**
- `points`: A tensor of 3D points with shape $(N, 3)$, where $N$ is the number of points.
- `full_proj_transform`: A $4 \times 4$ projection matrix mapping world coordinates to clip space.
- `depth_map`: A 2D tensor representing the reference depth map for the current view.
- `rgb_map`: A 2D tensor representing the reference RGB color map for the current view.
- `inv_contraction` (optional): A boolean flag or function indicating if the space is contracted (used in Unbounded TSDF Fusion).

**Outputs:**
- `sdf`: A tensor of shape $(N, 1)$ containing the signed distance values.
- `rgb`: A tensor of shape $(N, 3)$ containing the sampled RGB colors.
- `mask`: A boolean tensor of shape $(N, 1)$ indicating valid projections.

**Data Structures:**
- Points are stored as floating-point tensors.
- Depth and RGB maps are stored as 2D tensors (height $\times$ width).
- Masks are boolean tensors.

**Steps:**
1. **Homogenization**: Append a column of ones to `points` to create homogeneous coordinates $(x, y, z, 1)$.
2. **Clip Space Projection**: Multiply the homogeneous points by `full_proj_transform` to obtain clip space coordinates $(x_c, y_c, z_c, w_c)$.
3. **Perspective Division**: Compute pixel coordinates by dividing $x_c$ and $y_c$ by $z_c$ (assuming $w_c=1$ or already handled in projection matrix). Let $z_{proj} = z_c$.
   $$ u = \frac{x_c}{z_{proj}}, \quad v = \frac{y_c}{z_{proj}} $$
4. **Projection Masking**: Create a mask where:
   - $-1 \le u \le 1$
   - $-1 \le v \le 1$
   - $z_{proj} > 0$
   Points failing any condition are marked invalid.
5. **Depth Sampling**: Sample the `depth_map` at $(u, v)$ using bilinear interpolation (`grid_sample` with border padding). Reshape to $(N, 1)$. Let this be $d_{sampled}$.
6. **Color Sampling**: Sample the `rgb_map` at $(u, v)$ using bilinear interpolation (`grid_sample` with border padding). Reshape to $(N, 3)$. Let this be $c_{sampled}$.
7. **SDF Calculation**: Compute the signed distance as the difference between the sampled depth and the projected depth:
   $$ \text{SDF} = d_{sampled} - z_{proj} $$
   [S1, unofficial]

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
    x_c, y_c, z_c, w_c = clip_coords[:, 0], clip_coords[:, 1], clip_coords[:, 2], clip_coords[:, 3]
    
    # 3. Perspective division
    # Note: In standard OpenGL/DirectX projections, w_c is often -z or similar.
    # Assuming full_proj_transform handles w correctly for division.
    # If w_c is not 1, we divide by w_c. If the matrix is set up such that
    # z_c is already the depth after division, adjust accordingly.
    # Standard GL projection: z_c is clip z, w_c is -view_z.
    # Here we assume z_c represents the depth value to compare against depth_map.
    # The source implies z_c is the projected z.
    z_proj = z_c 
    
    # 4. Pixel coordinates
    u = x_c / z_proj
    v = y_c / z_proj
    
    # 5. Masking
    valid_u = (u >= -1.0) & (u <= 1.0)
    valid_v = (v >= -1.0) & (v <= 1.0)
    valid_z = z_proj > 0.0
    mask = valid_u & valid_v & valid_z
    
    # 6. Sampling
    # Normalize u, v to [-1, 1] for grid_sample if they are in [-1, 1] already.
    # grid_sample expects normalized coordinates in [-1, 1].
    # u, v are already in [-1, 1] for valid points.
    # Reshape to (N, 1, 1, 2) for grid_sample: (N, 1, 1, 2) -> (N, 1, 1, 2)
    # grid_sample input: (N, H, W, 2) or (N, 1, 1, 2) depending on implementation.
    # PyTorch grid_sample: grid is (N, H_out, W_out, 2).
    # We need to sample at specific points.
    
    # Create grid for sampling
    # u, v are (N,)
    # Reshape to (N, 1, 1, 2)
    grid = torch.stack([u, v], dim=-1).unsqueeze(1).unsqueeze(1)  # (N, 1, 1, 2)
    
    # Sample depth
    # depth_map needs to be (1, 1, H, W) for grid_sample
    depth_map_expanded = depth_map.unsqueeze(0).unsqueeze(0)  # (1, 1, H, W)
    sampled_depth = torch.nn.functional.grid_sample(
        depth_map_expanded, grid, mode='bilinear', padding_mode='border', align_corners=False
    ).squeeze(-1).squeeze(-1)  # (N, 1)
    
    # Sample RGB
    # rgb_map needs to be (1, 3, H, W)
    rgb_map_expanded = rgb_map.permute(2, 0, 1).unsqueeze(0)  # (1, 3, H, W)
    sampled_rgb = torch.nn.functional.grid_sample(
        rgb_map_expanded, grid, mode='bilinear', padding_mode='border', align_corners=False
    ).squeeze(-1).squeeze(-1)  # (N, 3)
    
    # 7. SDF Calculation
    sdf = sampled_depth - z_proj.unsqueeze(1)  # (N, 1)
    
    return sdf, sampled_rgb, mask
```

## Subtopics

### Unbounded TSDF Fusion
Fuses SDF values from multiple frames in contracted space with adaptive truncation, then uncontracts the space [S1, unofficial]. It differs from standard TSDF fusion by handling unbounded scenes via a contraction mapping that compresses infinity into a unit sphere [S1, unofficial]. Choose this when dealing with large-scale outdoor scenes or when the volume of interest extends beyond a fixed bounding box [S1, unofficial]. It calls `compute_sdf_perframe` for each view in the `viewpoint_stack` [S1, unofficial]. The truncation parameter `sdf_trunc` is set to $5 \times \text{voxel\_size}$ [S1, unofficial]. For points outside the unit sphere ($||x|| > 1$), the truncation is scaled by $1/(2-||x||)$ if `inv_contraction` is provided; otherwise, it remains constant [S1, unofficial]. The fusion uses a weighted average update: $new = (old \times w + val) / (w + 1)$, with weights initialized to 1 and incremented per view [S1, unofficial].

### Signed Ray Distance Function (SRDF)
A volumetric shape representation where the signed distance of a 3D point $X$ is defined by the depth prediction of a specific camera $j$ along the viewing ray passing through $X$, rather than the shortest distance in any direction [S2]. Unlike standard SDF which uses the shortest distance along any direction, or SDDF which uses a fixed direction, SRDF uses the direction of the viewing ray from a specific camera [S2]. It is defined as $SRDF_j(X) = D_j(X) - Z_j(X)$, where $D_j(X)$ is the predicted depth and $Z_j(X)$ is the point's depth along the ray [S2]. It is differentiable at any point in the 3D observation volume [S2].

## Operating characteristics
Time complexity is linear with respect to the number of points $N$ for a single view, dominated by the projection and bilinear sampling operations [S1, unofficial]. Memory complexity is $O(N)$ for storing the SDF and RGB values [S1, unofficial]. Performance scales with the resolution of the depth/RGB maps due to the sampling cost, but is generally efficient on GPU due to parallelizable tensor operations [S1, unofficial]. Strengths include simplicity and direct integration with differentiable rendering pipelines [S1, unofficial]. Limitations include sensitivity to depth map accuracy and occlusion handling, as it relies on a single view's depth map [S1, unofficial]. Not suitable for scenes with significant self-occlusion without multi-view fusion [S1, unofficial].

## Background
Originated in the context of 2D Gaussian Splatting and multi-view reconstruction, specifically in the `2d-gaussian-splatting` repository by Inria's GRAPHDECO research group [S1, unofficial]. The method is used to evaluate the consistency of 3D points against 2D observations, facilitating colorization and SDF-based regularization [S1, unofficial]. It builds on the concept of projecting 3D points to 2D to leverage 2D image data for 3D attribute estimation [S1, unofficial].

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] Multi-View Reconstruction Using Signed Ray Distance Functions (SRDF), https://doi.org/10.1109/cvpr52729.2023.01602, fetched 2026-10-26
