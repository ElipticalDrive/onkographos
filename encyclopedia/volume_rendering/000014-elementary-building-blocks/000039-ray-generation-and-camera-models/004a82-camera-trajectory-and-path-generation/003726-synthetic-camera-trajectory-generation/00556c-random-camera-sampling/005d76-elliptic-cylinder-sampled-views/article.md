# Elliptic Cylinder Sampled Views

Also called: Elliptic Cylinder View Sampling

## Overview
Elliptic Cylinder Sampled Views is a camera trajectory generation method that creates random viewpoints directed at a scene center for computing SDS (Score Distillation Sampling) loss, using an elliptical cylinder that fits the training view positions [S1]. It differs from its sibling [Warped Views](17410-warped-views.md) by generating novel, uncorrelated positions on a fitted cylinder rather than rotating existing camera poses [S1].

## How it works
**Inputs:** A set of $N$ training camera poses $\{P_i\}_{i=1}^N$, where each pose $P_i$ consists of a translation vector $t_i$ and a rotation matrix $R_i$ [S1].
**Outputs:** A single new camera pose $P_{new} = (t_{new}, R_{new})$ [S1].

**Data Structures:**
*   Camera Pose: Struct containing `translation` (vec3) and `rotation` (3x3 matrix) [G].
*   Scene Center: vec3, typically the mean of all training camera positions [G].

**Steps:**
1.  **Coordinate Alignment:** Transform all training camera positions to align with a world coordinate system where the positive Z-axis is perpendicular to the ground using Principal Component Analysis (PCA) [S1]. Let the transformed positions be $t'_i$.
2.  **Ellipse Fitting:** Determine the scaling factor for the elliptical axes by taking the 90th percentile of the absolute displacements from the scene center in the X-Y plane [S1]. Let this scale be $s$.
3.  **Vertical Bounds:** Set the vertical bounds for the Z-coordinate at the 10th and 90th percentiles of the Z-coordinates of the transformed training views, denoted as $z_{min}$ and $z_{max}$ [S1].
4.  **Random Translation Sampling:** Uniformly randomly select a camera translation vector $t_{new}$ in the aligned system [S1].
    *   Select an angle $\phi \in [0, 2\pi)$.
    *   Set $x_{new} = s \cos \phi$ and $y_{new} = s \sin \phi$.
    *   Set $z_{new}$ uniformly at random within $[z_{min}, z_{max}]$.
    *   $t_{new} = [x_{new}, y_{new}, z_{new}]^T$.
5.  **Rotation Computation:** Compute the camera’s rotation matrix $R_{new}$ [S1].
    *   **Look-at Vector:** Direct the look-at vector towards the scene center. In the aligned system, the direction is $-\text{normalize}(t_{new} - \text{center})$ [S1].
    *   **Up Vector:** Align the up vector with the average up vector of the input views. Compute the average up vector $\bar{u}$ from the training cameras' rotation matrices and normalize it [S1].
    *   Construct $R_{new}$ such that its columns correspond to the right, up, and forward vectors derived from the look-at and up vectors [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_elliptic_cylinder_view(training_poses, scene_center):
    """
    Generate a single random camera view using the Elliptic Cylinder method.
    
    Args:
        training_poses: List of dicts with 'translation' (vec3) and 'rotation' (3x3).
        scene_center: vec3 representing the center of the scene.
        
    Returns:
        dict with 'translation' (vec3) and 'rotation' (3x3).
    """
    # Step 1: Coordinate Alignment via PCA
    positions = np.array([p['translation'] for p in training_poses])
    mean_pos = np.mean(positions, axis=0)
    centered_pos = positions - mean_pos
    
    # Compute covariance matrix and eigenvectors
    cov = np.cov(centered_pos.T)
    eigenvalues, eigenvectors = np.linalg.eigh(cov)
    
    # Sort eigenvectors by eigenvalue (ascending)
    idx = np.argsort(eigenvalues)
    z_axis = eigenvectors[:, idx[0]]  # Smallest eigenvalue corresponds to ground normal
    
    # Construct rotation matrix to align z_axis with world Z [0,0,1]
    arbitrary = np.array([0, 0, 1]) if abs(z_axis[2]) < 0.9 else np.array([1, 0, 0])
    x_axis = np.cross(z_axis, arbitrary)
    x_axis = x_axis / np.linalg.norm(x_axis)
    y_axis = np.cross(x_axis, z_axis)
    
    R_align = np.column_stack((x_axis, y_axis, z_axis))
    aligned_positions = (R_align.T @ (positions - mean_pos).T).T
    
    # Step 2: Determine scaling factor (90th percentile of XY displacements)
    xy_displacements = np.sqrt(aligned_positions[:, 0]**2 + aligned_positions[:, 1]**2)
    scale = np.percentile(xy_displacements, 90)
    
    # Step 3: Set vertical bounds (10th and 90th percentiles of Z)
    z_coords = aligned_positions[:, 2]
    z_min = np.percentile(z_coords, 10)
    z_max = np.percentile(z_coords, 90)
    
    # Step 4: Uniformly randomly select a camera translation vector
    phi = np.random.uniform(0, 2 * np.pi)
    x_new = scale * np.cos(phi)
    y_new = scale * np.sin(phi)
    z_new = np.random.uniform(z_min, z_max)
    
    t_aligned = np.array([x_new, y_new, z_new])
    
    # Step 5: Compute rotation matrix
    # Look-at vector: towards scene center (origin in aligned space)
    look_at_dir = -t_aligned
    look_at_dir = look_at_dir / np.linalg.norm(look_at_dir)
    
    # Average up vector of input views
    up_vectors = np.array([p['rotation'][:, 1] for p in training_poses])
    avg_up = np.mean(up_vectors, axis=0)
    avg_up = avg_up / np.linalg.norm(avg_up)
    
    # Construct basis
    right = np.cross(avg_up, look_at_dir)
    right = right / np.linalg.norm(right)
    up = np.cross(look_at_dir, right)
    
    R_aligned = np.column_stack((right, up, look_at_dir))
    
    # Transform back to original coordinate system
    t_new = (R_align @ t_aligned) + mean_pos
    R_new = R_align @ R_aligned
    
    return {'translation': t_new, 'rotation': R_new}
```

**Parameters:**
*   Percentile for ellipse scaling: 90th [S1].
*   Percentile for vertical bounds: 10th and 90th [S1].
*   Number of sampled views: Fixed number, as per [Random Camera Sampling](15470-random-camera-sampling.md) [G].

**Edge Cases:**
*   **Degenerate Input:** If all training views are collinear or coplanar, PCA may fail to produce a stable ground normal [G]. Add a check for singular covariance matrix and fallback to a default up vector (e.g., $[0,1,0]$) [G].
*   **Precision:** Ensure numerical stability when computing cross products for near-parallel vectors [G].

## Subtopics

### [Warped Views](17410-warped-views.md)
Warped Views is a view sampling strategy that generates novel viewpoints by rotating training cameras around an estimated scene center axis. Unlike SDS sampled views, which may be too distant to provide sufficient overlap, Warped Views ensures large view overlapping by applying small, predefined angular perturbations to the original camera poses. It is primarily used in conjunction with Depth Warping Loss to enforce geometric consistency between slightly different perspectives of the same scene.

## Operating characteristics
*   **Time Complexity:** $O(N)$ for PCA and percentile calculations, where $N$ is the number of training views. Sampling a single view is $O(1)$ [G].
*   **Memory Complexity:** $O(N)$ to store training poses and intermediate statistics [G].
*   **Strengths:** Ensures sampled views are geometrically consistent with the input data distribution, providing good coverage for SDS loss [S1].
*   **Limitations:** Does not provide the large view overlapping required for depth warping losses, which is why Warped Views are used for those [S1].
*   **When to use:** When computing SDS loss for 3D Gaussian Splatting from sparse input views [S1].
*   **When not to use:** When large view overlapping is required for depth consistency losses [S1].

## Background
Originating from the SparseGS paper, this method is part of the UVR (Unsupervised View Regularization) framework designed to regularize 3DGS training from sampled views [S1]. It is specifically applied to compute the SDS loss, complementing other components like Pearson Depth Loss and Depth Warping Loss which use different sampling strategies [S1].

## Sources
[S1] Sparsegs: Sparse View Synthesis Using 3D Gaussian Splatting, https://doi.org/10.1109/3dv66043.2025.00100, fetched 2026-10-26
