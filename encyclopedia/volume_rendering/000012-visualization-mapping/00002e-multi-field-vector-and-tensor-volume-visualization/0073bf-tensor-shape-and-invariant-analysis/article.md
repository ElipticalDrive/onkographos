# Tensor Shape and Invariant Analysis

Also called: Tensor Invariant Analysis, Shape Space Visualization

## Overview
Tensor Shape and Invariant Analysis is a family of visualization mapping techniques that decompose second-order tensor fields into scalar invariants (such as eigenvalues, norms, or directional distributions) to reveal structural properties like anisotropy, stress type, or dominant orientation. It distinguishes itself from sibling methods like [Glyph-based Vector/Tensor Visualization](10535-glyph-based-vector-tensor-visualization.md) by abstracting the tensor into statistical or geometric descriptors rather than rendering local geometric primitives, and from [Diffusion Tensor Imaging (DTI) Visualization](5713-diffusion-tensor-imaging-dti-visualization.md) by offering robust Cartesian-based representations (Shape Space) that remain stable for indefinite tensors where spherical coordinates fail.

## Shared principles
All methods in this family evaluate a mapping function $M(\mathbf{x})$ at sample points $\mathbf{x}$ within the volume to produce visual attributes [G]. The core mechanism involves sampling the multi-component data field at $\mathbf{x}$, decomposing or normalizing the value (e.g., extracting magnitude, direction, eigenvalues, or phase), and applying a transfer function or geometric construction to map these features to rendering parameters such as color, opacity, or primitive geometry [G].

## Subtopics

### [Shape Space (Tensor Eigenvalue Space)](8885-shape-space-tensor-eigenvalue-space.md)
Shape Space is a visualization mapping technique that represents the shape of a second-order tensor by mapping its three eigenvalues to a point in a 3D vector space. It provides a coordinate system where the tensor's structural properties are explicit [G]. Unlike spherical coordinate systems (tensor norm, fractional anisotropy, tensor mode) used in Diffusion Tensor Imaging (DTI), which become unstable for tensors with small norms because angular coordinates are undefined at the origin, Shape Space uses Cartesian-like descriptors that remain stable for indefinite tensors where eigenvalues may be positive, negative, or zero.

### [Directional Histogram](8888-directional-histogram.md)
The Directional Histogram is a visualization method for analyzing the distribution of principal directions (such as eigenvectors or shear directions) within a tensor field by projecting them onto a unit sphere. It differs from sibling methods like [Domain Colouring](1833-domain-colouring.md) or [Glyph-based Vector/Tensor Visualization](10535-glyph-based-vector-tensor-visualization.md) by aggregating directional data into statistical bins on a sphere rather than mapping values directly to color at spatial locations or rendering geometric primitives. This allows for the identification of dominant orientations and anisotropy in the field.

### Shape Space Scatterplot
#### definition
- [S1, book] A 2D plot depicting the relation between two scalar invariants, specifically principal stresses, to quantify normal and shear stresses.
#### step
- [S1, book] Sort the three principal stresses in descending order: $\sigma_1 \ge \sigma_2 \ge \sigma_3$.
- [S1, book] Plot $\sigma_1$ on the x-axis and $\sigma_3$ on the y-axis, dividing the plot into four quadrants (A, B, C, D).
- [S1, book] Apply a sign-preserving logarithmic mapping to the eigenvalues: $f(\sigma_i) = \log(\sigma_i + 1)$ for $\sigma_i \ge 0$, and $f(\sigma_i) = -\log(1 - \sigma_i)$ for $\sigma_i < 0$.
- [S1, book] Linearly map the results of the logarithmic mapping to the range [-1, 1].
#### comparison
- [S1, book] Unlike standard logarithmic mappings that require positive domains, this method preserves signs to distinguish between positive (tensile) and negative (compressive) invariants.

### Multi-scale Structure Tensor Analysis
#### definition
- [S2] A method to estimate local frequency content and orientation by analyzing image structure across multiple scales using Gaussian blur kernels and aggregating normalized structure tensors weighted by texture energy.
#### step
- [S2] Construct a Gaussian scale space using kernels $G_{\tilde{\tau}_l}$ with increasing standard deviations $\tilde{\tau}_l = 1.5^l$ for $l \in 0, \dots, L$ (where $L=4$ in practice), maintaining full spatial resolution at all levels to yield images $I_l$.
- [S2] At each level $l$, compute per-pixel structure tensors $S_l$ using an integration scale $\rho_l = 3\tilde{\tau}_l$.
#### formula
- [S2] Normalize the tensor to remove influence of overall signal magnitude: $\hat{S}_l = \frac{S_l}{\text{tr}(S_l) + \epsilon}$, where $\epsilon > 0$ ensures numerical stability.
- [S2] Aggregate structure tensors weighted by energy and squared frequency: $\bar{S} = \frac{\sum_{l=0}^L E_l^\gamma \omega_l^2 \hat{S}_l}{\sum_{l=0}^L E_l^\gamma + \epsilon}$, where $\gamma=3.0$ controls weighting sharpness.

## Background
The concept of Shape Space and Mohr diagrams for stress tensor analysis was formalized in engineering visualization contexts, notably presented in "A Visual Approach to Analysis of Stress Tensor Fields" (2011) [S1, book]. This work introduced Shape Space Scatterplots and Mohr diagrams as tools to quantify normal and shear stresses, addressing the limitations of spherical coordinates for indefinite tensors. The application of multi-scale structure tensor analysis for frequency-aware visualization was later adapted for 3D Gaussian Splatting convergence in "Faster 3D Gaussian Splatting Convergence via Structure-Aware Densification" (2025) [S2], extending tensor analysis to scale-space representations for structure-aware densification.

## Sources
[S1] A Visual Approach to Analysis of Stress Tensor Fields, https://doi.org/10.4230/dfu.vol2.sciviz.2011.188, fetched 2026-10-26 (book)
[S2] Faster 3D Gaussian Splatting Convergence via Structure-Aware Densification, https://doi.org/10.1145/3799902.3811212, fetched 2026-10-30
