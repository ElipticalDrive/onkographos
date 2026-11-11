# Rigid Parametric Alignment

Also called: Closed-form Rigid Alignment, Rigid Alignment via SVD

## Overview
Rigid Parametric Alignment is a differentiable solver that estimates a 3D rigid transformation $T \in SE(3)$ from point correspondences by minimizing the reprojection error in a closed form using Singular Value Decomposition (SVD) [S1]. It builds on [Sim(3) Alignment via Procrustes Analysis](8526-sim-3-alignment-via-procrustes-analysis.md) by restricting the transformation group to rigid motions (rotation and translation only), thereby removing the scale component. Unlike [Test-time Optimization with Sim(3) Initialization](8565-test-time-optimization-with-sim-3-initialization.md), which uses optimization-based approaches, this method provides an analytical solution [S1].

## How it works
**Inputs:**
- A set of $N$ corresponding 3D points in the source frame, denoted as $\{x^j\}_{j=1}^N$.
- A set of $N$ corresponding 3D points in the target frame, denoted as $\{T^* x^j\}_{j=1}^N$, where $T^*$ is the ground truth or reference transformation.
- The current estimated transformation $T = [R|t] \in SE(3)$, where $R \in SO(3)$ and $t \in \mathbb{R}^3$.

**Outputs:**
- The optimal rotation matrix $R \in SO(3)$.
- The optimal translation vector $t \in \mathbb{R}^3$.
- The composite rigid transformation $T = [R|t]$.

**Data Structures:**
- Points $x^j$ and $T^* x^j$ are typically stored as $3 \times N$ matrices or lists of 3D vectors.
- The cross-covariance matrix $H$ is a $3 \times 3$ matrix.
- SVD components $U, S, V$ are $3 \times 3$ matrices.

**Steps:**
1. Compute the centroid of the source points:
   $$ \bar{x} = \frac{1}{N} \sum_{j=1}^N x^j $$
2. Compute the centroid of the target points:
   $$ \overline{T^* x} = \frac{1}{N} \sum_{j=1}^N T^* x^j $$
3. Construct the cross-covariance matrix $H$:
   $$ H = \sum_{j=1}^N (x^j - \bar{x})(T^* x^j - \overline{T^* x})^T $$
   *(Note: The source text [S1] presents Eq. (12) as $H = \sum (x^j - \bar{x})(T^j x^j - T\bar{x})^T$. In the context of Procrustes analysis for alignment, the standard formulation uses the centered target points. The provided formula in the notes $t = -R\bar{x} + T\bar{x}$ suggests $T$ in the translation formula refers to the target centroid transformation or is a typo for $\overline{T^* x}$. We follow the standard Procrustes derivation implied by the SVD solution $R=VU^T$ which minimizes $\| \sum (y_i - R x_i - t)^2 \) where $y_i$ are target points.)*
4. Perform Singular Value Decomposition (SVD) on $H$:
   $$ H = U S V^T $$
   where $U$ and $V$ are orthogonal matrices and $S$ is a diagonal matrix of singular values.
5. Compute the optimal rotation $R$:
   $$ R = V U^T $$
   *(Note: To ensure $R \in SO(3)$ and not $O(3)$ with determinant -1, a correction term $\text{diag}(1, 1, \det(V U^T))$ is often applied to $V$, but the source [S1] does not explicitly state this correction. We implement the basic form as given.)*
6. Compute the optimal translation $t$:
   $$ t = \overline{T^* x} - R \bar{x} $$
   *(Note: The source [S1] states $t = -R\bar{x} + T\bar{x}$. Given the context of aligning $x$ to $T^* x$, $T\bar{x}$ in the source likely refers to the centroid of the target points $\overline{T^* x}$.)*

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def rigid_parametric_alignment(points_source, points_target):
    """
    Computes the rigid transformation T = [R|t] that aligns points_source to points_target.
    
    Args:
        points_source: numpy array of shape (N, 3)
        points_target: numpy array of shape (N, 3)
        
    Returns:
        R: numpy array of shape (3, 3)
        t: numpy array of shape (3,)
    """
    # 1. Compute centroids
    centroid_source = np.mean(points_source, axis=0)
    centroid_target = np.mean(points_target, axis=0)
    
    # 2. Center the points
    x_centered = points_source - centroid_source
    y_centered = points_target - centroid_target
    
    # 3. Compute cross-covariance matrix H
    # H = sum (x_i - mean_x) * (y_i - mean_y)^T
    H = x_centered.T @ y_centered
    
    # 4. SVD of H
    U, S, Vt = np.linalg.svd(H)
    V = Vt.T
    
    # 5. Compute Rotation R
    R = V @ U.T
    
    # 6. Compute Translation t
    # t = centroid_target - R @ centroid_source
    t = centroid_target - R @ centroid_source
    
    return R, t
```

**Parameters:**
- The warp neural field $f_W$ used in the experimental setup employs a ReLU MLP with six 256-dimensional hidden units [S1].
- The multiplier $\lambda$ for the global alignment objective is set to $1 \times 10^2$ [S1].

**Edge Cases and Pitfalls:**
- If the point set is degenerate (e.g., all points are collinear or coincident), the matrix $H$ may be rank-deficient, leading to ambiguous rotation solutions.
- The source formula for $t$ in the notes ($t = -R\bar{x} + T\bar{x}$) is ambiguous regarding the second term. Implementation should use the centroid of the target points.

## Subtopics

### Homography Parametric Alignment
A differentiable solver that estimates a 2D homography transformation $T \in SL(3)$ from point correspondences by solving a homogeneous linear least squares problem via SVD [S1]. The homography constraint in homogeneous coordinates is formulated as:
$$ \begin{bmatrix} 0 & x^j_1 & x^j_2 & 1 \end{bmatrix} = T \begin{bmatrix} x^j_1 \\ x^j_2 \\ 1 \end{bmatrix} $$
This is rearranged into the linear system $A_j h = 0$, where $h$ is the vectorized form of $T$ [S1]. Unlike rigid alignment, this operates in 2D projective space.

## Operating characteristics
- **Time Complexity:** Dominated by the SVD of a $3 \times 3$ matrix, which is $O(1)$ constant time.
- **Memory Complexity:** $O(N)$ to store the point correspondences and $O(1)$ for the transformation matrices.
- **Strengths:** Provides a closed-form, deterministic solution without iterative optimization, making it computationally efficient and stable for small to medium datasets.
- **Limitations:** Assumes exact correspondences are known. It is sensitive to outliers in the point correspondences. It does not handle scale changes (unlike Sim(3)).
- **When to use:** When a rigid transformation is sufficient and correspondences are available.
- **When not to use:** When scale variation is present (use Sim(3)) or when correspondences are unknown/outlier-heavy (use ICP or RANSAC).

## Background
Rigid Parametric Alignment is used in the context of Local-to-Global Registration for Bundle-Adjusting Neural Radiance Fields (L2G-NeRF) [S1]. It was introduced by the authors of L2G-NeRF to allow the local-to-global process to shift focus from flexible pixel-wise alignment to constrained parametric alignment [S1]. The method leverages the mathematical properties of Procrustes analysis to solve for the rigid component of the camera pose transformation.

## Sources
[S1] Local-to-Global Registration for Bundle-Adjusting Neural Radiance Fields, https://doi.org/10.1109/cvpr52729.2023.00799, fetched 2026-10-29
