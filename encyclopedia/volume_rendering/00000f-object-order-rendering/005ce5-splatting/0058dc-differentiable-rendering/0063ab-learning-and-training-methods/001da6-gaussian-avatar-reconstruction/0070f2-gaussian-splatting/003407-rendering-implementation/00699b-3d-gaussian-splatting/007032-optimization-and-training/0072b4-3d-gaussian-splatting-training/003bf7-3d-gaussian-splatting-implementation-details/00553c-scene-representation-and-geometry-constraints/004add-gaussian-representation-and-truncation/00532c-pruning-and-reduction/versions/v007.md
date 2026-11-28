# Pruning and Reduction

Also called: Gaussian pruning, 3DGS pruning, vertex pruning, SplatQ

## Overview
Pruning and reduction methods remove redundant or low-impact 3D Gaussians from a trained or training model to reduce memory footprint and rendering latency while preserving visual fidelity. Unlike sibling methods that focus on geometric simplification of primitives or opacity-independent truncation, this family focuses on the selection and elimination of entire Gaussian entities based on their contribution to the rendered image or their geometric degeneracy. It is distinct from [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md) by actively modifying the set of Gaussians $\{G_i\}$ rather than just their rendering parameters.

## Shared principles
All methods in this family operate on the standard 3D Gaussian Splatting representation, where the scene is defined by a set of $N$ 3D Gaussians $\{G_i\}_{i=1}^N$. Each Gaussian $G_i$ is defined by its mean $\boldsymbol{\mu}_i$, covariance matrix $\boldsymbol{\Sigma}_i$ (derived from rotation $\mathbf{R}_i$ and scale $\mathbf{S}_i$), opacity $\alpha_i$, and view-dependent color $\mathbf{c}_i(\omega)$.

The rendering process evaluates the Gaussian density along a ray $\mathbf{r}(t) = \mathbf{o} + t\omega$ and integrates contributions. The Gaussian density function is:
$$ G(\mathbf{x}) = \exp\left(-\frac{1}{2}(\mathbf{x} - \boldsymbol{\mu})^\top \boldsymbol{\Sigma}^{-1} (\mathbf{x} - \boldsymbol{\mu})\right) $$

The core variation across methods lies in the definition, storage, and optimization of $\mathbf{c}_i(\omega)$ and potentially $\alpha_i$, moving beyond the fixed SH coefficients used in base 3DGS. Common strategies include:
1.  **Post-Processing Transformation:** Applying a learned affine transformation to the rendered image based on 3D position and view embedding [14740-color-and-appearance-modeling.md].
2.  **Factorized Illumination:** Separating scene lighting (via a tensor grid) from material properties (via BRDF features) to model specular effects [14740-color-and-appearance-modeling.md].
3.  **Image-Conditioned Modulation:** Using an MLP and per-image embeddings to modulate Gaussian color and opacity, enabling transient object removal and appearance synthesis [14740-color-and-appearance-modeling.md].
4.  **Neural Field Substitution:** Replacing per-Gaussian SH with a shared grid-based neural field to reduce parameters and improve generalization [14740-color-and-appearance-modeling.md].
5.  **Volumetric Attenuation:** Modifying the alpha-blending equation to incorporate medium properties, such as direct attenuation $B_d$ and backscatter parameters $B_\infty, B_b$, to model light transport in non-vacuum media [15148-physics-and-optical-modeling.md].

## Subtopics

### [Vertex-level Pruning in Mixed-Primitive Gaussian Splatting](13983-vertex-level-pruning-in-mixed-primitive-gaussian-splatting.md)
Vertex-level pruning is a geometric simplification mechanism applied to mixed-primitive splats (Gaussian triangles and lines) during the training of 3D Gaussian Splatting. It removes redundant vertices from primitives that have collapsed or degenerated into lower-dimensional shapes, converting them into simpler primitives (ellipses or lines) to refine the surface representation. Unlike standard Gaussian Splatting, which only performs primitive-level opacity pruning, this method operates on the vertex level within the mixed-primitive framework.

### [SplatQ](14415-splatq.md)
SplatQ is a post-training pruning framework for 3D Gaussian Splatting (3DGS) that addresses the spatial quality imbalance and loss of joint rendering effects inherent in global Top-K selection methods. Unlike sibling methods that rank Gaussians independently, SplatQ partitions primitives into size-limited blocks based on spatial and appearance similarity, allocates retention quotas to balance regional quality, and solves a Quadratic Unconstrained Binary Optimization (QUBO) problem within each block to jointly select Gaussians under a fixed global budget.

### Vertex Pruning
**Definition**
A pruning mechanism designed to support mixed-primitive-based learning, as standard pruning is incompatible with mixed primitives [S1, unofficial].

**Step**
Prune vertices of the mixed primitives to optimize the representation [S1, unofficial].

### Gaussian Reduction with Learnable Mask
**Definition**
A method to reduce the number of Gaussians using a learnable mask during optimization, targeting small-sized kernels for removal [S2].

**Step**
*   Apply binary masks $M$ to the opacities and scale attributes of the kernels at every densification step [S2].
*   Use residual vector quantization with $L$ stages of vector quantization for geometric attributes like scale and rotation [S2].
*   Represent view-dependent colors via a grid-based neural field using hash grids for input positions, features, and view direction, fed into an MLP [S2].

**Formula**
The end-to-end loss is:
$$ \mathcal{L} = \mathcal{L}_{ren} + \lambda_m \mathcal{L}_m + \mathcal{L}_r + \mathcal{L}_s $$
where $\mathcal{L}_{ren}$ is L1+SSIM loss, $\mathcal{L}_m$ is masking loss ($\frac{1}{N} \sum \sigma(m_n)$), and $\mathcal{L}_r, \mathcal{L}_s$ are rotation and scale objective functions [S2].

**Parameter**
*   $\epsilon$: Masking threshold for the sigmoid function in the learnable mask [S2].
*   $\lambda_m$: Weight for the masking loss term, typically $5 \times 10^{-4}$ [S2].

### 3DGS Visibility Pruning
**Definition**
Application of visibility-based pruning to 3D Gaussian Splatting, using Gaussian centroids as reference points for visibility calculation [S2].

**Step**
*   Represent each Gaussian kernel by its centroid position, scaling matrix $S$, and rotation matrix $R$ defining the covariance $\Sigma$ [S2].
*   Use the Gaussian centroids as the 3D points $p_i$ in the visibility mask computation algorithm [S2].

## Background
Pruning in 3D Gaussian Splatting emerged as a solution to the high memory consumption and computational cost associated with the large number of Gaussians required for high-fidelity reconstruction. Early methods relied on simple opacity thresholds, but later approaches introduced geometric constraints (vertex pruning for mixed primitives) and optimization-based selection (SplatQ, learnable masks) to maintain quality at lower densities. Visibility-based pruning leverages the fact that occluded Gaussians contribute little to the final image, allowing for aggressive reduction in complex scenes.

## Sources
[S1] A Mixed-Primitive-based Gaussian Splatting Method for Surface Reconstruction, https://core.ac.uk/works/308836756, fetched 2026-10-30 (unofficial)
[S2] Visibility-Based Geometry Pruning of Neural Plenoptic Scene Representations, https://doi.org/10.1109/tmm.2025.3618548, fetched 2026-10-26
