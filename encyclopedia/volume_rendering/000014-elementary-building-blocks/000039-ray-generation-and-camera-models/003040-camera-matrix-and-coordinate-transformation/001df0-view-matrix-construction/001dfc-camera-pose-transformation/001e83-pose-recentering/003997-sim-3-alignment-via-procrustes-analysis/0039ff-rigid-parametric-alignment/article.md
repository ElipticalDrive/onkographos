# Rigid Parametric Alignment

Also called: Closed-form Rigid Alignment, Horn's Method, Horn's Method for Trajectory Alignment, Least-squares transformation estimation, Point set registration, Rigid Alignment via SVD, Rigid Body Registration via SVD

## Overview
Rigid Parametric Alignment estimates a 3D rigid transformation $T \in SE(3)$ from point correspondences by minimizing alignment error in a closed form using Singular Value Decomposition (SVD) [S2]. It solves the absolute orientation problem in a single deterministic step, distinguishing it from iterative registration methods that require repeated correspondence matching or gradient descent [S1, unofficial][S2]. Unlike [Homography Parametric Alignment](8618-homography-parametric-alignment.md), which estimates a 2D projective transformation for planar distortions, this method operates in 3D Euclidean space to preserve metric distances and angles [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
- **Input:** Two sets of $N$ corresponding 3D points: model points $X = \{x_1, \dots, x_N\}$ and data points $Y = \{y_1, \dots, y_N\}$ [S1, unofficial]. Each set is represented as a $3 \times N$ matrix where columns are points [S1, unofficial].
- **Output:** A rotation matrix $R \in SO(3)$ and a translation vector $t \in \mathbb{R}^3$, forming the transformation $T = [R|t] \in SE(3)$ [S2].

**Data Structures:**
- Points are stored as 3D vectors packed into $3 \times N$ matrices for batched linear algebra operations [S1, unofficial].

**Steps:**
1. **Centroid Calculation:** Compute the centroid (mean) of the model points $\bar{x}$ and the data points $\bar{y}$:
   $$ \bar{x} = \frac{1}{N} \sum_{i=1}^{N} x_i, \quad \bar{y} = \frac{1}{N} \sum_{i=1}^{N} y_i $$
2. **Centering:** Subtract the centroids from the respective points to obtain zero-centered coordinates:
   $$ x'_i = x_i - \bar{x}, \quad y'_i = y_i - \bar{y} $$
3. **Covariance Matrix:** Compute the $3 \times 3$ cross-covariance matrix $H$ between the centered points:
   $$ H = \sum_{i=1}^{N} x'_i (y'_i)^T $$
   *(Note: Source [S1, unofficial] denotes this as $W$ and computes it via outer products of corresponding columns.)*
4. **SVD:** Perform Singular Value Decomposition on $H$:
   $$ H = U S V^T $$
   where $U$ and $V$ are orthogonal matrices and $S$ is a diagonal matrix of singular values [S2].
5. **Rotation Correction:** Initialize a correction matrix $D$ as the identity matrix $I_3$ [G]. To ensure a proper rotation ($\det(R)=1$) and avoid reflection:
   - If $\det(U V^T) < 0$, set the last element of $D$ to $-1$ (i.e., $D = \text{diag}(1, 1, -1)$) [S1, unofficial]. This handles the case where the SVD yields a reflection [G].
6. **Rotation Calculation:** Compute the optimal rotation matrix:
   $$ R = V D U^T $$
   *(Source [S2] states $R = V U^T$ assuming no reflection. Source [S1, unofficial] explicitly includes the sign flip logic.)*
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
    H = np.dot(model_centered.T, data_centered)
    
    # 4. SVD
    U, S, Vt = np.linalg.svd(H)
    
    # 5. Rotation Correction
    # Check for reflection
    if np.linalg.det(np.dot(U, Vt)) < 0:
        # Flip the sign of the last row of Vt (last column of V)
        Vt[-1, :] *= -1
        
    # 6. Rotation
    R = np.dot(Vt.T, U.T)
    
    # 7. Translation
    t = mean_data - np.dot(R, mean_model)
    
    return R, t
```

**Parameters:**
- **Landmark Count:** Minimum 4 non-coplanar points are required for a unique solution in 3D [S3, unofficial].
- **Hidden Units (Contextual):** In specific neural field implementations (e.g., L2G-NeRF), the rigid alignment is parameterized by a ReLU MLP with six 256-dimensional hidden units [S2]. This is a specific architectural choice for differentiable alignment, not a general parameter of the SVD method itself [G].

**Edge Cases and Pitfalls:**
- **Reflection:** If the point sets have different handedness, the SVD may yield a rotation with determinant -1. The correction step is critical to enforce $R \in SO(3)$ [S1, unofficial].
- **Degenerate Configurations:** If all points are collinear or coplanar, the covariance matrix $H$ is rank-deficient, leading to ambiguous rotation solutions [S3, unofficial].
- **Precision:** Floating-point errors in SVD can occasionally result in $R$ not being perfectly orthogonal. Re-orthogonalization (e.g., via SVD of $R$) may be needed for strict numerical stability [G].

## Subtopics

### [Homography Parametric Alignment](8618-homography-parametric-alignment.md)
Estimates a 2D projective transformation $T \in SL(3)$ from point correspondences by solving a homogeneous linear least squares problem via SVD. It is employed in camera-in-the-loop optimization to map captured camera coordinates to Spatial Light Modulator (SLM) pattern coordinates, ensuring the captured wavefront and SLM pattern reside in the same coordinate system. Unlike [Rigid Parametric Alignment](8609-rigid-parametric-alignment.md), which estimates a 3D rigid transformation in $SE(3)$, this method operates in 2D homogeneous space to handle planar or perspective distortions between the sensor and the display plane.

### [Iterative Closest Point (ICP)](10563-iterative-closest-point-icp.md)
A registration algorithm used to align two point sets, typically a reconstructed point cloud and a ground truth mesh, by finding an optimal rigid transformation. It builds on [Rigid Parametric Alignment](8609-rigid-parametric-alignment.md) by iteratively refining the transformation estimate through correspondence matching, rather than solving for a transformation from pre-established correspondences in a single step [G]. Unlike differentiable renderers which optimize pose via gradient descent on pixel errors, ICP operates directly in geometric space.

### [Coarse Camera Pose Estimation](15954-coarse-camera-pose-estimation.md)
An incremental registration method that estimates the initial extrinsic parameters (camera pose) for unposed target images relative to a registered reference image using a learned scene regressor. It solves the problem of initializing camera poses in neural radiance field training by converting predicted 3D points into 2D-3D correspondences and solving for the pose via Perspective-n-Point (PnP). It differs from [Iterative Closest Point (ICP)](10563-iterative-closest-point-icp.md) by relying on a neural scene representation to provide dense 3D structure rather than iterative point-to-point alignment, and from [Structure-from-Motion (SfM)](16341-structure-from-motion-sfm.md) by operating incrementally on a learned regressor rather than globally on feature matches.

### [N-Point Alignment Registration](17730-n-point-alignment-registration.md)
An automatic registration tool that aligns source and destination objects by finding a transformation that minimizes the error between corresponding landmarks. It solves the problem of registering volumetric objects when manual transformation is difficult, using a least-square approximation based on user-specified points. It differs from its parent [Rigid Parametric Alignment](8609-rigid-parametric-alignment.md) by relying on explicit landmark correspondences rather than closed-form SVD minimization of reprojection error, and from [Iterative Closest Point (ICP)](10563-iterative-closest-point-icp.md) by requiring manual point selection rather than automatic nearest-neighbor matching.

### [Geometric Moment Initialization](20995-geometric-moment-initialization.md)
Estimates the initial rigid transformation parameters (translation and rotation) for 3D volume registration by computing the center of gravity and principal axes of the source and target volumes. It solves the problem of poor convergence in mutual information (MI) optimization, where zero-initialization leads to a rugged error surface with many local extrema. It distinguishes itself from siblings like [Homography Parametric Alignment](8618-homography-parametric-alignment.md) and [Iterative Closest Point (ICP)](10563-iterative-closest-point-icp.md) by relying on global volume statistics rather than point correspondences or 2D projections, making it suitable for monomodal intra-subject registration where anatomical structures are rigid but displaced.

## Operating characteristics
- **Time Complexity:** $O(N)$ for centroid and covariance calculation, and $O(1)$ for the fixed-size $3 \times 3$ SVD [G]. Thus, the overall complexity is linear in the number of points $N$ [G].
- **Memory Complexity:** $O(N)$ to store the point sets and $O(1)$ for the covariance matrix [G].
- **Performance:** Provides an exact closed-form solution, making it extremely fast and deterministic [S1, unofficial][S2].
- **Strengths:** No initialization required; computationally efficient [G].
- **Limitations:** Requires known correspondences between points [G].
- **When to Use:** When correspondences are known and the transformation is strictly rigid (no scaling or shear) [G].
- **When Not to Use:** When correspondences are unknown (use ICP or feature matching); when the transformation includes scaling (use Sim(3) Alignment) [G].

## Background
Rigid Parametric Alignment is a fundamental building block in computer vision and robotics for pose estimation and registration [G]. In the context of neural rendering, it is used in methods like L2G-NeRF to align local neural fields to a global coordinate system [S2], and in SplaTAM for trajectory alignment [S1, unofficial].

## Sources
[S1] spla-tam/SplaTAM, https://github.com/spla-tam/SplaTAM, fetched 2026-10-27 (unofficial)
[S2] Local-to-Global Registration for Bundle-Adjusting Neural Radiance Fields, https://doi.org/10.1109/cvpr52729.2023.00799, fetched 2026-10-29
[S3] Intuitive Slice-based Exploration of Volumetric Medical Data, https://doi.org/10.17488/rmib.40.3.3, fetched 2026-10-29 (unofficial)
