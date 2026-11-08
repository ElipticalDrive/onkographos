# Depth to 3D Points Conversion

Also called: 3D Point Cloud Generation from Depth

## Overview
This method converts a 2D depth map and camera parameters into a 3D point cloud in either camera-relative or world coordinates. It solves the problem of lifting 2D pixel measurements into 3D space to establish geometric correspondence for downstream tasks like novel view synthesis or scene reconstruction. It differs from its sibling [Epipolar Ray Intersection](7644-epipolar-ray-intersection.md) by operating on dense depth maps rather than sparse epipolar line intersections, and from [Point Cloud Generation and Refinement](8042-point-cloud-generation-and-refinement.md) by focusing on the initial unprojection step rather than the offline refinement and hole-filling processes.

## How it works
The method takes a depth map and camera intrinsics (or pseudo-focal lengths) as input and outputs a set of 3D points. The core operation is back-projection of each pixel $(u, v)$ with depth $d$ into 3D space.

**Inputs:**
- `depth`: Depth map of shape $(B, H, W)$ or $(B, H, W, n)$ [S1, unofficial].
- `pseudo_focal`: Pseudo focal lengths of shape $[B, H, W]$, $[B, 2, H, W]$, or $[B, 1, H, W]$ [S1, unofficial].
- `principal_point` (`pp`): Optional tuple $(pp_x, pp_y)$. If `None`, defaults to $((W-1)/2, (H-1)/2)$ [S1, unofficial].
- `camera_pose`: Optional extrinsic matrix containing rotation $R_{cam2world}$ and translation $t_{cam2world}$ [S1, unofficial].

**Data Structures:**
- `grid_x`, `grid_y`: 2D grids of pixel coordinates ranging from $0$ to $W-1$ and $0$ to $H-1$ respectively, generated via `xy_grid` [S1, unofficial].

**Steps:**
1.  **Generate Grid Coordinates:** Create 2D arrays `grid_x` and `grid_y` representing the column and row indices of each pixel in the depth map [S1, unofficial].
2.  **Adjust for Principal Point:** Subtract the principal point coordinates from the grid coordinates to center the coordinate system at the optical center.
    -   $u' = grid\_x - pp_x$
    -   $v' = grid\_y - pp_y$
    -   If `pp` is not provided, $pp_x = (W-1)/2$ and $pp_y = (H-1)/2$ [S1, unofficial].
3.  **Compute Camera Coordinates:** Calculate the 3D coordinates $(x, y, z)$ in the camera frame using the depth $d$ and adjusted grid coordinates.
    -   $x = d \cdot \frac{u'}{pseudo\_focalx}$
    -   $y = d \cdot \frac{v'}{pseudo\_focaly}$
    -   $z = d$
    -   Here, $pseudo\_focalx$ and $pseudo\_focaly$ are extracted from the `pseudo_focal` input based on its shape [S1, unofficial].
4.  **Transform to World Coordinates (Optional):** If `camera_pose` is provided, transform the camera coordinates to world coordinates.
    -   Extract $R_{cam2world}$ and $t_{cam2world}$ from the pose [S1, unofficial].
    -   $X_{world} = R_{cam2world} \cdot X_{cam} + t_{cam2world}$ [S1, unofficial].
5.  **Output:** Return the resulting 3D points. If used in an offline pipeline, save the point cloud for inference [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def depth_to_3d_points(depth, pseudo_focal, principal_point=None, camera_pose=None):
    """
    Converts depth map to 3D points.
    
    Args:
        depth: np.ndarray of shape (B, H, W) or (B, H, W, n)
        pseudo_focal: np.ndarray of shape [B, H, W], [B, 2, H, W], or [B, 1, H, W]
        principal_point: tuple (pp_x, pp_y) or None
        camera_pose: dict or array containing 'R' (3x3) and 't' (3,) or None
        
    Returns:
        points_world: np.ndarray of shape (B, H, W, 3)
    """
    # 1. Generate grid coordinates
    B, H, W = depth.shape[:3]
    y_coords, x_coords = np.meshgrid(np.arange(H), np.arange(W), indexing='ij')
    
    # 2. Adjust for principal point
    if principal_point is None:
        pp_x = (W - 1) / 2.0
        pp_y = (H - 1) / 2.0
    else:
        pp_x, pp_y = principal_point
        
    u_prime = x_coords - pp_x
    v_prime = y_coords - pp_y
    
    # 3. Extract pseudo focal lengths
    # Handle different shapes of pseudo_focal
    if pseudo_focal.ndim == 4:
        if pseudo_focal.shape[1] == 2:
            pseudo_focalx = pseudo_focal[:, 0, :, :]
            pseudo_focaly = pseudo_focal[:, 1, :, :]
        else:
            # Assume single channel or broadcastable
            pseudo_focalx = pseudo_focal[:, 0, :, :]
            pseudo_focaly = pseudo_focal[:, 0, :, :]
    else:
        # Shape [B, H, W] or [B, 1, H, W]
        pseudo_focalx = pseudo_focal[:, 0, :, :] if pseudo_focal.ndim == 3 else pseudo_focal[:, 0, :, :]
        pseudo_focaly = pseudo_focalx # Assuming isotropic or same value
        
    # 4. Compute camera coordinates
    # Avoid division by zero
    safe_focalx = np.where(np.abs(pseudo_focalx) > 1e-6, pseudo_focalx, 1.0)
    safe_focaly = np.where(np.abs(pseudo_focaly) > 1e-6, pseudo_focaly, 1.0)
    
    x_cam = depth * (u_prime / safe_focalx)
    y_cam = depth * (v_prime / safe_focaly)
    z_cam = depth
    
    # Stack into (B, H, W, 3)
    points_cam = np.stack([x_cam, y_cam, z_cam], axis=-1)
    
    # 5. Transform to world coordinates if pose is provided
    if camera_pose is not None:
        R = camera_pose['R'] # Shape (3, 3)
        t = camera_pose['t'] # Shape (3,)
        
        # Reshape points_cam to (B*H*W, 3) for matrix multiplication
        flat_points = points_cam.reshape(-1, 3)
        
        # Apply transformation: X_world = R * X_cam + t
        # R is (3,3), X_cam is (N, 3) -> (N, 3)
        points_world_flat = (R @ flat_points.T).T + t
        
        # Reshape back to (B, H, W, 3)
        points_world = points_world_flat.reshape(B, H, W, 3)
    else:
        points_world = points_cam
        
    return points_world
```

**Parameters and Typical Values:**
- `pseudo_focal`: Typically derived from camera intrinsics. If exact intrinsics are unknown, pseudo-focal lengths are estimated or assumed equal to image width/height [S1, unofficial].
- `principal_point`: Often assumed to be the image center $(W/2, H/2)$ if not provided [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** If `pseudo_focal` is zero, division will fail. A small epsilon or fallback value should be used [G].
- **Invalid Depth:** Depth values of 0 or infinity should be masked out or handled to prevent invalid 3D points [G].
- **Coordinate System:** Ensure consistency between the depth map convention (e.g., Z-forward vs Z-backward) and the camera coordinate system definition [G].

## Subtopics

### Depth Map to Absolute Camera Coordinates
Converts a depth map, intrinsics, and camera pose to 3D points in world coordinates. It extends the basic conversion by applying the camera-to-world transformation. It is chosen when world-space coordinates are required for global scene alignment. It links to [Epipolar Ray Intersection](7644-epipolar-ray-intersection.md) as it uses similar extrinsic data.

### Global Scene Point Cloud Construction
Constructs a global scene point cloud $P \in \mathbb{R}^{N_p \times 3}$ in world coordinates by unprojecting pixels from $N$ calibrated images using intrinsic matrices $\{K_n\}_{n=1}^N$ and extrinsic matrices $\{T_n = R_n | t_n\}_{n=1}^N$. The formula is $\mathcal {P}=\bigcup ^N \mathbf {\pi }^{-1}\left (I_n, D_n, \mathbf {T}_n, \mathbf {K}_n\right )$ where $\pi^{-1}$ denotes pixel unprojection, $I_n$ is the image, $D_n$ is the depth, $T_n$ is the extrinsic matrix, and $K_n$ is the intrinsic matrix [S3]. After construction, a uniform filter is applied in 3D space to delete duplicate primitives while keeping the scene structure, followed by a statistical filter to remove floaters (noise) [S3]. This variant is used for building dense, clean global maps from multiple views.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of pixels in the depth map, as each pixel is processed independently.
- **Memory Complexity:** $O(N)$ to store the resulting 3D points.
- **Performance:** Fast and parallelizable. Performance depends on the resolution of the depth map and the efficiency of the matrix operations for world transformation.
- **Strengths:** Simple, direct conversion; works well with dense depth maps.
- **Limitations:** Sensitive to noise in depth maps; does not handle occlusions or missing depth data inherently; requires accurate intrinsics/extrinsics for world coordinates.
- **When to use:** When a quick 3D representation of a single view or a set of views is needed for initialization or visualization.
- **When not to use:** When high-precision global alignment is required without post-processing (use Global Scene Point Cloud Construction with filtering).

## Background
The method originates from standard computer vision techniques for back-projecting depth maps to 3D space. It is widely used in 3D reconstruction, SLAM, and novel view synthesis pipelines. In the context of neural rendering, it is used for surface-based sampling to initialize NeRF or Gaussian Splatting models [S2, S3]. The specific implementation details using pseudo-focal lengths are found in projects like InstantSplat [S1, unofficial].

## Sources
[S1] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S2] Improving Neural Radiance Fields Using Near-Surface Sampling with Point Cloud Generation, https://link.springer.com/article/10.1007/s11063-024-11654-5, fetched 2026-10-30
[S3] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
