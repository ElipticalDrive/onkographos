# Radiative and Tomographic Gaussians

Also called: Radiative 3DGS, Tomographic 3DGS, X-ray Gaussian Splatting

## Overview
Radiative and Tomographic Gaussians adapt 3D Gaussian Splatting (3DGS) for X-ray imaging by replacing view-dependent color and opacity with view-independent scalar density or radiation intensity [3040-radiative-gaussian-splatting.md]. This modification solves the problem of reconstructing volumetric data from 2D attenuation measurements, where radiation depends on the integrated density along the ray path rather than surface appearance [3040-radiative-gaussian-splatting.md]. Unlike standard 3DGS, which uses spherical harmonics for view-dependent appearance, these methods remove angular dependence to model isotropic density fields [3040-radiative-gaussian-splatting.md].

## Shared principles
The family shares the core 3DGS primitive representation but modifies the radiative transfer model to align with X-ray physics.

**Primitive Representation:**
The scene is represented by a set of $M$ 3D Gaussian kernels $\mathcal{G}_3 = \{G_i\}_{i=1}^M$ [3040-radiative-gaussian-splatting.md]. Each kernel is defined by:
*   $\mathbf{p}_i \in \mathbb{R}^3$: Position.
*   $\mathbf{R}_i \in \mathbb{R}^{3 \times 3}$: Rotation matrix.
*   $\mathbf{S}_i \in \mathbb{R}^{3 \times 3}$: Scale matrix.
*   The covariance matrix is $\mathbf{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^\top \mathbf{R}_i^\top$ [3040-radiative-gaussian-splatting.md].

**Radiative Model:**
Standard 3DGS uses view-dependent color $\mathbf{c}_i(\mathbf{d})$. Radiative variants replace this with a view-independent scalar:
*   **Central Density ($\rho_i$):** Used in [3040-radiative-gaussian-splatting.md] to model isotropic density fields for CT reconstruction [3040-radiative-gaussian-splatting.md].
*   **Radiation Intensity ($I_i$):** Used in [4525-x-gaussian.md] to represent direct radiation intensity [4525-x-gaussian.md].
*   **Feature Vector ($\mathbf{f}_i$):** Used in [5304-radiative-gaussian-point-cloud-model.md], mapped to intensity via a Radiation Intensity Response Function (RIRF) [5304-radiative-gaussian-point-cloud-model.md].

**Rendering Pipeline:**
1.  **Ray Space Transformation:** 3D Gaussians are transformed into ray space to align viewing rays with the coordinate axis [3040-radiative-gaussian-splatting.md].
2.  **Projection:** 3D Gaussians are projected to 2D image plane. The projected 2D covariance $\mathbf{\hat{\Sigma}}_i$ is computed using the Jacobian $J$ of the affine approximation of the projection and the rotation part $W$ of the extrinsic matrix:
    $$ \mathbf{\hat{\Sigma}}_i = J W \mathbf{\Sigma}_i W^T J^T $$
    [3040-radiative-gaussian-splatting.md]
3.  **Alpha Compositing:** The rendered intensity is computed by summing contributions of sorted Gaussians along the ray:
    $$ I_{\text{rendered}} = \sum_{i=1}^M c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    [3040-radiative-gaussian-splatting.md]
    In radiative models, $c_i$ is unity or related to source intensity, and attenuation is modeled via the scalar density/intensity [3040-radiative-gaussian-splatting.md].

## Subtopics

### [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md)
Adapts 3DGS for tomographic reconstruction using central density $\rho_i$. It replaces spherical harmonics with a scalar density to estimate 3D distribution from X-ray projections. Choose this for general CT reconstruction where volumetric density is the primary target.

### [X-Gaussian](4525-x-gaussian.md)
Optimizes a 3D Gaussian point cloud for X-ray novel view synthesis and sparse-view CT. It uses Automatic Camera and Uniform Initialization (ACUI) and Differentiable Radiative Rasterization (DRR). Choose this when accurate scanner parameters are known and fast inference for downstream algorithms (FDK, SART) is required.

### [Radiative Gaussian Point Cloud Model](5304-radiative-gaussian-point-cloud-model.md)
Uses a Radiation Intensity Response Function (RIRF) to map learnable feature vectors to scalar intensity. Choose this when a flexible, learnable mapping from features to radiative properties is needed without explicit spherical harmonics.

### [Volumetric Primitives Radiance Field](5900-volumetric-primitives-radiance-field.md)
A differentiable volume rendering integrator for radiance field optimization. It considers only primary rays and accumulates emission based on spherical harmonics within segments. Choose this for radiance field optimization tasks rather than general volumetric scattering [5900-volumetric-primitives-radiance-field.md].

## Background
The method originates from the adaptation of 3D Gaussian Splatting [1795-3d-gaussian-splatting.md] to X-ray imaging, where the physical model of light interaction (reflection/emission) is replaced by X-ray attenuation (absorption/scattering). Standard 3DGS assumes view-dependent appearance, which is invalid for X-rays where attenuation is path-integrated and isotropic [3040-radiative-gaussian-splatting.md]. This family enables differentiable rendering of X-ray projections, allowing for gradient-based optimization of 3D density fields from 2D projections [3040-radiative-gaussian-splatting.md].

## Sources
[S1] Radiative and Tomographic Gaussians source notes.
