# Gaussian Representation and Truncation

Also called: Gaussian Splatting Primitives, 3D Gaussian Primitives, Gaussian Splats

## Overview
This family defines the fundamental 3D Gaussian primitive used in differentiable volume rendering and its associated optimization, truncation, and management strategies. It solves the problem of representing complex 3D scenes as a set of anisotropic ellipsoids that can be efficiently rasterized and optimized via gradient descent. It differs from sibling methods by focusing on the *definition* of the primitive (covariance, opacity, position) and the *lifecycle* of these primitives (truncation, pruning, atomization, voxelization) rather than the rasterization kernel itself (e.g., [Inverse Multiquadric Splatting](9910-inverse-multiquadric-splatting.md)) or the rendering pipeline (e.g., [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md)).

## Shared principles
All methods in this family operate on a 3D Gaussian primitive $g$ defined by:
*   **Mean ($\mu$):** The 3D center position.
*   **Covariance ($\Sigma$):** A $3 \times 3$ positive-definite matrix defining the shape, orientation, and scale. Often parameterized as $\Sigma = R S S^T R^T$, where $R$ is a rotation matrix and $S$ is a scaling matrix.
*   **Opacity ($\alpha$ or $\omega$):** A scalar value in $[0, 1]$ representing the maximum density or transparency at the center.
*   **Color ($c$):** Appearance, typically represented by Spherical Harmonics (SH) coefficients.

The rendering contribution of a Gaussian at a pixel is determined by its 2D projected covariance $\Sigma'$ and the Mahalanobis distance $E$ from the pixel center to $\mu'$:
$$ E = (\mathbf{x} - \mu')^T (\Sigma')^{-1} (\mathbf{x} - \mu') $$
The spatial response $R$ is typically $R = \exp(-\frac{1}{2} E)$. The final alpha is $\alpha = \omega \cdot R$.

## Subtopics

### [Inverse Multiquadric Splatting](9910-inverse-multiquadric-splatting.md)
Replaces the Gaussian exponential kernel with an inverse multiquadric function ($1/(E+1)$) to modulate opacity. It solves over-saturation in high-opacity regions. Choose when standard Gaussians cause halo artifacts.

### [Opacity-Independent Gaussian Truncation](10266-opacity-independent-gaussian-truncation.md)
Decouples the spatial truncation threshold from the opacity parameter. It solves the issue where low-opacity Gaussians are culled before contributing gradients. Choose when aggressive culling is needed without losing gradient flow for transparent surfaces.

### [Directed Distance Field (DDF) for Gaussian Splatting](12854-directed-distance-field-ddf-for-gaussian-splatting.md)
Uses a neural implicit field (hash grid + MLP) to provide constant-time ray-surface queries for Gaussians. It solves the lack of efficient secondary ray support (shadows, GI) in explicit splatting. Choose for global illumination or shadow casting.

### [Atomized Proliferation](12924-atomized-proliferation.md)
Constrains fine-detail Gaussians into uniform "Atom Gaussians" and uses a warm-up split schedule. It solves noisy meshes and unstable densification. Choose for high-fidelity geometry reconstruction.

### [Voxel-based Gaussian Splatting](13071-voxel-based-gaussian-splatting.md)
Fixes Gaussian positions to a regular 3D voxel grid and removes SH coefficients. It solves inaccurate volume extraction for CT/X-ray data. Choose for medical imaging or volumetric data.

### [Gaussian Voxel Kernel Functions (GVKF)](13979-gaussian-voxel-kernel-functions-gvkf.md)
Replaces discrete opacity with Gaussian kernel functions decoded from voxel features. It solves memory overhead and enables direct mesh extraction. Choose for memory-efficient surface reconstruction.

### [3D Gaussian Particle Approximation of VDB Datasets](14723-3d-gaussian-particle-approximation-of-vdb-datasets.md)
Converts OpenVDB volumes to non-overlapping Gaussian particles accelerated by OptiX. It solves real-time rendering of large sparse volumetric data. Choose for scientific visualization of VDB data.

### [Pruning and Reduction](14909-pruning-and-reduction.md)
Removes redundant Gaussians based on opacity, geometric degeneracy, or rendering impact. It solves storage and rendering efficiency issues. Choose for compression or deployment.

### POG (Potentially Over-large Gaussians)
POG is a pruning strategy that identifies and removes potentially over-large Gaussians to reduce redundancy. [S1]
- [S1] Tanks&Temples ablation results: POG yields PSNR 26.26, SSIM 0.879, LPIPS 0.121; HG yields PSNR 26.36, SSIM 0.880, LPIPS 0.126.
- [S1] POG (Eq. 8) improves LPIPS slightly but degrades PSNR significantly. POG overgrows already learned areas, making them susceptible to redundant Gaussians, which causes PSNR degradation. HG (Eq. 9) avoids this.

### AtomGS
AtomGS is a training optimization that uses Atomized Proliferation and Geometry-Guided Optimization. [S2, unofficial]
- [S2, unofficial] Atomized Proliferation: Constrain ellipsoid Gaussians of various sizes into uniformly-sized Atom Gaussians. Apply a progressive split schedule to provide better coverage of scene details, emphasizing densification in accordance with scene details.
- [S2, unofficial] Atomized Proliferation: Retain larger anisotropic Gaussians to represent the background or geometric areas with few features, while smaller anisotropic Gaussians are constrained into Atom Gaussians.
- [S2, unofficial] Geometry-Guided Optimization: Integrate weights derived from a 2D edge detector into the curvature map to compute an Edge-Aware Normal Loss, imposing stricter constraints on Gaussian positioning aligned with flat surfaces while allowing flexibility for irregularly shaped areas.
- [S2, unofficial] Pruning: During the subsequent pruning phase, merge Gaussians covering large, less detailed surfaces, resulting in a similar or even reduced number of Gaussians compared to original 3DGS while retaining competitive rendering quality.
- [S2, unofficial] Unlike standard 3DGS which freely drifts and splits, AtomGS provides precise guidance on where to focus Gaussians for better 3D geometry optimization, balancing visual and geometric accuracy by refining underlying geometry alignment.

## Background
The 3D Gaussian Splatting representation was introduced by Kerbl et al. in 2023 [G]. It has since been extended by numerous works to address limitations in geometry, memory, and rendering capabilities, as detailed in the subtopics above.

## Sources
[S1] Pushing Rendering Boundaries: Hard Gaussian Splatting, https://arxiv.org/abs/2412.04826v1, fetched 2026-10-26
[S2] AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field, https://core.ac.uk/works/391753628, fetched 2026-10-30 (unofficial)
