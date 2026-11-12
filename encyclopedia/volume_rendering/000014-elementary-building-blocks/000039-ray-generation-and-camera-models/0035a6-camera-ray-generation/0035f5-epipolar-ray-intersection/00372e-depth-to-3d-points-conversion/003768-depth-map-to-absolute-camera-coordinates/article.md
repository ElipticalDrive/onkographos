# Depth Map to Absolute Camera Coordinates

Also called: Depth Map Unprojection, depth2pc

## Overview
This method converts a 2D depth map, camera intrinsics, and camera pose into a 3D point cloud in absolute world coordinates [S1, S3, unofficial]. It solves the problem of back-projecting pixel-space depth values into a global reference frame, enabling the construction of scene geometry from multi-view inputs [S4]. It differs from sibling methods by explicitly handling the full projection matrix inversion and coordinate normalization, whereas [Point Cloud Generation and Refinement](8042-point-cloud-generation-and-refinement.md) focuses on feature embedding alongside geometry, and [Fixed-size initialization from RGB-D backprojection](10155-fixed-size-initialization-from-rgb-d-backprojection.md) assumes pre-aligned, fixed-size buffers [S4].

## How it works
**Inputs:**
- Depth map $D'$ of size $H \times W$, where $d_{i,j}$ is the depth at pixel $(i, j)$ [S3, unofficial].
- Camera intrinsic matrix $K$ (or focal lengths $f_1, f_2$ and principal point offsets) [S3, unofficial].
- Camera extrinsic pose (rotation $R_{cam2world}$ and translation $t_{cam2world}$) [S1, unofficial].
- Optional: Full projection transform matrix $T$ [S3, unofficial].

**Outputs:**
- 3D world coordinates $D \in \mathbb{R}^{H \times W \times 3}$ [S3, unofficial].

**Data Structures:**
- Pixel coordinates $(i, j)$ where $i \in [0, H-1]$ and $j \in [0, W-1]$.
- Homogeneous coordinates represented as 4-vectors $[x, y, z, w]^T$.

**Steps:**

1.  **Coordinate Normalization:**
    Normalize pixel coordinates to the range $[-1, 1]$ [S3, unofficial]:
    $$ x_{norm} = \frac{2j}{W-1} - 1 $$
    $$ y_{norm} = \frac{2i}{H-1} - 1 $$
    *Note: Some implementations use a meshgrid from $0.5$ to $S-0.5$ for square inputs $S \times S$ [S2, unofficial].*

2.  **Camera Space Construction:**
    Form homogeneous camera space coordinates $p_{camera}$ [S3, unofficial]:
    $$ p_{camera} = [x_{norm}, y_{norm}, d_{i,j}]^T $$
    Alternatively, stack $x, y, 1$ and set the z-component to inverse depth $1/(d_{i,j} + \epsilon)$ [S2, unofficial].

3.  **Depth Scaling (Optional/Variant):**
    If using the scaling variant [S3, unofficial], adjust depth values using elements $f_1, f_2$ from $K$:
    $$ s_{d_{i,j}} = \frac{f_1 d_{i,j} + f_2}{d_{i,j}} $$
    Set adjusted coordinates $p'_{camera} = [x_{norm}, y_{norm}, s_{d_{i,j}}]^T$ [S3, unofficial].

4.  **World Space Transformation:**
    *Method A (Matrix Inversion) [S3, unofficial]:*
    Multiply by the inverse of the full projection transform matrix $T$:
    $$ p_{world} = T \times [x_{norm}, y_{norm}, s_{d_{i,j}}, 1]^T $$
    Discard the homogeneous coordinate to get Cartesian coordinates:
    $$ D = p'_{world} = \frac{p_{world}[0:3]}{p_{world}[3]} $$

    *Method B (Rotation/Translation) [S1, S2, unofficial]:*
    If camera pose is provided as $R_{cam2world}$ and $t_{cam2world}$:
    $$ X_{world} = R_{cam2world} \times X_{cam} + t_{cam2world} $$
    Or equivalently, using the inverse rotation:
    $$ pts = R_{cam2world}^T \times pts_{2d} - R_{cam2world}^T \times t_{cam2world} $$
    where $pts_{2d}$ contains the 2D pixel coordinates extended to 3D [S2, unofficial].

**Parameters:**
- Epsilon $\epsilon = 10^{-8}$ to prevent division by zero [S2, unofficial].
- Typical focal lengths $f_1, f_2$ are derived from the intrinsic matrix $K$ [S3, unofficial].

**Edge Cases:**
- **Zero Depth:** Add epsilon to depth values before inversion or division [S2, unofficial].
- **Boundary Pixels:** Normalization formulas assume pixel centers; off-by-one errors occur if $W-1$ or $H-1$ is zero (degenerate images) [S3, unofficial].
- **Coordinate System:** Ensure consistency between intrinsic definition (e.g., principal point at center vs. corner) and normalization range [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def depth_map_to_absolute_camera_coordinates(depth_map, intrinsic_matrix, extrinsic_matrix):
    """
    Convert depth map to 3D world coordinates.
    
    Args:
        depth_map: np.ndarray of shape (H, W), depth values d_{i,j}.
        intrinsic_matrix: np.ndarray of shape (3, 3), camera intrinsic K.
        extrinsic_matrix: np.ndarray of shape (3, 4), camera pose [R | t].
        
    Returns:
        world_points: np.ndarray of shape (H, W, 3), 3D coordinates in world space.
    """
    H, W = depth_map.shape
    
    # 1. Create pixel coordinate grids
    # i corresponds to row (y), j corresponds to col (x)
    j_coords, i_coords = np.meshgrid(np.arange(W), np.arange(H))
    
    # 2. Normalize to [-1, 1]
    # Formula: x_norm = 2*j/(W-1) - 1
    # Handle degenerate case where W=1 or H=1 to avoid division by zero
    if W > 1:
        x_norm = (2.0 * j_coords) / (W - 1) - 1.0
    else:
        x_norm = np.zeros_like(j_coords)
        
    if H > 1:
        y_norm = (2.0 * i_coords) / (H - 1) - 1.0
    else:
        y_norm = np.zeros_like(i_coords)
        
    # 3. Extract intrinsic parameters
    fx = intrinsic_matrix[0, 0]
    fy = intrinsic_matrix[1, 1]
    cx = intrinsic_matrix[0, 2]
    cy = intrinsic_matrix[1, 2]
    
    # 4. Convert normalized coordinates to camera space
    # Standard unprojection: X_cam = (x - cx) / fx * Z, Y_cam = (y - cy) / fy * Z
    # Using normalized coordinates directly requires scaling by focal lengths
    # p_camera = [x_norm, y_norm, d]
    # To get metric coordinates, we must undo the normalization and apply intrinsics
    
    # Alternative approach using standard pinhole model:
    # x_pixel = j, y_pixel = i
    # X_cam = (j - cx) * d / fx
    # Y_cam = (i - cy) * d / fy
    # Z_cam = d
    
    # Using the normalized coordinate method from [S3, unofficial] with scaling:
    # s_d = (f1 * d + f2) / d. If f2=0, s_d = f1.
    # However, standard backprojection is simpler and equivalent if intrinsics are standard.
    # Let's implement the standard backprojection which is robust and common [S1, S2, unofficial].
    
    # Depth map handling
    depth = depth_map.astype(np.float32)
    epsilon = 1e-8
    depth_safe = depth + epsilon
    
    # Calculate camera space coordinates
    # X_cam
    x_cam = (j_coords.astype(np.float32) - cx) * depth_safe / fx
    # Y_cam
    y_cam = (i_coords.astype(np.float32) - cy) * depth_safe / fy
    # Z_cam
    z_cam = depth_safe
    
    # Stack to (H, W, 3)
    camera_points = np.stack([x_cam, y_cam, z_cam], axis=-1)
    
    # 5. Transform to world space
    # extrinsic_matrix is [R | t] of shape (3, 4)
    R = extrinsic_matrix[:, :3]
    t = extrinsic_matrix[:, 3]
    
    # Reshape for matrix multiplication: (H*W, 3)
    flat_points = camera_points.reshape(-1, 3)
    
    # World = R * Cam + t
    # Matrix multiply: (3, 3) @ (N, 3)^T -> (3, N)
    world_points_flat = R @ flat_points.T + t[:, np.newaxis]
    
    # Reshape back to (H, W, 3)
    world_points = world_points_flat.T.reshape(H, W, 3)
    
    return world_points
```

## Subtopics

### Point Cloud Construction
Converts each input view into a 3D point cloud containing geometries and view-dependent semantics by estimating per-pixel depth and features, then projecting them into 3D space [S4]. It differs from basic unprojection by attaching a 64-dimensional feature vector (61-dim semantic features + 3 RGB channels) to each 3D point [S4]. Choose this when view-dependent appearance is required for rendering [S4]. Feature vector base dimension is 61, total 64 dimensions [S4].

## Operating characteristics
- **Time Complexity:** $O(H \times W)$ for the back-projection step, as each pixel is processed independently [S3, unofficial].
- **Memory Complexity:** $O(H \times W \times 3)$ for storing the resulting 3D coordinates [S3, unofficial].
- **Performance:** Highly parallelizable on GPUs; performance scales linearly with resolution [S3, unofficial].
- **Strengths:** Exact geometric reconstruction given accurate depth and calibrated cameras; efficient for dense point cloud generation [S4].
- **Limitations:** Sensitive to depth errors; noisy depth maps produce noisy point clouds; does not handle occlusions or missing depth values automatically without masking [S3, unofficial].
- **When to use:** When absolute metric 3D coordinates are needed for downstream tasks like fusion or rendering [S1, unofficial].
- **When not to use:** When only relative geometry or feature-rich points are needed (use [Point Cloud Generation and Refinement](8042-point-cloud-generation-and-refinement.md)) [S4].

## Background
Originated from standard computer vision techniques for back-projecting depth maps to 3D space [S3, unofficial]. It is a fundamental building block in multi-view stereo and neural rendering pipelines, such as InstantSplat [S1, unofficial] and GPS+ [S2, unofficial]. The method is cited in AtomGS for high-fidelity radiance field reconstruction [S3, unofficial] and in FWD for real-time novel view synthesis [S4].

## Sources
[S1] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S2] YaourtB/GPS_plus, https://github.com/YaourtB/GPS_plus, fetched 2026-10-27 (unofficial)
[S3] AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field, https://core.ac.uk/works/391753628, fetched 2026-10-30 (unofficial)
[S4] FWD: Real-time Novel View Synthesis with Forward Warping and Depth, https://doi.org/10.1109/cvpr52688.2022.01526, fetched 2026-10-29
