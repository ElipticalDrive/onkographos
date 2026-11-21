# Domain-Specific Adaptations
## Overview
Domain-Specific Adaptations extend 3D Gaussian Splatting to non-photometric domains by replacing view-dependent RGB color with scalar attributes or specialized geometric intersections [1875-volumetric-gaussian-splatting-avis.md], [3301-gaussian-splatting-for-microscopy.md], [5170-ultragauss-rendering.md]. Unlike standard 3D Gaussian Splatting which optimizes for novel-view synthesis from calibrated RGB images, these methods optimize against 3D volume data, microscopy stacks, or ultrasound probe planes using domain-specific loss functions and rendering primitives [1875-volumetric-gaussian-splatting-avis.md], [3301-gaussian-splatting-for-microscopy.md], [5170-ultragauss-rendering.md].

## Shared principles
All adaptations retain the core 3D Gaussian primitive representation: a set of $M$ anisotropic 3D Gaussian kernels $\mathcal{G} = \{G_i\}_{i=1}^M$ [Application-Specific Extensions](13465-application-specific-extensions.md). Each kernel is defined by:
*   $\mathbf{p}_i \in \mathbb{R}^3$: Position (mean) [Application-Specific Extensions](13465-application-specific-extensions.md).
*   $\mathbf{R}_i \in \mathbb{R}^{3 \times 3}$: Rotation matrix [Application-Specific Extensions](13465-application-specific-extensions.md).
*   $\mathbf{S}_i \in \mathbb{R}^{3 \times 3}$: Scale matrix [Application-Specific Extensions](13465-application-specific-extensions.md).
*   The covariance matrix is $\mathbf{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^\top \mathbf{R}_i^\top$ [Application-Specific Extensions](13465-application-specific-extensions.md).

The rendering pipeline's geometric projection remains consistent for ray-based methods: 3D Gaussians are transformed into ray space, and the projected 2D covariance $\mathbf{\hat{\Sigma}}_i$ is computed using the Jacobian $J$ of the affine approximation of the projection and the rotation part $W$ of the extrinsic matrix:
$$ \mathbf{\hat{\Sigma}}_i = J W \mathbf{\Sigma}_i W^T J^T $$
[Application-Specific Extensions](13465-application-specific-extensions.md)

However, the attribute representation and loss functions diverge significantly from the standard RGB opacity/color model [G]. Standard 3D Gaussian Splatting uses view-dependent color $\mathbf{c}_i(\mathbf{d})$ [G]. In radiative and domain-specific variants, this is replaced by view-independent scalar attributes or feature vectors [1875-volumetric-gaussian-splatting-avis.md], [3301-gaussian-splatting-for-microscopy.md], [5170-ultragauss-rendering.md].

## Subtopics

### [Volumetric Gaussian Splatting (AVIS)](1875-volumetric-gaussian-splatting-avis.md)
Adapts 3D Gaussian Splatting for medical volume data (CT/MR) by optimizing against a target 3D volume and ROI mask using volumetric losses on rasterized voxel grids, rather than photometric losses on camera views [1875-volumetric-gaussian-splatting-avis.md].

### [Gaussian Splatting for Microscopy](3301-gaussian-splatting-for-microscopy.md)
Decomposes $n$-dimensional fluorescence microscopy volumes into Gaussian primitives for efficient storage and web visualization, using blind-spot cross-validation to determine splat count and prevent overfitting to noise [3301-gaussian-splatting-for-microscopy.md].

### [UltraGauss Rendering](5170-ultragauss-rendering.md)
Renders 3D ultrasound volumes by treating the probe plane as a 3D intersection target, using 3D Mahalanobis distance to evaluate opacity and avoiding the occlusion modeling required in visual spectrum rendering [5170-ultragauss-rendering.md].

## Background
Not given in the sources.

## Sources
