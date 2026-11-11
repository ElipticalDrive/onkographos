# Depth Map to Absolute Camera Coordinates

Also called: Depth Map Unprojection

## Overview
This method converts a 2D depth map and camera parameters into a 3D point cloud in absolute world coordinates. It differs from sibling methods by explicitly handling the transformation from normalized pixel space to world space via an inverse projection matrix, often incorporating depth scaling for specific camera models [S2, unofficial]. It serves as the geometric foundation for initializing point clouds in neural rendering pipelines [S1, S2, unofficial].

## How it works
**Inputs:**
- Depth map $D'$ of size $H \times W$, where $d_{i,j}$ is the depth value at pixel $(i, j)$.
- Camera projection matrix $K$ (or elements $f_1, f_2$ derived from it).
- Camera pose $T_{cam2world}$ (optional, if world coordinates are required).

**Outputs:**
- 3D point cloud $D \in \mathbb{R}^{H \times W \times 3}$ in world coordinates.

**Data Structures:**
- $D'$: 2D array of floats.
- $K$: $3 \times 3$ matrix.
- $T$: $4 \times 4$ inverse projection transform matrix.

**Steps:**
1. **Normalization**: Normalize pixel coordinates $(i, j)$ to $[-1, 1]$ [S2, unofficial]:
   $$x_{norm} = \frac{2j}{W-1} - 1$$
   $$y_{norm} = \frac{2i}{H-1} - 1$$
   where $i$ is the row index and $j$ is the column index [S2, unofficial].

2. **Depth Scaling**: Scale depth values using $f_1, f_2$ from $K$ [S2, unofficial]:
   $$s_{d_{i,j}} = \frac{f_1 d_{i,j} + f_2}{d_{i,j}}$$
   [S2, unofficial]. Note: [S1, unofficial] does not mention this scaling step, using raw depth directly in camera coordinates.

3. **Homogeneous Camera Coordinates**: Form the vector in camera space [S2, unofficial]:
   $$p_{camera} = [x_{norm}, y_{norm}, s_{d_{i,j}}]^T$$
   [S2, unofficial]. If scaling is not applied, $s_{d_{i,j}} = d_{i,j}$.

4. **World Space Transformation**:
   - If camera pose is provided, transform to world coordinates.
   - Method A (from [S2, unofficial]): Multiply by the inverse of the full projection transform matrix $T$:
     $$p_{world} = T \times [x_{norm}, y_{norm}, s_{d_{i,j}}, 1]^T$$
     [S2, unofficial].
   - Method B (from [S1, unofficial]): If $T$ is not used, extract rotation $R_{cam2world}$ and translation $t_{cam2world}$ from the pose, and compute:
     $$X_{world} = R_{cam2world} X_{cam} + t_{cam2world}$$
     [S1, unofficial]. Here $X_{cam}$ is derived from $p_{camera}$ assuming standard pinhole projection.

5. **Discard Homogeneous Coordinate**: Convert to Cartesian coordinates [S2, unofficial]:
   $$D = p'_{world} = \frac{p_{world}[0:3]}{p_{world}[3]}$$
   [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def depth_map_to_absolute_camera_coordinates(depth_map, K, camera_pose=None):
    """
    Convert depth map to 3D world coordinates.
    
    Args:
        depth_map: H x W array of depth values.
        K: 3x3 camera projection matrix.
        camera_pose: 4x4 transformation matrix from camera to world (optional).
        
    Returns:
        points: H x W x 3 array of 3D world coordinates.
    """
    H, W = depth_map.shape
    points = np.zeros((H, W, 3))
    
    # Precompute normalization factors
    x_norm = (2 * np.arange(W) / (W - 1)) - 1
    y_norm = (2 * np.arange(H) / (H - 1)) - 1
    
    # Create meshgrid for pixel coordinates
    j_coords, i_coords = np.meshgrid(np.arange(W), np.arange(H))
    
    # Normalize pixel coordinates
    x_n = (2 * j_coords / (W - 1)) - 1
    y_n = (2 * i_coords / (H - 1)) - 1
    
    # Extract f1, f2 from K (assuming standard pinhole model)
    # Note: Exact mapping of f1, f2 to K elements depends on K's definition.
    # Typically f1 = K[0,0], f2 = K[0,2] or similar. 
    # Using generic f1, f2 as per [S2, unofficial].
    f1 = K[0, 0]
    f2 = K[0, 2]
    
    # Depth scaling
    d = depth_map
    s_d = (f1 * d + f2) / d
    
    # Homogeneous coordinates in camera space
    # p_camera = [x_n, y_n, s_d]
    
    if camera_pose is not None:
        # Method from [S2, unofficial]: Use inverse projection matrix T
        # T is the inverse of the full projection transform matrix.
        # This step assumes T is precomputed or derived from K and camera pose.
        # For implementation, T is often constructed as the inverse of the 
        # projection matrix combined with the pose.
        
        # Construct homogeneous vector [x_n, y_n, s_d, 1]
        # Shape: (H, W, 4)
        homo_vec = np.stack([x_n, y_n, s_d, np.ones_like(d)], axis=-1)
        
        # Apply transformation T (4x4)
        # p_world = T @ homo_vec.T -> (4, H*W)
        # Reshape to (H, W, 4)
        p_world = np.einsum('ij,hwj->hwi', camera_pose, homo_vec) # Simplified for 4x4 T
        
        # Discard homogeneous coordinate
        points = p_world[:, :, :3] / p_world[:, :, 3:4]
    else:
        # Method from [S1, unofficial]: Direct camera coordinates if no world pose
        # X_cam = [x_n * s_d, y_n * s_d, s_d] (assuming standard pinhole)
        # This is a placeholder for [S1, unofficial]'s logic which calls depthmap_to_camera_coordinates
        points = np.stack([x_n * s_d, y_n * s_d, s_d], axis=-1)
        
    return points
```

**Edge Cases and Pitfalls:**
- Division by zero in depth scaling if $d_{i,j} = 0$. Guard against this [S2, unofficial].
- Precision issues with large depth values in homogeneous division [S2, unofficial].
- Boundary conditions for $W-1$ and $H-1$ if dimensions are 1 [S2, unofficial].

## Operating characteristics
- **Time Complexity**: $O(H \times W)$ for pixel-wise operations.
- **Memory Complexity**: $O(H \times W)$ for storing the output point cloud.
- **Strengths**: Direct conversion to world coordinates; handles non-linear depth scaling if required [S2, unofficial].
- **Limitations**: Requires accurate camera intrinsics and pose [S1, S2, unofficial]. Sensitive to depth map noise [S2, unofficial].

## Background
This method is used in neural rendering pipelines to initialize point clouds from depth maps [S1, S2, unofficial]. It is a fundamental step in methods like InstantSplat [S1, unofficial] and AtomGS [S2, unofficial].

## Sources
[S1] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S2] AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field, https://core.ac.uk/works/391753628, fetched 2026-10-30 (unofficial)
