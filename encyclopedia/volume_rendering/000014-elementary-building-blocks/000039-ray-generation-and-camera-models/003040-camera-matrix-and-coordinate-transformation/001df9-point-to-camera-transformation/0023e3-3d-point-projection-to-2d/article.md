# 3D Point Projection to 2D

Also called: Point Projection

## Overview
Projects 3D points into multiple camera views to compute 2D pixel locations and visibility masks [S1, unofficial]. It differs from orthographic projection by introducing depth-dependent scaling (perspective division), which is essential for realistic camera models and visibility computation in volume rendering pipelines [S1, unofficial, S2, unofficial].

## How it works
**Inputs:**
- `xyz`: Tensor of 3D points with shape $[N, 3]$ (coordinates in world space).
- `train_cameras`: Tensor of shape $[V, 34]$ containing camera parameters for $V$ views [S1, unofficial]. The 34 values are structured as: 2 values for image size, 16 values for intrinsic matrix parameters, and 16 values for extrinsic matrix parameters [S1, unofficial].

**Outputs:**
- `pixel_locations`: Tensor of shape $[V, N, 2]$ containing the 2D pixel coordinates $(u, v)$ for each point in each view.
- `visibility_mask`: Boolean tensor of shape $[V, N]$ indicating if points are in front of the camera.

**Data Structures:**
- **Homogeneous Coordinates**: 3D points are converted to 4D homogeneous vectors by appending a 1.0.
- **Projection Matrix ($P$)**: Computed as $P = K \cdot T_{world\_to\_cam}^{-1}$, where $K$ is the intrinsic matrix and $T_{world\_to\_cam}^{-1}$ is the inverse of the extrinsic matrix (camera-to-world) [S1, unofficial].

**Steps:**
1. **Parse Camera Parameters**: Reshape the `train_cameras` tensor to extract intrinsic and extrinsic matrices. The intrinsics occupy indices 2 to 18 (16 values) and are reshaped into $[V, 4, 4]$ matrices $K$. The extrinsics occupy the last 16 values and are reshaped into $[V, 4, 4]$ matrices $C$ (camera-to-world) [S1, unofficial].
2. **Homogenize Points**: Append a column of 1s to `xyz` to create homogeneous coordinates $X_h$ of shape $[N, 4]$ [S1, unofficial].
3. **Compute Projection Matrix**: Calculate $P = K \cdot C^{-1}$ for each view [S1, unofficial].
4. **Project Points**: Multiply the projection matrix by the homogeneous points. The operation is performed as $P \cdot X_h^T$, resulting in projected coordinates $X_p$ of shape $[V, 4, N]$ [S1, unofficial].
5. **Perspective Division**: Compute pixel coordinates by dividing the x and y components by the z component (depth). The z component is clamped to a minimum value of $10^{-8}$ to prevent division by zero [S1, unofficial].
   $$ u = \frac{X_p[:, 0, :]}{\max(X_p[:, 2, :], 10^{-8})} $$
   $$ v = \frac{X_p[:, 1, :]}{\max(X_p[:, 2, :], 10^{-8})} $$
6. **Visibility Mask**: A point is considered visible (in front of the camera) if its projected z-coordinate is positive ($X_p[:, 2, :] > 0$) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

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
    # Intrinsics: indices 2 to 18 (16 values) reshaped to [V, 4, 4]
    train_intrinsics = train_cameras[:, 2:18].reshape(-1, 4, 4)
    
    # Extrinsics: last 16 values reshaped to [V, 4, 4] (Camera-to-World)
    train_poses = train_cameras[:, -16:].reshape(-1, 4, 4)
    
    # 2. Convert 3D points to homogeneous coordinates [N, 4]
    # Append 1s to the last dimension
    xyz_h = torch.cat([xyz, torch.ones_like(xyz[:, :1])], dim=-1)
    
    # 3. Compute projection matrix P = K @ C^-1
    # Inverse of the camera-to-world pose gives world-to-camera transformation
    inv_poses = torch.inverse(train_poses)
    P = torch.bmm(train_intrinsics, inv_poses)
    
    # 4. Apply projection
    # xyz_h is [N, 4]. We need to multiply P [V, 4, 4] by xyz_h [N, 4].
    # Transpose xyz_h to [4, N] for batch matrix multiplication.
    # Result shape: [V, 4, N]
    xyz_h_t = xyz_h.t()  # [4, N]
    # Repeat P to match N? No, bmm handles [V, 4, 4] @ [V, 4, N] if we expand xyz_h_t.
    # Actually, standard bmm: [V, 4, 4] @ [V, 4, N] -> [V, 4, N]
    # We need to expand xyz_h_t to [V, 4, N]
    xyz_h_expanded = xyz_h_t.unsqueeze(0).repeat(P.shape[0], 1, 1)
    
    projections = torch.bmm(P, xyz_h_expanded)  # [V, 4, N]
    
    # 5. Compute pixel coordinates (perspective division)
    # Extract x, y, z from the first 3 rows of the result
    x = projections[:, 0, :]  # [V, N]
    y = projections[:, 1, :]  # [V, N]
    z = projections[:, 2, :]  # [V, N]
    
    # Clamp z to min 1e-8 to avoid division by zero
    z_clamped = torch.clamp(z, min=1e-8)
    
    pixel_locations = torch.stack([x / z_clamped, y / z_clamped], dim=-1)  # [V, N, 2]
    
    # 6. Determine visibility mask
    # Points are valid if they are in front of the camera (z > 0)
    visibility_mask = z > 0
    
    return pixel_locations, visibility_mask
```

**Parameters:**
- `train_cameras`: Tensor of shape $[V, 34]$ [S1, unofficial].
- `min_depth_clamp`: $10^{-8}$ [S1, unofficial].

**Edge Cases:**
- **Division by Zero**: Handled by clamping the depth $z$ to a minimum of $10^{-8}$ [S1, unofficial].
- **Back-facing Points**: Points with $z \le 0$ are marked as invisible [S1, unofficial].

## Subtopics

### [Perspective Projection of Points](4037-perspective-projection-of-points.md)
Perspective projection maps 3D world points to 2D image coordinates by intersecting a ray from the center of projection through the point with an image plane. It differs from orthographic projection by introducing depth-dependent scaling (perspective division), which is essential for realistic camera models and visibility computation in volume rendering pipelines.

## Operating characteristics
- **Time Complexity**: $O(V \cdot N)$, where $V$ is the number of views and $N$ is the number of points. The dominant operations are matrix inversion ($O(V)$) and batch matrix multiplication ($O(V \cdot N)$).
- **Memory Complexity**: $O(V \cdot N)$ to store the projected coordinates and visibility masks.
- **Strengths**: Efficiently computes visibility and 2D locations for batched points across multiple views.
- **Limitations**: Does not account for occlusion by other geometry; only checks if the point is in front of the camera plane.
- **When to use**: When generating visibility masks or 2D projections for point-based representations in multi-view volume rendering.
- **When not to use**: When occlusion by other geometry is required (use ray marching or rasterization).

## Background
This method is implemented in the `vf-nerf` repository [S1, unofficial]. It is used in the context of rendering images from a dataset by projecting points to compute pixel locations and visibility.

## Sources
[S1] albertgassol1/vf-nerf, https://github.com/albertgassol1/vf-nerf, fetched 2026-10-26 (unofficial)
