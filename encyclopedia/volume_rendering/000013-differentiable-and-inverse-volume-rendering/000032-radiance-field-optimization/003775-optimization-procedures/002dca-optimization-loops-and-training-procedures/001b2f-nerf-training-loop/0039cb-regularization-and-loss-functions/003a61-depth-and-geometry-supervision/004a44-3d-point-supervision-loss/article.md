# 3D Point Supervision Loss

Also called: 3D Point Supervision, Point-based Pose Registration Loss

## Overview
A training objective that minimizes the distance between 3D points backprojected from rendered depth and ground-truth 3D points to enforce global geometric consistency. It solves the problem of pose refinement in sparse-view regimes where standard depth losses fail to register camera poses correctly [S1]. Unlike [Geometric Baseline Regularization](7475-geometric-baseline-regularization.md) which operates on density fields, this method explicitly aligns rendered geometry with known 3D landmarks [S1].

## How it works
**Inputs:**
*   Neural Radiance Field (NeRF) parameters.
*   Camera poses (intrinsics $\mathbf{K}$, extrinsics $\mathbf{T}$).
*   Ground-truth 3D points $\mathbf{P}_{gt}$ in the world frame.
*   Rendered depth $d$ for each pixel.

**Outputs:**
*   Scalar loss value $\mathcal{L}_{3D}$.

**Data Structures:**
*   $\mathbf{P}_{gt}$: Array of $N$ 3D points, shape $(N, 3)$.
*   $\mathbf{P}_{pred}$: Array of $M$ backprojected 3D points, shape $(M, 3)$.

**Steps:**
1.  **Render Depth:** For each training view, render the depth map $d(u, v)$ using the NeRF [S1].
2.  **Backproject:** Convert 2D pixel coordinates $(u, v)$ and rendered depth $d$ into 3D world coordinates $\mathbf{P}_{pred}$ using the camera intrinsics and extrinsics [S1].
    *   Formula: $\mathbf{P}_{pred} = \mathbf{T}^{-1} \mathbf{K}^{-1} [u, v, 1]^T \cdot d$ [G].
3.  **Compute Distance:** Calculate the L1 distance between the set of predicted points $\mathbf{P}_{pred}$ and the nearest ground-truth points $\mathbf{P}_{gt}$ [S1].
    *   Formula: $\mathcal{L}_{3D} = \frac{1}{M} \sum_{i=1}^{M} \min_{j} || \mathbf{P}_{pred}^{(i)} - \mathbf{P}_{gt}^{(j)} ||_1$ [G].
4.  **Minimize:** Add $\mathcal{L}_{3D}$ to the total loss with weight $\lambda_{3D}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_3d_point_loss(nerf, poses, intrinsics, gt_points_3d):
    """
    Computes the 3D point supervision loss.
    
    Args:
        nerf: Neural Radiance Field model.
        poses: List of camera poses (extrinsics) for training views.
        intrinsics: Camera intrinsic matrix K.
        gt_points_3d: Array of ground-truth 3D points (N, 3).
        
    Returns:
        loss: Scalar L1 distance between backprojected points and GT points.
    """
    total_loss = 0.0
    num_points = 0
    
    for pose in poses:
        # 1. Render depth map from NeRF
        # depth_map shape: (H, W)
        depth_map = nerf.render_depth(pose)
        
        # 2. Backproject depth to 3D world coordinates
        # Create meshgrid of pixel coordinates (u, v)
        u, v = np.meshgrid(np.arange(depth_map.shape[1]), np.arange(depth_map.shape[0]))
        u = u.flatten()
        v = v.flatten()
        depth_vals = depth_map.flatten()
        
        # Filter out invalid depths (e.g., background or zero)
        valid_mask = depth_vals > 0
        u = u[valid_mask]
        v = v[valid_mask]
        depth_vals = depth_vals[valid_mask]
        
        if len(u) == 0:
            continue
            
        # Convert to homogeneous coordinates
        # k_coords shape: (M, 3)
        k_coords = np.stack([u, v, np.ones_like(u)], axis=-1)
        
        # Inverse projection to camera space
        # cam_coords shape: (M, 3)
        cam_coords = np.linalg.inv(intrinsics) @ k_coords.T
        cam_coords = cam_coords.T * depth_vals[:, np.newaxis]
        
        # Transform to world frame
        # world_coords shape: (M, 3)
        world_coords = (pose @ np.vstack([cam_coords.T, np.ones((1, len(cam_coords)))]))[:3, :].T
        
        # 3. Compute distance to nearest GT point
        # For each predicted point, find min L1 distance to any GT point
        # Using broadcasting for efficiency: (M, 1, 3) - (1, N, 3) -> (M, N, 3)
        dists = np.abs(world_coords[:, np.newaxis, :] - gt_points_3d[np.newaxis, :, :])
        min_dists = np.min(np.sum(dists, axis=-1), axis=-1)
        
        total_loss += np.sum(min_dists)
        num_points += len(min_dists)
        
    if num_points == 0:
        return 0.0
        
    return total_loss / num_points
```

**Parameters:**
*   $\lambda_{3D}$: Weight for the 3D point loss. Not given in the sources.

**Edge Cases:**
*   **No GT Points:** If $\mathbf{P}_{gt}$ is empty, the loss is undefined. [G]
*   **Invalid Depth:** Background pixels or occluded regions may produce invalid depth values. These must be filtered before backprojection. [G]
*   **Sparse GT:** If GT points are sparse, the nearest-neighbor search may not represent the full surface geometry accurately. [S1]

## Subtopics

## Operating characteristics
*   **Time Complexity:** Dominated by depth rendering and nearest-neighbor search. Not given in the sources.
*   **Memory Complexity:** Requires storing backprojected points and GT points. Not given in the sources.
*   **Performance:** On DTU with 3 views and noisy poses, Photo + L1 GT 3D points yields PSNR 18.8 (20.3 masked), SSIM 0.75 (0.84 masked), LPIPS 0.21 (0.11 masked), DE 0.07 [S1]. This outperforms Photo + L1 GT depth which yields PSNR 13.8 (14.0 masked), SSIM 0.54 (0.70 masked), LPIPS 0.46 (0.27 masked), DE 0.17 [S1].
*   **Strengths:** Enforces global geometric connection between training images, successfully registering poses in sparse-view regimes [S1].
*   **Limitations:** Requires ground-truth 3D points, which may not be available in all datasets [S1].
*   **When to use:** Sparse-view regimes with noisy poses where standard depth losses fail to refine poses [S1].
*   **When not to use:** Dense-view regimes with accurate poses, or when GT 3D points are unavailable [S1].

## Background
Originated in SPARF (Neural Radiance Fields from Sparse and Noisy Poses) [S1]. The method was developed to address the issue of overfitting and degenerate geometry in sparse-view NeRF training, particularly when camera poses are noisy [S1]. It builds on the insight that enforcing explicit geometric connections between training images and underlying scene geometry is crucial in sparse-view settings [S1].

## Sources
[S1] SPARF: Neural Radiance Fields from Sparse and Noisy Poses, https://doi.org/10.1109/cvpr52729.2023.00408, fetched 2026-10-26
