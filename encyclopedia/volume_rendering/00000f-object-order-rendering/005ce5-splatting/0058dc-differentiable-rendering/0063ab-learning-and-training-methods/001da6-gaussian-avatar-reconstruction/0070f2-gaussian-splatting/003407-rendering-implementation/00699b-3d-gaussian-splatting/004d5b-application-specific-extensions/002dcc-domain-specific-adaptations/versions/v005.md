# Domain-Specific Adaptations


## Overview
Domain-Specific Adaptations modify the standard 3D Gaussian Splatting (3DGS) framework to handle non-RGB, non-photometric data such as medical volumes, microscopy stacks, and ultrasound probe intersections [1875-volumetric-gaussian-splatting-avis.md], [3301-gaussian-splatting-for-microscopy.md], [5170-ultragauss-rendering.md]. While standard 3DGS optimizes for novel-view synthesis of RGB scenes using calibrated camera poses and photometric losses, these adaptations replace the projection target with volumetric supervision, voxel-space rasterization, or 3D geometric intersections [1875-volumetric-gaussian-splatting-avis.md], [3301-gaussian-splatting-for-microscopy.md], [5170-ultragauss-rendering.md]. They solve the problem of reconstructing and visualizing high-dimensional or physically distinct data fields where traditional camera-based rendering is inapplicable or insufficient [G].

## Shared principles
The family shares the core primitive: a set of $K$ anisotropic 3D Gaussians parameterized by mean $\mu_k$, covariance $\Sigma_k$ (often via Cholesky factor $L_k$), and a scalar attribute (intensity $a_k$, opacity $\alpha_k$, or color $c_k$) [G].

1.  **Volumetric Supervision:** Instead of minimizing photometric error between rendered pixels and input images, loss is computed between the Gaussian representation and a target 3D volume $V$ [G]. The prediction $\hat{V}(x)$ at a voxel $x$ is typically a sum of Gaussian contributions:
    $$ \hat{V}(x) = \sum_{k=1}^{K} a_k \exp\left(-\frac{1}{2} (x - \mu_k)^T \Sigma_k^{-1} (x - \mu_k)\right) $$
    Loss functions include L1, MSE, Dice, or Tversky, applied to voxel grids or sampled points [1875-volumetric-gaussian-splatting-avis.md], [3301-gaussian-splatting-for-microscopy.md].

2.  **Voxel-Space Rasterization/Sampling:** Rendering involves sampling the Gaussian field at voxel centers or within a voxel grid, often using trilinear interpolation of the volume data to supervise the Gaussians, rather than projecting Gaussians onto a 2D image plane [1875-volumetric-gaussian-splatting-avis.md].

3.  **Initialization from Volume:** Initial Gaussian positions are seeded directly from the input volume data (e.g., gradient-weighted samples, uniform grid, or ROI mask sampling) rather than from Structure-from-Motion (SfM) points [1875-volumetric-gaussian-splatting-avis.md], [3301-gaussian-splatting-for-microscopy.md].

4.  **Normalization:** Input volumes are normalized to $[0, 1]$ to stabilize optimization, often with dataset-specific outlier clipping [3301-gaussian-splatting-for-microscopy.md].

## Subtopics

### [Volumetric Gaussian Splatting (AVIS)](1875-volumetric-gaussian-splatting-avis.md)
A method for medical-style data (CT/MR) that optimizes 3D Gaussians against a target volume inside a Region of Interest (ROI) mask [1875-volumetric-gaussian-splatting-avis.md]. It uses volumetric and mask-aware supervision on rasterized voxel grids. It differs from [Gaussian Splatting for Microscopy](3301-gaussian-splatting-for-microscopy.md) by focusing on general medical volume supervision rather than specific microscopy constraints, and from [UltraGauss Rendering](5170-ultragauss-rendering.md) by its direct voxel-space rasterization supervision [1875-volumetric-gaussian-splatting-avis.md]. Choose this for 3D medical volume reconstruction and visualization.

### [Gaussian Splatting for Microscopy](3301-gaussian-splatting-for-microscopy.md)
Decomposes $n$-dimensional fluorescence microscopy volumes into Gaussian primitives for interactive web visualization [3301-gaussian-splatting-for-microscopy.md]. It initializes splats from voxel data and uses blind-spot cross-validation to determine the optimal splat count, preventing overfitting to noise. It differs from [AVIS](1875-volumetric-gaussian-splatting-avis.md) by its specific handling of microscopy noise and blind-spot validation, and from [UltraGauss](5170-ultragauss-rendering.md) by its focus on $n$-dimensional data storage and visualization rather than probe-plane intersection [3301-gaussian-splatting-for-microscopy.md]. Choose this for large-scale microscopy data compression and viewing.

### [UltraGauss Rendering](5170-ultragauss-rendering.md)
A volume rendering method for 3D ultrasound that treats the ultrasound probe plane as a 3D intersection target [5170-ultragauss-rendering.md]. It evaluates Gaussian opacity by computing the 3D Mahalanobis distance to the probe plane, avoiding occlusion modeling. It differs from [AVIS](1875-volumetric-gaussian-splatting-avis.md) and [Microscopy](3301-gaussian-splatting-for-microscopy.md) by replacing photometric/volumetric projection with volumetric intersection logic specific to ultrasound physics [5170-ultragauss-rendering.md]. Choose this for 3D ultrasound visualization.

## Background
Standard 3D Gaussian Splatting was introduced for novel-view synthesis of RGB scenes [1795-3d-gaussian-splatting.md]. Domain-specific adaptations emerged to extend this efficient rasterization primitive to non-RGB domains [G]. AVIS and Microscopy adaptations address the need for compact, interactive visualization of large 3D/4D scientific datasets (medical CT/MR, fluorescence microscopy) where voxel grids are too large and ray-marching is too slow [1875-volumetric-gaussian-splatting-avis.md], [3301-gaussian-splatting-for-microscopy.md]. UltraGauss addresses the specific physics of ultrasound, where data is acquired as intersecting planes rather than projected views [5170-ultragauss-rendering.md].

## Sources
