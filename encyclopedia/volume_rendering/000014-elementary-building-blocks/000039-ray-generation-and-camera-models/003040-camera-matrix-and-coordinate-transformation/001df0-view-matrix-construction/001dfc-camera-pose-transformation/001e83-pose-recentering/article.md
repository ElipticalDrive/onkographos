# Pose Recentering

Also called: LLFF Pose Normalization for NDC

## Overview
Pose Recentering transforms a set of camera poses so that their average pose becomes the identity transformation, effectively centering the scene at the origin and aligning the coordinate axes. This solves the problem of arbitrary camera coordinate systems in Structure-from-Motion (SfM) outputs, which can cause numerical instability and incorrect depth sampling in volume rendering. It differs from [RotateY](1720-rotatey.md) and [Rotation Transformation](2279-rotation-transformation.md) by simultaneously handling both translation (centering) and rotation (axis alignment) via a single rigid transformation derived from the statistical properties of the pose set, rather than applying fixed or incremental rotations.

## How it works
**Inputs:** A set of $N$ camera poses, typically stored as an array `poses` of shape $(N, 3, 4)$, where each row $i$ is a $3 \times 4$ matrix $[R_i | t_i]$ representing the world-to-camera or camera-to-world transformation (specifically, camera-to-world $c2w$ in the standard LLFF/NeRF context).
**Outputs:** A recentered set of poses `poses_centered` of shape $(N, 3, 4)$.

The method constructs an average pose matrix $P_{avg}$ that represents the "center" of the camera distribution, then applies the inverse of this matrix to all poses.

### Data Structures
- `poses`: NumPy array of shape $(N, 3, 4)$.
- `poses_homo`: NumPy array of shape $(N, 4, 4)$, created by appending a row $[0, 0, 0, 1]$ to each pose.
- `pose_avg`: NumPy array of shape $(3, 4)$, representing the average transformation.
- `pose_avg_homo`: NumPy array of shape $(4, 4)$, the homogeneous version of `pose_avg`.

### Steps
1. **Homogenize Input Poses:**
   Append the row $[0, 0, 0, 1]$ to each of the $N$ poses to create `poses_homo` of shape $(N, 4, 4)$.

2. **Compute Scene Center:**
   Calculate the mean of all translation vectors (the fourth column of the $3 \times 4$ poses, or the last row of the $4 \times 4$ homogeneous poses excluding the last element).
   $$ \text{center} = \frac{1}{N} \sum_{i=1}^{N} t_i $$
   where $t_i$ is the translation part of pose $i$. In code, this is `poses[..., 3].mean(0)`.

3. **Compute Average Z-Axis:**
   Calculate the mean of all pose z-axes (the third column of the $3 \times 4$ poses, or the third row of the $4 \times 4$ homogeneous poses excluding the last element). Normalize this vector.
   $$ z = \frac{\sum_{i=1}^{N} r_{i,2}}{\|\sum_{i=1}^{N} r_{i,2}\|} $$
   where $r_{i,2}$ is the z-axis vector of pose $i$.

4. **Compute Intermediate Y-Axis:**
   Calculate the mean of all pose y-axes (the second column of the $3 \times 4$ poses).
   $$ y' = \frac{1}{N} \sum_{i=1}^{N} r_{i,1} $$
   where $r_{i,1}$ is the y-axis vector of pose $i$.

5. **Compute X-Axis:**
   Compute the normalized cross product of $y'$ and $z$ to ensure orthogonality.
   $$ x = \frac{y' \times z}{\|y' \times z\|} $$
   Note: $y'$ is not used directly as the y-axis because it is not guaranteed to be orthogonal to $z$.

6. **Compute Final Y-Axis:**
   Compute the final y-axis using the cross product of $z$ and $x$ to maintain a right-handed orthonormal basis.
   $$ y = z \times x $$

7. **Construct Average Pose Matrix:**
   Stack the computed axes and center into a $3 \times 4$ matrix.
   $$ P_{avg} = [x, y, z, \text{center}] $$
   Shape: $(3, 4)$.

8. **Create Homogeneous Average Pose:**
   Append the row $[0, 0, 0, 1]$ to $P_{avg}$ to create $P_{avg\_homo}$ of shape $(4, 4)$.

9. **Invert Average Pose:**
   Compute the inverse of $P_{avg\_homo}$.
   $$ T_{inv} = P_{avg\_homo}^{-1} $$

10. **Transform Poses:**
    Left-multiply the inverse average pose with all homogenized poses.
    $$ P_{centered\_homo} = T_{inv} \times P_{homo} $$
    This results in an array of shape $(N, 4, 4)$.

11. **Truncate to 3x4:**
    Remove the last row (the homogeneous row) from each pose.
    $$ \text{poses\_centered} = P_{centered\_homo}[:, :3] $$
    Shape: $(N, 3, 4)$.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def recenter_poses(poses: np.ndarray) -> np.ndarray:
    """
    Recenter a set of camera poses so their average pose is identity.
    
    Args:
        poses: np.ndarray of shape (N, 3, 4), camera-to-world poses.
        
    Returns:
        np.ndarray of shape (N, 3, 4), recentered poses.
    """
    # Step 1: Homogenize input poses
    # Append [0, 0, 0, 1] to each pose to make them 4x4
    ones = np.ones((poses.shape[0], 1, 4))
    poses_homo = np.concatenate([poses, ones], axis=1)  # Shape: (N, 4, 4)
    
    # Step 2: Compute scene center (mean of translation vectors)
    # Translation is the 4th column of the 3x4 pose, or the last row of 4x4 pose (indices 0,1,2)
    center = poses_homo[:, 3, :3].mean(axis=0)  # Shape: (3,)
    
    # Step 3: Compute average z-axis
    # Z-axis is the 3rd column of the 3x4 pose, or the 3rd row of 4x4 pose (index 2)
    z_axes = poses_homo[:, 2, :3]  # Shape: (N, 3)
    z = z_axes.mean(axis=0)
    z = z / np.linalg.norm(z)  # Normalize
    
    # Step 4: Compute intermediate y-axis
    # Y-axis is the 2nd column of the 3x4 pose, or the 2nd row of 4x4 pose (index 1)
    y_axes = poses_homo[:, 1, :3]  # Shape: (N, 3)
    y_prime = y_axes.mean(axis=0)  # Shape: (3,)
    
    # Step 5: Compute x-axis
    x = np.cross(y_prime, z)
    x = x / np.linalg.norm(x)  # Normalize
    
    # Step 6: Compute final y-axis
    y = np.cross(z, x)
    
    # Step 7: Construct average pose matrix (3x4)
    pose_avg = np.stack([x, y, z, center], axis=1)  # Shape: (3, 4)
    
    # Step 8: Create homogeneous average pose (4x4)
    pose_avg_homo = np.vstack([pose_avg, [0, 0, 0, 1]])  # Shape: (4, 4)
    
    # Step 9: Invert average pose
    pose_avg_homo_inv = np.linalg.inv(pose_avg_homo)
    
    # Step 10: Transform all poses
    # Left-multiply: T_inv @ P_homo
    poses_centered_homo = pose_avg_homo_inv @ poses_homo  # Shape: (N, 4, 4)
    
    # Step 11: Truncate to 3x4
    poses_centered = poses_centered_homo[:, :3]  # Shape: (N, 3, 4)
    
    return poses_centered
```

### Parameters
- None. The method is deterministic based on the input poses.

### Edge Cases and Pitfalls
- **Non-Orthogonal Y-Axis:** The intermediate y-axis $y'$ is not guaranteed to be orthogonal to $z$. Using $y'$ directly would result in a non-orthonormal basis. The algorithm explicitly derives $x$ and $y$ via cross products to ensure orthonormality [S2, unofficial].
- **Degenerate Scene:** If all cameras are collinear or coplanar in a way that makes the mean z-axis zero or the cross product zero, normalization will fail. This is rare in typical SfM outputs but should be guarded against [G].
- **Coordinate System Convention:** The method assumes the input poses are camera-to-world ($c2w$). If the input is world-to-camera ($w2c$), the transformation logic must be inverted or adapted [S1, unofficial].

## Subtopics

### [Sim(3) Alignment via Procrustes Analysis](8526-sim-3-alignment-via-procrustes-analysis.md)
A method to align estimated camera poses to ground truth by computing a 3D similarity transformation (Sim(3)) using Procrustes analysis on camera pose locations. Unlike Pose Recentering which only applies a rigid transformation (rotation and translation) to center the scene, this method also scales the scene to match the ground truth scale. It is chosen when absolute scale information is available and needs to be recovered, whereas Pose Recentering is used when only relative pose structure is available and scale is arbitrary [8526-sim-3-alignment-via-procrustes-analysis.md].

### PCA Pose Alignment
A variant that transforms poses so that the principal components of the camera positions lie on the XYZ axes. It recenters the camera positions by subtracting their mean and then computes the eigendecomposition of the covariance matrix of the centered positions. The eigenvectors are sorted by eigenvalue and assigned to the rotation matrix. If the determinant of the rotation matrix is negative, the third axis is flipped to ensure a proper rotation. The transform matrix is constructed by concatenating the rotation matrix and the rotated negative mean position. A final check flips the y and z axes if the mean z-component of the y-axis of the recentered poses is negative, ensuring a consistent orientation. This method aligns the coordinate system with the data's principal axes rather than the average camera orientation, which can be beneficial for scenes with specific geometric structures [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ for computing means and cross products, where $N$ is the number of poses. The matrix inversion is $O(1)$ for a fixed $4 \times 4$ matrix. The final transformation is $O(N)$ for matrix multiplication.
- **Memory Complexity:** $O(N)$ to store the input and output poses. Temporary storage for homogenized poses is also $O(N)$.
- **Measured Performance:** The method is extremely fast, typically taking less than a millisecond for thousands of poses on a modern CPU. It is often used as a preprocessing step in NeRF pipelines [S1, unofficial].
- **Strengths:** Simple to implement, numerically stable for typical SfM outputs, effectively centers the scene for NDC-based rendering.
- **Limitations:** Assumes the average pose is a good representation of the scene center. May not work well for highly non-uniform camera distributions or scenes with large outliers. Does not handle scale recovery [S2, unofficial].
- **When to Use:** When preparing camera poses for volume rendering with NDC coordinates, especially for datasets like LLFF or Blender synthetic data where the camera distribution is roughly centered around the scene.
- **When Not to Use:** When absolute scale is required (use [Sim(3) Alignment via Procrustes Analysis](8526-sim-3-alignment-via-procrustes-analysis.md)), or when the camera distribution is highly skewed or degenerate.

## Background
Pose Recentering originated in the context of NeRF (Neural Radiance Fields) and LLFF (Look-Lock-Focus) datasets. It was popularized by the official NeRF implementation and subsequent variants like NerfPl. The method addresses the issue that SfM pipelines (like COLMAP) produce camera poses in an arbitrary coordinate system, which can lead to poor performance in neural rendering if the scene is not centered at the origin. The specific algorithm described here, often referred to as "LLFF Pose Normalization," is designed to work well with the NDC (Normalized Device Coordinates) system used in many NeRF implementations. It was first documented in the unofficial NeRF repository and later adopted by many other projects [S1, unofficial].

## Sources
[S1] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S2] kwea123/nerf_pl, https://github.com/kwea123/nerf_pl, fetched 2026-10-26 (unofficial)
[S3] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
