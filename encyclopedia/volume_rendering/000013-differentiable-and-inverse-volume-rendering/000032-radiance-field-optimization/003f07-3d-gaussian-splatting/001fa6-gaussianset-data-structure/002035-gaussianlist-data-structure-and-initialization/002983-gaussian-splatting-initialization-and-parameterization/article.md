# Gaussian Splatting Initialization and Parameterization

Also called: SfM Initialization, Structure from Motion Initialization

## Overview
Gaussian Splatting Initialization and Parameterization establishes the initial state of a 3D Gaussian Splatting representation from input data, typically a sparse point cloud, to begin differentiable optimization [S3, S6, unofficial]. It differs from [Random Large-Variance Initialization](6129-random-large-variance-initialization.md) by using structured geometric priors rather than random noise, and from [Gaussian Geometry Parameters Prediction](8442-gaussian-geometry-parameters-prediction.md) by relying on explicit data mapping rather than learned prediction networks [S3, S6, unofficial]. It serves as the foundational step for [Gaussian Splatting Initialization and Parameterization](2313-gaussian-splatting-initialization-and-parameterization.md).

## How it works
**Inputs:**
1.  Input images (for SfM) or a pre-computed 3D point cloud (e.g., from COLMAP) containing XYZ coordinates and RGB colors [S3, S5, S6, unofficial].
2.  Camera parameters (intrinsics and extrinsics) if performing SfM [S6, unofficial].
3.  Optional depth maps and normal maps if using depth-guided initialization [S2, unofficial].

**Outputs:**
A set of $N$ learnable 3D Gaussian parameters ready for training:
*   Positions: $\mathbf{xyz} \in \mathbb{R}^{N \times 3}$
*   Scales: $\mathbf{s} \in \mathbb{R}^{N \times 3}$ (log-space pre-activation)
*   Rotations: $\mathbf{q} \in \mathbb{R}^{N \times 4}$ (quaternions)
*   Opacities: $\alpha \in \mathbb{R}^{N}$ (logit pre-activation)
*   Colors: $\mathbf{c} \in \mathbb{R}^{N \times 3}$ (SH degree 0) and $\mathbf{f}_{rest} \in \mathbb{R}^{N \times (N_{SH}-1) \times 3}$ (higher degrees) [S3, S5, unofficial].

**Data Structures:**
Parameters are stored as differentiable tensors (e.g., `nn.Parameter` in PyTorch) on the GPU [S5, unofficial].
*   `_xyz`: Shape $(N, 3)$, float32.
*   `_scaling`: Shape $(N, 3)$, float32.
*   `_rotation`: Shape $(N, 4)$, float32.
*   `_opacity`: Shape $(N, 1)$, float32.
*   `_features_dc`: Shape $(N, 1, 3)$, float32.
*   `_features_rest`: Shape $(N, N_{SH}-1, 3)$, float32 [S5, unofficial].

**Steps:**

1.  **Position Initialization:**
    *   If using SfM/COLMAP: Extract XYZ coordinates from the 3D point cloud and copy them directly to the GPU memory as the initial means $\mathbf{xyz}$ [S1, S5, S6, unofficial].
    *   If using Depth-guided: Estimate normal maps from spatial gradients of depth maps (e.g., from Depth Anything) to orient initial surfels. Initialize positions from the depth map points [S2, unofficial].
    *   If using Random: Generate points in a bounding volume (not covered in detail here, see [Random Large-Variance Initialization](6129-random-large-variance-initialization.md)) [S3, unofficial].

2.  **Rotation Initialization:**
    *   **Identity (Standard):** Initialize all quaternions to identity $[1, 0, 0, 0]$ [S3, S5, unofficial].
    *   **Normal-Aligned (Depth-guided):** Compute the quaternion that aligns the local z-axis with the estimated surface normal for each point [S2, unofficial].
    *   **Random (Fallback):** If no geometric prior is available, generate random unit quaternions using the method from [Sho94]:
        $$
        \begin{aligned}
        q_0 &= \sqrt{1-u} \sin(2\pi v) \\
        q_1 &= \sqrt{1-u} \cos(2\pi v) \\
        q_2 &= \sqrt{u} \sin(2\pi w) \\
        q_3 &= \sqrt{u} \cos(2\pi w)
        \end{aligned}
        $$
        where $u, v, w$ are uniform random numbers in $[0, 1]$ [S1, unofficial].

3.  **Scale Initialization:**
    *   Compute the distance to the $k$-nearest neighbors (KNN) for each point in the input cloud. Common $k$ is 50 [S3, unofficial].
    *   Calculate the mean KNN distance $d$.
    *   Apply a scaling factor $s_{init}$ (typically 1.5 for depth-guided, or derived from a base distance) [S2, S5, unofficial].
    *   Compute the log-scale pre-activation:
        $$
        \mathbf{s} = \log(\sqrt{d^2}) \quad \text{or} \quad \mathbf{s} = \log(\sqrt{\max(d^2, \epsilon)})
        $$
        where $\epsilon = 10^{-7}$ prevents numerical instability [S5, unofficial].
    *   Replicate this value across the three dimensions (x, y, z) for isotropic scaling, or use distinct distances for anisotropic scaling [S1, S3, unofficial].
    *   **Clamping:** During training, actual scales are clamped to a maximum value (e.g., 10.0) to prevent excessively large Gaussians [S5, unofficial].

4.  **Opacity Initialization:**
    *   Initialize the logit pre-activation value.
    *   **Standard:** Set to 8.0, which yields $\sigma(8.0) \approx 0.9997$ [S3, unofficial].
    *   **Alternative:** Set to the logit of 0.1, i.e., $\log(0.1 / 0.9)$ [S1, unofficial].
    *   **Clamping:** Clamp the initial logit value to the range $[\log(10^{-4}/0.99), \log(0.99/10^{-4})]$ to prevent extreme values during optimization [S5, unofficial].

5.  **Color (SH) Initialization:**
    *   **Degree 0 ($f_{dc}$):** Convert input RGB colors (normalized to $[-0.5, 0.5]$) by dividing by the SH basis constant $C_0 = 0.28209479177387814$:
        $$
        f_{dc} = \frac{\mathbf{c}_{rgb}}{C_0}
        $$
    *   **Higher Degrees ($f_{rest}$):** Initialize to zeros [S1, S5, unofficial].
    *   **Metallic/Roughness (if applicable):** Initialize metallic to 0.5 and roughness to 0.1 [S2, unofficial].

6.  **Covariance Construction (Runtime/Forward Pass):**
    *   Construct the diagonal scaling matrix $S = \text{diag}(e^{\mathbf{s}})$ [S5, unofficial].
    *   Normalize the rotation quaternion $\mathbf{q}$ to unit length and convert to a rotation matrix $R \in \mathbb{R}^{3 \times 3}$ [S5, unofficial].
    *   Compute the 3D covariance matrix:
        $$
        \Sigma = R S S^T R^T
        $$
    *   **2D Projection:** For rendering, transform to camera space. Let $W$ be the world-to-camera rotation matrix and $J$ be the Jacobian of the perspective projection:
        $$
        J = \begin{bmatrix}
        f/z & 0 & -f x/z^2 \\
        0 & f/z & -f y/z^2
        \end{bmatrix}
        $$
        where $f$ is focal length, $z$ is depth (clamped with $10^{-7}$), and $(x,y)$ are camera-space coordinates [S5, unofficial].
    *   Compute 2D covariance:
        $$
        \Sigma' = J W \Sigma W^T J^T
        $$
    *   Add numerical stability bias $\beta = 0.3$ to the diagonal elements:
        $$
        \Sigma'_{ii} \leftarrow \Sigma'_{ii} + 0.3 \quad \text{for } i \in \{0, 1\}
        $$
        [S5, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def initialize_gaussians(point_cloud_xyz, point_cloud_rgb, num_neighbors=50):
    """
    Initialize 3D Gaussian parameters from a COLMAP-style point cloud.
    
    Args:
        point_cloud_xyz: numpy array of shape (N, 3)
        point_cloud_rgb: numpy array of shape (N, 3), values in [0, 1]
        num_neighbors: int, number of neighbors for KNN distance estimation
        
    Returns:
        dict of initialized torch tensors
    """
    N = point_cloud_xyz.shape[0]
    device = torch.device('cuda')
    
    # 1. Positions
    xyz = torch.tensor(point_cloud_xyz, dtype=torch.float32, device=device)
    
    # 2. Scales (Isotropic from KNN)
    # Compute KNN distances (simplified: using Euclidean distance to all points for demo)
    # In practice, use a KD-tree for efficiency
    dists = torch.cdist(xyz, xyz)
    # Exclude self (distance 0) by setting diagonal to inf
    dists.fill_diagonal_(float('inf'))
    # Get mean of k nearest neighbors
    k_dists, _ = torch.topk(dists, k=num_neighbors, largest=False)
    mean_knn_dist = torch.mean(k_dists, dim=1) # Shape (N,)
    
    # Log-scale initialization
    # Clamp minimum distance to avoid log(0)
    min_dist = 1e-7
    log_scales = torch.log(torch.sqrt(torch.clamp(mean_knn_dist, min=min_dist)))
    
    # Replicate to 3 dimensions for isotropic scaling
    scaling = log_scales.unsqueeze(1).repeat(1, 3)
    
    # 3. Rotations (Identity)
    rotation = torch.zeros((N, 4), dtype=torch.float32, device=device)
    rotation[:, 0] = 1.0 # w=1, x=y=z=0
    
    # 4. Opacity (Logit of 0.5 or 8.0)
    # Using 8.0 as per [S3, unofficial]
    opacity = torch.full((N, 1), 8.0, dtype=torch.float32, device=device)
    
    # 5. Colors (SH Degree 0)
    C0 = 0.28209479177387814
    # Normalize RGB to [-0.5, 0.5] then divide by C0
    rgb_normalized = torch.tensor(point_cloud_rgb, dtype=torch.float32, device=device) * 2.0 - 1.0
    features_dc = rgb_normalized / C0
    features_dc = features_dc.unsqueeze(1) # Shape (N, 1, 3)
    
    # Higher degree SH coefficients initialized to 0
    # Assuming degree 3, there are 15 coefficients total, 1 is dc, so 14 rest
    num_rest_coeffs = 14
    features_rest = torch.zeros((N, num_rest_coeffs, 3), dtype=torch.float32, device=device)
    
    return {
        'xyz': xyz,
        'scaling': scaling,
        'rotation': rotation,
        'opacity': opacity,
        'features_dc': features_dc,
        'features_rest': features_rest
    }

def compute_covariance(scaling, rotation):
    """
    Compute 3D covariance matrix from scaling and rotation.
    
    Args:
        scaling: tensor (N, 3)
        rotation: tensor (N, 4)
        
    Returns:
        covariance: tensor (N, 3, 3)
    """
    # Convert quaternions to rotation matrices
    # Assuming a standard quaternion_to_rotation_matrix function
    R = quaternion_to_rotation_matrix(rotation) # Shape (N, 3, 3)
    
    # Create diagonal scaling matrix
    # S = diag(exp(scaling))
    s = torch.exp(scaling) # Shape (N, 3)
    S = torch.diag_embed(s) # Shape (N, 3, 3)
    
    # Sigma = R S S^T R^T
    # Since S is diagonal, S S^T = diag(s^2)
    SS_T = torch.diag_embed(s**2)
    covariance = R @ SS_T @ R.transpose(1, 2)
    
    return covariance

def quaternion_to_rotation_matrix(q):
    """
    Convert quaternions to rotation matrices.
    q: (N, 4) with format [w, x, y, z]
    """
    # Normalize quaternions
    q = torch.nn.functional.normalize(q, dim=1)
    w, x, y, z = q[:, 0], q[:, 1], q[:, 2], q[:, 3]
    
    # Standard formula
    R = torch.zeros((q.shape[0], 3, 3), device=q.device)
    R[:, 0, 0] = 1 - 2 * (y**2 + z**2)
    R[:, 0, 1] = 2 * (x*y - w*z)
    R[:, 0, 2] = 2 * (x*z + w*y)
    R[:, 1, 0] = 2 * (x*y + w*z)
    R[:, 1, 1] = 1 - 2 * (x**2 + z**2)
    R[:, 1, 2] = 2 * (y*z - w*x)
    R[:, 2, 0] = 2 * (x*z - w*y)
    R[:, 2, 1] = 2 * (y*z + w*x)
    R[:, 2, 2] = 1 - 2 * (x**2 + y**2)
    
    return R
```

## Subtopics

### [Gaussian Model Initialization](2349-gaussian-model-initialization.md)
Gaussian Model Initialization establishes the initial state of a 3D Gaussian Splatting representation from input data, typically a sparse point cloud, to begin differentiable optimization. It differs from [Random Large-Variance Initialization](6129-random-large-variance-initialization.md) by using structured geometric priors rather than random noise, and from [Gaussian Geometry Parameters Prediction](8442-gaussian-geometry-parameters-prediction.md) by relying on explicit data mapping rather than learned prediction networks. It serves as the foundational step for [Gaussian Splatting Initialization and Parameterization](2313-gaussian-splatting-initialization-and-parameterization.md).

### [Random Large-Variance Initialization](6129-random-large-variance-initialization.md)
Random Large-Variance Initialization (RAIN-GS) relaxes the strict Structure-from-Motion (SfM) point cloud assumption required by standard 3D Gaussian Splatting by introducing sparse, large-variance random initialization. Unlike standard methods that rely solely on dense SfM points, this approach allows optimization to proceed even when SfM data is sparse or absent, serving as a fallback or complementary initialization strategy.

### [Temporal Point Cloud Initialization via Chamfer Distance Minimization](6954-temporal-point-cloud-initialization-via-chamfer-distance-min.md)
This method reduces training time for dynamic video scenes by initializing the point cloud of the current frame using the learned model from the previous frame, leveraging temporal correlation between nearby frames. It sets itself apart from static initialization methods by explicitly optimizing geometric alignment between consecutive frames via Chamfer distance minimization before refining radiance parameters.

### [Gaussian Geometry Parameters Prediction](8442-gaussian-geometry-parameters-prediction.md)
Gaussian Geometry Parameters Prediction is a feed-forward method that predicts the geometric attributes (position, opacity, and covariance) of 3D Gaussians from a global volumetric feature representation, rather than regressing them per-pixel from 2D features. It solves the problem of multi-view inconsistencies and ghosting artifacts common in pixel-aligned 3DGS methods by accumulating depth information into a unified global volume and refining Gaussian locations in 3D space. Unlike standard initialization which fixes geometry from point clouds, this method learns to adjust initial noisy positions via recursive offset prediction to achieve faithful reconstruction.

### [Background Hemisphere Initialization](8446-background-hemisphere-initialization.md)
Background Hemisphere Initialization addresses the problem of rendering distant background regions (typically hundreds of meters away) in large-scale urban or outdoor scenes where standard foreground Gaussian splatting fails due to sparsity and occlusion. It models the background as a fixed-radius hemisphere that moves with the camera to maintain a constant relative distance, allowing for efficient regression of Gaussian parameters from projected 2D colors. It differs from sibling methods like [Gaussian Model Initialization](2349-gaussian-model-initialization.md) and [Random Large-Variance Initialization](6129-random-large-variance-initialization.md) by explicitly structuring the background geometry as a moving hemisphere rather than initializing points within the foreground volume or randomly in space.

### [ReSplat Initialization](10957-resplat-initialization.md)
ReSplat Initialization predicts an initial set of 3D Gaussians from a sparse point cloud by aggregating local and global 3D context to compensate for information loss incurred by downsampling the input point cloud by 16x. Unlike naive prediction methods that directly map point features to Gaussian parameters, this method uses a transformer-based encoder to enrich features with neighborhood context before decoding. It sets the stage for the subsequent Recurrent Gaussian Update by providing a structured initial state $\mathcal{G}^0$.

### Neural Point Cloud Initialization from COLMAP
A method to initialize the neural point cloud and their features for Point-NeRF using an external point cloud reconstruction (specifically COLMAP). It solves the problem of how to provide initial positions, variances, and feature vectors for the neural points. Initialize the variance parameter $\gamma_i$ for each point $p_i$ to 0.3. During optimization, the confidence score of valid points is pushed to 1. To acquire the feature vector $f_i$ for a point $p_i$, first rule out all views where the point is occluded by other points. Find the view where the camera is closest to the point $p_i$. Unproject the point $p_i$ onto the feature maps extracted by the Image Feature Extraction 2D CNN (Gf) from the selected closest view to obtain $f_i$ [S7].

### Binary Gaussian Hierarchy Construction
The hierarchy is a binary tree. Leaf nodes contain the full set of optimized Gaussian properties (base color, SH coefficients, position, covariance). Internal nodes store merged approximations of their two child nodes [S8, unofficial].

### Single Ellipsoid Initialization
Initializes a single ellipsoid from a set of 3D points using Principal Component Analysis (PCA) to determine orientation, center, and radii. $r_{\min}$: Minimum allowed radius to avoid numerical problems. Typical value: 0.005. $\alpha$: Scaling factor to ensure points are mostly covered. Typical value: 3 [S9].

## Operating characteristics
**Time and Memory Complexity:**
*   Initialization time is dominated by the SfM process if not pre-computed. SfM initialization takes approximately 6.0 minutes on average scenes, compared to 0.1 seconds for random initialization [S4, unofficial].
*   Memory usage is linear with respect to the number of Gaussians $N$, storing position, scale, rotation, opacity, and SH coefficients [S5, unofficial].

**Measured Performance:**
*   On the Average-30K dataset, SfM initialization achieves a PSNR of 26.05, while random initialization drops to 20.42 [S6, unofficial].
*   In EndoGaussian, COLMAP initialization yields 35.201 PSNR but with slow inference (60.28 FPS), while their optimized method achieves 37.849 PSNR with fast inference (195.09 FPS) [S4, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Provides geometrically accurate starting points, significantly improving reconstruction quality and convergence speed compared to random initialization [S6, unofficial].
*   **Limitations:** Requires high-quality SfM data; fails or degrades background quality if SfM points are sparse or missing [S6, unofficial]. Random initialization avoids complete failure but degrades background quality [S6, unofficial].

**When to Use:**
*   Use SfM initialization for static scenes with sufficient image overlap and texture.
*   Use Random or Depth-guided initialization when SfM fails or for dynamic scenes where temporal initialization is preferred [S1, S2, unofficial].

## Background
Originating from the 3D Gaussian Splatting paper (Kerbl et al., 2023), the method relies on Structure from Motion (SfM) to provide initial 3D points [S6, unofficial]. SfM estimates camera poses, intrinsic parameters, and 3D feature point locations from input images [S6, unofficial]. These points serve as the initial positions for the Gaussian distribution, replacing the random initialization used in earlier neural rendering methods [S6, unofficial]. The approach has been extended to depth-guided initialization using normal maps from depth estimators like Depth Anything [S2, unofficial], and to hierarchical or predictive methods for large-scale or dynamic scenes [S1, S2, S8, unofficial].

## Sources
[S1] Show HN: Msplat – 3D Gaussian Splatting training in ~90s on M4 Max, native Metal, https://github.com/rayanht/msplat, fetched 2026-10-26 (unofficial)
[S2] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
[S3] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S4] EndoGaussian: Real-time Gaussian Splatting for Dynamic Endoscopic Scene Reconstruction, https://huggingface.co/papers/2401.12561, fetched 2026-10-30 (unofficial)
[S5] metin-yat/mini-splat, https://github.com/metin-yat/mini-splat, fetched 2026-10-28 (unofficial)
[S6] GaussianSplatting論文のAblationについての解説・考察, https://qiita.com/kotauchisunsun/items/f1e89f28aa770b8d3b19, fetched 2026-10-29 (unofficial)
[S7] Point-NeRF: Point-based Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00536, fetched 2026-10-26
[S8] A LoD of Gaussians: Unified Training and Rendering for Ultra-Large Scale Reconstruction with External Memory, https://doi.org/10.48550/arXiv.2507.01110, fetched 2026-10-28 (unofficial)
[S9] Learning Scene-Level Signed Directional Distance Function With Ellipsoidal Priors and Neural Residuals., https://doi.org/10.1109/tpami.2026.3688658, fetched 2026-10-26
