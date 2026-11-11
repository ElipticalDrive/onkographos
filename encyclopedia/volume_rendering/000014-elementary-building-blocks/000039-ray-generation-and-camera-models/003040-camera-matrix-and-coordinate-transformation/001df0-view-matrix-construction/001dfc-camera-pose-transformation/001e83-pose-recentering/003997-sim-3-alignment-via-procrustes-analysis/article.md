# Sim(3) Alignment via Procrustes Analysis

Also called: Pose Accuracy Measurement, Procrustes alignment, Sim(3) BARF

## Overview
Sim(3) Alignment via Procrustes Analysis computes a 3D similarity transformation (rotation, translation, and uniform scale) to align estimated camera poses to ground truth poses by analyzing camera location coordinates [S1, unofficial]. It is primarily used for pose quality assessment in Neural Radiance Fields (NeRF) and Structure-from-Motion (SfM) pipelines [S1, unofficial]. Unlike [PCA Pose Alignment](7626-pca-pose-alignment.md), which aligns based on principal component directions, this method explicitly solves for the similarity transform that minimizes the distance between corresponding camera centers [S1, unofficial].

## How it works
**Inputs:**
- Set of estimated camera pose locations: $P_{est} = \{p_1, p_2, \dots, p_N\}$, where $p_i \in \mathbb{R}^3$.
- Set of ground truth camera pose locations: $P_{gt} = \{q_1, q_2, \dots, q_N\}$, where $q_i \in \mathbb{R}^3$.
- Note: Only the translation components (camera centers) of the poses are used for the Procrustes analysis [S1, unofficial].

**Outputs:**
- A 3D similarity transformation $T \in Sim(3)$, defined by a rotation matrix $R \in SO(3)$, a translation vector $t \in \mathbb{R}^3$, and a scale factor $s \in \mathbb{R}^+$.

**Data Structures:**
- The input points are typically stored as $3 \times N$ matrices (or $N \times 3$ depending on convention). Let $X$ be the matrix of estimated centers and $Y$ be the matrix of ground truth centers.

**Algorithm Steps:**
1. **Centroid Calculation:** Compute the centroid of the estimated locations $\mu_X = \frac{1}{N} \sum_{i=1}^N p_i$ and the centroid of the ground truth locations $\mu_Y = \frac{1}{N} \sum_{i=1}^N q_i$ [S1, unofficial].
2. **Centering:** Subtract the centroids from the respective points to obtain centered matrices $X_c = X - \mu_X$ and $Y_c = Y - \mu_Y$ [S1, unofficial].
3. **Covariance Matrix:** Compute the covariance matrix $H = X_c Y_c^T$ (or $Y_c X_c^T$ depending on the definition of the transformation direction; typically $H = \sum (x_i - \mu_X)(y_i - \mu_Y)^T$) [S1, unofficial].
4. **Singular Value Decomposition (SVD):** Perform SVD on $H$: $H = U \Sigma V^T$ [S1, unofficial].
5. **Rotation Calculation:** Compute the rotation matrix $R = V U^T$. To ensure a proper rotation (determinant +1) and avoid reflection, check the determinant. If $\det(R) = -1$, multiply the last column of $V$ by $-1$ before computing $R$ [S1, unofficial].
6. **Scale Calculation:** Compute the scale factor $s = \frac{\text{trace}(\Sigma)}{\sum_{i=1}^N \|p_i - \mu_X\|^2}$ [S1, unofficial].
7. **Translation Calculation:** Compute the translation vector $t = \mu_Y - s R \mu_X$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def sim3_procrustes(X, Y):
    """
    Compute Sim(3) alignment from X (estimated) to Y (ground truth).
    X, Y: numpy arrays of shape (N, 3) representing camera centers.
    Returns: R (3x3), s (float), t (3,)
    """
    N = X.shape[0]
    
    # 1. Centroids
    mu_X = np.mean(X, axis=0)
    mu_Y = np.mean(Y, axis=0)
    
    # 2. Centering
    X_c = X - mu_X
    Y_c = Y - mu_Y
    
    # 3. Covariance Matrix
    H = X_c.T @ Y_c
    
    # 4. SVD
    U, S, Vt = np.linalg.svd(H)
    V = Vt.T
    
    # 5. Rotation
    R = V @ U.T
    
    # Handle reflection case
    if np.linalg.det(R) < 0:
        V[:, -1] *= -1
        R = V @ U.T
        
    # 6. Scale
    # Sum of squared norms of centered X
    sum_sq_X = np.sum(X_c ** 2)
    if sum_sq_X == 0:
        s = 1.0 # Degenerate case: all points are identical
    else:
        s = np.trace(S) / sum_sq_X
        
    # 7. Translation
    t = mu_Y - s * R @ mu_X
    
    return R, s, t
```

**Parameters:**
- None. The method is deterministic given the input points.

**Edge Cases and Pitfalls:**
- **Degenerate Configurations:** Procrustes analysis is unreliable when all camera locations lie on a line or in a plane, or when the camera translation contains significant noise [S1, unofficial].
- **Precision:** Ensure numerical stability in SVD, especially when the scale factor is very small or the point distribution is ill-conditioned [S1, unofficial].

## Subtopics

### [Test-time Optimization with Sim(3) Initialization](8565-test-time-optimization-with-sim-3-initialization.md)
A testing procedure involving Sim(3) alignment initialization followed by additional test-time optimization to adjust test poses for view synthesis. It sets itself apart from sibling methods like [Rigid Parametric Alignment](8609-rigid-parametric-alignment.md) by explicitly decoupling the coarse alignment (via Procrustes analysis on locations) from the fine-grained pose refinement (via gradient-based optimization). Used in BARF [20] to initialize poses before optimization.

### [Rigid Parametric Alignment](8609-rigid-parametric-alignment.md)
Rigid Parametric Alignment is a differentiable solver that estimates a 3D rigid transformation $T \in SE(3)$ from point correspondences by minimizing the reprojection error in a closed form using Singular Value Decomposition (SVD). It builds on [Sim(3) Alignment via Procrustes Analysis](8526-sim-3-alignment-via-procrustes-analysis.md) by restricting the transformation group to rigid motions (rotation and translation only), thereby removing the scale component. Unlike [Test-time Optimization with Sim(3) Initialization](8565-test-time-optimization-with-sim-3-initialization.md), which uses optimization-based approaches, this method provides an analytical solution.

### [Pose Registration Metrics](12703-pose-registration-metrics.md)
Pose Registration Metrics quantify the geometric discrepancy between estimated camera poses and ground-truth poses after alignment. They solve the problem of evaluating the accuracy of Structure-from-Motion (SfM) or Sim(3) alignment methods by providing scalar measures of rotational and translational deviation. Unlike sibling methods that focus on alignment computation or optimization, these metrics serve as the evaluation standard for registration quality.

### Sim(3) with Rotation Alignment
A variant of Sim(3) alignment that adds a virtual point to mitigate Procrustes analysis failures in degenerate cases, using camera rotation during alignment [S1, unofficial].
**Steps:**
1. Add a virtual point at coordinates $(0, 0, 1)$ in the camera coordinate system of each image [S1, unofficial].
2. Use the camera parameter to transform this virtual point to the world coordinate [S1, unofficial].
3. Use the camera rotation during the alignment process [S1, unofficial].
**Pitfalls:** This approach causes more translation errors compared to standard Sim(3) [S1, unofficial].
**Comparison:** Reduces rotation errors ($\Delta R$) but harms translation accuracy ($\Delta T$) when pose accuracy is high [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Dominated by the SVD of a $3 \times 3$ matrix (or $3 \times N$ covariance), which is $O(1)$ effectively for fixed dimensionality, or $O(N)$ for computing the covariance matrix.
- **Memory Complexity:** $O(N)$ to store the camera centers.
- **Strengths:** Provides a closed-form solution for aligning camera trajectories; effective for pose quality assessment [S1, unofficial].
- **Limitations:** Unreliable for degenerate camera configurations (e.g., collinear cameras) or noisy translations [S1, unofficial]. Standard Sim(3) may cause fewer translation errors than Sim(3) with rotation, but Sim(3) with rotation reduces rotation errors at the cost of translation accuracy [S1, unofficial].
- **When to use:** When evaluating pose accuracy in NeRF pipelines (e.g., BARF, L2G-NeRF) or initializing poses for test-time optimization [S1, unofficial].
- **When not to use:** When camera centers are collinear or highly noisy, or when a rigid transformation (no scale) is strictly required without the overhead of scale estimation.

## Background
The method originates from the application of Procrustes analysis to camera pose alignment in the context of Neural Radiance Fields. It was notably employed in the BARF (Bundle-Adjusting NeRF) paper [20] and L2G-NeRF [6] for pose quality assessment [S1, unofficial]. The variant "Sim(3) with rotation" was introduced in CF-NeRF [41] to address degeneracies in Procrustes analysis [S1, unofficial].

## Sources
[S1] CT-NeRF: Incremental Optimizing Neural Radiance Field and Poses with Complex Trajectory, https://doi.org/10.48550/arxiv.2404.13896, fetched 2026-10-26 (unofficial)
[S2] DBARF: Deep Bundle-Adjusting Generalizable Neural Radiance Fields, https://doi.org/10.1109/cvpr52729.2023.00011, fetched 2026-10-29
