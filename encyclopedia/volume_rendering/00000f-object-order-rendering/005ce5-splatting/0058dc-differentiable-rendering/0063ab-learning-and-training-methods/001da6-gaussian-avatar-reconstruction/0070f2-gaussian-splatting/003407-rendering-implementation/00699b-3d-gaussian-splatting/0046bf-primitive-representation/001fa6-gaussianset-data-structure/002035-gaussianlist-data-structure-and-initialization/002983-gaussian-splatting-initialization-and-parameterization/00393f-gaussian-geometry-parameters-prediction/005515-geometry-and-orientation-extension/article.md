# Geometry and orientation extension


## Overview
This family extends the standard 3D Gaussian primitive to better model explicit geometry, surface orientation, and directional asymmetry, which standard isotropic Gaussians lack [G]. It sets itself apart from standard 3D Gaussian Splatting by introducing learnable normals [9621-oriented-gaussian-splatting.md], skewness [10045-3d-skew-normal-splatting.md], or mesh-constrained parameterizations [15283-barycentric-splat-parameterization.md] to enable surface wrapping, mesh extraction, and view-dependent effects [S1].

## Shared principles
The standard 3D Gaussian Splatting primitive is defined by a center $\mu_i$, a covariance matrix $\Sigma_i$ (parameterized by rotation $R_i$ and scaling $S_i$), opacity $\alpha_i$, and color $c_i$ [Parent]. The covariance is typically $\Sigma_i = R_i S_i S_i^\top R_i^\top$ [G]. The unnormalized probability density function is:
$$ G_i(\mathbf{x}) = \alpha_i \exp\left(-\frac{1}{2}(\mathbf{x} - \mu_i)^\top \Sigma_i^{-1} (\mathbf{x} - \mu_i)\right) $$
Extensions in this family modify the geometric or orientation parameters to address specific limitations:
1. **Orientation:** Adding a normal vector $n_i$ [9621-oriented-gaussian-splatting.md] or skewness parameter $k_i$ [10045-3d-skew-normal-splatting.md] to define directional attenuation or asymmetry.
2. **Constraint:** Constraining centers $\mu_i$ to lie on a mesh surface using barycentric coordinates [15283-barycentric-splat-parameterization.md].
3. **Dimensionality:** Extending the Gaussian to higher dimensions (e.g., 6D for space+direction) to model view-dependent effects [S1].

## Subtopics

### [Oriented Gaussian Splatting](9621-oriented-gaussian-splatting.md)
Augments the standard 3D Gaussian with a learnable normal vector $n_i$ to create a reciprocal attenuation model, enabling explicit geometric field derivation (normal and occupancy) via the Objects as Volumes (OaV) framework. It sets itself apart by restoring reciprocity ($\sigma(x, w) = \sigma(x, -w)$), which is broken by standard isotropic attenuation, thereby allowing for consistent surface wrapping and mesh extraction.

### [3D Skew-Normal Splatting](10045-3d-skew-normal-splatting.md)
Replaces the standard symmetric 3D Gaussian kernel with the Azzalini Skew-Normal distribution to model directional asymmetry, sharp boundaries, and one-sided surfaces. It sets itself apart from [Oriented Gaussian Splatting](9621-oriented-gaussian-splatting.md) by using a continuous skewness parameter rather than a hard half-space truncation, allowing smooth interpolation between symmetric Gaussians and half-ellipsoids while preserving analytical tractability under affine transformations.

### [Barycentric Splat Parameterization](15283-barycentric-splat-parameterization.md)
Constrains 2D Gaussian Splat centers to lie on triangular mesh faces using barycentric coordinates, allowing splats to move freely on the face while preserving per-splat attributes. It differs from sibling methods by deriving Gaussian geometry directly from mesh topology via heuristic midpoint sampling rather than learning orientation vectors or skewness parameters.

### [Anisotropic Covariance](11879-anisotropic-covariance.md)
Allows 3D Gaussian primitives to scale anisotropically along the x, y, and z axes and rotate in 3D space, rather than remaining spherical, to better fit scene geometry. It solves the problem of geometric mismatch where isotropic primitives fail to represent elongated structures, resulting in blurry or rounded artifacts. It sets itself apart from the isotropic baseline by introducing 7 additional optimization variables per primitive (3 scales, 4 quaternion components) to model directional attenuation.

### 6D Gaussian Splatting
An extension of 3DGS that models view-dependent effects by adding directional dimensions, representing scene elements as 6D Gaussians (space+direction) [S1]. It parameterizes the full 6D covariance via Cholesky decomposition $\Sigma = LL^\top$ with $L$ lower-triangular and positive diagonal entries to guarantee positive definiteness [S1].

### In-Plane Rotation for Splats
A parameterization for the orientation of Gaussian Splats on the face plane using an in-plane rotation parameter $R_{2d}$, transformed into world space [S2]. The final rotation matrix is $R = R_s \cdot R_{2d}$, where $R$ is the final rotation matrix used for rasterization, $R_s$ is the rotation matrix of the triangle faces, and $R_{2d}$ is the in-plane rotation parameter [S2].

## Background
Standard 3D Gaussian Splatting [Parent] uses isotropic or anisotropic Gaussians for view synthesis but lacks explicit geometric primitives for surface reconstruction. Recent extensions have addressed limitations: 4D Gaussian Splatting incorporates temporal dynamics, while 6D Gaussian Splatting models view-dependent effects by adding directional dimensions [S1]. Oriented Gaussian Splatting introduces normals for surface wrapping [9621-oriented-gaussian-splatting.md]. 3D Skew-Normal Splatting uses skew-normal distributions for asymmetry [10045-3d-skew-normal-splatting.md]. Barycentric Splat Parameterization binds Gaussians to meshes using barycentric coordinates [15283-barycentric-splat-parameterization.md]. Anisotropic covariance allows 3D Gaussians to scale along x, y, and z axes and rotate in 3D space to better fit scene geometry [11879-anisotropic-covariance.md]. In-plane rotation is used in mesh-constrained frameworks like OMeGa [S2].

## Sources
[S1] 7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting, https://arxiv.org/abs/2503.07946v1, fetched 2026-10-26
[S2] OMeGa: Joint Optimization of Explicit Meshes and Gaussian Splats for Robust Scene-Level Surface Reconstruction, https://doi.org/10.1109/WACV61042.2026.00427, fetched 2026-10-30
