# Representation and Kernel Modifications

Also called: Kernel Design, Primitive Variants

## Overview
Representation and Kernel Modifications extend the standard 3D Gaussian Splatting primitive to address specific limitations in geometry, appearance, and surface reconstruction. While the base method uses symmetric, isotropic or anisotropic Gaussians for volume rendering, this family introduces negative Gaussians for high-frequency detail, tangent-plane densification for geometric smoothness, decaying radial basis functions for alternative kernel shapes, and oriented primitives for watertight mesh extraction. These modifications set apart the family by altering the fundamental mathematical definition of the primitive or its spatial distribution strategy, rather than just optimizing the parameters of the standard Gaussian.

## Shared principles
The shared principles of this family are minimal, as the modifications are often mutually exclusive or address distinct problems. However, all methods retain the core differentiable rasterization pipeline of 3D Gaussian Splatting, where Gaussians are projected to 2D, sorted by depth, and composited via alpha blending. The key shared aspect is the use of 3D Gaussians (or Gaussian-like primitives) as the primary scene representation, optimized via gradient descent from multi-view images. The covariance matrix $\Sigma$ is typically parameterized by a rotation matrix $R$ and a scale vector $S$, such that $\Sigma = R S S^T R^T$. The rendering equation remains:
$$ C = \sum_{i \in N} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
where $c_i$ is the color, $\alpha_i$ is the opacity, and $N$ is the set of sorted Gaussians. Modifications occur in the definition of $c_i$, the support of the Gaussian kernel, or the densification rules.

## Subtopics

### [Negative Gaussian Splatting](9152-negative-gaussian-splatting.md)
Negative Gaussian Splatting (NegGS) introduces negative Gaussians to model high-frequency elements and complex nonlinear structures [9152-negative-gaussian-splatting.md]. It differs from standard Gaussian Splatting by allowing two Gaussians with identical parameters and inverse colors to reduce each other, providing greater flexibility in shape representation and color modulation [9152-negative-gaussian-splatting.md]. It is chosen when standard Gaussians fail to capture sharp edges or high-frequency details due to their smooth, positive-only nature. See [Negative Gaussian Splatting](9152-negative-gaussian-splatting.md).

### [GeoGaussian](9187-geogaussian.md)
GeoGaussian improves geometric consistency and surface smoothness by densifying Gaussians on the tangent plane of existing Gaussians and enforcing smoothness of geometric properties between neighboring Gaussians [9187-geogaussian.md]. It distinguishes itself by specifically targeting geometric regularity and surface continuity, unlike Negative Gaussian Splatting which focuses on color modulation. It is chosen when the output geometry is noisy or inconsistent. See [GeoGaussian](9187-geogaussian.md).

### [DARBF-Splatting](9911-darbf-splatting.md)
DARBF-Splatting generalizes reconstruction kernels beyond the exponential family by using Decaying Anisotropic Radial Basis Functions (DARBFs) [9911-darbf-splatting.md]. DARBFs are non-negative functions of the Mahalanobis distance that support anisotropic behavior and differentiability, enabling differentiable rendering while approximating the Gaussian function's closed-form integration advantage. It is chosen when the Gaussian kernel's specific decay properties are insufficient for the scene's geometry. See [DARBF-Splatting](9911-darbf-splatting.md).

### Scaffold-GS
Scaffold-GS initializes a voxel grid and attaches learnable features onto each voxel point, determining all attributes of Gaussians by interpolated features from the voxel grid and lightweight neural networks [S1].

### Gaussian Shell Maps
Gaussian Shell Maps combine CNN-based generators with 3DGS to recreate virtual humans with sophisticated details such as clothing and accessories [S1].

### Gaussian Wrapping
Gaussian Wrapping is a framework that reinterprets 3D Gaussian primitives as oriented probabilistic surface elements representing geometry only within an oriented half-space, allowing derivation of closed-form occupancy, vacancy, and normal fields for watertight mesh extraction [S2].

#### step
- Model surface points with asymmetric primitives where the Gaussian models density decay on the outward-facing side, while the inward-facing side is considered fully occupied [S2].
- Identify geometric gaps where the oriented surface assumption is violated and densify the representation to obtain a watertight shell of oriented primitives [S2].
- Introduce a learnable oriented normal at each Gaussian element and define an adapted attenuation formulation to enable closed-form expressions for normal and occupancy fields [S2].
- Introduce a novel consistency loss and a dedicated densification strategy to enforce Gaussians to wrap the entire surface by closing geometric holes [S2].
- Modify the differentiable rasterizer to output depth as an isosurface of the continuous model [S2].
- Use Primal Adaptive Meshing for Region-of-Interest meshing at arbitrary resolution [S2].

## Background
The standard 3D Gaussian Splatting representation was introduced by Kerbl et al. [2023] as a fast, explicit scene representation. Subsequent work identified limitations in geometric fidelity, surface reconstruction, and high-frequency detail capture. Negative Gaussian Splatting [9152-negative-gaussian-splatting.md] and GeoGaussian [9187-geogaussian.md] address geometric and appearance artifacts. DARBF-Splatting [9911-darbf-splatting.md] explores alternative kernel functions. Gaussian Wrapping [S2] adapts the representation for surface reconstruction, drawing inspiration from the Objects as Volumes framework [39]. Scaffold-GS [S1] and Gaussian Shell Maps [S1] integrate neural networks with Gaussian splatting for structured and human-centric scenes.

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians, https://doi.org/10.1007/978-3-032-37095-2_4, fetched 2026-10-29
