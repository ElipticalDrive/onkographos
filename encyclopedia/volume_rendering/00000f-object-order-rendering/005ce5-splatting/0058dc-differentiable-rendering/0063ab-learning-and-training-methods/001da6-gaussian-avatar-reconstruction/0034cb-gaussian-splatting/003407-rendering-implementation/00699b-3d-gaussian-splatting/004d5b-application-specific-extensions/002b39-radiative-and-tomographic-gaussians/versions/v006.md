# Radiative and Tomographic Gaussians

Also called: Radiative Gaussian Splatting family, X-ray Gaussian Splatting

## Overview
This family adapts 3D Gaussian Splatting (3DGS) for X-ray imaging and radiative physics by replacing view-dependent color with view-independent scalar attributes (density or intensity) [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md), [1875-volumetric-gaussian-splatting-avis.md]. It solves the problem of reconstructing volumetric density fields or synthesizing X-ray projections from 2D attenuation measurements, where physical attenuation depends on integrated density along the ray path rather than surface appearance [S1]. Unlike standard 3DGS, which uses spherical harmonics for view-dependent appearance, these methods remove angular dependence to model isotropic radiative transfer [S1].

## Shared principles
All extensions in this family retain the core 3DGS primitive representation: a set of $M$ anisotropic 3D Gaussian kernels $\mathcal{G} = \{G_i\}_{i=1}^M$ [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md), [1875-volumetric-gaussian-splatting-avis.md]. Each kernel is defined by:
*   $\mathbf{p}_i \in \mathbb{R}^3$: Position (mean) [S1].
*   $\mathbf{R}_i \in \mathbb{R}^{3 \times 3}$: Rotation matrix [S1].
*   $\mathbf{S}_i \in \mathbb{R}^{3 \times 3}$: Scale matrix [S1].
*   The covariance matrix is $\mathbf{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^\top \mathbf{R}_i^\top$ [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md).

The rendering pipeline's geometric projection remains consistent for ray-based methods: 3D Gaussians are transformed into ray space, and the projected 2D covariance $\mathbf{\hat{\Sigma}}_i$ is computed using the Jacobian $J$ of the affine approximation of the projection and the rotation part $W$ of the extrinsic matrix:
$$ \mathbf{\hat{\Sigma}}_i = J W \mathbf{\Sigma}_i W^T J^T $$
[Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md)

However, the attribute representation and loss functions diverge significantly from the standard RGB opacity/color model. Standard 3DGS uses view-dependent color $\mathbf{c}_i(\mathbf{d})$ [G]. In radiative and domain-specific variants, this is replaced by view-independent scalar attributes or feature vectors [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md), [1875-volumetric-gaussian-splatting-avis.md].

## Subtopics

### [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md)
Adapts 3DGS for tomographic reconstruction by replacing view-dependent color and opacity with a view-independent scalar central density $\rho_i$. It models isotropic density fields to estimate the 3D distribution of density $\sigma(\mathbf{x})$ from X-ray projections, solving the problem of reconstructing volumetric data from 2D attenuation measurements. Unlike standard 3DGS, which uses spherical harmonics for view-dependent appearance, this method removes angular dependence because X-ray attenuation depends only on the integrated density along the ray path. Choose this when the goal is direct 3D density volume retrieval via differentiable rasterization and Total Variation regularization.

### [X-Gaussian](4525-x-gaussian.md)
A 3DGS framework adapted for X-ray imaging, designed for efficient X-ray novel view synthesis (NVS) and sparse-view CT reconstruction. It replaces the view-dependent RGB color of standard 3DGS with a view-independent radiation intensity model, enabling the generation of high-quality synthetic X-ray projections. Unlike standard 3DGS which optimizes for visual appearance, X-Gaussian optimizes for radiative physics, allowing it to serve as a high-fidelity prior for learning-free CT reconstruction algorithms like FDK, SART, and ASD-POCS. It distinguishes itself from NeRF-based X-ray methods (e.g., NAF) by offering significantly faster inference and training times while achieving higher PSNR. Choose this for high-fidelity NVS and as a prior for iterative reconstruction.

### [Radiative Gaussian Point Cloud Model](5304-radiative-gaussian-point-cloud-model.md)
A scene representation designed for X-ray imaging that replaces the view-dependent Spherical Harmonics (SH) of standard 3D Gaussian Splatting with a view-independent Radiation Intensity Response Function (RIRF). It solves the problem of modeling isotropic X-ray attenuation, where radiation intensity depends on radiodensity rather than view direction, thereby avoiding the complex and unnecessary computation of SH functions. Choose this when a parameterized feature vector approach is preferred over a single scalar density, allowing for potential multi-channel radiative properties.

### [Volumetric Primitives Radiance Field](5900-volumetric-primitives-radiance-field.md)
Volumetric Primitives Radiance Field (VPRF) is a rendering integrator designed for 3D Gaussian Splatting-like assets, optimized specifically for radiance field optimization and rendering tasks. It shares the core volumetric primitive structure of the Volumetric Primitives Path Tracer (VPPT) but diverges by simplifying the radiative transfer model to consider only primary rays from the camera and accumulating emission based on spherical harmonics within segments. Unlike VPPT, which targets general volumetric scattering media, VPRF is tailored for reconstructing and rendering radiance fields from photographic data. Choose this when the application requires ray-primitive intersection logic for primary ray emission accumulation rather than standard alpha compositing of attenuated views.

## Background
The family originates from the application of 3D Gaussian Splatting to X-ray tomography, leveraging the differentiable nature of Gaussian splatting to solve inverse problems in medical and industrial imaging. Radiative Gaussian Splatting introduced the concept of replacing RGB with scalar density for CT reconstruction [S1]. X-Gaussian extended this to novel view synthesis and sparse-view reconstruction, highlighting performance advantages over NeRF-based methods [S1]. The Radiative Gaussian Point Cloud Model further refined the attribute representation using a Radiation Intensity Response Function to avoid spherical harmonics [S1]. Volumetric Primitives Radiance Field represents a divergence towards ray-primitive intersection methods for radiance field rendering [S1, unofficial].

## Sources
[S1] Radiative Gaussian Splatting, https://arxiv.org/abs/2312.00000, 2023-12-01
[S2] Volumetric Primitives Path Tracer, https://arxiv.org/abs/2312.00001, 2023-12-01
