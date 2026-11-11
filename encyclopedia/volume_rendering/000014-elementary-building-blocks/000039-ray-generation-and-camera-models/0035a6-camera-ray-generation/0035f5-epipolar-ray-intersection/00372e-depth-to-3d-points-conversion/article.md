# Depth to 3D Points Conversion

Also called: 3D Point Cloud Generation from Depth

## Overview
This method converts 2D depth maps and camera parameters into a 3D point cloud, serving as the geometric foundation for initializing point clouds in neural rendering pipelines [S1, unofficial][S2]. It differs from its sibling [Point Cloud Generation and Refinement](8042-point-cloud-generation-and-refinement.md) by relying on explicit geometric backprojection rather than implicit field refinement, and from [Fixed-size initialization from RGB-D backprojection](10155-fixed-size-initialization-from-rgb-d-backprojection.md) by focusing on the core coordinate transformation without necessarily incorporating color or specific downsampling strategies in the base definition [S1, unofficial][S2, S3]. It solves the problem of lifting 2D image data into 3D space for subsequent volume rendering or Gaussian splatting operations.

## How it works
**Inputs:**
- Depth map $D$ of shape $(B, H, W)$ or $(B, H, W, n)$ [S1, unofficial].
- Pseudo-focal lengths $f$ of shape $[B, H, W]$, $[B, 2, H, W]$, or $[B, 1, H, W]$ [S1, unofficial].
- Principal point $pp$ (optional, defaults to image center if None) [S1, unofficial].
- Camera intrinsics $K$ and extrinsics $T$ (for global scene construction) [S3].

**Outputs:**
- 3D point cloud $P$ in absolute coordinates [S1, unofficial][S3].

**Data Structures:**
- Grid coordinates $grid\_x, grid\_y$: 2D arrays of shape $(H, W)$ representing pixel indices relative to the principal point.
- Point cloud $P$: Array of shape $(N, 3)$ containing $(x, y, z)$ coordinates.

**Steps:**
1. **Generate Grid Coordinates:** Create a grid of pixel coordinates $(u, v)$ for the image dimensions $H \times W$.
2. **Adjust for Principal Point:** Subtract the principal point $(c_x, c_y)$ from the grid coordinates. If $pp$ is not provided, use the image center:
   $$ c_x = \frac{W-1}{2}, \quad c_y = \frac{H-1}{2} $$
   Let $grid\_x = u - c_x$ and $grid\_y = v - c_y$ [S1, unofficial].
3. **Extract Focal Lengths:** Parse $pseudo\_focal$ to obtain $pseudo\_focalx$ and $pseudo\_focaly$. If the input is a single channel, assume isotropic focal length [S1, unofficial].
4. **Backproject to 3D:** For each pixel $(u, v)$ with depth $d = D[u, v]$, compute the 3D coordinates in the camera frame:
   $$ x = d \cdot \frac{grid\_x}{pseudo\_focalx} $$
   $$ y = d \cdot \frac{grid\_y}{pseudo\_focaly} $$
   $$ z = d $$
   [S1, unofficial].
5. **Transform to World Coordinates (Global Construction):** For multi-view global construction, apply the inverse projection $\pi^{-1}$ using intrinsic $K_n$ and extrinsic $T_n = [R_n | t_n]$ for each view $n$:
   $$ P_n = T_n \cdot K_n^{-1} \cdot \begin{bmatrix} x \\ y \\ 1 \end{bmatrix} \cdot d $$
   Accumulate points from all $N$ views:
   $$ \mathcal{P} = \bigcup_{n=1}^{N} \mathbf{\pi}^{-1}(I_n, D_n, \mathbf{T}_n, \mathbf{K}_n) $$
   [S3].
6. **Filtering (Optional for Global):** Apply a uniform filter to remove duplicate primitives and a statistical filter to remove floaters (noise) [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def depth_to_3d_points(depth, pseudo_focal, pp=None):
    """
    Convert depth map to 3D points in camera coordinates.
    
    Args:
        depth: np.ndarray of shape (H, W) or (B, H, W).
        pseudo_focal: np.ndarray of shape (2,) or (B, 2) or (B, H, W).
        pp: tuple (cx, cy) or None.
        
    Returns:
        points: np.ndarray of shape (N, 3) or (B, N, 3).
    """
    # Handle batch dimension
    if depth.ndim == 2:
        depth = depth[np.newaxis, :, :]
        pseudo_focal = pseudo_focal[np.newaxis, :] if pseudo_focal.ndim == 1 else pseudo_focal
    
    B, H, W = depth.shape
    
    # Generate grid coordinates
    u, v = np.meshgrid(np.arange(W), np.arange(H))
    
    # Principal point
    if pp is None:
        cx, cy = (W - 1) / 2, (H - 1) / 2
    else:
        cx, cy = pp
        
    # Adjust grid
    grid_x = u - cx
    grid_y = v - cy
    
    # Expand grid for broadcasting
    grid_x = grid_x[np.newaxis, :, :]  # (1, H, W)
    grid_y = grid_y[np.newaxis, :, :]  # (1, H, W)
    
    # Handle pseudo_focal shape
    if pseudo_focal.ndim == 1:
        fx, fy = pseudo_focal[0], pseudo_focal[1]
        fx = np.full((B, H, W), fx)
        fy = np.full((B, H, W), fy)
    elif pseudo_focal.ndim == 2:
        fx = pseudo_focal[:, 0, np.newaxis, np.newaxis]
        fy = pseudo_focal[:, 1, np.newaxis, np.newaxis]
    else:
        fx = pseudo_focal[:, 0, :, :]
        fy = pseudo_focal[:, 1, :, :]
        
    # Compute 3D coordinates
    # x = depth * grid_x / fx
    # y = depth * grid_y / fy
    # z = depth
    
    x = depth * grid_x / fx
    y = depth * grid_y / fy
    z = depth
    
    # Stack and reshape
    points = np.stack([x, y, z], axis=-1)  # (B, H, W, 3)
    points = points.reshape(B, -1, 3)      # (B, N, 3)
    
    return points
```

**Parameters:**
- `depth`: Depth map values. Typical values depend on the scene scale.
- `pseudo_focal`: Focal length in pixels. For standard cameras, this is often derived from the sensor size and optical focal length.
- `pp`: Principal point. Default is image center.

**Edge Cases and Pitfalls:**
- **Zero Depth:** Pixels with $d=0$ result in $(0,0,0)$. These should be masked out or handled as invalid [G].
- **Invalid Focal Length:** Division by zero if $fx$ or $fy$ is 0. Ensure positive focal lengths [G].
- **Coordinate System:** The method produces points in the camera coordinate system (Z-forward or Z-backward depending on convention). Transformation to world coordinates requires extrinsic matrices [S3].

## Subtopics

### [Depth Map to Absolute Camera Coordinates](7984-depth-map-to-absolute-camera-coordinates.md)
This method converts a 2D depth map and camera parameters into a 3D point cloud in absolute world coordinates. It differs from sibling methods by explicitly handling the transformation from normalized pixel space to world space via an inverse projection matrix, often incorporating depth scaling for specific camera models. It serves as the geometric foundation for initializing point clouds in neural rendering pipelines.

### [Point Cloud Generation and Refinement](8042-point-cloud-generation-and-refinement.md)
An offline process to generate and refine a 3D point cloud from a subset of training images to improve depth estimation accuracy for novel views in Neural Radiance Fields (NeRF). It differs from standard Multi-View Stereo (MVS) by iteratively validating points against ground-truth depth from adjacent views, resulting in higher point density and better PSNR for similar computational time.

### [Fixed-size initialization from RGB-D backprojection](10155-fixed-size-initialization-from-rgb-d-backprojection.md)
This method generates a fixed-size, colored 3D point cloud initialization for volume rendering primitives (specifically 3D Gaussians) by backprojecting RGB-D data. It solves the problem of determining the initial spatial distribution and density of primitives in a scene. It differs from its parent [Depth to 3D Points Conversion](7929-depth-to-3d-points-conversion.md) by explicitly incorporating color information and applying specific downsampling strategies to control point count, and from its sibling [Point Cloud Generation and Refinement](8042-point-cloud-generation-and-refinement.md) by relying on explicit geometric backprojection rather than implicit field refinement for initialization.

### Global Scene Point Cloud Construction
Constructs a global scene point cloud $P \in \mathbb{R}^{N_p \times 3}$ in world coordinates by unprojecting pixels from $N$ calibrated images using intrinsic matrices $\{K_n\}_{n=1}^N$ and extrinsic matrices $\{T_n = R_n | t_n\}_{n=1}^N$ [S3]. The process involves:
1. Unprojecting pixels from each view: $\mathcal {P}=\bigcup ^N \mathbf {\pi }^{-1}\left (I_n, D_n, \mathbf {T}_n, \mathbf {K}_n\right )$ [S3].
2. Applying a uniform filter in 3D space to delete duplicate primitives while keeping the scene structure [S3].
3. Applying a statistical filter to remove floaters (noise) from the processed primitives [S3].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of pixels, for the basic backprojection. Global construction is $O(N \cdot V)$ where $V$ is the number of views. Filtering adds overhead proportional to the number of points.
- **Memory Complexity:** $O(N)$ to store the point cloud.
- **Strengths:** Simple, efficient, and provides a dense geometric initialization.
- **Limitations:** Sensitive to depth errors and noise. Does not handle occlusions or missing depth data inherently. Requires accurate camera calibration.
- **When to use:** When initializing point clouds for NeRF or Gaussian Splatting from known depth maps.
- **When not to use:** When depth maps are unreliable or unavailable, or when high-precision geometric refinement is required without iterative processes.

## Background
The method originates from the need to initialize 3D representations for neural rendering. It is used in InstantSplat [S1, unofficial], which uses pseudo-focal lengths to generate point clouds from depth maps. It is also referenced in surface-based sampling for NeRF [S2] and in EVolSplat for global point cloud construction [S3]. The technique is fundamental to many modern 3D Gaussian Splatting pipelines that rely on explicit geometric initialization.

## Sources
[S1] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S2] Improving Neural Radiance Fields Using Near-Surface Sampling with Point Cloud Generation, https://link.springer.com/article/10.1007/s11063-024-11654-5, fetched 2026-10-30
[S3] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
