# Gaussian Model Initialization

Also called: Flame-initialized Gaussian Initialization, LIVGaussMap, LIVGaussMap Initialization, LiDAR-based Gaussian Initialization, PSAvatar

## Overview
Gaussian Model Initialization establishes the initial state of a 3D Gaussian Splatting representation from input data, typically a sparse point cloud, to begin differentiable optimization. It differs from [Random Large-Variance Initialization](6129-random-large-variance-initialization.md) by using structured geometric priors rather than random noise, and from [Gaussian Geometry Parameters Prediction](8442-gaussian-geometry-parameters-prediction.md) by relying on explicit data mapping rather than learned prediction networks. It serves as the foundational step for [Gaussian Splatting Initialization and Parameterization](2313-gaussian-splatting-initialization-and-parameterization.md).

## How it works
The method initializes a set of 3D Gaussians by mapping input geometric and photometric data to learnable parameters. The standard pipeline involves creating a `GaussianModel` instance, populating it with point cloud data, and setting up the optimizer.

### Inputs and Outputs
*   **Inputs:**
    *   Point cloud data: A set of $N$ points with 3D coordinates $\mathbf{xyz} \in \mathbb{R}^{N \times 3}$ and RGB colors $\mathbf{rgb} \in \mathbb{R}^{N \times 3}$ [S2, unofficial].
    *   Optional: LiDAR data or explicit face models (e.g., FLAME) for specific domains [S1].
*   **Outputs:**
    *   A `GaussianModel` object with initialized tensors for position ($\mathbf{\mu}$), opacity ($\alpha$), scaling ($\mathbf{s}$), rotation ($\mathbf{q}$), and spherical harmonic coefficients ($\mathbf{c}_{sh}$) [S2, unofficial].

### Data Structures
The `GaussianModel` maintains the following learnable parameters (fields) for $N$ Gaussians:
1.  `_xyz`: Position, shape $(N, 3)$, initialized from input point cloud coordinates [S2, unofficial].
2.  `_features_dc`: Zeroth-order spherical harmonic coefficients, shape $(N, 3)$, initialized from input RGB colors [S1].
3.  `_features_rest`: Higher-order SH coefficients, shape $(N, 15 \times 3)$, initialized to zeros [S1].
4.  `_scaling`: Log-scale parameters, shape $(N, 3)$, initialized to small random values or derived from nearest-neighbor distances [S3].
5.  `_rotation`: Quaternions, shape $(N, 4)$, initialized to identity $(1, 0, 0, 0)$ or random uniform [S1, S3].
6.  `_opacity`: Opacity values, shape $(N, 1)$, initialized to a small constant (e.g., 0.1) or derived from density [S2, unofficial][S3].

### Steps
1.  **Model Instantiation:** Create the `GaussianModel` and move it to the target device (CPU or CUDA) [S2, unofficial].
2.  **Point Cloud Population:** Call `create_from_pcd(xyz, rgb)` to populate the model. This function:
    *   Sets `_xyz` to the input `xyz` [S2, unofficial].
    *   Sets `_features_dc` to the input `rgb` [S1].
    *   Initializes `_features_rest` to zeros [S1].
    *   Initializes `_opacity` to a small constant (e.g., 0.1) [S2, unofficial].
    *   Initializes `_scaling` and `_rotation` to default values (small random scale, identity rotation) [S2, unofficial].
3.  **Optimizer Setup:** Configure the optimizer with specific learning rates for different parameter groups [S2, unofficial]:
    *   Learning rate for `_xyz`: $1.6 \times 10^{-4}$ [S2, unofficial].
    *   Learning rate for `_features_dc`: $2.5 \times 10^{-3}$ [S2, unofficial].
    *   Learning rates for other parameters (scaling, rotation, opacity) follow standard 3DGS defaults unless specified otherwise [S3].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

class GaussianModel:
    def __init__(self, device='cpu'):
        self.device = device
        self._xyz = None
        self._features_dc = None
        self._features_rest = None
        self._scaling = None
        self._rotation = None
        self._opacity = None

    def create_from_pcd(self, xyz, rgb):
        """
        Initialize GaussianModel from point cloud data.
        xyz: np.ndarray or torch.Tensor of shape (N, 3)
        rgb: np.ndarray or torch.Tensor of shape (N, 3)
        """
        N = xyz.shape[0]
        
        # Move to device if necessary
        if isinstance(xyz, np.ndarray):
            xyz = torch.tensor(xyz, dtype=torch.float32, device=self.device)
        if isinstance(rgb, np.ndarray):
            rgb = torch.tensor(rgb, dtype=torch.float32, device=self.device)
            
        # Initialize position
        self._xyz = xyz.clone()
        
        # Initialize color (DC SH)
        self._features_dc = rgb.clone()
        
        # Initialize higher-order SH to zeros
        self._features_rest = torch.zeros((N, 15, 3), device=self.device)
        
        # Initialize opacity to a small constant
        self._opacity = torch.full((N, 1), 0.1, device=self.device)
        
        # Initialize scaling to small random values
        self._scaling = torch.rand((N, 3), device=self.device) * 0.1
        
        # Initialize rotation to identity quaternions
        self._rotation = torch.zeros((N, 4), device=self.device)
        self._rotation[:, 0] = 1.0
        
        # Verify covariance symmetry (Pitfall check)
        # Covariance = R * S * S^T * R^T
        # This should be symmetric by construction if implemented correctly.
        # The test script checks torch.allclose(cov0, cov0.T, atol=1e-6) [S2, unofficial]

    def get_covariance(self):
        """
        Compute covariance matrices from scaling and rotation.
        Returns shape (N, 3, 3)
        """
        # Extract scaling and rotation
        scales = torch.exp(self._scaling) # Log-exp trick
        quats = self._rotation / self._rotation.norm(dim=1, keepdim=True)
        
        # Build rotation matrices from quaternions
        # R is shape (N, 3, 3)
        R = quat_to_rotation_matrix(quats)
        
        # Build scaling matrices
        S = torch.zeros((self._xyz.shape[0], 3, 3), device=self.device)
        for i in range(3):
            S[:, i, i] = scales[:, i]
            
        # Covariance = R * S * S^T * R^T
        # Since S is diagonal, S * S^T = S^2
        cov = R @ (S @ S) @ R.transpose(1, 2)
        return cov

def quat_to_rotation_matrix(quats):
    """
    Convert quaternions (w, x, y, z) to rotation matrices.
    quats: (N, 4)
    Returns: (N, 3, 3)
    """
    w, x, y, z = quats[:, 0], quats[:, 1], quats[:, 2], quats[:, 3]
    
    # Standard quaternion to rotation matrix formula
    # See [S2, unofficial] for implementation details
    # This is a standard math operation, no library calls
    xx, yy, zz = x*x, y*y, z*z
    xy, xz, yz = x*y, x*z, y*z
    wx, wy, wz = w*x, w*y, w*z
    
    R = torch.zeros((quats.shape[0], 3, 3), device=quats.device)
    R[:, 0, 0] = 1 - 2 * (yy + zz)
    R[:, 0, 1] = 2 * (xy - wz)
    R[:, 0, 2] = 2 * (xz + wy)
    R[:, 1, 0] = 2 * (xy + wz)
    R[:, 1, 1] = 1 - 2 * (xx + zz)
    R[:, 1, 2] = 2 * (yz - wx)
    R[:, 2, 0] = 2 * (xz - wy)
    R[:, 2, 1] = 2 * (yz + wx)
    R[:, 2, 2] = 1 - 2 * (xx + yy)
    return R

# Example Usage
# model = GaussianModel(device='cuda')
# model.create_from_pcd(xyz, rgb)
# optimizer = torch.optim.Adam([
#     {'params': model._xyz, 'lr': 0.00016},
#     {'params': model._features_dc, 'lr': 0.0025},
#     # ... other params
# ], lr=0.0001)
```

### Parameters
*   **Learning Rate for `_xyz`:** $1.6 \times 10^{-4}$ [S2, unofficial].
*   **Learning Rate for `_features_dc`:** $2.5 \times 10^{-3}$ [S2, unofficial].
*   **Initial Opacity:** $0.1$ [S2, unofficial].

### Edge Cases and Pitfalls
*   **Covariance Symmetry:** The covariance matrix computed from scaling and rotation must be symmetric. If the rotation matrix construction or scaling application is flawed, the resulting covariance may not be symmetric, causing numerical instability in rasterization. A test script checks `torch.allclose(cov0, cov0.T, atol=1e-6)` to ensure this property holds [S2, unofficial].
*   **LiDAR/SLAM Specifics:** For LiDAR-based initialization (e.g., LIVGaussMap), an optimizable size-adaptive voxel grid is built for the global map to stabilize localization and mapping [S1].
*   **Facial Geometry:** For human head modeling, explicit FLAME face models can be used to initialize Gaussians to capture high-fidelity geometry and volumetric objects like glasses [S1].

## Subtopics

### [Spherical Harmonics Initialization](3163-spherical-harmonics-initialization.md)
Spherical Harmonics Initialization sets the initial view-dependent color coefficients for 3D Gaussian splats by assigning input RGB colors to the zeroth-order (DC) spherical harmonic term while zeroing all higher-order coefficients. It differs from [Angle-pose Cuboid Uniform Initialization (ACUI)](4503-angle-pose-cuboid-uniform-initialization-acui.md) and [DreamGaussian](5998-dreamgaussian.md) by focusing exclusively on the color parameterization rather than spatial position, scale, or rotation. Unlike [Spherical Harmonics Evaluation](3848-spherical-harmonics-evaluation.md), which computes color at runtime, this method prepares the static coefficient tensor for optimization.

### [Angle-pose Cuboid Uniform Initialization (ACUI)](4503-angle-pose-cuboid-uniform-initialization-acui.md)
Angle-pose Cuboid Uniform Initialization (ACUI) is a deterministic initialization strategy for 3D Gaussian Splatting in X-ray novel view synthesis that replaces Structure-from-Motion (SfM) by computing camera parameters directly from scanner specifications and uniformly sampling Gaussian centers within a bounding cuboid. It solves the problem of feature detection failure in low-contrast, overlapping X-ray projections, offering a significant reduction in training time compared to SfM-based initialization.

### [DreamGaussian](5998-dreamgaussian.md)
DreamGaussian is a text-to-3D generation method that replaces the MipNeRF representation in the DreamFusion framework with 3D Gaussian Splatting (3DGS), using Score Distillation Sampling (SDS) loss to optimize 3D Gaussians. It is distinguished from its siblings by its two-stage pipeline: initial SDS optimization of Gaussians followed by UV-texture refinement on an extracted mesh, enabling the generation of a textured mesh in approximately 2 minutes. Unlike [Gaussian Asset Generation](7625-gaussian-asset-generation.md) which may use feed-forward models, DreamGaussian relies on iterative optimization.

### [Gaussian Asset Generation](7625-gaussian-asset-generation.md)
Gaussian Asset Generation is a deterministic method for creating a reproducible 3D Gaussian Splatting PLY asset, specifically a dense "rainbow sphere" shell, without using randomness or external data. It solves the problem of needing a standardized, visually recognizable test asset for validating parsers and renderers (such as SceneView's PlyParser) by generating a synthetic point cloud with specific color, scale, opacity, and rotation parameters. It differs from other initialization methods like [Gaussian Model Initialization](2349-gaussian-model-initialization.md) or [DreamGaussian](5998-dreamgaussian.md) by being purely synthetic and deterministic, rather than derived from input images or learned optimization.

### [Gaussian Initialization and Warm-up](9657-gaussian-initialization-and-warm-up.md)

### R2-Gaussian Initialization
R2-Gaussian Initialization is a method for tomographic reconstruction that initializes radiative Gaussians using a low-quality volume reconstructed via the FDK algorithm in less than 1 second [S3]. It excludes empty spaces with a density threshold $\tau$ and randomly samples $M$ points as kernel positions [S3]. The scales of Gaussians are set as the nearest neighbor distances, and rotation is assumed to be zero [S3]. Central densities are queried from the FDK volume and scaled down with $k$ to compensate for overlay between kernels [S3]. Typical parameters are $M=50k$, $\tau=0.05$, and $k=0.15$ [S3].

### GS-W Sampling Coordinate Initialization
GS-W Sampling Coordinate Initialization equips each Gaussian point with $K$ learnable sampling coordinates to enable adaptive feature sampling on $K$ feature maps [S4]. It assigns each Gaussian point $i$ $K$ sampling coordinates $(sc_1^i, sc_2^i, \dots, sc_K^i) \in \mathbb{R}^{K \times 2}$ [S4]. It randomly generates $K$ matrices $(M^1, M^2, \dots, M^K) \in \mathbb{R}^{K \times 2 \times 3}$ and constrains each matrix such that the sum of its rows equals 1 [S4]. The initial sampling coordinates are computed as $(sc_1^1, sc_2^2, \dots, sc_k^K)_{init} = (M^1 X_i, M^2 X_i, \dots, M^K X_i)$ [S4]. The number of feature maps $K$ is typically set to 3 to balance efficiency and effectiveness [S4].

### Persistent Material Descriptors
Persistent Material Descriptors attach samples $(m_{ik}, b_{ik})$ with weights based on initial Gaussian support at sampled voxels, keeping identifiers and weights fixed during optimization [S5, unofficial]. The weight is computed as $w_{ik} \propto \exp\left(-\frac{1}{2}(\mathbf{x}_{ik} - \boldsymbol{\mu}_i)^\top \Sigma_i^{-1}(\mathbf{x}_{ik} - \boldsymbol{\mu}_i)\right)$, normalized such that $\sum_k w_{ik} = 1$ [S5, unofficial]. Identifiers and weights remain fixed as each primitive's geometry is optimized [S5, unofficial]. Clone and split children inherit descriptors and factor rows from the nearest pre-existing primitive, and pruning applies one mask to all attributes [S5, unofficial]. Equation 12 makes the descriptor scale-aware: a sub-voxel primitive concentrates on its own material, whereas a larger primitive integrates over its footprint [S5, unofficial].

### PhysGaussian
PhysGaussian is a method using the Material Point Method (MPM) over Gaussians, providing one representation for both rendering and simulation of materials like cloth, jelly, sand, and snow [S6, unofficial]. It was introduced in CVPR'24 [S6, unofficial].

### GaussianEditor
GaussianEditor is a text-driven local editing method using Hierarchical Gaussian Semantic Tracing and diffusion-guided refinement [S6, unofficial]. It was introduced by Chen et al. in CVPR'24 [S6, unofficial].

### GMM to Fuzzy Metaball Conversion
GMM to Fuzzy Metaball Conversion is a method to convert other representations (meshes, point clouds) into Fuzzy Metaballs (Gaussian Mixture Models) by fitting a GMM [S7]. Two types are described: surface GMM (sGMM) and volumetric GMM (vGMM) [S7]. To build an sGMM, points are sampled from the surface of the object and a GMM is fit using Expectation-Maximization [S7]. To build a vGMM, points are sampled from the interior of a watertight mesh and a GMM is fit using Expectation-Maximization [S7]. The resulting GMM is optimized using a differentiable renderer to refine the fit [S7]. A mesh can be extracted by running marching cubes with an optimized iso-surface level [S7]. The authors use vGMMs in their experiments, although both sGMM and vGMM initialization work well [S7].

### Geometry Reconstruction via LTM
Geometry Reconstruction via LTM is a method to extract a lightweight mesh with moderate triangle count from a neural field for initializing Gaussian splats, using aggressive mesh decimation based on the Quadric Error Metric to avoid redundant splats in distorted regions [S8]. It jointly trains two multilayer perceptrons (MLPs) to encode the signed distance field $f_{sdf}$ and the appearance field $g_{color}$, optimized using differentiable volumetric rendering and Eikonal regularization [S8]. The Marching Cube algorithm is employed to obtain a mesh $M = \{V, F\}$ with vertices $V$ and faces $F$ extracted from $f_{sdf}$ [S8]. Aggressive mesh decimation based on the Quadric Error Metric is applied to efficiently represent moderate geometry with fewer triangles [S8]. The LTM code is used for extracting mesh from neural field and decimating those mesh [S8].

### Shorter Gaussian Lists
Shorter Gaussian Lists investigates training from a tiny initial set of Gaussians to achieve comparable quality and accelerated pipeline [S9, unofficial].

## Operating characteristics
*   **Time Complexity:** Initialization is generally $O(N)$ where $N$ is the number of points in the input point cloud [S2, unofficial].
*   **Memory Complexity:** $O(N)$ to store the Gaussian parameters [S2, unofficial].
*   **Performance:** Standard initialization from COLMAP point clouds is efficient and widely used [S2, unofficial]. LiDAR-based initialization can be faster for large-scale scenes but requires additional processing for voxel grid construction [S1].
*   **Strengths:** Provides a geometrically meaningful starting point for optimization, leading to faster convergence and better final quality compared to random initialization [S1][S2, unofficial].
*   **Limitations:** Depends on the quality of the input point cloud. Poor quality or sparse point clouds can lead to suboptimal initialization [S1].
*   **When to use:** When high-quality input data (COLMAP, LiDAR) is available.
*   **When not to use:** When input data is unavailable or of very poor quality, consider [Random Large-Variance Initialization](6129-random-large-variance-initialization.md) or [Gaussian Geometry Parameters Prediction](8442-gaussian-geometry-parameters-prediction.md).

## Background
Gaussian Model Initialization was introduced as part of the original 3D Gaussian Splatting paper [S2, unofficial]. It has since been extended with various domain-specific methods, such as LiDAR-based initialization for SLAM [S1], FLAME-based initialization for human avatars [S1], and FDK-based initialization for tomographic reconstruction [S3].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] metin-yat/mini-splat, https://github.com/metin-yat/mini-splat, fetched 2026-10-28 (unofficial)
[S3] R$^2$-Gaussian: Rectifying Radiative Gaussian Splatting for Tomographic Reconstruction, https://arxiv.org/abs/2405.20693v2, fetched 2026-10-26
[S4] Gaussian in the Wild: 3D Gaussian Splatting for Unconstrained Image Collections, https://doi.org/10.1007/978-3-031-73116-7_20, fetched 2026-10-27
[S5] FactorSplat: Appearance-Controllable Gaussian Proxies for Medical Volume Rendering, https://doi.org/10.48550/arxiv.2610.02382, fetched 2026-10-26 (unofficial)
[S6] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S7] Approximate Differentiable Rendering with Algebraic Surfaces, https://doi.org/10.1007/978-3-031-19824-3_35, fetched 2026-10-26
[S8] MeshGS: Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering, https://arxiv.org/abs/2410.08941v1, fetched 2026-10-26
[S9] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
