# Pruning and Reduction

Also called: Gaussian Splatting Pruning, 3DGS Reduction, Vertex Pruning, SplatQ

## Overview
Pruning and reduction methods remove redundant or low-impact 3D Gaussians from a trained model to reduce memory footprint and rendering latency while preserving visual fidelity [S1, unofficial][S2]. Unlike standard opacity-based removal, this family includes vertex-level geometric simplification for mixed primitives [S1, unofficial], joint optimization of retention via Quadratic Unconstrained Binary Optimization (QUBO) to handle spatial quality imbalance [S14415], and learnable masking or visibility-based filtering during optimization [S2].

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
2.  **Geometric Redundancy:** Removing vertices or primitives that are spatially coincident or collinear [S1, unofficial].
3.  **Visibility:** Removing Gaussians occluded by others from all training viewpoints [S2].
4.  **Optimization Impact:** Removing Gaussians that contribute minimally to the loss function or rendering error [S2].

## Subtopics

### [Vertex-level Pruning in Mixed-Primitive Gaussian Splatting](13983-vertex-level-pruning-in-mixed-primitive-gaussian-splatting.md)
Applies vertex-level pruning to mixed-primitive Gaussians (points, lines, triangles) to remove redundant vertices, converting triangles to lines or ellipses, and lines to ellipses based on proximity and collinearity thresholds.

### [SplatQ](14415-splatq.md)
A post-training pruning framework that partitions Gaussians into blocks based on spatial/appearance similarity, allocates retention quotas, and solves a QUBO problem within each block to jointly select Gaussians that minimize reconstruction error under a fixed global count, addressing spatial quality imbalance.

### Vertex Pruning
#### definition
- [S1, unofficial] A pruning mechanism designed to support mixed-primitive-based learning, as standard pruning is incompatible with mixed primitives.
#### step
- [S1, unofficial] Prune vertices of the mixed primitives to optimize the representation.

### Gaussian Reduction with Learnable Mask
#### definition
- [S2] A method to reduce the number of Gaussians using a learnable mask during optimization, targeting small-sized kernels for removal.
#### step
- [S2] Apply binary masks M to the opacities and scale attributes of the kernels at every densification step.
- [S2] Use residual vector quantization with L stages of vector quantization for geometric attributes like scale and rotation.
- [S2] Represent view-dependent colors via a grid-based neural field using hash grids for input positions, features, and view direction, fed into an MLP.
#### formula
- [S2] The end-to-end loss is: L = L_ren + λ_m L_m + L_r + L_s, where L_ren is L1+SSIM loss, L_m is masking loss (1/N Σ σ(m_n)), and L_r, L_s are rotation and scale objective functions.
#### parameter
- [S2] ε: Masking threshold for the sigmoid function in the learnable mask.
- [S2] λ_m: Weight for the masking loss term, typically 5 × 10^-4.

### 3DGS Visibility Pruning
#### definition
- [S2] Application of visibility-based pruning to 3D Gaussian Splatting, using Gaussian centroids as reference points for visibility calculation.
#### step
- [S2] Represent each Gaussian kernel by its centroid position, scaling matrix $S$, and rotation matrix $R$ defining the covariance $\Sigma$.
- [S2] Use the Gaussian centroids as the 3D points $p_i$ in the visibility mask computation algorithm.

## Background
Pruning in 3D Gaussian Splatting emerged as the method's high memory consumption became a bottleneck for large-scale scenes. Early implementations relied on simple opacity thresholds [G]. Subsequent work introduced geometric redundancy checks for mixed primitives [S1, unofficial] and visibility-based culling using centroid projections [S2]. More recent approaches like SplatQ address the limitations of global Top-K selection by using block-wise QUBO optimization to preserve spatial quality balance [S14415].

## Sources
[S1] A Mixed-Primitive-based Gaussian Splatting Method for Surface Reconstruction, https://core.ac.uk/works/308836756, fetched 2026-10-30 (unofficial)
[S2] Visibility-Based Geometry Pruning of Neural Plenoptic Scene Representations, https://doi.org/10.1109/tmm.2025.3618548, fetched 2026-10-26
