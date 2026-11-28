# Domain-Specific Adaptations
## Overview
Domain-Specific Adaptations extend 3D Gaussian Splatting to non-photometric domains by replacing view-dependent RGB color with scalar attributes or specialized geometric intersections . Unlike standard 3D Gaussian Splatting which optimizes for novel-view synthesis from calibrated RGB images, these methods optimize against 3D volume data, microscopy stacks, or ultrasound probe planes using domain-specific loss functions and rendering primitives .

## Shared principles
The primitive representation, covariance parameterization, and geometric projection pipeline are inherited from [Application-Specific Extensions](13465-application-specific-extensions.md). The attribute representation and loss functions diverge significantly from the standard RGB opacity/color model [G]. Standard 3D Gaussian Splatting uses view-dependent color $\mathbf{c}_i(\mathbf{d})$ [G]. In radiative and domain-specific variants, this is replaced by view-independent scalar attributes or feature vectors .

## Subtopics

### [Volumetric Gaussian Splatting (AVIS)](1875-volumetric-gaussian-splatting-avis.md)
Adapts 3D Gaussian Splatting for medical volume data (CT/MR) by optimizing against a target 3D volume and ROI mask using volumetric losses on rasterized voxel grids, rather than photometric losses on camera views.

### [Gaussian Splatting for Microscopy](3301-gaussian-splatting-for-microscopy.md)
Decomposes $n$-dimensional fluorescence microscopy volumes into Gaussian primitives for efficient storage and web visualization, using blind-spot cross-validation to determine splat count and prevent overfitting to noise.

### [UltraGauss Rendering](5170-ultragauss-rendering.md)
Renders 3D ultrasound volumes by treating the probe plane as a 3D intersection target, using 3D Mahalanobis distance to evaluate opacity and avoiding the occlusion modeling required in visual spectrum rendering.

## Background
Not given in the sources.

## Sources
[S1] Volumetric Gaussian Splatting (AVIS)
[S2] Gaussian Splatting for Microscopy
[S3] UltraGauss Rendering
