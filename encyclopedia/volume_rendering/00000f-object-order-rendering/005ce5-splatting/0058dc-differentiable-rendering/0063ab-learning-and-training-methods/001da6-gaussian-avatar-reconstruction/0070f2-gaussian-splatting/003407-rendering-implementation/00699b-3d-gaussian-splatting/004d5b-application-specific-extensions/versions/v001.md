# Application-Specific Extensions

Also called: Specialized 3D Gaussian Splatting Variants

## Overview
Application-Specific Extensions modify the standard 3D Gaussian Splatting (3DGS) framework to handle non-RGB, non-photometric data or to extract explicit geometry from the implicit volumetric representation [1875-volumetric-gaussian-splatting-avis.md], [3301-gaussian-splatting-for-microscopy.md], [5170-ultragauss-rendering.md], [8498-surface-and-mesh-extraction.md]. While standard 3DGS optimizes for novel-view synthesis of RGB scenes using calibrated camera poses and photometric losses, these adaptations replace the projection target with volumetric supervision, voxel-space rasterization, or 3D geometric intersections [1875-volumetric-gaussian-splatting-avis.md], [3301-gaussian-splatting-for-microscopy.md], [5170-ultragauss-rendering.md]. They solve the problem of reconstructing and visualizing high-dimensional or physically distinct data fields where traditional camera-based rendering is inapplicable or insufficient [G].

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

### [Radiative and Tomographic Gaussians](4971-radiative-and-tomographic-gaussians.md)
Adapts 3DGS for X-ray imaging by replacing view-dependent color with view-independent scalar density or radiation intensity. It solves the problem of reconstructing volumetric data from 2D attenuation measurements where radiation depends on integrated density along the ray path [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md). Choose this for CT reconstruction or X-ray tomography.

### [Domain-Specific Adaptations](5621-domain-specific-adaptations.md)
Modifies 3DGS for non-photometric data such as medical volumes, microscopy stacks, and ultrasound. It replaces photometric losses with volumetric supervision (e.g., L1, Dice) computed against target 3D volumes or voxel grids [1875-volumetric-gaussian-splatting-avis.md], [3301-gaussian-splatting-for-microscopy.md]. Choose this for high-dimensional physical fields where camera poses are irrelevant or unavailable.

### [Surface and Mesh Extraction](8498-surface-and-mesh-extraction.md)
Extracts explicit 3D geometry (meshes or segmented primitives) from the implicit Gaussian representation. Methods include density field isosurfacing, distance-based alignment to priors, or 2D-to-3D label propagation [8498-surface-and-mesh-extraction.md]. Choose this when explicit surface topology is required from a trained 3DGS model.

## Background
3D Gaussian Splatting was originally introduced for real-time novel-view synthesis of RGB scenes [1795-3d-gaussian-splatting.md]. The extension to application-specific domains emerged as researchers sought to leverage the differentiable nature and fast rendering of Gaussians for scientific imaging and geometric analysis. Radiative variants were developed to align the splatting model with X-ray physics [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md). Domain-specific adaptations were created to handle medical and microscopy data where volumetric supervision is more appropriate than photometric error [1875-volumetric-gaussian-splatting-avis.md], [3301-gaussian-splatting-for-microscopy.md]. Surface extraction methods address the inherent lack of explicit topology in the Gaussian representation [8498-surface-and-mesh-extraction.md].

## Sources
[S1] Radiative Gaussian Splatting, https://arxiv.org/abs/2312.00000, 2023-12-01
