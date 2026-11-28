# 3D Feature Construction via Lifting and Voxelization


## Overview
This method constructs a dense 3D feature volume from multi-view 2D features by lifting unstructured point clouds into a global coordinate system and aggregating them into a regular voxel grid [S1]. It differs from local feature aggregation by first projecting 2D features into 3D space via depth unprojection and rigid transformation, then using average pooling to aggregate features within voxels, providing global contextual information for downstream volume rendering tasks [S1].

## How it works
**Inputs:**
- Set of $N$ input views [S1].
- For each view $i \in \{1, \dots, N\}$:
  - Image feature map $F_i \in \mathbb{R}^{H \times W \times C}$ [S1].
  - Predicted depth map $D_i \in \mathbb{R}^{H \times W}$, where $D_i(u, v)$ is the depth at pixel $(u, v)$ [S1].
  - Camera intrinsics $K \in \mathbb{R}^{3 \times 3}$ [S1].
  - Camera extrinsics: rotation matrix $R_i \in \mathbb{R}^{3 \times 3}$ and translation vector $T_i \in \mathbb{R}^3$ [S1].
- Voxel size $v_s$ (scalar) [S1].

**Outputs:**
- Voxel feature grid $V$ with per-voxel feature vectors $V_{i,j,k} \in \mathbb{R}^C$ [S1].

**Data Structures:**
- A dense point cloud $P$ in world space, where each 3D point is associated with its corresponding image feature [S1].
- A mapping from voxel indices $(i, j, k)$ to a set of features $S_{i,j,k}$ and a count $|S_{i,j,k}|$ [S1].

**Steps:**

1. **Unprojection to Camera Coordinates:**
   For each view $i$ and each pixel $(u, v)$:
   - Retrieve depth $d = D_i(u, v)$ and image feature $f = F_i(u, v)$ [S1].
   - Compute normalized camera coordinates:
     $$
     \begin{bmatrix} x_c \\ y_c \\ z_c \end{bmatrix} = d \cdot K^{-1} \begin{bmatrix} u \\ v \\ 1 \end{bmatrix}
     $$
     [S1]

2. **Transformation to World Coordinates:**
   Transform the camera-space point $P_{cam} = [x_c, y_c, z_c]^T$ to world space:
   $$
   P_{world} = R_i P_{cam} + T_i
   $$
   [S1]
   Store the tuple $(P_{world}, f)$ in the global point cloud $P$ [S1].

3. **Voxelization:**
   For each point $p = (x_p, y_p, z_p) \in P$:
   - Compute integer voxel indices:
     $$
     i = \text{rnd}\left(\frac{x_p}{v_s}\right), \quad j = \text{rnd}\left(\frac{y_p}{v_s}\right), \quad k = \text{rnd}\left(\frac{z_p}{v_s}\right)
     $$
     where $\text{rnd}(\cdot)$ denotes rounding to the nearest integer [S1].
   - Assign the feature $f$ to the set $S_{i,j,k}$ associated with voxel $(i, j, k)$ [S1].

4. **Feature Aggregation:**
   For each occupied voxel $(i, j, k)$:
   - Let $S_{i,j,k}$ be the set of features $\{f_p\}$ from all points falling into this voxel [S1].
   - Compute the aggregated voxel feature $V_{i,j,k}$ by averaging along the channel dimension:
     $$
     V_{i,j,k} = \frac{1}{|S_{i,j,k}|} \sum_{p \in S_{i,j,k}} f_p
     $$
     [S1]
   - If $|S_{i,j,k}| = 0$, the voxel is empty [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def construct_feature_volume(views, voxel_size):
    """
    views: list of dicts, each containing:
        'F': image feature map (H, W, C)
        'D': depth map (H, W)
        'K': camera intrinsics (3, 3)
        'R': rotation matrix (3, 3)
        'T': translation vector (3,)
    voxel_size: float
    Returns:
        V: voxel feature grid (I, J, K, C)
        grid_shape: tuple (I, J, K)
    """
    world_points = []
    world_features = []

    for view in views:
        F = view['F']
        D = view['D']
        K_inv = np.linalg.inv(view['K'])
        R = view['R']
        T = view['T']

        H, W, C = F.shape
        u, v = np.meshgrid(np.arange(W), np.arange(H), indexing='xy')
        u = u.flatten()
        v = v.flatten()
        d = D.flatten()
        features = F.reshape(-1, C)

        # Unproject to camera space: P_cam = d * K^{-1} @ [u, v, 1]^T
        ones = np.ones_like(u)
        cam_coords = d[:, np.newaxis] * (K_inv @ np.stack([u, v, ones], axis=0)).T

        # Transform to world space: P_world = R * P_cam + T
        world_coords = (cam_coords @ R.T) + T

        world_points.append(world_coords)
        world_features.append(features)

    if not world_points:
        return np.array([]), (0, 0, 0)

    all_points = np.vstack(world_points)
    all_features = np.vstack(world_features)

    # Voxelization: indices = round(point / voxel_size)
    indices = np.round(all_points / voxel_size).astype(np.int32)

    # Determine grid bounds
    min_idx = indices.min(axis=0)
    max_idx = indices.max(axis=0)
    grid_shape = tuple((max_idx - min_idx + 1).tolist())

    # Offset indices to be non-negative
    relative_indices = indices - min_idx

    # Aggregation
    voxel_data = {}
    for idx, feat in zip(relative_indices, all_features):
        key = tuple(idx)
        if key not in voxel_data:
            voxel_data[key] = {'sum': np.zeros_like(feat), 'count': 0}
        voxel_data[key]['sum'] += feat
        voxel_data[key]['count'] += 1

    # Initialize output grid with zeros
    V = np.zeros((*grid_shape, all_features.shape[1]))

    for key, data in voxel_data.items():
        if data['count'] > 0:
            V[key] = data['sum'] / data['count']

    return V, grid_shape
```

**Parameters:**
- $v_s$: Voxel size. Not given in the sources.
- $K$: Camera intrinsics. Provided by the camera model. [G]
- $R_i, T_i$: Camera extrinsics. Provided by the pose estimation or SfM pipeline. [G]
- $D_i$: Predicted depth. Output of a depth estimation network. [G]

**Edge Cases and Pitfalls:**
- **Empty Voxels:** Voxels with no points have no defined feature. [G] Typically initialized to zero.
- **Precision:** Floating-point errors in unprojection and transformation may cause points near voxel boundaries to jitter. Using `round` (nearest integer) is specified, which is more robust than floor for centered voxels. [S1]
- **Multi-view Conflict:** Points from different views may project to the same voxel. Average pooling resolves this by blending features. [S1]

## Subtopics

### [Global Feature Volume Construction](14754-global-feature-volume-construction.md)
Builds a dense 3D feature volume $F_v$ from multi-view 2D features to provide global contextual information for downstream volume rendering tasks. It differs from local feature aggregation by first lifting 2D features into a 3D grid via projection and statistical pooling, then regularizing the sparse result with a 3D U-Net.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Converts unstructured point clouds into a regular grid suitable for 3D convolutions. Handles multi-view data naturally via averaging. [G]
- **Limitations:** Resolution is limited by voxel size. Fine details may be lost if $v_s$ is large. Sparse regions may have noisy features if few points project to them. [G]
- **When to use:** When a dense 3D feature representation is needed for 3D CNNs or volume rendering, and multi-view consistency is desired. [G]
- **When not to use:** When high-frequency details are critical and voxelization causes aliasing, or when memory is extremely constrained. [G]

## Background
Originated in VolSplat, which rethinks feed-forward 3D Gaussian Splatting with voxel-aligned prediction [S1]. The method leverages standard computer vision techniques (unprojection, rigid transformation, voxelization) to bridge 2D image features and 3D volumetric reasoning [S1].

## Sources
[S1] VolSplat: Rethinking Feed-Forward 3D Gaussian Splatting with Voxel-Aligned Prediction, https://doi.org/10.1007/978-3-032-37232-1_1, fetched 2026-10-26
