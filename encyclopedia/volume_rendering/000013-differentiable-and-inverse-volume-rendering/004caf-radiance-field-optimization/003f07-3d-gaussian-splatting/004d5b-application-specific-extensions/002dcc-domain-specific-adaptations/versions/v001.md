# Domain-Specific Adaptations


## Overview
Domain-Specific Adaptations modify the standard 3D Gaussian Splatting framework to handle non-standard input modalities, such as volumetric medical imaging (CT/MR), fluorescence microscopy, or 3D ultrasound, where traditional multi-view RGB photometric supervision is unavailable or inappropriate [S1, unofficial]. Unlike standard 3DGS which optimizes for novel-view synthesis from calibrated camera poses, these methods optimize Gaussian parameters against target volumes, fixed-pool microscopy constraints, or 3D intersection targets, often replacing rasterization-based rendering with direct voxel-space supervision or specialized distance metrics [S1, unofficial].

## Shared principles
The family shares the core representation of 3D Gaussians (position, covariance, opacity, intensity) but diverges in supervision signals and optimization constraints. Common mechanisms include:
1.  **Volumetric Supervision:** Replacing image-plane photometric loss with voxel-space losses (e.g., MSE, Dice, Tversky) between rasterized Gaussian projections and target volume data [S1, unofficial].
2.  **Normalization:** Input volumes are typically normalized to $[0, 1]$ to stabilize gradient descent, often discarding physical units (e.g., Hounsfield Units) unless explicitly preserved [S1, unofficial].
3.  **Initialization:** Gaussians are initialized from voxel data (e.g., inside a mask or from gradient-weighted edges) rather than from Structure-from-Motion (SfM) points [S1, unofficial].
4.  **Optimization Constraints:** Use of fixed-pool optimizers, blind-spot cross-validation, or specific density control strategies tailored to the data's noise floor or compression requirements [S1, unofficial].

## Subtopics

### [Volumetric Gaussian Splatting (AVIS)](1875-volumetric-gaussian-splatting-avis.md)
Adapts 3DGS for medical volume supervision (CT/MR) using voxel-grid rasterization and mask-aware losses. Chosen when training on aligned 3D volumes with ROI masks rather than multi-view images.

### [Gaussian Splatting for Microscopy](3301-gaussian-splatting-for-microscopy.md)
Decomposes microscopy volumes into oriented Gaussians for compression and streaming, using a fixed-pool optimizer and blind-spot cross-validation. Chosen for scientific data compression and web visualization.

### [UltraGauss Rendering](5170-ultragauss-rendering.md)
Treats the ultrasound probe plane as a 3D intersection target using 3D Mahalanobis distance for opacity, avoiding occlusion modeling. Chosen for 3D ultrasound volume rendering.

## Background
This family emerges from the need to apply the efficiency of 3D Gaussian Splatting to domains where camera calibration and multi-view consistency are either absent or irrelevant, such as medical imaging and microscopy. The adaptations leverage the differentiable nature of Gaussian splatting to optimize directly against volumetric ground truth.

## Sources
[S1] Unofficial implementation notes and documentation for Volumetric Gaussian Splatting (AVIS), Gaussian Splatting for Microscopy, and UltraGauss Rendering.
