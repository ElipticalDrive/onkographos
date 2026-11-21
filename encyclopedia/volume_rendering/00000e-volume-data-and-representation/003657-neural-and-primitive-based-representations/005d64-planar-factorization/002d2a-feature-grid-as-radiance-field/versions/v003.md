# Feature Grid as Radiance Field

Also called: EgoNeRF representation, Tensor Factorization

## Overview
Feature Grid as Radiance Field represents a 3D scene using explicit, low-rank tensor-decomposed feature grids for density and appearance, replacing the implicit coordinate-to-value mapping of standard MLPs. It sets itself apart from sibling methods like [Hash Grid Neural Fields](3855-hash-grid-neural-fields.md) and [MLP Neural Fields](3854-mlp-neural-fields.md) by storing features in a structured, factorized grid format that reduces space complexity from cubic to quadratic while maintaining regular angular components via spherical parameterization [S1].

## How it works
**Inputs:** A set of egocentric images with corresponding camera parameters.
**Outputs:** A scene representation defined by density $\sigma(\mathbf{x})$ and color $c(\mathbf{x}, \mathbf{d})$ for any 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$.

**Data Structures:**
The representation consists of two primary grids, $\mathcal{G}_\sigma$ and $\mathcal{G}_a$, defined on a "balanced spherical grid" with resolution $2N_y^r \times N_y^\theta \times N_y^\phi$.
1.  **Density Grid ($\mathcal{G}_\sigma$):** A 3D tensor of size $2N_y^r \times N_y^\theta \times N_y^\phi$ with a single channel storing volume density values.
2.  **Appearance Grid ($\mathcal{G}_a$):** A 3D tensor of size $2N_y^r \times N_y^\theta \times N_y^\phi \times C$ storing $C$-dimensional neural appearance features.

Both grids are decomposed into low-rank tensor factors (vectors and matrices) to reduce memory usage. The decomposition is performed per "Yin" and "Yang" hemisphere ($y \in Y$, where $Y=\{\text{Yin, Yang}\}$).

**Decomposition Formulas:**
The density grid component for a hemisphere $y$ is decomposed as [S1]:
$$ \mathcal{G}_\sigma^y = \sum_{n=1}^{N_\sigma} \mathbf{v}_{\sigma,n}^{y,R} \otimes \mathbf{M}_{\sigma,n}^{y,\Theta\Phi} + \mathbf{v}_{\sigma,n}^{y,\Theta} \times \mathbf{M}_{\sigma,n}^{y,\Phi R} + \mathbf{v}_{\sigma,n}^{y,\Phi} \otimes \mathbf{M}_{\sigma,n}^{y,R\Theta} $$
where $\otimes$ is the outer product, $\mathbf{v}$ are vectors, and $\mathbf{M}$ are matrices. The indices $R, \Theta, \Phi$ correspond to radial, polar, and azimuthal dimensions.

The appearance grid component for a hemisphere $y$ is decomposed as [S1]:
$$ \mathcal{G}_a^y = \sum_{n=1}^{N_a} \mathcal{A}_{a,n}^{y,R} \otimes \mathbf{b}_{3n-2}^y + \mathcal{A}_{a,n}^{y,\Theta} \otimes \mathbf{b}_{3n-1}^y + \mathcal{A}_{a,n}^{y,\Phi} \otimes \mathbf{b}_{3n}^y $$
where $\mathcal{A}$ are matrices and $\mathbf{b}$ are vectors.

The final grids are the union of Yin and Yang components:
$$ \mathcal{G}_\sigma = \bigcup_{y \in Y} \mathcal{G}_\sigma^y, \quad \mathcal{G}_a = \bigcup_{y \in Y} \mathcal{G}_a^y $$

**Query Process:**
1.  **Trilinear Interpolation:** For a query position $\mathbf{x}$, retrieve features from the decomposed grids using trilinear interpolation $\mathcal{T}$. Note that $\mathcal{T}$ operates on the reconstructed full tensor or is implemented via efficient lookup/sum of interpolated factors [S1].
2.  **Density Retrieval:** $\sigma(\mathbf{x}) = \mathcal{T}(\mathcal{G}_\sigma, \mathbf{x})$.
3.  **Feature Retrieval:** $\mathbf{f} = \mathcal{T}(\mathcal{G}_a, \mathbf{x})$.
4.  **Color Decoding:** Pass the retrieved feature $\mathbf{f}$ and viewing direction $\mathbf{d}$ through a tiny Multi-Layer Perceptron (MLP) $f_{\text{MLP}}$ to obtain color: $c(\mathbf{x}, \mathbf{d}) = f_{\text{MLP}}(\mathbf{f}, \mathbf{d})$ [S1].

**Parameters:**
*   $N_y^r, N_y^\theta, N_y^\phi$: Grid resolutions for radius, theta, and phi.
*   $N_\sigma, N_a$: Number of terms in the tensor decomposition sum.
*   $C$: Dimension of appearance features.
*   Typical values are not explicitly given in the sources for all parameters, but the method emphasizes reducing space complexity from $O(n^3)$ to $O(n^2)$ [S1].

**Edge Cases and Pitfalls:**
*   **Spherical Parameterization:** The use of balanced spherical grids requires careful handling of coordinate mapping from Cartesian $\mathbf{x}$ to spherical $(r, \theta, \phi)$ to ensure correct interpolation, especially near the origin or poles [S1].
*   **Yin/Yang Union:** The scene is split into Yin and Yang hemispheres; queries must correctly identify which hemisphere $\mathbf{x}$ falls into to select the appropriate grid components [S1].

## Subtopics

### [Masked Wavelet Representation for Compact Neural Radiance Fields](9619-masked-wavelet-representation-for-compact-neural-radiance-fi.md)
Reduces memory footprint by transforming grid coefficients into the wavelet domain and applying a trainable binary mask. Sets itself apart by leveraging wavelet compactness for smooth signals and per-scene optimized masking, achieving state-of-the-art performance within a 2 MB memory budget.

## Operating characteristics
*   **Time Complexity:** Query time depends on the cost of trilinear interpolation and the tiny MLP decoding. The tensor decomposition allows for efficient storage but requires reconstruction or specialized interpolation logic.
*   **Memory Complexity:** The low-rank tensor factorization reduces space complexity from $O(n^3)$ to $O(n^2)$, where $n$ is the grid resolution [S1].
*   **Strengths:** Reduces memory footprint significantly compared to dense grids or large MLPs; maintains regular angular components; suitable for egocentric view synthesis [S1].
*   **Limitations:** Requires spherical parameterization; decomposition adds complexity to the storage and retrieval structure compared to dense grids.
*   **When to use:** When memory is constrained but high-quality novel view synthesis is required, particularly for egocentric scenes [S1].

## Background
The method was introduced in the paper "Balanced Spherical Grid for Egocentric View Synthesis" (EgoNeRF) [S1]. It aims to reconstruct 3D scene representations and synthesize novel view images from egocentric images. The approach builds on the idea of explicit feature grids but introduces balanced spherical grids and low-rank tensor factorization to address the high memory cost of standard 3D grids [S1].

## Sources
[S1] Balanced Spherical Grid for Egocentric View Synthesis, https://doi.org/10.1109/cvpr52729.2023.01592, fetched 2026-10-29
