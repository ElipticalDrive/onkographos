# Rigid Parametric Alignment

Also called: Closed-form Rigid Alignment, Horn's Method, Horn's Method for Trajectory Alignment, Rigid Alignment via SVD

## Overview
Rigid Parametric Alignment computes the optimal 3D rigid transformation $T \in SE(3)$ that aligns a set of model points to a set of data points by minimizing the sum of squared Euclidean distances between corresponding landmarks [S1, unofficial][S2]. It solves the absolute orientation problem in a single closed-form step using Singular Value Decomposition (SVD), distinguishing it from iterative methods like ICP or optimization-based approaches that require gradient descent [S1, unofficial][S2]. Unlike [Homography Parametric Alignment](8618-homography-parametric-alignment.md), which estimates a 2D projective transformation for planar distortions, this method operates in 3D Euclidean space to preserve metric distances and angles [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
- **Input:** Two sets of $N$ corresponding 3D points: model points $X = \{x_1, \dots, x_N\}$ and data points $Y = \{y_1, \dots, y_N\}$. Each set is represented as a $3 \times N$ matrix where columns are points.
- **Output:** A rotation matrix $R \in SO(3)$ and a translation vector $t \in \mathbb{R}^3$, forming the transformation $T = [R|t]$.

**Data Structures:**
- Points are stored as 3D vectors. In implementation, they are often packed into $3 \times N$ matrices for batched linear algebra operations.

**Steps:**
1. **Centroid Calculation:** Compute the centroid (mean) of the model points $\bar{x}$ and the data points $\bar{y}$:
   $$ \bar{x} = \frac{1}{N} \sum_{i=1}^{N} x_i, \quad \bar{y} = \frac{1}{N} \sum_{i=1}^{N} y_i $$
2. **Centering:** Subtract the centroids from the respective points to obtain zero-centered coordinates:
   $$ x'_i = x_i - \bar{x}, \quad y'_i = y_i - \bar{y} $$
3. **Covariance Matrix:** Compute the $3 \times 3$ covariance matrix $H$ (also denoted as $W$ in some implementations) between the centered points:
   $$ H = \sum_{i=1}^{N} x'_i (y'_i)^T $$
   *(Note: Source [S2] defines $H$ as $\sum (x_j - \bar{x})(T_j y_j - T\bar{x})^T$ in the context of general registration, but for standard rigid alignment where $T$ is the unknown, the cross-covariance of centered points is the standard derivation. Source [S1, unofficial] computes $W$ as sum of outer products of centered columns.)*
4. **SVD:** Perform Singular Value Decomposition on $H$:
   $$ H = U S V^T $$
   where $U$ and $V$ are orthogonal matrices and $S$ is a diagonal matrix of singular values.
5. **Rotation Correction:** Initialize a correction matrix $D$ as the identity matrix $I_3$. To ensure a proper rotation (determinant +1) and avoid reflection:
   - If $\det(U V^T) < 0$, set the last element of $D$ to $-1$ (i.e., $D = \text{diag}(1, 1, -1)$). This handles the case where the SVD yields a reflection.
   - Specifically, if $\det(U) \cdot \det(V) < 0$, flip the sign of the smallest singular value's corresponding axis in the rotation calculation [S1, unofficial].
6. **Rotation Calculation:** Compute the optimal rotation matrix:
   $$ R = V D U^T $$
   *(Source [S2] states $R = V U^T$, assuming no reflection or implicit handling. Source [S1, unofficial] explicitly includes the sign flip logic.)*
7. **Translation Calculation:** Compute the optimal translation vector:
   $$ t = \bar{y} - R \bar{x} $$
8. **Transformation Construction:** The final rigid transformation is:
   $$ T = \begin{bmatrix} R & t \\ 0 & 1 \end{bmatrix} \in SE(3) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def rigid_parametric_alignment(model_points, data_points):
    """
    Aligns model_points to data_points using closed-form SVD.
    
    Args:
        model_points: np.ndarray of shape (N, 3)
        data_points: np.ndarray of shape (N, 3)
        
    Returns:
        R: np.ndarray of shape (3, 3), rotation matrix
        t: np.ndarray of shape (3,), translation vector
    """
    # 1. Centroids
    mean_model = np.mean(model_points, axis=0)
    mean_data = np.mean(data_points, axis=0)
    
    # 2. Centering
    model_centered = model_points - mean_model
    data_centered = data_points - mean_data
    
    # 3. Covariance Matrix H
    # H = sum(x'_i * y'_i^T)
    H = np.dot(model_centered.T, data_centered)
    
    # 4. SVD
    U, S, Vt = np.linalg.svd(H)
    
    # 5. Rotation Correction
    # Check for reflection
    det_UVt = np.linalg.det(np.dot(U, Vt))
    if det_UVt < 0:
        # Flip the sign of the last column of V (or last row of Vt)
        # Corresponds to flipping the sign of the smallest singular value's axis
        Vt[-1, :] *= -1
        
    # 6. Rotation
    R = np.dot(Vt.T, U.T)
    
    # 7. Translation
    t = mean_data - np.dot(R, mean_model)
    
    return R, t
```

**Parameters:**
- **Landmark Count:** Minimum 4 non-coplanar points are required for a unique solution in 3D [S3]. More points improve robustness to noise [S3].
- **Hidden Units (Contextual):** In specific neural field implementations (e.g., L2G-NeRF), the rigid alignment is parameterized by a ReLU MLP with six 256-dimensional hidden units [S2]. This is a specific architectural choice for differentiable alignment, not a general parameter of the SVD method itself.

**Edge Cases and Pitfalls:**
- **Reflection:** If the point sets have different handedness, the SVD may yield a rotation with determinant -1. The correction step (flipping the sign of the last singular value's axis) is critical to enforce $R \in SO(3)$ [S1, unofficial].
- **Degenerate Configurations:** If all points are collinear or coplanar, the covariance matrix $H$ is rank-deficient, leading to ambiguous rotation solutions. At least 4 non-coplanar points are required [S3].
- **Precision:** Floating-point errors in SVD can occasionally result in $R$ not being perfectly orthogonal. Re-orthogonalization (e.g., via SVD of $R$) may be needed for strict numerical stability [G].

## Subtopics

### [Homography Parametric Alignment](8618-homography-parametric-alignment.md)
Estimates a 2D projective transformation $T \in SL(3)$ from point correspondences by solving a homogeneous linear least squares problem via SVD. It is employed in camera-in-the-loop optimization to map captured camera coordinates to Spatial Light Modulator (SLM) pattern coordinates, ensuring the captured wavefront and SLM pattern reside in the same coordinate system. Unlike Rigid Parametric Alignment, which estimates a 3D rigid transformation in $SE(3)$, this method operates in 2D homogeneous space to handle planar or perspective distortions between the sensor and the display plane.

### [Iterative Closest Point (ICP)](10563-iterative-closest-point-icp.md)
A registration algorithm used to align two point sets, typically a reconstructed point cloud and a ground truth mesh, by finding an optimal rigid transformation. It builds on Rigid Parametric Alignment by iteratively refining the transformation estimate through correspondence matching, rather than solving for a transformation from pre-established correspondences in a single step [G]. Unlike differentiable renderers which optimize pose via gradient descent on pixel errors, ICP operates directly in geometric space.

### [Coarse Camera Pose Estimation](15954-coarse-camera-pose-estimation.md)
An incremental registration method that estimates the initial extrinsic parameters (camera pose) for unposed target images relative to a registered reference image using a learned scene regressor. It solves the problem of initializing camera poses in neural radiance field training by converting predicted 3D points into 2D-3D correspondences and solving for the pose via Perspective-n-Point (PnP). It differs from ICP by relying on a neural scene representation to provide dense 3D structure rather than iterative point-to-point alignment, and from Structure-from-Motion (SfM) by operating incrementally on a learned regressor rather than globally on feature matches.

### N-Point Alignment Registration
An automatic registration tool that aligns source and destination objects by finding a transformation that minimizes the error between corresponding landmarks [S3]. It requires identifying at least four corresponding landmarks in both the source and destination objects and computes the transformation using the least-square approximation method based on these landmarks [S3]. The more landmarks specified, the better the fit [S3].

## Operating characteristics
- **Time Complexity:** $O(N)$ for centroid and covariance calculation, and $O(1)$ for the fixed-size $3 \times 3$ SVD. Thus, the overall complexity is linear in the number of points $N$.
- **Memory Complexity:** $O(N)$ to store the point sets and $O(1)$ for the covariance matrix.
- **Performance:** Provides an exact closed-form solution, making it extremely fast and deterministic compared to iterative methods like ICP [S1, unofficial][S2].
- **Strengths:** No initialization required; robust to moderate noise; computationally efficient.
- **Limitations:** Requires known correspondences between points; sensitive to outliers in the point sets (minimizes squared error, so outliers have large influence); fails if points are degenerate (collinear/coplanar) [S3].
- **When to Use:** When correspondences are known and the transformation is strictly rigid (no scaling or shear).
- **When Not to Use:** When correspondences are unknown (use ICP or feature matching); when the transformation includes scaling (use Sim(3) Alignment); when outliers are prevalent (use RANSAC-based variants).

## Background
Rigid Parametric Alignment, often referred to as Horn's Method, originates from the work of B. K. P. Horn in 1987, who published a closed-form solution for the absolute orientation problem using quaternions, later shown to be equivalent to the SVD approach [S1, unofficial][S2]. It is a fundamental building block in computer vision and robotics for pose estimation and registration. In the context of neural rendering, it is used in methods like L2G-NeRF to align local neural fields to a global coordinate system [S2], and in SplaTAM for trajectory alignment [S1, unofficial].

## Sources
[S1] spla-tam/SplaTAM, https://github.com/spla-tam/SplaTAM, fetched 2026-10-27 (unofficial)
[S2] Local-to-Global Registration for Bundle-Adjusting Neural Radiance Fields, https://doi.org/10.1109/cvpr52729.2023.00799, fetched 2026-10-29
[S3] Multimodal volume-based tumor neurosurgery planning in the virtual workbench, https://doi.org/10.1007/bfb0056289, fetched 2026-10-26
