# Gaussian Representation and Truncation

Also called: Gaussian Splatting Primitive Management, Gaussian Optimization Strategies

## Overview
This family encompasses methods that modify the definition, spatial extent, or lifecycle of 3D Gaussian primitives to improve rendering fidelity, geometric accuracy, or computational efficiency. While standard 3D Gaussian Splatting (3DGS) uses fixed isotropic/anisotropic Gaussians with opacity-dependent truncation, this family introduces alternative kernel shapes, opacity-independent culling, voxel-based constraints, and aggressive pruning or atomization strategies. It differs from [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md) by focusing on the *properties* of the primitives themselves rather than the *order* of their processing, and from [Data Compression and Storage Optimization](12943-data-compression-and-storage-optimization.md) by altering the training dynamics and representation density rather than just the file format.

## Shared principles
All methods in this family operate on the standard 3D Gaussian primitive $g$ defined by:
*   **Mean ($\mu$):** The 3D center position.
*   **Covariance ($\Sigma$):** A $3 \times 3$ positive-definite matrix defining the shape, orientation, and scale. Often parameterized as $\Sigma = R S S^T R^T$, where $R$ is a rotation matrix and $S$ is a scaling matrix.
*   **Opacity ($\alpha$ or $\omega$):** A scalar value in $[0, 1]$ representing the maximum density or transparency at the center.
*   **Color ($c$):** Appearance, typically represented by Spherical Harmonics (SH) coefficients.

The rendering contribution of a Gaussian at a pixel is determined by its 2D projected covariance $\Sigma'$ and the Mahalanobis distance $E$ from the pixel center to $\mu'$:
$$ E = (\mathbf{x} - \mu')^T (\Sigma')^{-1} (\mathbf{x} - \mu') $$
The spatial response $R$ is typically $R = \exp(-\frac{1}{2} E)$. The final alpha is $\alpha = \omega \cdot R$.

Pruning criteria generally rely on one or more of the following metrics:
1.  **Opacity Threshold:** Removing Gaussians where $\alpha < \tau_{\alpha}$ [G].
2.  **Geometric Redundancy:** Removing vertices or primitives that are spatially coincident or collinear [S1].
3.  **Visibility:** Removing Gaussians occluded by others from all training viewpoints [S2, unofficial].
4.  **Optimization Impact:** Removing Gaussians that contribute minimally to the loss function or rendering error [S2, unofficial].

## Subtopics

### [Inverse Multiquadric Splatting](9910-inverse-multiquadric-splatting.md)
Replaces the Gaussian radial basis function with an inverse multiquadric function to define spatial response. It solves artifact generation in high-curvature regions by providing a different decay profile. Choose this when standard Gaussians produce artifacts on sharp edges.

### [Opacity-Independent Gaussian Truncation](10266-opacity-independent-gaussian-truncation.md)
Decouples spatial truncation from opacity, checking the spatial response $R$ against a threshold $\tau_\alpha$ before multiplying by opacity. This allows consistent truncation at fewer standard deviations (e.g., $1\sigma$) regardless of opacity, enabling aggressive optimization. Choose this to reduce Gaussian count and improve training stability.

### [Directed Distance Field (DDF) for Gaussian Splatting](12854-directed-distance-field-ddf-for-gaussian-splatting.md)
A neural implicit representation mapping ray origin/direction to distance and visibility. It serves as a constant-time ray oracle for secondary rays (shadows, AO), eliminating the need for mesh extraction or BVHs. Choose this for global illumination or shadowing in Gaussian scenes.

### [Atomized Proliferation](12924-atomized-proliferation.md)
Constrains fine-detail Gaussians into uniform "Atom Gaussians" and prioritizes their proliferation. It uses a warm-up phase for aggressive densification followed by merging of large surfaces. Choose this for superior mesh quality and geometric precision.

### [Voxel-based Gaussian Splatting](13071-voxel-based-gaussian-splatting.md)
Fixes Gaussian positions to a regular voxel grid and removes view-dependent color. It solves instability in CT data by regressing opacity/scale/rotation from transformer tokens. Choose this for differentiable X-ray rendering and CT extraction.

### [Gaussian Voxel Kernel Functions (GVKF)](13979-gaussian-voxel-kernel-functions-gvkf.md)
Uses a dynamic sparse voxel grid where Gaussian attributes are decoded from feature vectors via MLPs. It manages Gaussian growth through gradient-based voxel subdivision. Choose this for high-fidelity surface reconstruction in open scenes with reduced memory usage.

### [3D Gaussian Particle Approximation of VDB Datasets](14723-3d-gaussian-particle-approximation-of-vdb-datasets.md)
Converts OpenVDB volumetric datasets into non-overlapping 3D Gaussian particles. It enables memory-efficient, lossy representation with LOD support via hardware ray tracing. Choose this for rendering large sparse volumetric data compatible with traditional transfer functions.

### [Pruning and Reduction](14909-pruning-and-reduction.md)
Removes redundant or low-impact primitives based on geometric redundancy, visibility, or optimization scores. Unlike sibling methods that modify truncation or kernel shape, this focuses on *selection* and *removal*. Choose this to reduce memory footprint and rendering latency.

#### POG (Potentially Over-large Gaussians)
POG identifies potentially over-large Gaussians to mitigate redundant growth in already learned areas. In ablation studies on Tanks&Temples, POG yields PSNR 26.26, SSIM 0.879, LPIPS 0.121, whereas Hard Gaussians (HG) yield PSNR 26.36, SSIM 0.880, LPIPS 0.126 [S1]. POG improves LPIPS slightly but degrades PSNR significantly because overgrown areas become susceptible to redundant Gaussians [S1].

#### AtomGS
AtomGS employs Atomized Proliferation to constrain ellipsoid Gaussians into uniformly-sized Atom Gaussians, emphasizing densification for fine details [S2, unofficial]. It retains larger anisotropic Gaussians for background/low-feature areas and uses an Edge-Aware Normal Loss (integrating 2D edge detector weights into curvature maps) to constrain positioning on flat surfaces [S2, unofficial]. During pruning, it merges Gaussians covering large, less detailed surfaces, resulting in a similar or reduced Gaussian count compared to original 3DGS while retaining competitive quality [S2, unofficial]. Unlike standard 3DGS which freely drifts and splits, AtomGS provides precise guidance for 3D geometry optimization [S2, unofficial].

## Background
The standard 3D Gaussian Splatting method [G] introduced explicit 3D Gaussians as a primitive for real-time radiance field rendering. Subsequent research identified limitations in geometric accuracy, memory usage, and handling of specific data types (e.g., CT volumes, VDB datasets). This led to the development of alternative kernel functions (Inverse Multiquadric), modified truncation strategies (Opacity-Independent), and structural constraints (Voxel-based, Atomized) to address these issues. The family also includes methods for secondary ray querying (DDF) and post-training optimization (Pruning).

## Sources
[S1] Pushing Rendering Boundaries: Hard Gaussian Splatting, https://arxiv.org/abs/2412.04826v1, fetched 2026-10-26
[S2] AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field, https://core.ac.uk/works/391753628, fetched 2026-10-30 (unofficial)
