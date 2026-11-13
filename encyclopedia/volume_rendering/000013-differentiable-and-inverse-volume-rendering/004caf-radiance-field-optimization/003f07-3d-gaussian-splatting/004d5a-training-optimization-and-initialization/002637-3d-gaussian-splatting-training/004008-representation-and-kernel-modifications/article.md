# Representation and Kernel Modifications

Also called: Kernel Design, Primitive Shape Modification, Negative Gaussians, Scaffold-GS, GeoGaussian, DARBF, Grid-based Blur

## Overview
This family of methods modifies the fundamental representation or kernel functions of 3D Gaussian Splatting (3DGS) to address limitations in geometric fidelity, memory efficiency, and modeling capacity. While standard 3DGS relies on symmetric, positive-valued Gaussian primitives with closed-form integration shortcuts, these variants introduce negative colors [9152-negative-gaussian-splatting.md], voxel-based latent scaffolds [9186-scaffold-gs.md], tangent-plane densification [9187-geogaussian.md], non-Gaussian radial basis functions [9911-darbf-splatting.md], and spatially-varying blur kernels [14244-grid-based-blur-kernel.md]. They set themselves apart from sibling methods like [Base 3DGS Optimization](7549-base-3dgs-optimization.md) by altering the primitive's mathematical definition or structural initialization rather than just the training schedule or loss functions.

## Shared principles
All methods in this family retain the core differentiable rasterization pipeline of 3DGS, where 3D primitives are projected to 2D ellipses and alpha-blended. However, they diverge in how the primitive's contribution to the color buffer is calculated or how its attributes are parameterized.

1.  **Primitive Parameterization:** Standard 3DGS stores mean $\boldsymbol{\mu}$, covariance $\boldsymbol{\Sigma}$, opacity $\alpha$, and color $\mathbf{c}$ directly. Variants may store these indirectly (e.g., via voxel features [9186-scaffold-gs.md]) or modify the sign of $\mathbf{c}$ [9152-negative-gaussian-splatting.md].
2.  **Kernel Function:** The standard kernel is the Gaussian PDF. Variants replace this with other functions of the Mahalanobis distance $d_M(\mathbf{x}; \boldsymbol{\mu}, \boldsymbol{\Sigma}) = [(\mathbf{x} - \boldsymbol{\mu})^T \boldsymbol{\Sigma}^{-1} (\mathbf{x} - \boldsymbol{\mu})]^{\frac{1}{2}}$, such as DARBFs [9911-darbf-splatting.md] or oriented half-space models [Gaussian Wrapping].
3.  **Integration Shortcut:** Standard 3DGS uses the property that the projection of a 3D Gaussian is a 2D Gaussian, allowing efficient rendering by truncating the covariance matrix. Methods using non-Gaussian kernels (e.g., DARBFs) must approximate this shortcut using correction factors [9911-darbf-splatting.md] or accept higher computational costs.

## Subtopics

### [Negative Gaussian Splatting](9152-negative-gaussian-splatting.md)
See [Negative Gaussian Splatting](9152-negative-gaussian-splatting.md).

### [Scaffold-GS](9186-scaffold-gs.md)
See [Scaffold-GS](9186-scaffold-gs.md).

### [GeoGaussian](9187-geogaussian.md)
See [GeoGaussian](9187-geogaussian.md).

### [DARBF-Splatting](9911-darbf-splatting.md)
See [DARBF-Splatting](9911-darbf-splatting.md).

### [Grid-based Blur Kernel](14244-grid-based-blur-kernel.md)
See [Grid-based Blur Kernel](14244-grid-based-blur-kernel.md).

### Gaussian Wrapping
Gaussian Wrapping reinterprets 3D Gaussian primitives as oriented probabilistic surface elements representing geometry only within an oriented half-space, allowing the derivation of closed-form occupancy, vacancy, and normal fields for watertight mesh extraction [S2]. It models surface points with asymmetric primitives where the Gaussian models density decay on the outward-facing side, while the inward-facing side is considered fully occupied [S2]. The method identifies geometric gaps where the oriented surface assumption is violated and densifies the representation to obtain a watertight shell of oriented primitives [S2]. It introduces a learnable oriented normal at each Gaussian element and defines an adapted attenuation formulation to enable closed-form expressions for normal and occupancy fields [S2]. A novel consistency loss and a dedicated densification strategy enforce Gaussians to wrap the entire surface by closing geometric holes [S2]. The differentiable rasterizer is modified to output depth as an isosurface of the continuous model, and Primal Adaptive Meshing is used for Region-of-Interest meshing at arbitrary resolution [S2].

### Gaussian Shell Maps
Gaussian Shell Maps (GSMs) combine CNN-based generators with 3DGS to recreate virtual humans with sophisticated details such as clothing and accessories [S1].

## Background
The standard 3D Gaussian Splatting method [G] introduced a highly efficient differentiable rasterizer for 3D Gaussians. However, subsequent research identified limitations in handling high-frequency details, complex nonlinear structures (like shadows), sparse initialization, and watertight surface reconstruction. Negative Gaussian Splatting [9152-negative-gaussian-splatting.md] was proposed to handle rapid color transitions and complex shapes via cancellation effects. Scaffold-GS [9186-scaffold-gs.md] addressed memory inefficiency in sparse regions by decoupling geometry from primitives. GeoGaussian [9187-geogaussian.md] improved geometric consistency during densification. DARBF-Splatting [9911-darbf-splatting.md] explored non-exponential kernels for potentially faster convergence. Gaussian Wrapping [S2] and Gaussian Shell Maps [S1] extended the representation for specific applications like surface reconstruction and human avatars.

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians, https://doi.org/10.1007/978-3-032-37095-2_4, fetched 2026-10-29
