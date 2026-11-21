# Pose Registration Metrics

Also called: Pose Error Metrics, Rotation Error, Translation Error

## Overview
Pose Registration Metrics quantify the alignment accuracy between estimated camera poses and ground-truth poses, serving as the evaluation standard for methods like [Sim(3) Alignment via Procrustes Analysis](8526-sim-3-alignment-via-procrustes-analysis.md). They solve the problem of measuring how well an estimated trajectory matches reality in terms of both orientation and position. Unlike sibling methods that focus on alignment optimization or parametric fitting, these metrics provide the scalar error values used to judge the success of the alignment.

## How it works
The method computes two primary error components: translation error and rotation error. These are calculated per pose pair or aggregated over a trajectory.

**Inputs:**
- $T_{gt}$: Ground-truth translation vector (3D position).
- $\hat{T}$: Estimated translation vector (3D position).
- $R_{gt}$: Ground-trotation rotation matrix ($3 \times 3$).
- $\hat{R}$: Estimated rotation matrix ($3 \times 3$).

**Outputs:**
- $T_{err}$: Translation error (scalar).
- $R_{err}$: Rotation error (scalar, in radians).

**Data Structures:**
- Rotation matrices are stored as $3 \times 3$ orthogonal matrices with determinant 1.
- Translation vectors are stored as 3-element floating-point arrays.

**Formulas:**

1.  **Translation Error ($T_{err}$):**
    The Euclidean distance between the estimated and ground-truth positions [S1, unofficial][S2]:
    $$ T_{err} = \| \hat{T} - T_{gt} \|_2 $$
    Note: On datasets where positions are not in metric space, this error has no physical units [S2].

2.  **Rotation Error ($R_{err}$):**
    The geodesic distance on the rotation manifold, representing the angle required to align the estimated rotation with the ground truth [S1, unofficial][S2]:
    $$ R_{err} = \arccos \left( \frac{\text{tr}(R_{gt}^\top \hat{R}) - 1}{2} \right) $$
    Alternatively expressed using the inverse [S2]:
    $$ R_{err} = \arccos \left( \frac{\text{tr}(R_{gt}^{-1} \hat{R}) - 1}{2} \right) $$
    Where $\text{tr}(\cdot)$ denotes the trace of a matrix. The result is in radians [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def compute_translation_error(est_pos: np.ndarray, gt_pos: np.ndarray) -> float:
    """
    Computes Euclidean distance between estimated and ground truth positions.
    Args:
        est_pos: 3D array, estimated position [x, y, z]
        gt_pos: 3D array, ground truth position [x, y, z]
    Returns:
        float: Euclidean distance
    """
    diff = est_pos - gt_pos
    return np.linalg.norm(diff)

def compute_rotation_error(est_rot: np.ndarray, gt_rot: np.ndarray) -> float:
    """
    Computes geodesic rotation error in radians.
    Args:
        est_rot: 3x3 numpy array, estimated rotation matrix
        gt_rot: 3x3 numpy array, ground truth rotation matrix
    Returns:
        float: Rotation error in radians
    """
    # Compute trace of R_gt^T * R_est
    # Note: R_gt^T is equivalent to R_gt^-1 for rotation matrices
    product = gt_rot.T @ est_rot
    trace_val = np.trace(product)
    
    # Clamp value to [-1, 1] to handle floating point errors in arccos
    cos_angle = (trace_val - 1.0) / 2.0
    cos_angle = np.clip(cos_angle, -1.0, 1.0)
    
    return math.acos(cos_angle)

def evaluate_pose_pair(est_pose, gt_pose):
    """
    Evaluates a single pose pair.
    Args:
        est_pose: dict or object with 'pos' (3D) and 'rot' (3x3)
        gt_pose: dict or object with 'pos' (3D) and 'rot' (3x3)
    Returns:
        tuple: (translation_error, rotation_error)
    """
    t_err = compute_translation_error(est_pose['pos'], gt_pose['pos'])
    r_err = compute_rotation_error(est_pose['rot'], gt_pose['rot'])
    return t_err, r_err
```

**Parameters:**
- No specific tuning parameters for the basic error calculation. Aggregation metrics (like RMSE) may require trajectory length $N$.

**Edge Cases:**
- **Numerical Stability:** The argument to $\arccos$ must be clamped to $[-1, 1]$ to prevent NaNs due to floating-point precision errors in matrix multiplication [G].
- **Non-Metric Space:** Translation error units are undefined if the coordinate system is not metric [S2].

## Subtopics

### [Absolute Trajectory Error Computation](12727-absolute-trajectory-error-computation.md)
Computes the Root Mean Square Error (RMSE) of the translation component between a ground truth trajectory and an estimated trajectory after optimal alignment. It quantifies the global scale and drift of an ego-motion estimation system by measuring the integrated distance between pose estimates and ground truth. Unlike relative error metrics, ATE evaluates the absolute consistency of the trajectory in the world frame.

## Operating characteristics
- **Time Complexity:** $O(N)$ for $N$ poses, as each pose is processed independently.
- **Memory Complexity:** $O(1)$ per pose pair; $O(N)$ if storing all errors for aggregation.
- **Strengths:** Simple to implement; provides intuitive geometric interpretation (meters for translation, radians for rotation).
- **Limitations:** Translation error lacks physical meaning in non-metric spaces [S2]. Does not account for temporal consistency or drift direction.
- **When to use:** Standard evaluation for pose estimation accuracy after alignment.
- **When not to use:** When relative drift or scale consistency is the primary concern (use Relative Pose Error instead).

## Background
Originating from the evaluation of visual odometry and SLAM systems, these metrics have become standard in novel view synthesis and 3D reconstruction papers. The specific formulation of rotation error as the geodesic distance on the rotation manifold is widely cited in recent works such as SPARF [S2] and VoxelTTO [S1, unofficial]. The translation error is a standard Euclidean norm, though its interpretation varies based on the dataset's coordinate system [S2].

## Sources
[S1] VoxelTTO: Voxel-Aligned Feed-Forward 3D Gaussian Splatting with Test-Time Optimization, https://doi.org/10.48550/arxiv.2609.21498, fetched 2026-10-26 (unofficial)
[S2] SPARF: Neural Radiance Fields from Sparse and Noisy Poses, https://doi.org/10.1109/cvpr52729.2023.00408, fetched 2026-10-26
[S3] CNNs Based Viewpoint Estimation for Volume Visualization, https://doi.org/10.1145/3309993, fetched 2026-10-29
