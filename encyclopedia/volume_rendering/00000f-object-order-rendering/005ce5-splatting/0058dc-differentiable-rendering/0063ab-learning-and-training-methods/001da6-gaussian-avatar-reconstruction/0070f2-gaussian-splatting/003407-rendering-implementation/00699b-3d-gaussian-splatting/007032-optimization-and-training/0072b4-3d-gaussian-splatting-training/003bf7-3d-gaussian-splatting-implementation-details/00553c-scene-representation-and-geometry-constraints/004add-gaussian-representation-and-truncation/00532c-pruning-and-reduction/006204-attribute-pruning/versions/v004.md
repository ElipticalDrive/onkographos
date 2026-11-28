# Attribute Pruning

Also called: SH Degree Adjustment, Spherical Harmonics Pruning

## Overview
Attribute pruning selectively reduces the storage and computational burden of per-Gaussian attributes, primarily spherical harmonics (SH) coefficients, to optimize memory usage and rendering efficiency [S1]. Unlike sibling methods that remove primitives entirely, this technique retains the geometric structure of the scene while adapting the fidelity of view-dependent color representation to local complexity [S1]. It distinguishes itself from [SplatQ](14415-splatq.md) by focusing on dynamic coefficient reduction rather than quantization, and from [Vertex-level Pruning in Mixed-Primitive Gaussian Splatting](13983-vertex-level-pruning-in-mixed-primitive-gaussian-splatting.md) by operating on the standard Gaussian representation's attribute fields rather than modifying the primitive topology.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ 3D Gaussians $\{G_i\}_{i=1}^N$, where each Gaussian $G_i$ has mean $\boldsymbol{\mu}_i$, covariance $\boldsymbol{\Sigma}_i$, opacity $\alpha_i$, and a vector of SH coefficients $\mathbf{c}_i$ [S1].
*   **Output:** A modified set of Gaussians with reduced attribute storage, where the number of SH coefficients per Gaussian varies based on scene complexity [S1].

**Data Structures:**
Each Gaussian $G_i$ stores attributes. In standard 3DGS, SH coefficients for RGB data at degree 3 require 48 coefficients (16 per channel), consuming 81% of the single-Gaussian storage size (48 out of 59 attributes) [S1]. The pruning method modifies the size or content of the coefficient vector $\mathbf{c}_i$.

**Steps:**
1.  **Analyze Scene Complexity:** Evaluate the complexity of the region associated with each Gaussian. This is often determined by the variance of color across different viewing angles or the magnitude of higher-order SH coefficients [S1].
2.  **Determine SH Degree:** Assign a SH degree $d_i$ to each Gaussian $G_i$ based on the analysis:
    *   For less complex regions (e.g., uniform surfaces, low view-dependent variation), set $d_i = 0$ (basic RGB color, 3 coefficients) [S1].
    *   For complex regions (e.g., specular highlights, intricate lighting), retain higher degrees (e.g., $d_i = 3$, 48 coefficients) [S1].
3.  **Adjust Coefficients:** Truncate or zero-out higher-order SH coefficients for Gaussians assigned lower degrees. If $d_i < 3$, coefficients corresponding to degrees $> d_i$ are discarded or ignored during rendering [S1].
4.  **Render:** During splatting, evaluate the color $\mathbf{c}_i(\omega)$ using only the retained SH coefficients for the assigned degree [S1].

**Formulas:**
The color contribution of a Gaussian $G_i$ at viewing direction $\omega$ is computed using the retained SH coefficients $\mathbf{c}_i^{(d_i)}$:
$$ \mathbf{c}_i(\omega) = \sum_{l=0}^{d_i} \sum_{m=-l}^{l} c_{i,l,m} Y_{l,m}(\omega) $$
where $Y_{l,m}$ are the spherical harmonic basis functions [S1].

**Parameters:**
*   **SH Degree:** Typically ranges from 0 to 3. Degree 3 is the standard in 3DGS but is often reduced to 0 or 1 for static or simple regions [S1].
*   **Thresholds:** Not given in the sources. Implementation requires thresholds to decide when a region is "complex" enough to warrant higher SH degrees.

**Edge Cases and Pitfalls:**
*   **Precision Loss:** Reducing SH degree can introduce visible artifacts in regions with high-frequency view-dependent effects if the threshold is too aggressive [S1].
*   **Uniformity:** Applying a uniform SH degree across the entire scene negates the benefits of pruning; the method requires per-Gaussian or per-region adaptation [S1].
*   **Alternative Representations:** Completely removing SH in favor of NeRF-like MLPs or classical shading is possible but requires custom, potentially slower rendering pipelines [S1].

## Subtopics

### Reduced3DGS
An approach that extends attribute pruning by combining SH band adjustment with primitive reduction and quantization [S1]. It introduces a scale- and resolution-aware redundant primitive removal method that uses a redundancy score to identify low-impact primitives, extending opacity-based pruning [S1]. For attributes, it adaptively adjusts SH bands by evaluating color consistency across views, reducing higher-order bands when view-dependent effects are minimal [S1]. It further applies K-means clustering to create a codebook and uses 16-bit half-float quantization for remaining uncompressed floating-point values [S1].

## Operating characteristics
*   **Memory:** Significantly reduces storage size. Standard 3DGS uses 48 SH coefficients for degree 3, consuming 81% of per-Gaussian storage [S1]. Pruning can reduce this to 3 coefficients (degree 0) for simple regions, offering substantial memory savings [S1].
*   **Compute:** Reduces the number of coefficients evaluated during rendering, lowering computational cost per Gaussian [S1].
*   **Quality:** Trade-off between memory/compute and visual fidelity. Lower SH degrees may cause blurring or loss of specular highlights in complex regions [S1].
*   **Use Cases:** Ideal for memory-constrained devices (e.g., VR headsets) or large scenes where uniform high-fidelity SH is unnecessary [S1].

## Background
Attribute pruning was introduced as a method to address the large storage footprint of 3D Gaussian Splatting, where SH coefficients dominate the memory usage [S1]. It builds on the observation that not all parts of a scene require high-degree SH for accurate view-dependent rendering [S1]. The concept is discussed in the context of compression and optimization techniques for 3DGS [S1].

## Sources
[S1] 3DGS.zip: A survey on 3D Gaussian Splatting Compression Methods, https://arxiv.org/abs/2407.09510v5, fetched 2026-10-26
