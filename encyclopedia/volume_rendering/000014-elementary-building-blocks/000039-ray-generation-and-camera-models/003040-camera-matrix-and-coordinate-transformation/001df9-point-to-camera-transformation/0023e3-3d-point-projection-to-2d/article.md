# 3D Point Projection to 2D

Also called: Point Projection

## Overview
Projects 3D points into multiple camera views to compute 2D pixel locations and visibility masks [S1, unofficial]. It differs from orthographic projection by introducing depth-dependent scaling (perspective division), which is essential for realistic camera models and visibility computation in volume rendering pipelines [G].

## How it works
**Inputs:**
- `xyz`: Tensor of 3D points with shape $[N, 3]$ in world coordinates [S1, unofficial].
- `train_cameras`: Tensor of shape $[V, 34]$ containing camera parameters for $V$ views [S1, unofficial]. The 34 values are structured as 2 values for image size, 16 values for intrinsic matrix parameters, and 16 values for extrinsic matrix parameters [S1, unofficial].

**Outputs:**
- `pixel_locations`: Tensor of shape $[V, N, 2]$ containing the 2D pixel coordinates $(u, v)$ for each point in each view [S1, unofficial].
- `visibility_mask`: Boolean tensor of shape $[V, N]$ indicating if points are in front of the camera [S1, unofficial].

**Data Structures:**
- **Homogeneous Coordinates**: 3D points are converted to 4D homogeneous vectors by appending a 1.0 [S1, unofficial].
- **Projection Matrix ($P$)**: Computed as $P = K \cdot C^{-1}$, where $K$ is the intrinsic matrix and $C$ is the camera-to-world extrinsic matrix [S1, unofficial].

**Steps:**
1. **Parse Camera Parameters**: Reshape the `train_cameras` tensor to extract intrinsic and extrinsic matrices. The intrinsics occupy indices 2 to 18 (16 values) and are reshaped into $[V, 4, 4]$ matrices $K$. The extrinsics occupy the last 16 values and are reshaped into $[V, 4, 4]$ matrices $C$ (camera-to-world) [S1, unofficial].
2. **Homogenize Points**: Append a column of 1s to `xyz` to create homogeneous coordinates $X_h$ of shape $[N, 4]$ [S1, unofficial].
3. **Compute Projection Matrix**: Calculate $P = K \cdot C^{-1}$ for each view [S1, unofficial].
4. **Project Points**: Multiply the projection matrix by the homogeneous points. The operation is performed as $P \cdot X_h^T$, repeated across views and permuted, resulting in projected coordinates $X_p$ of shape $[V, 4, N]$ [S1, unofficial].
5. **Perspective Division**: Compute pixel coordinates by dividing the x and y components by the z component (depth). The z component is clamped to a minimum value of $10^{-8}$ to prevent division by zero [S1, unofficial].
   $$ u = \frac{X_p[:, 0, :]}{\max(X_p[:, 2, :], 10^{-8})} $$
   $$ v = \frac{X_p[:, 1, :]}{\max(X_p[:, 2, :], 10^{-8})} $$
6. **Visibility Mask**: A point is considered visible (in front of the camera) if its projected z-coordinate is positive ($X_p[:, 2, :] > 0$) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def project_points_to_2d(xyz: torch.Tensor, train_cameras: torch.Tensor) -> tuple:
    """
    Projects 3D points to 2D pixel coordinates for multiple camera views.
    
    Args:
        xyz: Tensor of shape [N, 3] containing 3D world coordinates.
        train_cameras: Tensor of shape [V, 34] containing camera parameters.
                       Format: [img_w, img_h, 16 intrinsic params, 16 extrinsic params].
    
    Returns:
        pixel_locations: Tensor of shape [V, N, 2] with (u, v) coordinates.
        visibility_mask: Tensor of shape [V, N] with boolean values.
    """
    # 1. Parse camera parameters
    train_intrinsics = train_cameras[:, 2:18].reshape(-1, 4, 4)
    train_poses = train_cameras[:, -16:].reshape(-1, 4, 4)
    
    # 2. Convert 3D points to homogeneous coordinates [N, 4]
    xyz_h = torch.cat([xyz, torch.ones_like(xyz[:, :1])], dim=-1)
    
    # 3. Compute projection matrix P = K @ C^-1
    inv_poses = torch.inverse(train_poses)
    P = torch.bmm(train_intrinsics, inv_poses)
    
    # 4. Apply projection
    # xyz_h is [N, 4]. Transpose to [4, N], repeat for V views to [V, 4, N],
    # then permute to [V, N, 4] for batch matrix multiplication with P [V, 4, 4].
    xyz_h_t = xyz_h.t().repeat(P.shape[0], 1, 1).permute(0, 2, 1)
    projections = torch.bmm(P, xyz_h_t)  # [V, 4, N]
    
    # 5. Compute pixel coordinates (perspective division)
    x = projections[:, 0, :]  # [V, N]
    y = projections[:, 1, :]  # [V, N]
    z = projections[:, 2, :]  # [V, N]
    
    z_clamped = torch.clamp(z, min=1e-8)
    pixel_locations = torch.stack([x / z_clamped, y / z_clamped], dim=-1)  # [V, N, 2]
    
    # 6. Determine visibility mask
    visibility_mask = z > 0
    
    return pixel_locations, visibility_mask
```

**Parameters:**
- `train_cameras`: Tensor of shape $[V, 34]$ [S1, unofficial].
- `min_depth_clamp`: $10^{-8}$ [S1, unofficial].

**Edge Cases:**
- **Division by Zero**: Handled by clamping the projected depth $z$ to a minimum of $10^{-8}$ [S1, unofficial].
- **Back-facing Points**: Points with a projected depth $z \le 0$ are marked as invisible [S1, unofficial].

## Subtopics

### Depth Estimation from Point Cloud
Projects a generated 3D point cloud to the image plane of a novel viewpoint [S2]. If multiple 3D points project to the same pixel, the closest 3D point to the camera location is used for distance calculation [S2].

### Reprojection of 2D Features to 3D Space
Retrieves 2D pixel features for a query 3D point $p$ from an observed view $I_m$ using camera pose $\xi_m$ and intrinsics $K_m$ [S3]. If the projected point lies inside the image boundaries, the nearest pixel's features are duplicated to the 3D point using nearest neighbor interpolation [S3]. If it lies outside, a zero vector is assigned to indicate no observed information [S3]. Advanced techniques like bilinear interpolation or kernel functions may address discretization issues, but nearest neighbor is simpler and empirically effective for small-scale structures [S3].

## Operating characteristics
Time and memory complexity: Not given in the sources.

## Background
This method is used in the context of rendering images from a dataset by projecting points to compute pixel locations and visibility [S1, unofficial].

## Sources
[S1] albertgassol1/vf-nerf, https://github.com/albertgassol1/vf-nerf, fetched 2026-10-26 (unofficial)
[S2] Improving Neural Radiance Fields Using Near-Surface Sampling with Point Cloud Generation, https://link.springer.com/article/10.1007/s11063-024-11654-5, fetched 2026-10-30
[S3] GRF: Learning a General Radiance Field for 3D Representation and Rendering, https://doi.org/10.1109/iccv48922.2021.01490, fetched 2026-10-26
