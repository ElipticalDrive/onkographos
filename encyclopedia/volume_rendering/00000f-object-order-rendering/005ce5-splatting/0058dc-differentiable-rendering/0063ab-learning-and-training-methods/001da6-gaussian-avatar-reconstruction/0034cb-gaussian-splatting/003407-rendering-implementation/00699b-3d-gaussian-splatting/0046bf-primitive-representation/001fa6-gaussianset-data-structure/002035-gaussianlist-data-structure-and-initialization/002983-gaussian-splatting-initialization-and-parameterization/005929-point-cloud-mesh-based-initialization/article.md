# Point Cloud/Mesh-based Initialization
Also called: Geometry-based Gaussian Seeding

## Overview
This family of methods initializes 3D Gaussian Splatting (3DGS) primitives using pre-existing 3D geometry (point clouds or meshes) rather than relying solely on sparse Structure-from-Motion (SfM) outputs. It solves the problem of initializing high-quality radiance fields when multi-view images are unavailable or when the user possesses dense, surface-aligned volumetric data (e.g., MPEG plenoptic point clouds or meshes) [S1]. It differs from [Random Large-Variance Initialization](6129-random-large-variance-initialization.md) by anchoring Gaussians to observed geometry, and from [Gaussian Model Initialization](2349-gaussian-model-initialization.md) by supporting dense mesh inputs and optional fixed-position constraints [S1].

## Shared principles
All methods in this family map input geometric primitives (vertices or points) to Gaussian centers. The core shared mechanism is the derivation of initial covariance parameters (scale and rotation) and color features from the input geometry or associated rendering data.

1.  **Position Mapping:** Input vertices or points $\{p_i\}$ are assigned directly to the `_xyz` tensor of the Gaussian model [S1].
2.  **Rotation Initialization:** The rotation quaternion is initialized to the identity $(1, 0, 0, 0)$ for all Gaussians, implying an initial isotropic orientation unless local normals are available [S1].
3.  **Color Initialization:** The zeroth-order spherical harmonic coefficient (`_features_dc`) is set to the RGB color of the corresponding point or rendered pixel. Higher-order coefficients (`_features_rest`) are zero-initialized [S1].
4.  **Scale Estimation:** Scales are derived from local density or depth. Common heuristics include using the median distance to $k$-nearest neighbors or computing an isometric radius based on focal length and depth [S1].
5.  **Opacity Initialization:** Opacity is initialized to a low value (e.g., raw parameter $\approx -2.197$ corresponding to 0.1) to prevent early overconfidence and allow gradient propagation .

## Subtopics

### [Gaussian Model Initialization](2349-gaussian-model-initialization.md)
Standard procedure for seeding 3DGS from a sparse SfM point cloud (e.g., COLMAP). It maps each 3D point to a Gaussian, initializing position, SH coefficients, scale, rotation, and opacity. It differs from [Random Large-Variance Initialization](6129-random-large-variance-initialization.md) by anchoring to observed geometry, and from [Gaussian Geometry Parameters Prediction](8442-gaussian-geometry-parameters-prediction.md) by using heuristic density estimation rather than neural networks.
*   **When to choose:** When starting with standard SfM sparse points and no dense mesh is available.
*   **Key Details:**
    *   **Scale:** Estimated via $k$-nearest neighbor distances (e.g., $k=20$). A common heuristic sets the scale to a fraction (e.g., 0.1) of the median nearest-neighbor distance.
    *   **Rotation:** Identity quaternion.
    *   **Opacity:** Raw parameter $\approx -2.197$ (target opacity 0.1).
    *   **Learning Rates:** Position $\alpha_{xyz} = 0.00016$, Color $\alpha_{color} = 0.0025$.
    *   **Pitfall:** Covariance must be symmetric; ensure quaternion normalization.

### [3DGS Custom Initialization from Point Cloud/Mesh](16108-3dgs-custom-initialization-from-point-cloud-mesh.md)
Initializes 3DGS from dense 3D point clouds or meshes, replacing the standard SfM initialization. It solves the problem of creating models from existing volumetric data when original multi-view images are unavailable. Unlike standard 3DGS, it uses dense, surface-aligned points, resulting in cleaner models and faster convergence.
*   **When to choose:** When starting with a known dense mesh or plenoptic point cloud, and the original camera poses/images are missing or insufficient.
*   **Key Details:**
    *   **Input:** 3D point cloud or mesh. View-dependent colors are preferred.
    *   **Process:** Renders multi-view images from the geometry to provide training data and camera parameters.
    *   **Adaptive Density Control:** Deactivated if input is dense/clean to preserve geometry; activated if noisy/sparse.
    *   **Position Learning Rate:** Set to $0$ to fix splats on the original surface.
    *   **Other Learning Rates:** $> 0$ for opacity, scale, rotation, color.

### 3D Gaussian Initialization
A variant focusing on isometric spherical initialization based on depth.
*   **Rotation:** Identity matrix [S1].
*   **Scale:** Isometric spheres. Radius $r$ is computed as:
    $$ r = \frac{f \cdot D^{\text{init}}}{H} $$
    where $f$ is the vertical focal length, $D^{\text{init}}$ is the initial depth, and $H$ is the image height [S1].
*   **Color:** SH DC set to pixel color; other coefficients zero [S1].
*   **Opacity:** Depends on view count $N$: $N=2 \rightarrow 0.6$, $N=3 \rightarrow 0.5$, $N=4 \rightarrow 0.35$. Opacity decreases with view count to ensure gradient propagation [S1].

### Background
The standard 3DGS initialization relies on sparse SfM points . Methods using dense point clouds or meshes were developed to leverage existing high-quality geometric representations (e.g., from MPEG or monocular depth estimators) when multi-view data is limited or unavailable [S1]. CoherentGS introduced the isometric scale calculation based on depth to handle sparse novel view synthesis [S1].

### Sources
[S1] CoherentGS: Sparse Novel View Synthesis with Coherent 3D Gaussians, https://doi.org/10.1007/978-3-031-73404-5_2, fetched 2026-10-27


