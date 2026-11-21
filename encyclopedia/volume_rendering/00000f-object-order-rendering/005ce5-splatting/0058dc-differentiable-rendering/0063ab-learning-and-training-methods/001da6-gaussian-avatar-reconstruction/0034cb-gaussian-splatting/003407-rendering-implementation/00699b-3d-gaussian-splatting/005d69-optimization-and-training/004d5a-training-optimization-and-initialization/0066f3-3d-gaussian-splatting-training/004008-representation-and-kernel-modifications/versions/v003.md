# Representation and Kernel Modifications


## Overview
This family of methods modifies the fundamental representation of 3D Gaussian Splatting primitives or the kernel functions used during differentiable volume rendering to improve geometric consistency, rendering efficiency, or surface reconstruction capabilities. While standard 3DGS relies on symmetric positive Gaussians and exponential kernels, these variants introduce negative color values, voxel-based scaffolding, tangent-plane constraints, or alternative radial basis functions to address limitations in handling high-frequency details, sparse geometry, and watertight mesh extraction.

## Shared principles
All methods in this family operate within the standard differentiable rasterization pipeline of 3D Gaussian Splatting [Parent](3761-3d-gaussian-splatting.md). They retain the core data structure of Gaussian primitives defined by mean position $\boldsymbol{\mu}$, covariance matrix $\boldsymbol{\Sigma}$ (or rotation/scale decomposition), opacity $\alpha$, and view-dependent color (typically via spherical harmonics). The rendering equation remains the standard alpha compositing formula:
$$ C = \sum_{i \in N} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
where $N$ is the set of sorted primitives. The primary divergence lies in the definition of the opacity contribution $\alpha_i$ (via kernel modification or structural constraints) or the optimization dynamics of the primitive attributes (via regularization or alternative parameterizations).

## Subtopics

### [Negative Gaussian Splatting](9152-negative-gaussian-splatting.md)
[Negative Gaussian Splatting](9152-negative-gaussian-splatting.md) extends the standard representation by allowing Gaussians to have negative color values $c_i$. This creates a "Diff-Gaussian" effect where positive and negative components can cancel each other out, enabling the modeling of high-frequency details and complex nonlinear structures that symmetric positive Gaussians struggle to represent efficiently. It differs from standard 3DGS by introducing a negative mask to track these components and initializing approximately 20% of Gaussians with negative colors.

### [Scaffold-GS](9186-scaffold-gs.md)
[Scaffold-GS](9186-scaffold-gs.md) replaces the direct storage of Gaussian attributes with a structured, voxel-based representation. It initializes a voxel grid and attaches learnable feature vectors to each voxel. Gaussian attributes (color, opacity, covariance) are not stored directly but are predicted by lightweight neural networks (MLPs) from interpolated features of surrounding voxels. This decouples the geometric scaffold from the primitive attributes, improving generalization in sparse regions and reducing memory overhead compared to storing full attributes for every Gaussian.

### [GeoGaussian](9187-geogaussian.md)
[GeoGaussian](9187-geogaussian.md) focuses on geometric consistency by modifying the densification and initialization strategies. It enforces normal consistency between neighboring Gaussians and constrains new Gaussians to lie on the tangent planes of existing parent Gaussians during densification. This prioritizes surface smoothness over raw appearance fidelity, addressing the noisy or discontinuous geometry often produced by standard 3DGS optimization. It introduces a normal consistency regularization term $L_{smooth} = \sum (1 - n_i \cdot n_j)$ and plane constraints to stabilize the geometry.

### [DARBF-Splatting](9911-darbf-splatting.md)
[DARBF-Splatting](9911-darbf-splatting.md) replaces the standard exponential Gaussian kernel with a broader class of non-exponential functions called Decaying Anisotropic Radial Basis Functions (DARBFs), such as half-cosine or inverse multiquadric kernels. These kernels span larger regions with higher opacity for the same variance, reducing the number of primitives required. A key challenge with non-Gaussian kernels is the lack of closed-form 2D projection integration; DARBF-Splatting addresses this by introducing a scalar correction factor $\psi$ to approximate the closure property, allowing the use of the standard 2D covariance projection shortcut while maintaining rendering accuracy.

### Gaussian Shell Maps
Gaussian Shell Maps (GSM) combines CNN-based generators with 3D Gaussian Splatting to reconstruct virtual humans with sophisticated details such as clothing and accessories [S1]. By leveraging 2D CNNs, it captures high-frequency appearance details that are difficult to model with standard spherical harmonics, enabling realistic garment dynamics and texture synthesis for animated humans.

### Gaussian Wrapping
[Gaussian Wrapping](https://doi.org/10.1007/978-3-032-37095-2_4) reinterprets 3D Gaussian primitives as oriented probabilistic surface elements representing geometry only within an oriented half-space. The Gaussian models density decay on the outward-facing side, while the inward-facing side is considered fully occupied. This allows for the derivation of closed-form occupancy, vacancy, and normal fields, enabling watertight mesh extraction. It introduces a learnable oriented normal, a consistency loss, and a dedicated densification strategy to close geometric holes, ensuring a complete shell of oriented primitives. It also modifies the rasterizer to output depth as an isosurface and uses Primal Adaptive Meshing for region-of-interest meshing.

## Background
3D Gaussian Splatting (3DGS) was introduced by Kerbl et al. in 2023 as a real-time explicit scene representation method [Parent](3761-3d-gaussian-splatting.md). Subsequent research identified limitations in geometric smoothness, surface reconstruction, and the expressive power of symmetric positive Gaussians. Negative Gaussian Splatting and DARBF-Splatting address the representational limits of the kernel and color values. Scaffold-GS and GeoGaussian address geometric coherence and memory efficiency. Gaussian Wrapping adapts the primitives for explicit surface reconstruction, drawing inspiration from the "Objects as Volumes" framework [S2]. Gaussian Shell Maps integrates neural rendering techniques for human avatars [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians, https://doi.org/10.1007/978-3-032-37095-2_4, fetched 2026-10-29
