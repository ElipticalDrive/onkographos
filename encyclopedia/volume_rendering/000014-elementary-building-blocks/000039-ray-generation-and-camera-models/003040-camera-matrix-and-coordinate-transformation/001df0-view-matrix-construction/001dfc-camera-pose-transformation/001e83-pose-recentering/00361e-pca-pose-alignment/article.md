# PCA Pose Alignment

Also called: PCA Pose Transformation, principal component pose transform

## Overview
PCA Pose Alignment transforms a set of camera poses so that the principal components of the camera positions lie on the XYZ axes, effectively recentering and rotating the coordinate system based on the data distribution [S1, S2, unofficial]. It sets the origin at the centroid of the camera positions and aligns the axes with the directions of maximum variance [S1, S2, unofficial]. This method is distinct from [Sim(3) Alignment via Procrustes Analysis](8526-sim-3-alignment-via-procrustes-analysis.md), which minimizes reconstruction error between corresponding points, and [Camera Space Normalization via Similarity Transform](11417-camera-space-normalization-via-similarity-transform.md), which explicitly aligns the world up axis to the camera's negative Y axis [S1, unofficial]. It is a standard step in [Pose Recentering](1786-pose-recentering.md) pipelines to normalize input data for neural rendering [S1, S2, unofficial].

## How it works
**Inputs:** A set of $N$ camera poses, represented as $N$ matrices of shape $(3, 4)$ or $(4, 4)$, where each matrix $P_i$ maps world coordinates to camera coordinates (or vice versa, depending on convention; here we assume $P_i$ represents the transformation from world to camera, or the pose in world space. The sources extract translation vectors $t$ from $P_i[:, :3, 3]$, which typically corresponds to the camera center in world coordinates if $P_i$ is a World-to-Camera matrix, or the camera position if $P_i$ is a Camera-to-World matrix. Given the context of "camera positions" and "recentering", we treat $t$ as the 3D position of the camera in the world coordinate system).

**Outputs:** A transformed set of poses $P'_i$ and a $4 \times 4$ transformation matrix $T$ such that $P'_i = T P_i$ (applied appropriately to homogeneous coordinates).

**Data Structures:**
- `poses`: Array of shape $(N, 3, 4)$ or $(N, 4, 4)$. We assume $(N, 4, 4)$ for consistency with homogeneous transformations, where the last row is $[0, 0, 0, 1]$.
- `t`: Array of shape $(N, 3)$ containing the camera positions.
- `t_mean`: Array of shape $(3,)$.
- `cov`: Array of shape $(3, 3)$.
- `eigenvalues`: Array of shape $(3,)$.
- `eigenvectors`: Array of shape $(3, 3)$, where columns are eigenvectors.
- `rot`: Array of shape $(3, 3)$.
- `transform`: Array of shape $(4, 4)$.

**Steps:**

1.  **Extract Positions:** Extract the translation vectors (camera positions) $t$ from the poses. If poses are $(N, 4, 4)$, $t_i = P_i[:3, 3]$.
2.  **Compute Mean:** Calculate the mean position $t_{mean} = \frac{1}{N} \sum_{i=1}^N t_i$.
3.  **Center Positions:** Subtract the mean from each position: $t'_i = t_i - t_{mean}$.
4.  **Compute Covariance:** Calculate the covariance matrix $C = (t')^T t'$. Note: Some implementations use $t' (t')^T$, but the sources specify `t.T @ t`, which implies $t'$ is $(N, 3)$ and $C$ is $(3, 3)$.
5.  **Eigendecomposition:** Compute the eigenvalues $\lambda$ and eigenvectors $v$ of $C$.
6.  **Sort Eigenvectors:** Sort the eigenvectors in descending order of their corresponding eigenvalues. Let the sorted eigenvectors be $v_1, v_2, v_3$. Construct the rotation matrix $R = [v_1, v_2, v_3]$.
7.  **Ensure Proper Rotation:** If $\det(R) < 0$, flip the sign of the third eigenvector (z-axis) to ensure a proper rotation (determinant +1). Specifically, $R = \text{diag}(1, 1, -1) R$.
8.  **Construct 3x4 Transform:** Construct the linear part of the transformation. The rotation is $R$. The translation is $-R t_{mean}$. The $3 \times 4$ matrix is $M = [R | -R t_{mean}]$.
9.  **Apply Transformation:** Apply $M$ to the centered poses. If poses are in world-to-camera format, the transformation might need to be applied inversely or carefully. However, the sources describe applying the transform to "padded poses". Assuming the poses represent the camera's position and orientation in the world, and we want to transform the world coordinates, the new pose $P'_i$ is obtained by transforming the camera center and orientation. The sources state: `poses_recentered = unpad_poses(transform @ pad_poses(poses))`. This implies a left-multiplication by the $4 \times 4$ transform matrix.
10. **Coordinate System Flip:** Check the mean z-component of the y-axis of the recentered poses. If the average z-component of the camera's y-axis (up vector) is negative, flip the coordinate system to ensure a consistent handedness or orientation. The flip matrix is $F = \text{diag}(1, -1, -1, 1)$. Update $P' = F P'$ and $T = F T$.
11. **Scale:** Scale the poses to fit within the $[-1, 1]^3$ cube. Calculate the scale factor $s = 1 / \max_i (\|t'_i\|_\infty)$ or $s = 1 / \max_i (\max(|t'_{ix}|, |t'_{iy}|, |t'_{iz}|))$. The sources say `max(abs(poses_recentered[:, :3, 3]))`, which is the maximum absolute value of any coordinate among all camera centers. Update the positions: $t''_i = s t'_i$. Update the transform matrix $T$ by pre-multiplying with $\text{diag}(s, s, s, 1)$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def pca_pose_alignment(poses):
    """
    Aligns camera poses using PCA on camera positions.
    
    Args:
        poses: np.ndarray of shape (N, 4, 4) or (N, 3, 4). 
               If (N, 3, 4), it is padded to (N, 4, 4) with [0,0,0,1] as the last row.
               
    Returns:
        transformed_poses: np.ndarray of shape (N, 4, 4)
        transform_matrix: np.ndarray of shape (4, 4)
    """
    # Handle input shape
    if poses.shape[1] == 3:
        # Pad to 4x4
        padded = np.zeros((poses.shape[0], 4, 4))
        padded[:, :3, :] = poses
        padded[:, 3, 3] = 1.0
        poses = padded
    
    # Step 1: Extract camera positions (translation vectors)
    # Assuming poses are World-to-Camera or Camera-to-World?
    # Sources extract t from poses[:, :3, 3]. 
    # In standard CV, if P is World-to-Camera, P[:3, 3] is -R^T * t_world.
    # If P is Camera-to-World, P[:3, 3] is t_world.
    # The context "camera positions" and "recentering" suggests t is the world position.
    # We assume the input poses provide the camera center in world coordinates at [:3, 3].
    t = poses[:, :3, 3]  # Shape (N, 3)
    
    # Step 2: Compute mean
    t_mean = np.mean(t, axis=0)  # Shape (3,)
    
    # Step 3: Center positions
    t_centered = t - t_mean  # Shape (N, 3)
    
    # Step 4: Compute covariance matrix
    # Covariance C = (t_centered)^T @ t_centered
    # Note: t_centered is (N, 3). t_centered.T is (3, N).
    # C is (3, 3).
    cov = t_centered.T @ t_centered
    
    # Step 5 & 6: Eigendecomposition and Sorting
    eigenvalues, eigenvectors = np.linalg.eigh(cov)
    # eigh returns sorted in ascending order. We need descending.
    idx = np.argsort(eigenvalues)[::-1]
    eigenvalues = eigenvalues[idx]
    eigenvectors = eigenvectors[:, idx]
    
    # Construct rotation matrix R from sorted eigenvectors (columns)
    R = eigenvectors  # Shape (3, 3)
    
    # Step 7: Ensure proper rotation (det(R) == 1)
    if np.linalg.det(R) < 0:
        # Flip the z-axis (third column)
        R[:, 2] *= -1
        
    # Step 8: Construct 3x4 transform matrix [R | -R @ t_mean]
    translation_part = -R @ t_mean
    M_3x4 = np.hstack([R, translation_part.reshape(3, 1)])  # Shape (3, 4)
    
    # Step 9: Apply transformation to poses
    # We need to apply this to the homogeneous coordinates.
    # The transform matrix T is 4x4.
    # T = [ R, -R @ t_mean ]
    #     [ 0, 1           ]
    T = np.eye(4)
    T[:3, :3] = R
    T[:3, 3] = translation_part
    
    # Apply T to poses. 
    # If poses are 4x4 matrices, we multiply T @ pose.
    # This transforms the coordinate system.
    transformed_poses = T @ poses  # Shape (N, 4, 4)
    
    # Step 10: Coordinate system flip based on y-axis orientation
    # Check mean z-component of the y-axis of the transformed poses.
    # The y-axis of the camera is the second column of the rotation part of the pose.
    # If poses are World-to-Camera, the rotation part is R_cam.
    # If poses are Camera-to-World, the rotation part is R_world.
    # The sources say "mean z-component of the y-axis of poses_recentered".
    # Let's assume the rotation part of the pose is in the top-left 3x3.
    # The y-axis is the second column (index 1).
    y_axes = transformed_poses[:, :3, 1]  # Shape (N, 3)
    mean_z_y_axis = np.mean(y_axes[:, 2])
    
    if mean_z_y_axis < 0:
        # Flip matrix diag(1, -1, -1, 1)
        F = np.diag([1, -1, -1, 1])
        transformed_poses = F @ transformed_poses
        T = F @ T
        
    # Step 11: Scale to fit in [-1, 1]^3
    # Calculate max absolute value of any coordinate in the camera centers
    centers = transformed_poses[:, :3, 3]
    max_abs = np.max(np.abs(centers))
    
    if max_abs > 0:
        scale_factor = 1.0 / max_abs
    else:
        scale_factor = 1.0
        
    # Update transform matrix with scaling
    S = np.diag([scale_factor, scale_factor, scale_factor, 1])
    T = S @ T
    
    # Update poses with scaling
    # Only scale the positions and the rotation part? 
    # Scaling the entire pose matrix T @ pose scales the translation and rotation.
    # However, rotation matrices should remain orthonormal. 
    # The sources say: "Scale positions: poses_recentered[:, :3, 3] *= scale_factor."
    # And "Update transform: transform = diag(...) @ transform".
    # This implies the transform matrix T accumulates the scale, and the poses are transformed by T.
    # So we just need to ensure T is correct. The poses are already transformed by T.
    # But wait, if we scale T, we must re-apply or update the poses?
    # The sources update T, then presumably use T for future transformations.
    # The returned `transformed_poses` should reflect the final state.
    # Let's re-apply the scale to the poses to be safe, or just return T.
    # The sources return `poses_recentered` and `transform`.
    # If we scaled T, the poses transformed by the *previous* T are not scaled.
    # We need to apply the scale to the poses as well.
    transformed_poses[:, :3, 3] *= scale_factor
    # Note: Scaling the rotation part is not done, only the translation.
    # But T includes the rotation. If we return T, it includes the scale.
    # If we return transformed_poses, they are scaled.
    
    return transformed_poses, T
```

**Parameters:**
- None. The method is deterministic given the input poses.

**Edge Cases and Pitfalls:**
- **Degenerate Input:** If all camera positions are identical, the covariance matrix is zero, and eigenvectors are undefined. [G] In this case, return the identity transform or raise an error.
- **Precision:** Eigendecomposition can be sensitive to numerical precision if the covariance matrix is ill-conditioned. [G] Use `np.linalg.eigh` for symmetric matrices.
- **Determinant Flip:** If the eigenvectors form a left-handed system, the determinant is -1. The flip step ensures a right-handed system.
- **Coordinate System Ambiguity:** The definition of "camera position" depends on whether the pose matrix is World-to-Camera or Camera-to-World. The sources assume $t$ is extracted from $[:3, 3]$. If the pose is World-to-Camera, $t$ is not the world position but $-R^T t_{world}$. The PCA should be performed on the world positions. If the input poses are World-to-Camera, one must first convert them to Camera-to-World (or extract the world position) before applying PCA. The sources do not explicitly state this conversion, implying the input `poses` already contain the world positions in $[:3, 3]$.

## Subtopics

### [Camera Space Normalization via Similarity Transform](11417-camera-space-normalization-via-similarity-transform.md)
Computes a 4x4 similarity transform matrix $T$ and a scalar scale to normalize camera poses (c2w) and optionally point clouds, aligning the world up axis to the camera's negative Y axis and centering the scene. It sets the coordinate system such that the average camera orientation defines the vertical axis, distinct from [PCA Pose Alignment](7626-pca-pose-alignment.md) which aligns principal components of the data distribution.

## Operating characteristics
- **Time Complexity:** $O(N)$ to extract positions and compute mean. $O(1)$ for covariance and eigendecomposition (fixed 3x3 matrix). $O(N)$ to apply transformation. Overall $O(N)$.
- **Memory Complexity:** $O(N)$ to store positions and transformed poses.
- **Strengths:** Simple, fast, and effective for normalizing camera distributions. It aligns the coordinate system with the data's principal axes, which can help neural networks learn more easily.
- **Limitations:** It only considers camera positions, not orientations. If the camera orientations are highly correlated with positions in a non-PCA way, this might not be optimal. It assumes the camera positions are the primary factor for normalization.
- **When to Use:** When the camera positions are the main source of variation and you want to normalize the scene geometry.
- **When Not to Use:** When the camera orientations are critical and not aligned with the position variance, or when the scene is not well-represented by the camera positions (e.g., sparse views).

## Background
PCA Pose Alignment is a technique used in multi-view stereo and neural rendering pipelines to normalize input data. It is commonly used in frameworks like [MultiNeRF](https://github.com/google-research/multinerf) [S1, unofficial] and [VolSplat](https://github.com/ziplab/VolSplat) [S2, unofficial]. The method leverages Principal Component Analysis to find the optimal rotation and translation for the camera positions.

## Sources
[S1] google-research/multinerf, https://github.com/google-research/multinerf, fetched 2026-10-26 (unofficial)
[S2] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
