# Camera Space Normalization via Similarity Transform

Also called: dataset normalization

## Overview
Computes a 4x4 similarity transform matrix $T$ and a scalar scale to normalize camera poses (c2w) and optionally point clouds, aligning the world up axis to the camera's negative Y axis and centering the scene [S1, unofficial]. It establishes a canonical coordinate system based on the average camera orientation, distinct from [PCA Pose Alignment](7626-pca-pose-alignment.md) which aligns principal components of the data distribution [S1, unofficial].

## How it works
**Inputs:**
- `c2w`: Array of shape $(N, 4, 4)$ containing camera-to-world matrices.
- `points`: Optional array of shape $(M, 3)$ containing world-space points.
- `strict_scaling`: Boolean flag determining the scaling function (`np.max` if `True`, `np.median` otherwise) [S1, unofficial].

**Outputs:**
- `T`: 4x4 similarity transform matrix.
- `scale`: Scalar scaling factor.
- `transformed_c2w`: Normalized camera matrices.
- `transformed_points`: Normalized points (if provided).

**Data Structures:**
- `c2w`: $4 \times 4$ homogeneous transformation matrices. Layout:
  $$
  \begin{bmatrix}
  R_{00} & R_{01} & R_{02} & t_x \\
  R_{10} & R_{11} & R_{12} & t_y \\
  R_{20} & R_{21} & R_{22} & t_z \\
  0 & 0 & 0 & 1
  \end{bmatrix}
  $$
  where $R$ is the $3 \times 3$ rotation submatrix and $t$ is the translation vector [S1, unofficial].

**Steps:**
1. **Extract Translation and Rotation:** Extract $R_i = \text{c2w}[i, :3, :3]$ and $t_i = \text{c2w}[i, :3, 3]$ for all cameras [S1, unofficial].
2. **Estimate World Up Axis:** Compute the average camera up vector in world space. The camera up axis in camera space is $[0, -1, 0]$ [S1, unofficial].
   $$
   \text{ups} = \sum_{i=0}^{N-1} R_i \cdot \begin{bmatrix} 0 \\ -1 \\ 0 \end{bmatrix}, \quad \text{world\_up} = \frac{\text{ups}}{\|\text{ups}\|}
   $$
3. **Compute Alignment Rotation ($R_{\text{align}}$):** Align `world_up` to the target camera up axis $\text{up\_camspace} = [0, -1, 0]^T$ using Rodrigues' formula [S1, unofficial].
   $$
   c = \text{up\_camspace} \cdot \text{world\_up}, \quad \text{cross} = \text{world\_up} \times \text{up\_camspace}
   $$
   Construct the skew-symmetric matrix $K$ from `cross`:
   $$
   K = \begin{bmatrix}
   0 & -k_z & k_y \\
   k_z & 0 & -k_x \\
   -k_y & k_x & 0
   \end{bmatrix}
   $$
   $$
   R_{\text{align}} = I + K + K^2 \cdot \frac{1}{1 + c}
   $$
4. **Handle Degenerate Case:** If $c \le -1$ (indicating the original data has a Y+ up axis), set $R_{\text{align}}$ to a 180-degree rotation about the X axis:
   $$
   R_{\text{align}} = \begin{bmatrix}
   -1 & 0 & 0 \\
   0 & 1 & 0 \\
   0 & 0 & 1
   \end{bmatrix}
   $$
   [S1, unofficial]
5. **Construct Translation Component:** Center the scene by computing a translation vector. If points are provided, center on the points; otherwise, center on the camera positions [G].
   $$
   \text{translate} = -\text{center}, \quad \text{transform}[:3, 3] = \text{translate}, \quad \text{transform}[:3, :3] = R_{\text{align}}
   $$
   [S1, unofficial]
6. **Rescale the Scene:** Compute the scale factor using the norms of the translated camera centers [S1, unofficial].
   $$
   \text{scale} = \frac{1.0}{\text{scale\_fn}(\|\text{t} + \text{translate}\|)}
   $$
   where $\text{scale\_fn}$ is `np.max` if `strict_scaling` is `True`, else `np.median`. Multiply the rotation and translation parts of the transform by `scale`:
   $$
   \text{transform}[:3, :] = \text{transform}[:3, :] \cdot \text{scale}
   $$
7. **Apply Transform:** Transform all camera matrices and points using $T$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def similarity_from_cameras(c2w, points=None, strict_scaling=True):
    """
    Computes a similarity transform to normalize camera poses.
    
    Args:
        c2w: np.ndarray of shape (N, 4, 4)
        points: np.ndarray of shape (M, 3) or None
        strict_scaling: bool, if True use max norm for scaling, else median
        
    Returns:
        T: np.ndarray of shape (4, 4)
        scale: float
        transformed_c2w: np.ndarray of shape (N, 4, 4)
        transformed_points: np.ndarray of shape (M, 3) or None
    """
    N = c2w.shape[0]
    
    # 1. Extract R and t
    Rs = c2w[:, :3, :3]  # (N, 3, 3)
    ts = c2w[:, :3, 3]   # (N, 3)
    
    # 2. Estimate world up axis
    cam_up = np.array([0.0, -1.0, 0.0])
    ups = np.sum(Rs @ cam_up, axis=0)
    world_up = ups / np.linalg.norm(ups)
    
    # 3. Compute R_align
    up_camspace = cam_up
    c = np.dot(up_camspace, world_up)
    cross = np.cross(world_up, up_camspace)
    
    K = np.array([
        [0, -cross[2], cross[1]],
        [cross[2], 0, -cross[0]],
        [-cross[1], cross[0], 0]
    ])
    
    if c <= -1.0:
        R_align = np.array([
            [-1.0, 0.0, 0.0],
            [0.0, 1.0, 0.0],
            [0.0, 0.0, 1.0]
        ])
    else:
        R_align = np.eye(3) + K + (K @ K) * (1.0 / (1.0 + c))
        
    # 4. Construct Transform Matrix
    # [G] Center the scene: use points if provided, else camera centers
    if points is not None:
        center = np.mean(points, axis=0)
    else:
        center = np.mean(ts, axis=0)
        
    translate = -center
    transform = np.eye(4)
    transform[:3, :3] = R_align
    transform[:3, 3] = translate
    
    # 5. Rescale
    norms = np.linalg.norm(ts + translate, axis=1)
    scale_val = 1.0 / (np.max(norms) if strict_scaling else np.median(norms))
    transform[:3, :] *= scale_val
    
    # 6. Apply to cameras
    transformed_c2w = np.einsum('ij,njk->nik', transform, c2w)
    
    transformed_points = None
    if points is not None:
        points_h = np.hstack([points, np.ones((points.shape[0], 1))])
        transformed_points_h = (transform @ points_h.T).T
        transformed_points = transformed_points_h[:, :3]
        
    return transform, scale_val, transformed_c2w, transformed_points
```

## Subtopics

### Full Normalization Pipeline
Combines camera-based similarity normalization and principal axes alignment to normalize both camera poses and point clouds [S1, unofficial].
1. Compute initial similarity transform $T_1$ from cameras using `similarity_from_cameras` [S1, unofficial].
2. Transform cameras using $T_1$ [S1, unofficial].
3. If points are provided, transform them using $T_1$ [S1, unofficial].
4. Compute principal axes transform $T_2$ from points using `align_principle_axes` (see [PCA Pose Alignment](7626-pca-pose-alignment.md)) [S1, unofficial].
5. Transform cameras and points using $T_2$ [S1, unofficial].
6. Return transformed cameras, points, and combined transform $T_2 @ T_1$ [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ for extracting rotations and computing the average up axis, where $N$ is the number of cameras [G].
- **Memory Complexity:** $O(N)$ to store the camera matrices and intermediate vectors [G].
- **Strengths:** Robustly aligns the vertical axis of the scene with the camera's up direction, which is crucial for rendering pipelines that assume a specific world orientation [G].
- **Limitations:** Relies on the assumption that the camera up vectors are consistent. If cameras are randomly oriented, the average up axis may be ill-defined [G]. The scaling factor choice (`max` vs `median`) affects the final scene size; `strict_scaling` ensures all points fit within a unit radius, while `median` is less sensitive to outliers [G].
- **When to use:** When initializing a neural rendering scene from SfM data to ensure consistent orientation and scale before training [G].

## Background
Implemented in [S1, unofficial] to standardize camera poses from Structure-from-Motion (SfM) outputs, which vary in scale, orientation, and centering, into a canonical coordinate system suitable for neural rendering optimization [G].

## Sources
[S1] meshroomHub/mrGSplat, https://github.com/meshroomHub/mrGSplat, fetched 2026-10-27 (unofficial)
