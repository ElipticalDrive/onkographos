# PCA Pose Alignment

Also called: PCA Pose Transformation, Pose Transformation via PCA, principal component pose transform

## Overview
PCA Pose Alignment transforms a set of camera-to-world poses so that the principal components of their translation vectors lie on the XYZ axes [S1, unofficial][S2, unofficial][S3, unofficial]. It re-centers the poses, rotates them based on the eigenvectors of the covariance matrix, ensures a consistent coordinate system orientation, and scales them to fit within the $[-1, 1]^3$ cube [S1, unofficial]. It is distinct from [Sim(3) Alignment via Procrustes Analysis](8526-sim-3-alignment-via-procrustes-analysis.md) which aligns poses using Procrustes analysis [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: `poses`, an array of shape $(N, 3, 4)$ representing $N$ camera-to-world transformation matrices [S3, unofficial].
- Output: `poses_recentered`, the transformed poses of shape $(N, 3, 4)$, and `transform`, a $4 \times 4$ transformation matrix [S1, unofficial][S3, unofficial].

**Data Structures:**
- `poses`: $(N, 3, 4)$ float array. Rows 0-2 are rotation/scale, row 3 is translation.
- `transform`: $4 \times 4$ float array.
- `t`: $(N, 3)$ float array of translation vectors.
- `t_mean`: $(3,)$ float array of mean translation.
- `rot`: $3 \times 3$ float array of rotation matrix.

**Steps:**
1. Extract translation vectors $t$ from `poses` ($t = \text{poses}[:, :3, 3]$) [S1, unofficial][S2, unofficial][S3, unofficial].
2. Compute mean translation $t_{\text{mean}} = \frac{1}{N} \sum_{i=1}^N t_i$ [S1, unofficial][S2, unofficial][S3, unofficial].
3. Center translations: $t = t - t_{\text{mean}}$ [S1, unofficial][S2, unofficial][S3, unofficial].
4. Compute covariance matrix $C = t^T t$ [S1, unofficial][S2, unofficial][S3, unofficial].
5. Perform eigendecomposition on $C$ to get eigenvalues $\lambda$ and eigenvectors $v$ [S1, unofficial][S2, unofficial][S3, unofficial].
6. Sort eigenvectors in descending order of eigenvalues: $v_{\text{sorted}} = v[:, \text{argsort}(\lambda)[::-1]]$ [S2, unofficial][S3, unofficial].
7. Set rotation matrix $\text{rot} = v_{\text{sorted}}^T$ [S3, unofficial].
8. If $\det(\text{rot}) < 0$, flip the third axis: $\text{rot} = \text{diag}([1, 1, -1]) \cdot \text{rot}$ [S1, unofficial][S2, unofficial][S3, unofficial].
9. Construct $3 \times 4$ transform matrix: $T_{3 \times 4} = [\text{rot} \mid \text{rot} \cdot (-t_{\text{mean}})]$ [S1, unofficial][S3, unofficial].
10. Apply transform to padded poses: `poses_recentered` = $\text{unpad\_poses}(T_{3 \times 4} \cdot \text{pad\_poses}(\text{poses}))$ [S1, unofficial][S3, unofficial].
11. Pad $T_{3 \times 4}$ to $4 \times 4$: $\text{transform} = \begin{bmatrix} T_{3 \times 4} \\ 0 & 0 & 0 & 1 \end{bmatrix}$ [S1, unofficial][S3, unofficial].
12. Check mean y-axis z-component of `poses_recentered`. If $\text{mean}(\text{poses\_recentered}[:, 2, 1]) < 0$, flip coordinate system:
    - `poses_recentered` = $\text{diag}([1, -1, -1, 1]) \cdot \text{poses\_recentered}$ [S1, unofficial][S2, unofficial][S3, unofficial].
    - $\text{transform} = \text{diag}([1, -1, -1, 1]) \cdot \text{transform}$ [S1, unofficial][S2, unofficial][S3, unofficial].
13. Calculate scale factor: $s = 1 / \max(|\text{poses\_recentered}[:, :3, 3]|)$ [S1, unofficial].
14. Scale positions: $\text{poses\_recentered}[:, :3, 3] *= s$ [S1, unofficial].
15. Update transform: $\text{transform} = \text{diag}(s, s, s, 1) \cdot \text{transform}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def transform_poses_pca(poses: np.ndarray) -> tuple[np.ndarray, np.ndarray]:
    """
    Transforms camera poses so that the principal components of the camera positions 
    lie on the XYZ axes, re-centers them, ensures consistent orientation, and scales 
    to fit in [-1, 1]^3.
    
    Args:
        poses: (N, 3, 4) array of camera-to-world transforms.
        
    Returns:
        poses_recentered: (N, 3, 4) transformed poses.
        transform: (4, 4) transformation matrix.
    """
    # 1. Extract translations and compute mean
    t = poses[:, :3, 3]  # (N, 3)
    t_mean = np.mean(t, axis=0)  # (3,)
    
    # 2. Center translations
    t_centered = t - t_mean  # (N, 3)
    
    # 3. Compute covariance matrix and eigendecomposition
    cov = t_centered.T @ t_centered  # (3, 3)
    eigenvalues, eigenvectors = np.linalg.eigh(cov)  # eigenvalues: (3,), eigenvectors: (3, 3)
    
    # 4. Sort eigenvectors in descending order of eigenvalues
    sorted_indices = np.argsort(eigenvalues)[::-1]
    rot = eigenvectors[:, sorted_indices].T  # (3, 3)
    
    # 5. Ensure proper rotation (determinant = 1)
    if np.linalg.det(rot) < 0:
        rot[2, :] *= -1  # Flip third axis
        
    # 6. Construct 3x4 transform matrix
    # T_3x4 = [rot | rot @ -t_mean]
    translation_part = rot @ (-t_mean)  # (3,)
    T_3x4 = np.hstack([rot, translation_part.reshape(3, 1)])  # (3, 4)
    
    # 7. Apply transform to poses
    # Pad poses to (N, 4, 4) by appending [0, 0, 0, 1]
    ones = np.ones((poses.shape[0], 1, 4))
    ones[:, 0, 3] = 1.0
    poses_padded = np.vstack([poses, ones])  # (N+1, 4, 4)
    
    # Apply transform
    poses_transformed_padded = T_3x4 @ poses_padded  # (N+1, 4, 4)
    
    # Unpad: take first 3 rows
    poses_recentered = poses_transformed_padded[:3, :, :]  # (3, N+1, 4) -> transpose to (N+1, 3, 4)
    poses_recentered = poses_recentered.transpose(1, 0, 2)  # (N+1, 3, 4)
    
    # Remove the homogeneous row (last element)
    poses_recentered = poses_recentered[:-1]  # (N, 3, 4)
    
    # 8. Pad transform to 4x4
    transform = np.vstack([T_3x4, np.array([0, 0, 0, 1])])  # (4, 4)
    
    # 9. Check and flip coordinate system if needed
    # Check mean z-component of the y-axis of recentered poses
    mean_y_z = np.mean(poses_recentered[:, 2, 1])
    if mean_y_z < 0:
        flip_matrix = np.diag([1, -1, -1, 1])
        poses_recentered_padded = np.vstack([poses_recentered, np.eye(4)])
        poses_recentered_padded = flip_matrix @ poses_recentered_padded
        poses_recentered = poses_recentered_padded[:-1]
        transform = flip_matrix @ transform
        
    # 10. Scale to fit in [-1, 1]^3
    max_abs_pos = np.max(np.abs(poses_recentered[:, :3, 3]))
    if max_abs_pos == 0:
        scale_factor = 1.0
    else:
        scale_factor = 1.0 / max_abs_pos
        
    poses_recentered[:, :3, 3] *= scale_factor
    scale_matrix = np.diag([scale_factor, scale_factor, scale_factor, 1])
    transform = scale_matrix @ transform
    
    return poses_recentered, transform
```

**Parameters:**
- `poses`: Input camera-to-world transforms array of shape $(N, 3, 4)$ [S3, unofficial].

**Edge Cases and Pitfalls:**
- Improper rotation handedness can occur if the determinant of the rotation matrix is negative. The algorithm explicitly corrects this by flipping the third axis [S3, unofficial].
- Coordinate system chirality may be inverted relative to the original capture setup. The algorithm detects this by checking the mean z-component of the y-axis and applies a $[1, -1, -1, 1]$ diagonal flip to restore consistency [S3, unofficial].
- If all camera positions are identical, the covariance matrix is zero, and eigendecomposition may be unstable. Not given in the sources.

## Subtopics

### [Camera Space Normalization via Similarity Transform](11417-camera-space-normalization-via-similarity-transform.md)
Computes a 4x4 similarity transform matrix $T$ and a scalar scale to normalize camera poses (c2w) and optionally point clouds, aligning the world up axis to the camera's negative Y axis and centering the scene. It establishes a canonical coordinate system based on the average camera orientation, distinct from [PCA Pose Alignment](7626-pca-pose-alignment.md) which aligns principal components of the data distribution.

## Operating characteristics
- Time complexity: $O(N)$ for mean and covariance computation, $O(1)$ for eigendecomposition (fixed 3x3 matrix), $O(N)$ for transformation application.
- Memory complexity: $O(N)$ for storing translations and transformed poses.
- Measured performance: Not given in the sources.
- Scaling: Linear with number of poses $N$.
- Strengths: Simple, fast, and effective for aligning camera poses based on spatial distribution.
- Limitations: Sensitive to outliers in camera positions. Assumes the principal components are meaningful for alignment.
- When to use: When camera positions are distributed along principal axes and a canonical coordinate system is needed.
- When not to use: When camera orientations are more important than positions, or when outliers are present.

## Background
Origin: Not given in the sources.
History and development: Not given in the sources.
Where it is used: Used in NeRF-related projects for pose normalization [S1, unofficial][S2, unofficial][S3, unofficial].

## Sources
[S1] google-research/multinerf, https://github.com/google-research/multinerf, fetched 2026-10-26 (unofficial)
[S2] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
[S3] pulangk97/VoLoDSplat, https://github.com/pulangk97/VoLoDSplat, fetched 2026-10-27 (unofficial)
