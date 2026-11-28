# Pose Registration Metrics

Also called: Localization Evaluation Metrics, Pose Error Metrics, Rotation Error, Terr, Translation Error

## Overview
Pose Registration Metrics evaluate camera pose estimation accuracy using absolute translation and rotation errors [S1][S2, unofficial][S3].

## How it works
**Inputs:**
- $t$: Ground-truth translation vector (3D position).
- $\hat{t}$: Estimated translation vector (3D position).
- $R$: Ground-truth rotation matrix ($3 \times 3$).
- $\hat{R}$: Estimated rotation matrix ($3 \times 3$).

**Outputs:**
- $\Delta t$: Translation error (scalar).
- $R_{err}$: Rotation error (scalar, in radians).

**Data Structures:**
Not given in the sources.

**Numbered Steps:**
1. Compute the Euclidean distance between the ground-truth and estimated translation vectors [S1][S2, unofficial][S3].
2. Compute the matrix product of the ground-truth rotation transpose and the estimated rotation [S1][S2, unofficial][S3].
3. Calculate the trace of the product matrix [S1][S2, unofficial][S3].
4. Apply the arccosine formula to the trace to obtain the geodesic rotation angle [S1][S2, unofficial][S3].
5. Clamp the arccosine argument to $[-1, 1]$ to handle floating-point precision errors [G].

**Formulas:**
1. **Translation Error ($\Delta t$):**
   The Euclidean distance between the estimated and ground-truth positions [S1]:
   $$ \Delta t = \| t - \hat{t} \|_2 $$
   Measured as the Euclidean distance between predicted and ground truth [S3]. On datasets where positions are not in metric space, the translation error has no units [S3].

2. **Rotation Error ($R_{err}$):**
   The geodesic distance on the rotation manifold, representing the angle required to align the estimated rotation with the ground truth [S2, unofficial]:
   $$ R_{err} = \arccos \left( \frac{\text{Tr}(R^\top \hat{R}) - 1}{2} \right) $$
   Alternatively expressed using the inverse [S3]:
   $$ R_{err} = \arccos \left( \frac{\text{Tr}(R^{-1} \hat{R}) - 1}{2} \right) $$
   Where $\text{Tr}(\cdot)$ denotes the trace of a matrix [S1]. The result is in radians [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def compute_translation_error(t: np.ndarray, t_hat: np.ndarray) -> float:
    """
    Computes Euclidean distance between estimated and ground truth positions.
    Args:
        t: 3D array, ground truth position [x, y, z]
        t_hat: 3D array, estimated position [x, y, z]
    Returns:
        float: Euclidean distance
    """
    return float(np.linalg.norm(t - t_hat))

def compute_rotation_error(R: np.ndarray, R_hat: np.ndarray) -> float:
    """
    Computes geodesic rotation error in radians.
    Args:
        R: 3x3 numpy array, ground truth rotation matrix
        R_hat: 3x3 numpy array, estimated rotation matrix
    Returns:
        float: Rotation error in radians
    """
    product = R.T @ R_hat
    trace_val = np.trace(product)
    cos_angle = (trace_val - 1.0) / 2.0
    cos_angle = np.clip(cos_angle, -1.0, 1.0)
    return float(math.acos(cos_angle))

def evaluate_pose_pair(t: np.ndarray, t_hat: np.ndarray, R: np.ndarray, R_hat: np.ndarray):
    """
    Evaluates a single pose pair.
    Args:
        t: Ground truth translation
        t_hat: Estimated translation
        R: Ground truth rotation
        R_hat: Estimated rotation
    Returns:
        tuple: (translation_error, rotation_error)
    """
    t_err = compute_translation_error(t, t_hat)
    r_err = compute_rotation_error(R, R_hat)
    return t_err, r_err
```

**Parameters:**
Not given in the sources.

**Edge Cases:**
- **Numerical Stability:** The argument to $\arccos$ must be clamped to $[-1, 1]$ to prevent NaNs due to floating-point precision errors in matrix multiplication [G].
- **Non-Metric Space:** Translation error units are undefined if the coordinate system is not metric [S3].

## Subtopics

### [Absolute Trajectory Error Computation](12727-absolute-trajectory-error-computation.md)
Computes the Root Mean Square Error (RMSE) of the translation component between a ground truth trajectory and an estimated trajectory after optimal alignment. It quantifies the global scale and drift of an ego-motion estimation system by measuring the integrated distance between pose estimates and ground truth. Unlike relative error metrics, ATE evaluates the absolute consistency of the trajectory in the world frame.

### Viewpoint Accuracy Evaluation
An evaluation metric based on geodesic distance tolerance between estimated and ground-truth viewpoints [S4]. The probability for viewpoint label $v$ is modeled as $P_v(s) \sim \mathcal{N}(v_\mu, v_\sigma^2)$, where $v_\mu = \arg \max_{v \in V} P_v(s)$ [S4]. A prediction is correct if the geodesic distance between $v_\mu$ and the ground-truth viewpoint $v_s$ is within a specified tolerance [S4]. Standard tolerances are $2^\circ$, $5^\circ$, $8^\circ$, $11^\circ$, and $15^\circ$ [S4].

## Operating characteristics
Not given in the sources.

## Background
These metrics are commonly used to measure localization accuracy in visual localization and novel view synthesis systems [S1][S2, unofficial][S3]. The rotation error formulation as a geodesic distance on the rotation manifold is standard in recent works [S2, unofficial][S3].

## Sources
[S1] SplatLoc: 3D Gaussian Splatting-based Visual Localization for Augmented Reality., https://doi.org/10.1109/tvcg.2025.3549563, fetched 2026-10-26
[S2] VoxelTTO: Voxel-Aligned Feed-Forward 3D Gaussian Splatting with Test-Time Optimization, https://doi.org/10.48550/arxiv.2609.21498, fetched 2026-10-26 (unofficial)
[S3] SPARF: Neural Radiance Fields from Sparse and Noisy Poses, https://doi.org/10.1109/cvpr52729.2023.00408, fetched 2026-10-26
[S4] CNNs Based Viewpoint Estimation for Volume Visualization, https://doi.org/10.1145/3309993, fetched 2026-10-29
