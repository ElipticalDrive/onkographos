# Gaussian Representation and Truncation

Also called: Gaussian Splatting Primitive Management, 3DGS Densification and Pruning, Gaussian Optimization Strategies

## Overview
Gaussian Representation and Truncation encompasses the mechanisms for managing the lifecycle of 3D Gaussian primitives during the training of radiance field models, specifically addressing how Gaussians are initialized, densified (split/cloned), pruned, and truncated during rendering. It solves the problem of balancing scene representation fidelity with computational efficiency by dynamically adjusting the number and size of Gaussians to match scene complexity. It differs from its siblings by focusing on the *optimization dynamics* and *structural constraints* of the Gaussian primitives themselves, rather than the rendering kernel function (e.g., Inverse Multiquadric Splatting), the truncation threshold logic (e.g., Opacity-Independent Truncation), or the acceleration structures (e.g., DDF).

## Shared principles
The family shares the core data structure of a 3D Gaussian defined by mean position $\mu$, covariance matrix $\Sigma$ (parameterized by scale $s$ and rotation $q$), opacity $\alpha$, and color $c$ (often represented by spherical harmonics coefficients). The optimization process typically involves an iterative loop where Gaussians are:
1.  **Projected:** $\Sigma$ is projected to 2D using the camera Jacobian to form $\Sigma'$.
2.  **Rasterized:** Contributions are accumulated along rays or via screen-space splatting.
3.  **Evaluated:** Loss is computed against ground truth images.
4.  **Updated:** Gradients update $\mu, \Sigma, \alpha, c$.
5.  **Managed:** Densification (splitting/cloning) and pruning occur based on gradients, opacity, and geometric constraints.

Key shared mathematical concepts include the Mahalanobis distance for spatial decay, the exponential opacity model $\alpha = \omega \exp(-\frac{1}{2} d^T C d)$, and the use of opacity thresholds for culling.

## Subtopics

### [Inverse Multiquadric Splatting](9910-inverse-multiquadric-splatting.md)
Replaces the standard Gaussian kernel with an inverse multiquadric radial basis function to provide slower, algebraic decay, solving tailing artifacts. It requires specific corrections to the covariance matrix scaling (factor 1.38) and gradient computations.

### [Opacity-Independent Gaussian Truncation](10266-opacity-independent-gaussian-truncation.md)
Decouples the truncation threshold from the Gaussian's opacity scalar, checking the unnormalized Gaussian response against the threshold before applying opacity. This allows for consistent truncation at fewer standard deviations (e.g., 1σ or 2σ) and enables gradient flow for near-zero opacity Gaussians.

### [Directed Distance Field (DDF) for Gaussian Splatting](12854-directed-distance-field-ddf-for-gaussian-splatting.md)
Uses a neural implicit representation (Multiresolution Hash Grid + MLP) as a constant-time ray oracle for secondary rays (shadows, AO), enabling global illumination without BVH or mesh extraction.

### [Atomized Proliferation](12924-atomized-proliferation.md)
Constrains fine-detail Gaussians into "Atom Gaussians" (isotropic spheroids) and prioritizes their proliferation, while merging large smooth surfaces. It uses a warm-up strategy for aggressive initial densification followed by merging, stabilizing the Gaussian count.

### [Voxel-based Gaussian Splatting](13071-voxel-based-gaussian-splatting.md)
Fixes Gaussian positions to a regular voxel grid and removes view-dependent color, solving instability in CT extraction. It regresses attributes from transformer tokens, enabling direct volume extraction via indexing.

### [Gaussian Voxel Kernel Functions (GVKF)](13979-gaussian-voxel-kernel-functions-gvkf.md)
Replaces discrete opacity values with Gaussian kernel functions decoded from feature vectors via MLPs, enabling continuous scene representation and direct mesh extraction with reduced memory usage.

### [Vertex-level Pruning in Mixed-Primitive Gaussian Splatting](13983-vertex-level-pruning-in-mixed-primitive-gaussian-splatting.md)
Extends truncation logic to the vertex level within mixed-primitive frameworks (triangles, lines, ellipses), converting primitives to simpler forms based on vertex proximity and collinearity.

### [3D Gaussian Particle Approximation of VDB Datasets](14723-3d-gaussian-particle-approximation-of-vdb-datasets.md)
Converts OpenVDB volumetric datasets into non-overlapping 3D Gaussian particles for memory-efficient, lossy representation with LOD support, rendered via forward ray marching using NVIDIA OptiX.

### [Pruning and Reduction](14909-pruning-and-reduction.md)
General strategies for reducing Gaussian count, including learnable masks and residual vector quantization for geometric attributes.

### POG (Potentially Over-large Gaussians)
POG is a variant of Gaussian growth that identifies potentially over-large Gaussians to address blurring and needle-like artifacts. It improves LPIPS slightly but degrades PSNR significantly compared to Hard Gaussians (HG) because it overgrows already learned areas, making them susceptible to redundant Gaussians [S1]. On Tanks&Temples, POG yields PSNR 26.26, SSIM 0.879, LPIPS 0.121, whereas HG yields PSNR 26.36, SSIM 0.880, LPIPS 0.126 [S1].

### AtomGS
AtomGS introduces Atomized Proliferation, constraining ellipsoid Gaussians of various sizes into uniformly-sized Atom Gaussians to enhance fine feature representation [S2, unofficial]. It retains larger anisotropic Gaussians for backgrounds and uses a progressive split schedule [S2, unofficial]. It also employs Geometry-Guided Optimization with an Edge-Aware Normal Loss, integrating weights from a 2D edge detector into the curvature map to constrain Gaussian positioning on flat surfaces while allowing flexibility for irregular areas [S2, unofficial]. During pruning, it merges Gaussians covering large, less detailed surfaces, resulting in a similar or reduced Gaussian count compared to original 3DGS while retaining competitive quality [S2, unofficial]. Unlike standard 3DGS which freely drifts and splits, AtomGS provides precise guidance for 3D geometry optimization [S2, unofficial].

### Gaussian Reduction with Learnable Mask
This method reduces the number of Gaussians using a learnable mask during optimization, targeting small-sized kernels for removal [S3]. Binary masks $M$ are applied to opacities and scale attributes at every densification step:
$$ M_n = \text{sg}(\mathbb{1}[\sigma(m_n) > \epsilon] - \sigma(m_n)) + \sigma(m_n) $$
$$ \hat{s}_n = M_n s_n, \quad \hat{o}_n = M_n o_n $$
where $\epsilon$ is the masking threshold, $\text{sg}(\cdot)$ is the stop-gradient operator, $\mathbb{1}$ is the indicator function, and $\sigma(\cdot)$ is the sigmoid function [S3]. Residual vector quantization with $L$ stages is used for geometric attributes (scale, rotation), and view-dependent colors are represented via a grid-based neural field using hash grids [S3]. The end-to-end loss is:
$$ L = L_{ren} + \lambda_m L_m + L_r + L_s $$
where $L_{ren}$ is L1+SSIM loss, $L_m = \frac{1}{N} \sum_{n=1}^N \sigma(m_n)$ is the masking loss, and $L_r, L_s$ are rotation and scale objective functions [S3]. The weight $\lambda_m$ is typically $5 \times 10^{-4}$ [S3].

## Background
3D Gaussian Splatting (3DGS) was introduced by Kerbl et al. in 2023 [G]. The original method used a fixed set of Gaussians initialized from Structure-from-Motion (SfM) points, with densification based on position gradients and opacity thresholds. Subsequent research has explored various ways to optimize this representation, including modifying the kernel function (Inverse Multiquadric), changing truncation logic (Opacity-Independent), using neural fields for attributes (GVKF, DDF), and constraining geometry (AtomGS, Voxel-based). The field continues to evolve with methods for compression, mixed primitives, and volumetric data integration.

## Sources
[S1] Pushing Rendering Boundaries: Hard Gaussian Splatting, https://arxiv.org/abs/2412.04826v1, fetched 2026-10-26
[S2] AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field, https://core.ac.uk/works/391753628, fetched 2026-10-30 (unofficial)
[S3] Visibility-Based Geometry Pruning of Neural Plenoptic Scene Representations, https://doi.org/10.1109/tmm.2025.3618548, fetched 2026-10-26
