# Gaussian Representation and Truncation

Also called: Gaussian Truncation, Gaussian Densification Control, Gaussian Pruning

## Overview
This family covers methods that modify the spatial extent, density, and survival criteria of 3D Gaussian primitives during the training and optimization of radiance fields. While standard 3D Gaussian Splatting uses a fixed opacity threshold for truncation and a heuristic-based densification strategy, these methods introduce algebraic decay functions, opacity-independent culling, geometry-guided proliferation, and structured pruning to improve geometric fidelity, reduce redundancy, and stabilize training. They differ from sibling articles on rendering pipeline optimization or compression by focusing specifically on the *representation* of the Gaussians themselves—how they are created, scaled, truncated, and removed.

## Shared principles
All methods in this family operate on the standard 3D Gaussian Splatting representation, where each primitive is defined by a mean position $\mu$, a covariance matrix $\Sigma$ (parameterized by scale $s$ and rotation $r$), an opacity $\omega$, and spherical harmonic coefficients for color. The core problem addressed is the mismatch between the Gaussian's geometric support and its visual contribution, or the instability of the densification process.

The shared mechanism involves modifying one or more of the following:
1.  **Truncation/Culling:** Determining when a Gaussian contributes to a pixel or should be discarded. Standard 3DGS uses $\alpha = \omega \cdot \exp(-\frac{1}{2} d_M^2)$ and culls if $\alpha < \tau$. Variants modify the basis function (e.g., inverse multiquadric) or the culling condition (opacity-independent).
2.  **Densification Control:** Deciding when to clone or split Gaussians based on gradients, loss, or geometric properties. Variants introduce scale constraints, atomization, or geometry-guided thresholds.
3.  **Pruning/Reduction:** Removing Gaussians based on opacity, redundancy, or visibility. Variants introduce vertex-level pruning, block-based optimization, or learnable masks.

## Subtopics

### [Inverse Multiquadric Splatting](9910-inverse-multiquadric-splatting.md)
Replaces the Gaussian radial basis function with an inverse multiquadric function to modulate opacity. It provides a smoother, algebraic decay profile compared to the exponential decay of standard Gaussians, reducing hard-edged artifacts. Choose this when visual quality suffers from overly sharp Gaussian contributions.

### [Opacity-Independent Gaussian Truncation](10266-opacity-independent-gaussian-truncation.md)
Modifies the truncation logic to cull Gaussians based solely on their geometric response $R$, decoupling it from the opacity parameter $\omega$. This allows consistent truncation at lower standard deviations (e.g., $1\sigma$) regardless of opacity, addressing the issue where low-opacity Gaussians receive no gradients. Choose this to stabilize gradient flow for transparent surfaces.

### [Atomized Proliferation](12924-atomized-proliferation.md)
A densification strategy that constrains fine-detail Gaussians into isotropic "Atom Gaussians" with fixed, uniform scales, while retaining larger anisotropic Gaussians for backgrounds. It uses a warm-up phase and geometric scale decay to align densification with scene geometry. Choose this to improve mesh fidelity and reduce noisy artifacts in complex details.

### [Vertex-level Pruning in Mixed-Primitive Gaussian Splatting](13983-vertex-level-pruning-in-mixed-primitive-gaussian-splatting.md)
Refines mixed-primitive representations (triangles, lines, ellipses) by removing redundant vertices. It collapses triangles to ellipses if vertices are close, or to lines if collinear. Choose this when using mixed primitives to reduce storage and representation complexity without losing geometric structure.

### [SplatQ](14415-splatq.md)
A post-training pruning framework that partitions Gaussians into blocks and solves a quadratic optimization problem to select a subset under a fixed budget, modeling joint deletion effects. Choose this for efficient model compression while maintaining high rendering quality, avoiding the spatial imbalance of global Top-K methods.

### POG (Potentially Over-large Gaussians)
A variant of densification control that identifies "potentially over-large" Gaussians to prevent them from overgrowing already learned areas. According to [S1], POG improves LPIPS slightly but degrades PSNR significantly (Tanks&Temples: PSNR 26.26 vs HG 26.36) because it makes learned areas susceptible to redundant Gaussians. It is generally inferior to Hard Gaussian Splatting (HG) for PSNR.

### AtomGS
A method that integrates Atomized Proliferation with Geometry-Guided Optimization [S2, unofficial]. It constrains ellipsoid Gaussians into uniformly-sized Atom Gaussians for fine details while retaining larger anisotropic Gaussians for backgrounds. It introduces an Edge-Aware Normal Loss, derived from 2D edge detector weights, to impose stricter constraints on flat surfaces. This provides precise guidance for 3D geometry optimization, balancing visual and geometric accuracy better than standard 3DGS which freely drifts and splits.

### Vertex Pruning
A pruning mechanism designed specifically for mixed-primitive-based learning, as standard opacity-based pruning is incompatible with mixed primitives [S3, unofficial]. It optimizes the representation by pruning vertices of mixed primitives (triangles/lines) to promote surface reconstruction learning.

### Gaussian Reduction with Learnable Mask
A method to reduce Gaussian count using a learnable binary mask $M$ applied to opacities and scales during optimization [S4]. It targets small-sized kernels for removal. The masked attributes are computed as $\hat{s}_n = M_n s_n$ and $\hat{o}_n = M_n o_n$, where $M_n = \text{sg}(\mathbb{1}[\sigma(m_n) > \epsilon] - \sigma(m_n)) + \sigma(m_n)$. The loss function is $\mathcal{L} = \mathcal{L}_{ren} + \lambda_m \mathcal{L}_m + \mathcal{L}_r + \mathcal{L}_s$, with $\lambda_m = 5 \times 10^{-4}$ and masking threshold $\epsilon$. It also uses residual vector quantization for geometry and hash grids for color.

### 3DGS Visibility Pruning
Applies visibility-based pruning to 3DGS by using Gaussian centroids as reference points for visibility calculation [S4]. It computes a visibility mask by projecting centroids to 2D and sorting by depth, removing occluded kernels when their opacity is reset. This reduces the number of kernels and leads to more compact representations.

## Background
3D Gaussian Splatting (3DGS) introduced a novel way to render radiance fields using explicit 3D Gaussians [Kerbl et al., 2023]. However, the original method's fixed truncation threshold and heuristic densification led to issues with geometric accuracy, redundant Gaussians, and training instability. Subsequent research focused on refining the Gaussian representation itself. Hard Gaussian Splatting [S1] addressed hard-to-render areas by growing specific Gaussians. AtomGS [S2, unofficial] and Atomized Proliferation introduced scale constraints for geometric precision. Mixed-primitive approaches [S3, unofficial] and pruning methods like SplatQ [S1] and Visibility Pruning [S4] aimed to reduce redundancy and improve storage efficiency. These methods collectively form the "Gaussian Representation and Truncation" family, focusing on the lifecycle and properties of the Gaussian primitives rather than just the rendering pipeline.

## Sources
[S1] Pushing Rendering Boundaries: Hard Gaussian Splatting, https://arxiv.org/abs/2412.04826v1, fetched 2026-10-26
[S2] AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field, https://core.ac.uk/works/391753628, fetched 2026-10-30 (unofficial)
[S3] A Mixed-Primitive-based Gaussian Splatting Method for Surface Reconstruction, https://core.ac.uk/works/308836756, fetched 2026-10-30 (unofficial)
[S4] Visibility-Based Geometry Pruning of Neural Plenoptic Scene Representations, https://doi.org/10.1109/tmm.2025.3618548, fetched 2026-10-26
