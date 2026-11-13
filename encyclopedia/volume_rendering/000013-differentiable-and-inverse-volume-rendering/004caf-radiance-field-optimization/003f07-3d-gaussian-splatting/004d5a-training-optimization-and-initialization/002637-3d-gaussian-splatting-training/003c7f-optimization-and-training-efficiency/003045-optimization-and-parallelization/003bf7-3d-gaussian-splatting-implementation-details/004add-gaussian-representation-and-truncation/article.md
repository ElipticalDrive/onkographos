# Gaussian Representation and Truncation

Also called: Gaussian Splatting Optimization, Gaussian Truncation, Gaussian Densification

## Overview
This family of techniques modifies the core Gaussian representation, culling thresholds, and densification strategies of 3D Gaussian Splatting to improve rendering fidelity, geometric accuracy, and training stability. It solves the problems of aggressive opacity falloff, inefficient truncation of low-opacity primitives, and uncontrolled Gaussian proliferation that leads to redundant or poorly shaped primitives. It differs from its siblings by focusing on the mathematical definition of the Gaussian kernel, the conditions under which Gaussians are culled or split, and the structural constraints applied to Gaussian shapes during optimization.

## Shared principles
The methods in this family operate on the standard 3D Gaussian Splatting representation, where each primitive is defined by a mean position $\mu$, a covariance matrix $\Sigma$, an opacity $\alpha$, and color coefficients. The rendering process involves projecting these 3D Gaussians into 2D screen-space ellipses, computing their contribution to each pixel via alpha blending, and backpropagating gradients to update the parameters.

The shared principles include:
1.  **2D Projection:** The 3D covariance $\Sigma$ is projected to 2D using the camera Jacobian $J$, resulting in $\Sigma' = J \Sigma J^T$. The 2D covariance is often represented by its inverse, the conic matrix $C = (\Sigma'_{2 \times 2})^{-1}$, stored as symmetric elements $(a, b, c)$ and opacity $w$.
2.  **Truncation/Culling:** To optimize performance, Gaussians are only processed if they contribute significantly to a pixel. This is typically determined by a threshold on the Gaussian response or the resulting alpha value.
3.  **Densification:** During training, Gaussians are cloned, split, or pruned based on gradient norms, opacity values, and geometric properties to adapt the representation density to scene complexity.

## Subtopics

### [Inverse Multiquadric Splatting](9910-inverse-multiquadric-splatting.md)
Replaces the standard Gaussian radial basis function with an inverse multiquadric function to provide a heavier tail, allowing for smoother blending and better representation of large-scale structures. It differs from siblings by using a specific algebraic decay form rather than exponential decay, requiring a correction factor of $1.38$ for the covariance matrix. Choose this when standard Gaussian falloff is too aggressive for the scene's scale.

### [Opacity-Independent Gaussian Truncation](10266-opacity-independent-gaussian-truncation.md)
Decouples the visibility threshold from the Gaussian's opacity parameter, solving the problem where low-opacity Gaussians are discarded before receiving gradients and high-opacity Gaussians are truncated too late. It sets itself apart by retaining the Gaussian basis functions but altering the early-out condition during rasterization to check the unnormalized response $r$ against a transparency threshold $\tau_\alpha = 1/255$. Choose this to enable consistent truncation at fewer than ~3.03 standard deviations.

### [Atomized Proliferation](12924-atomized-proliferation.md)
Enhances geometric precision by constraining Gaussians representing fine details into "Atom Gaussians" (isotropic spheroids with uniform size) and prioritizing their proliferation to align with scene geometry, followed by pruning to merge large smooth surfaces. Unlike standard 3DGS which struggles to stabilize densification, AtomGS uses a warm-up strategy that aggressively densifies initially, then merges, leading to a declining and stabilizing trend in Gaussian count. Choose this for high-fidelity geometry reconstruction.

### [Vertex-level Pruning in Mixed-Primitive Gaussian Splatting](13983-vertex-level-pruning-in-mixed-primitive-gaussian-splatting.md)
Removes redundant vertices from Gaussian triangles and lines during training, complementing standard primitive-level opacity pruning. It distinguishes itself by operating on the vertex geometry of mixed primitives (triangles, lines, ellipses) rather than just global primitive existence, allowing the representation to dynamically simplify from triangles to lines or ellipses based on local geometric coherence. Choose this when using mixed-primitive representations to reduce redundancy.

### POG (Potentially Over-large Gaussians)
POG is a variant of Gaussian growing that identifies potentially over-large Gaussians to address blurring and needle-like artifacts. In ablation studies on Tanks&Temples, POG yields PSNR 26.26, SSIM 0.879, LPIPS 0.121, whereas Hard Gaussians (HG) yield PSNR 26.36, SSIM 0.880, LPIPS 0.126 [S1]. POG improves LPIPS slightly but degrades PSNR significantly because it overgrows already learned areas, making them susceptible to redundant Gaussians [S1].

### AtomGS
AtomGS introduces Atomized Proliferation and Geometry-Guided Optimization. Atomized Proliferation constrains ellipsoid Gaussians of various sizes into uniformly-sized Atom Gaussians, emphasizing densification in accordance with scene details. Larger anisotropic Gaussians are retained for background or low-feature areas. Geometry-Guided Optimization integrates weights from a 2D edge detector into a curvature map to compute an Edge-Aware Normal Loss, imposing stricter constraints on Gaussian positioning for flat surfaces while allowing flexibility for irregular shapes. During pruning, Gaussians covering large, less detailed surfaces are merged, resulting in a similar or reduced number of Gaussians compared to original 3DGS while retaining competitive quality [S2, unofficial]. Unlike standard 3DGS which freely drifts and splits, AtomGS provides precise guidance on where to focus Gaussians for better 3D geometry optimization [S2, unofficial].

### Vertex Pruning
Vertex pruning is a mechanism designed to support mixed-primitive-based learning, as standard pruning is incompatible with mixed primitives. It prunes vertices of the mixed primitives to optimize the representation [S3, unofficial].

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [S1]. Subsequent works have explored various optimizations to its representation and training. Hard Gaussian Splatting (HGS) introduced the concept of Potentially Over-large Gaussians and Hard Gaussians to handle challenging areas [S1]. AtomGS, proposed by Liu et al., introduced Atomized Proliferation for high-fidelity radiance fields [S2, unofficial]. Mixed-Primitive Gaussian Splatting introduced vertex-level pruning to support different types of primitives [S3, unofficial]. These methods address specific limitations of the original 3DGS implementation regarding geometric accuracy, training stability, and representation efficiency.

## Sources
[S1] Pushing Rendering Boundaries: Hard Gaussian Splatting, https://arxiv.org/abs/2412.04826v1, fetched 2026-10-26
[S2] AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field, https://core.ac.uk/works/391753628, fetched 2026-10-30 (unofficial)
[S3] A Mixed-Primitive-based Gaussian Splatting Method for Surface Reconstruction, https://core.ac.uk/works/308836756, fetched 2026-10-30 (unofficial)
