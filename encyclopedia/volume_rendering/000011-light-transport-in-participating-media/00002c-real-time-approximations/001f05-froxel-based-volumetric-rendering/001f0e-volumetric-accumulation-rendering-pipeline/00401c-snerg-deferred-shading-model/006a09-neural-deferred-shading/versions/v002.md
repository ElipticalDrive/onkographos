# Neural Deferred Shading

Also called: Deferred Radiance Field Rendering

## Overview
Neural Deferred Shading is a decoding strategy for radiance fields that models view-dependent appearance using a lightweight shared Multi-Layer Perceptron (MLP) applied per pixel [S1][S2, unofficial]. It separates view-independent opacity from view-dependent color, allowing the network to decode final pixel color from accumulated feature vectors and ray direction without further positional encoding [S1]. The method provides an efficient decoding strategy that avoids high-order spherical harmonic bases while maintaining competitive novel-view synthesis quality [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Rasterized per-pixel feature maps (accumulated from primitives), ray direction $\mathbf{d}$, and MLP weights $\theta$ [S1][S2, unofficial].
*   **Output:** Final pixel color $\mathbf{c}$ [S1].

**Data Structures:**
*   **Primitive Features:** Each primitive carries a learnable feature vector $\mathbf{f}_i$ [S1][S2, unofficial]. In triangle-based rasterization frameworks, $N_{feat} = 7$ [S2, unofficial].
*   **Accumulated Features:** Per-pixel accumulation of transformed primitive features, weighted by opacity $\alpha_i$ and transmittance $T_i$ [S1].
*   **MLP:** A shallow, lightweight shared MLP $\text{MLP}_\theta$ [S1][S2, unofficial].

**Steps:**
1.  **Encode Ray Direction:** Compute the second-degree spherical harmonics encoding of the ray direction $\mathbf{d}$, denoted as $\text{SH}_2(\mathbf{d}) \in \mathbb{R}^9$ [S1].
2.  **Rasterize and Accumulate Features:** Rasterize the primitive feature vectors $\mathbf{f}_i$ to yield per-pixel feature maps [S2, unofficial]. For each pixel, accumulate contributions from intersected primitives $\mathcal{G}$ using opacity $\alpha_i$ and transmittance $T_i$ [S1]. The accumulated vector $\mathbf{A}$ is computed as:
    $$ \mathbf{A} = \sum_{i \in \mathcal{G}} \alpha_i T_i \begin{bmatrix} \sin(\mathbf{f}_i) \\ \cos(\mathbf{f}_i) \end{bmatrix} $$
    where $\mathbf{f}_i$ is the primitive feature interpolated at the point of maximum response $p^*_i$, and $\sin(\cdot)$ and $\cos(\cdot)$ are applied element-wise [S1].
3.  **Concatenate Inputs:** Concatenate the accumulated harmonic sum $\mathbf{A}$ with the scaled spherical harmonics encoding of the ray direction $k \cdot \text{SH}_2(\mathbf{d})$ [S1]. In triangle-based rasterization, the rasterized feature vector is concatenated directly with the viewing direction [S2, unofficial].
4.  **Decode Color:** Pass the concatenated input through the shallow MLP $\text{MLP}_\theta$ to produce the final shaded color $\mathbf{c}$ [S1][S2, unofficial].

**Formula:**
The rendering equation for the final color $\mathbf{c}$ is:
$$ \mathbf{c} = \text{MLP}_\theta \left( \sum_{i \in \mathcal{G}} \alpha_i T_i \begin{bmatrix} \sin(\mathbf{f}_i) \\ \cos(\mathbf{f}_i) \end{bmatrix}, k \cdot \text{SH}_2(\mathbf{d}) \right) $$
where:
*   $\text{MLP}_\theta$ is the shallow MLP with weights $\theta$ [S1].
*   $\mathcal{G}$ is the set of primitives intersected by the ray [S1].
*   $\alpha_i$ is the primitive opacity [S1].
*   $T_i$ is the accumulated transmittance [S1].
*   $\mathbf{f}_i$ is the primitive feature vector [S1].
*   $k$ is a scaling factor applied to the spherical harmonics encoding [S1].
*   $\text{SH}_2(\mathbf{d})$ is the second-degree spherical harmonics encoding of the ray direction $\mathbf{d}$ [S1].

**Parameters:**
*   $N_{feat}$: Dimension of the learnable feature vector. Value: 7 [S2, unofficial].
*   $k$: Scaling factor for $\text{SH}_2(\mathbf{d})$. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Opacity Handling:** Opacity is stored separately (e.g., in a scalar texture) and kept outside the neural network to ensure efficient access during stochastic opacity masking, where opacity must be evaluated for all fragments, but only the "winning" fragment's color participates in differentiation [S2, unofficial].
*   **Precision:** Feature vectors are stored as 8-bit PNG images after sigmoid activation to reduce storage [S2, unofficial].
*   **Input Dimensionality:** The input to the MLP depends on the specific implementation (accumulated harmonics vs. rasterized features) [S1][S2, unofficial].

## Subtopics

### Deep Shading
Deep Shading employs deferred shading buffers as input and uses deep neural networks as renderers to produce specific rendering outcomes such as environment shading [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Decoding requires a single MLP evaluation per pixel [S1][S2, unofficial].
*   **Memory Complexity:** Storage is reduced by using learnable feature vectors instead of storing full RGB values or high-order SH coefficients per primitive [S2, unofficial]. Features and opacity can fit into two 8-bit RGBA textures [S2, unofficial].
*   **Strengths:** Efficient decoding, reduced storage, and view-dependent appearance modeling without high-order bases [S1][S2, unofficial].
*   **Limitations:** Requires training an MLP, which may introduce optimization challenges [S1][S2, unofficial].
*   **Scaling:** Not given in the sources.
*   **Measured Performance:** Competitive PSNR and SSIM scores compared to baseline methods on the MipNeRF360 dataset using 15K primitives [S2, unofficial].
*   **When to Use:** Not given in the sources.
*   **When Not to Use:** Not given in the sources.

## Background
Neural Deferred Shading was introduced as a decoding strategy for radiance fields to efficiently model view-dependent appearance [S1][S2, unofficial]. It builds on the idea of separating view-independent and view-dependent components in radiance field representations [S1][S2, unofficial]. The method has been applied to radiance-field novel-view synthesis, semantic scene reconstruction, and high-resolution image fitting [S1]. It is also utilized in extreme radiance field simplification using triangle soups [S2, unofficial].

## Sources
[S1] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S2] DiffSoup: Direct Differentiable Rasterization of Triangle Soup for Extreme Radiance Field Simplification, https://doi.org/10.48550/arXiv.2603.27151, fetched 2026-10-30 (unofficial)
[S3] Neural Rendering and Its Hardware Acceleration: A Review, https://doi.org/10.48550/arxiv.2402.00028, fetched 2026-10-26 (unofficial)
