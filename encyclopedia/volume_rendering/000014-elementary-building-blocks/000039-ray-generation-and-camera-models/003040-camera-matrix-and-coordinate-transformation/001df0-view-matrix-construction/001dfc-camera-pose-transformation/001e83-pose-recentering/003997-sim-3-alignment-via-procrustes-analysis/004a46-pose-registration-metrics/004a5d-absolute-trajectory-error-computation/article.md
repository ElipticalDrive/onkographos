# Absolute Trajectory Error Computation

Also called: ATE, Absolute Trajectory Error

## Overview
Computes the Root Mean Square Error (RMSE) of the translation component between a ground truth trajectory and an estimated trajectory after optimal alignment [S2, unofficial]. It evaluates the global consistency of a SLAM or visual odometry system by measuring the integrated distance between pose estimates and ground truth [S1][S5].

## How it works
**Inputs:**
- Ground truth trajectory: A sequence of $N$ poses, typically stored as $4 \times 4$ transformation matrices or quaternions [S2, unofficial].
- Estimated trajectory: A sequence of $N$ poses corresponding to the ground truth frames [S2, unofficial].
- `monocular`: Boolean flag indicating if scale correction is required during alignment [S3, unofficial].

**Outputs:**
- `ate_rmse`: The scalar ATE RMSE value in the scene's units [S2, unofficial].
- `e_trans`: An array of per-frame translation error magnitudes [S4, unofficial].

**Data Structures:**
- Translation vectors: Extracted from the $4 \times 4$ pose matrices (specifically noted as the 3rd column, 4th row in some implementations) [S2, unofficial].

**Steps:**
1. **Preprocessing:** Iterate over frames; skip any frame where the ground truth pose contains NaN values [S2, unofficial]. Normalize estimated camera rotation quaternions if provided in quaternion form [S2, unofficial].
2. **Alignment:** Align the estimated trajectory to the ground truth trajectory using a similarity transformation (Sim(3)) or rigid transformation (SE(3)) to minimize alignment error, typically via Horn's method or Procrustes analysis [S2, unofficial][S6]. If `monocular` is true, pass `correct_scale=True` to enable scale drift correction [S3, unofficial].
3. **Error Computation:** For each frame $i$, compute the translation error vector $e_{trans, i} = p_{gt, i} - p_{est, aligned, i}$ and its magnitude $e_{trans, i} = \sqrt{\sum (e_{trans, i})^2}$ [S4, unofficial].
4. **RMSE Calculation:** Compute the ATE RMSE as the square root of the mean of the squared translation errors: $\text{ATE RMSE} = \sqrt{\frac{1}{N} \sum_{i=1}^N (e_{trans, i})^2}$ [S2, unofficial]. Some implementations report the mean translation error directly, but RMSE is the standard metric for ATE in SLAM evaluation [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_ate_rmse(gt_poses, est_poses, monocular=False):
    """
    Compute Absolute Trajectory Error (ATE) RMSE.
    
    Args:
        gt_poses: List of N 4x4 numpy arrays (ground truth poses).
        est_poses: List of N 4x4 numpy arrays (estimated poses).
        monocular: Boolean flag to enable scale correction [S3, unofficial].
        
    Returns:
        ate_rmse: Float, the ATE RMSE.
        e_trans: Array of N floats, per-frame translation errors [S4, unofficial].
    """
    # Extract translation vectors from 4x4 pose matrices [S2, unofficial]
    # Assuming row-major convention where translation is at index [3, 0:3]
    gt_trans = np.array([pose[3, 0:3] for pose in gt_poses])
    est_trans = np.array([pose[3, 0:3] for pose in est_poses])

    # Filter out frames with NaN ground truth poses [S2, unofficial]
    valid_mask = ~np.any(np.isnan(gt_trans), axis=1)
    gt_trans = gt_trans[valid_mask]
    est_trans = est_trans[valid_mask]
    N = len(gt_trans)

    if N == 0:
        return 100.0, np.array([]) # Pitfall: set to 100.0 if evaluation fails [S2, unofficial]

    # Align estimated trajectory to ground truth using Procrustes/Horn's method [S2, unofficial][S6]
    mu_gt = np.mean(gt_trans, axis=0)
    mu_est = np.mean(est_trans, axis=0)
    
    gt_c = gt_trans - mu_gt
    est_c = est_trans - mu_est
    
    H = est_c.T @ gt_c
    U, _, Vt = np.linalg.svd(H)
    d = np.linalg.det(Vt.T @ U.T)
    D = np.eye(3)
    D[2, 2] = d
    R = Vt.T @ D @ U.T
    
    t = mu_gt - R @ mu_est
    
    # Scale factor for monocular/Sim(3) alignment [S3, unofficial]
    scale = 1.0
    if monocular:
        scale = np.sqrt(np.sum(gt_c**2) / np.sum(est_c**2))
        
    # Apply alignment
    est_aligned = scale * (est_c @ R.T) + mu_gt

    # Compute per-frame translation errors [S4, unofficial]
    e_trans_vec = gt_trans - est_aligned
    e_trans = np.sqrt(np.sum(e_trans_vec**2, axis=1))

    # Compute RMSE [S2, unofficial]
    ate_rmse = np.sqrt(np.mean(e_trans**2))

    return ate_rmse, e_trans
```

**Parameters:**
- `monocular`: Boolean flag; if True, enables scale correction during alignment to handle monocular scale drift [S3, unofficial].
- Baseline processing: Some evaluations apply a Gaussian blur kernel of size $9 \times 9 \times 9$ voxels with $\sigma=4/3$ to the map before computing ATE [S1].

**Edge Cases and Pitfalls:**
- If trajectory evaluation fails or contains no valid poses, ATE RMSE is set to 100.0 [S2, unofficial].
- Precision issues may arise with very large or small trajectories; centering data during alignment mitigates this [G].

## Subtopics

### Mean Angular Error (MAE)
Measures the average error between predicted angles and true angles, commonly used in camera pose estimation and target tracking [S5]. Its calculation formula is $MAE = \frac{1}{n} \sum_{i=1}^n |\theta_i - \hat{\theta}_i|$ [S5], where $n$ is the number of observation points, $\theta_i$ is the true angle of the $i$-th observation point, and $\hat{\theta}_i$ is the predicted angle [S5]. It is used in computer vision and robot navigation for tasks involving angle estimation, such as camera pose estimation, target tracking, and motion capture [S5].

## Operating characteristics
- **Time Complexity:** $O(N)$ for error computation and alignment, where $N$ is the number of frames [S4, unofficial].
- **Memory Complexity:** $O(N)$ to store translation vectors and per-frame errors [S4, unofficial].
- **Strengths:** Provides a single scalar metric for global trajectory accuracy and scale consistency [S5].
- **Limitations:** Does not capture local drift or oscillations as well as relative error metrics like RPE [S5]. Sensitive to global scale errors if Sim(3) alignment is omitted [S3, unofficial].
- **When to use:** Evaluating the global consistency of SLAM systems, visual odometry, or pose estimation pipelines [S5].
- **When not to use:** When local accuracy or drift over short intervals is more important than global consistency [S5].

## Background
- **Origin:** Introduced as a standard metric for evaluating ego-motion estimation in SLAM and visual odometry [S1].
- **Development:** Initially proposed to measure the integrated distance between pose estimates and ground truth, it has become a de facto standard in the field [S1].
- **Usage:** Commonly used in RGB-D SLAM, Structure from Motion (SfM), and NeRF/Gaussian Splatting pose estimation evaluations [S1][S2, unofficial][S3, unofficial]. Evaluated on datasets like the freiburg-1 collection, yielding baseline Mean ATE of $0.10 \pm 0.17$ m and Median ATE of $0.038$ m [S1].

## Sources
[S1] Compressed Voxel-Based Mapping Using Unsupervised Learning, https://doi.org/10.3390/robotics6030015, fetched 2026-10-26
[S2] spla-tam/SplaTAM, https://github.com/spla-tam/SplaTAM, fetched 2026-10-27 (unofficial)
[S3] muskie82/MonoGS, https://github.com/muskie82/MonoGS, fetched 2026-10-27 (unofficial)
[S4] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S5] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
[S6] SAMURAI: Shape and Material from Unconstrained Real-World Arbitrary Image Collections, https://doi.org/10.52202/068431-1914, fetched 2026-10-29
