# Gaussian Model Initialization

Also called: Flame-initialized Gaussian Initialization, LIVGaussMap, LIVGaussMap Initialization, LiDAR-based Gaussian Initialization, PSAvatar

## Overview
Gaussian Model Initialization establishes the initial state of a 3D Gaussian Splatting (3DGS) representation from input data, typically a sparse point cloud, to enable differentiable rendering optimization. It solves the problem of providing valid geometric and photometric starting parameters (position, scale, rotation, opacity, and color) for the Gaussians. Unlike [Random Large-Variance Initialization](6129-random-large-variance-initialization.md), which scatters Gaussians broadly, this method anchors Gaussians to observed data. It differs from [Gaussian Geometry Parameters Prediction](8442-gaussian-geometry-parameters-prediction.md) by relying on explicit point cloud attributes rather than learned prediction networks, and from [ReSplat Initialization](10957-resplat-initialization.md) by focusing on the initial population rather than re-initialization from existing splats.

## How it works
The standard initialization populates a `GaussianModel` from a Structure-from-Motion (SfM) point cloud (e.g., COLMAP). The process involves creating the model instance, mapping point cloud attributes to learnable parameters, and ensuring mathematical consistency of the covariance matrix.

**Inputs:**
- Point cloud positions $P \in \mathbb{R}^{N \times 3}$ (xyz).
- Point cloud colors $C \in \mathbb{R}^{N \times 3}$ (rgb).
- Device (CPU or CUDA) [S2, unofficial].

**Outputs:**
- `GaussianModel` instance with initialized learnable parameters:
  - `_xyz`: Position tensor.
  - `_features_dc`: Zeroth-order spherical harmonic coefficients (color).
  - `_scaling`: Log-scale tensor.
  - `_rotation`: Quaternion tensor.
  - `_opacity`: Opacity tensor.

**Data Structures:**
The `GaussianModel` stores parameters as PyTorch tensors.
- `_xyz`: Shape $(N, 3)$, dtype float32.
- `_features_dc`: Shape $(N, 1, 3)$, dtype float32.
- `_scaling`: Shape $(N, 3)$, dtype float32.
- `_rotation`: Shape $(N, 4)$, dtype float32 (quaternions).
- `_opacity`: Shape $(N, 1)$, dtype float32.

**Steps:**
1. **Model Instantiation:** Create the `GaussianModel` object and move it to the target device [S2, unofficial].
2. **Population:** Call `create_from_pcd(xyz, rgb)` to populate the model [S2, unofficial].
   - **Position:** Set `_xyz` directly from input $P$.
   - **Color:** Set `_features_dc` from input $C$. The first channel (index 0) is set to the RGB values; other channels are zero [S1].
   - **Opacity:** Initialize `_opacity` to a low value, typically 0.1 [S9].
   - **Scaling:** Initialize `_scaling` based on the local point density. The scale is derived from the distance to the $K_{nn}$ nearest neighbors. Specifically, the scale is set to the nearest neighbor distance [S3, S9].
   - **Rotation:** Initialize `_rotation` to identity quaternions (no rotation) [S3]. For methods requiring surface alignment (e.g., DART), normals are estimated via local PCA on the $K_{nn}$ neighbors to define tangent frames [S9].
3. **Covariance Verification:** Ensure the computed covariance matrix $\Sigma$ is symmetric. The covariance is computed from scaling and rotation as $\Sigma = R S S^T R^T$. A test check is `torch.allclose(cov0, cov0.T, atol=1e-6)` [S2, unofficial].

**Parameters:**
- Learning rate for `_xyz`: 0.00016 [S2, unofficial].
- Learning rate for `_features_dc`: 0.0025 [S2, unofficial].
- $K_{nn}$ (Neighborhood size for scale/normal estimation): 20 [S9].
- Initial opacity: 0.1 [S9].

**Pitfalls:**
- **Covariance Symmetry:** The covariance matrix must be symmetric. If the rotation or scaling implementation is flawed, symmetry checks fail [S2, unofficial].
- **Lack of Normals:** Standard COLMAP point clouds lack surface normals. Surface normals for tangent frames must be estimated via local PCA if required by the specific initialization variant (e.g., DART) [S9].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def initialize_gaussian_model(xyz, rgb, device='cuda'):
    """
    Initialize a GaussianModel from a point cloud.
    xyz: Tensor of shape (N, 3)
    rgb: Tensor of shape (N, 3)
    """
    # 1. Instantiate Model
    model = GaussianModel()
    model = model.to(device)
    
    # 2. Populate from PCD
    # This internal function handles the mapping of xyz, rgb to learnable params
    model.create_from_pcd(xyz, rgb)
    
    # 3. Verify Covariance Symmetry [S2, unofficial]
    covariances = model.get_covariance()
    # covariances shape: (N, 3, 3)
    cov0 = covariances[0]
    assert torch.allclose(cov0, cov0.T, atol=1e-6), "Covariance is not symmetric"
    
    return model
```

## Subtopics

### [Spherical Harmonics Initialization](3163-spherical-harmonics-initialization.md)
Spherical Harmonics Initialization sets the initial view-dependent appearance parameters for 3D Gaussians by populating only the zeroth-order (DC) spherical harmonic coefficients with input RGB colors. It solves the problem of providing a valid, view-independent color baseline for the radiance field before optimization begins. Unlike [Spherical Harmonics Evaluation](3848-spherical-harmonics-evaluation.md), which computes full view-dependent color from existing coefficients, this method initializes the storage structure itself. It differs from [Spherical Harmonics Rotation](3985-spherical-harmonics-rotation.md) by not performing coordinate transformations, and from [Spherical Harmonics Decoding](7901-spherical-harmonics-decoding.md) by focusing on the initial assignment rather than unpacking.

### [Angle-pose Cuboid Uniform Initialization (ACUI)](4503-angle-pose-cuboid-uniform-initialization-acui.md)
Angle-pose Cuboid Uniform Initialization (ACUI) is a deterministic initialization strategy for 3D Gaussian Splatting in X-ray novel view synthesis that replaces Structure-from-Motion (SfM) by computing camera parameters directly from scanner specifications and uniformly sampling Gaussian centers within a bounding cuboid. It solves the problem of feature detection failure in low-contrast, overlapping X-ray projections, offering a significant reduction in training time compared to SfM-based initialization.

### [DreamGaussian](5998-dreamgaussian.md)
DreamGaussian is a text-to-3D generation method that replaces the MipNeRF representation in the DreamFusion framework with 3D Gaussian Splatting (3DGS), using Score Distillation Sampling (SDS) loss to optimize 3D Gaussians. It is distinguished from its siblings by its two-stage pipeline: initial SDS optimization of Gaussians followed by UV-texture refinement on an extracted mesh, enabling the generation of a textured mesh in approximately 2 minutes. Unlike [Gaussian Asset Generation](7625-gaussian-asset-generation.md) which may use feed-forward models, DreamGaussian relies on iterative optimization.

### [GaussianEditor](6004-gaussianeditor.md)
GaussianEditor is a text-driven local editing method for 3D Gaussian Splatting that leverages 2D image-editing models to drive updates of 3D Gaussians. It solves the problem of localized appearance and geometry modification in 3D scenes by repeatedly rendering views, applying 2D edits, and back-propagating the differences to Gaussian parameters. It is distinguished from sibling methods like GScalpel (which uses paint-bucket/boolean logic) and Style-GS (which focuses on stylization) by its use of Hierarchical Gaussian Semantic Tracing and diffusion-guided refinement for precise text-driven localization.

### [Gaussian Asset Generation](7625-gaussian-asset-generation.md)
Gaussian Asset Generation is a deterministic method for creating a reproducible 3D Gaussian Splatting PLY asset, specifically a dense "rainbow sphere" shell, without using randomness or external data. It solves the problem of needing a standardized, visually recognizable test asset for validating parsers and renderers (such as SceneView's PlyParser) by generating a synthetic point cloud with specific color, scale, opacity, and rotation parameters. It differs from other initialization methods like [Gaussian Model Initialization](2349-gaussian-model-initialization.md) or [DreamGaussian](5998-dreamgaussian.md) by being purely synthetic and deterministic, rather than derived from input images or learned optimization.

### [Learning-based and Generative Initialization](12356-learning-based-and-generative-initialization.md)

### [Geometry-aware Gaussian Splatting](12476-geometry-aware-gaussian-splatting.md)
A geometry-aware Gaussian Splatting method that initializes thin Gaussians aligned with surfaces in smoothly connected areas and uses constrained optimization to preserve scene geometry and texture, particularly in low-textured regions.

### [Sparse and Efficient Initialization](12944-sparse-and-efficient-initialization.md)

### R2-Gaussian Initialization
R2-Gaussian Initialization initializes Gaussians for tomographic reconstruction using an analytical volume rather than SfM points [S3]. It uses the Feldkamp-Davis-Kress (FDK) algorithm to reconstruct a low-quality volume in under 1 second [S3]. Empty spaces are excluded using a density threshold $\tau$ (typically 0.05), and $M$ points (typically 50k) are randomly sampled as kernel positions [S3]. Scales are set to nearest neighbor distances, and rotation is assumed to be zero [S3]. Central densities are queried from the FDK volume and scaled down by a factor $k$ (typically 0.15) to compensate for kernel overlay [S3].

### GS-W Sampling Coordinate Initialization
GS-W Sampling Coordinate Initialization assigns $K$ learnable sampling coordinates per Gaussian point to enable adaptive feature sampling on $K$ feature maps [S4]. For each Gaussian point $i$ with position $X_i \in \mathbb{R}^3$, $K$ random matrices $M^k \in \mathbb{R}^{2 \times 3}$ are generated such that the sum of each matrix's rows equals 1 [S4]. The initial sampling coordinates are computed as $(sc_1^1, \dots, sc_K^K)_{init} = (M^1 X_i, \dots, M^K X_i)$ [S4]. The number of feature maps $K$ is typically set to 3 to balance efficiency and effectiveness [S4].

### Persistent Material Descriptors
Persistent Material Descriptors attach samples $(m_{ik}, b_{ik})$ with weights based on initial Gaussian support at sampled voxels, keeping identifiers and weights fixed during optimization [S5, unofficial]. The weight is computed as $w_{ik} \propto \exp\left(-\frac{1}{2}(\mathbf{x}_{ik} - \boldsymbol{\mu}_i)^\top \Sigma_i^{-1}(\mathbf{x}_{ik} - \boldsymbol{\mu}_i)\right)$, normalized such that $\sum_k w_{ik} = 1$, where $(\boldsymbol{\mu}_i, \Sigma_i)$ are the initial center and covariance [S5, unofficial]. Identifiers and weights remain fixed as geometry is optimized. Clone and split children inherit descriptors from the nearest pre-existing primitive [S5, unofficial]. Equation 12 makes the descriptor scale-aware: sub-voxel primitives concentrate on their own material, while larger primitives integrate over their footprint [S5, unofficial].

### PhysGaussian
PhysGaussian is a method introduced in CVPR'24 that uses the Material Point Method (MPM) over Gaussians, providing a single representation for both rendering and simulation of materials like cloth, jelly, sand, and snow [S6, unofficial].

### GMM to Fuzzy Metaball Conversion
GMM to Fuzzy Metaball Conversion converts meshes or point clouds into Fuzzy Metaballs (Gaussian Mixture Models) by fitting a GMM [S7]. For a surface GMM (sGMM), points are sampled from the object's surface and fitted using Expectation-Maximization [S7]. For a volumetric GMM (vGMM), points are sampled from the interior of a watertight mesh and fitted similarly [S7]. The resulting GMM is optimized using a differentiable renderer, and meshes can be extracted via marching cubes with an optimized iso-surface level [S7]. Both forms work well, but vGMMs are used in the source experiments [S7].

### Geometry Reconstruction via LTM
Geometry Reconstruction via LTM extracts a lightweight mesh from a neural field for initializing Gaussian splats, using aggressive mesh decimation to avoid redundant splats [S8]. Two MLPs jointly encode the signed distance field $f_{sdf}$ and appearance field $g_{color}$, optimized with differentiable volumetric rendering and Eikonal regularization [S8]. The Marching Cube algorithm extracts a mesh $M=\{V, F\}$ from $f_{sdf}$ [S8]. Aggressive mesh decimation based on the Quadric Error Metric is applied to reduce triangle count while maintaining moderate geometry quality [S8]. The LTM code is used for extraction and decimation [S8].

### DART Initialization
DART Initialization rewrites the SfM cloud $P$ into an augmented cloud $P_{aug}$ using a replication multiplier $m$ (default 10) [S9]. Each augmented point becomes one 3D Gaussian [S9]. Covariance is estimated from local point density using a tangent-disk radius bounded by $\rho_{min}=0.008$ and $\rho_{max}=0.04$ (fractions of scene radius $R$) [S9]. Opacity is initialized at 0.1, and color from SfM color [S9]. Surface normals are estimated via local PCA with $K_{nn}=20$ neighbors, as COLMAP clouds lack usable normals [S9]. Weiszfeld iterations $T_w=8$ are used in the augmentation process [S9].

## Operating characteristics
- **Time Complexity:** Initialization is generally $O(N)$ for population from a point cloud of size $N$, with $O(N \log N)$ or $O(N)$ for nearest neighbor searches depending on the data structure used [S3, S9].
- **Memory Complexity:** $O(N)$ to store the Gaussian parameters [S2, unofficial].
- **Performance:** SfM-based initialization provides high-quality starting points for textured scenes but fails in low-texture or volumetric settings [S3]. LiDAR-based initialization (LIVGaussMap) stabilizes SLAM systems by providing scale-aware anchors [S1].
- **Strengths:** Provides physically meaningful starting positions and colors, accelerating convergence compared to random initialization [S2, unofficial].
- **Limitations:** Dependent on the quality of the input point cloud; sparse or noisy SfM clouds lead to poor initialization [S9].

## Background
Gaussian Model Initialization originated with the introduction of 3D Gaussian Splatting, which relies on Structure-from-Motion (SfM) point clouds (e.g., from COLMAP) for initialization [S2, unofficial]. Subsequent works have adapted this for specific domains: LIVGaussMap uses LiDAR for SLAM [S1], R2-Gaussian uses FDK volumes for tomography [S3], and PhysGaussian integrates physics simulation [S6, unofficial]. The method is a foundational step in the 3DGS pipeline, ensuring that the differentiable renderer starts with a valid geometric representation.

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] metin-yat/mini-splat, https://github.com/metin-yat/mini-splat, fetched 2026-10-28 (unofficial)
[S3] R$^2$-Gaussian: Rectifying Radiative Gaussian Splatting for Tomographic Reconstruction, https://arxiv.org/abs/2405.20693v2, fetched 2026-10-26
[S4] Gaussian in the Wild: 3D Gaussian Splatting for Unconstrained Image Collections, https://doi.org/10.1007/978-3-031-73116-7_20, fetched 2026-10-27
[S5] FactorSplat: Appearance-Controllable Gaussian Proxies for Medical Volume Rendering, https://doi.org/10.48550/arxiv.2610.02382, fetched 2026-10-26 (unofficial)
[S6] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S7] Approximate Differentiable Rendering with Algebraic Surfaces, https://doi.org/10.1007/978-3-031-19824-3_35, fetched 2026-10-26
[S8] MeshGS: Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering, https://arxiv.org/abs/2410.08941v1, fetched 2026-10-26
[S9] DART-3DGS: Distance-Aware Replication and Age-Gated Densification for 3D Gaussian Splatting, https://europepmc.org/article/PMC/PMC13705904, fetched 2026-10-27
