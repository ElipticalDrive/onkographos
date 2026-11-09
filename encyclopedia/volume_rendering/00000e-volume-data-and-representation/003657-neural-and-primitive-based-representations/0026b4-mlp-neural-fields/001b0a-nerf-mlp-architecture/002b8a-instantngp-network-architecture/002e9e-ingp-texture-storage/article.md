# iNGP Texture Storage

Also called: Texture Storage for Real-Time Rendering, iNGP Texture Storage for Real-Time Rendering

## Overview
This method optimizes the rendering performance of Instant-NGP (INGP) by replacing hash-table-based feature lookups with explicit texture storage, leveraging hardware-accelerated texture interpolation and memory locality [S2]. It reduces the number of texture fetches per MLP evaluation from 24 to 8 by storing presummed multi-resolution features, enabling a 3× reduction in fetch overhead [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Multi-resolution feature grids $g(v, l)$ trained during the model fitting phase, where $v$ is the spatial coordinate and $l$ is the resolution level index.
*   **Output:** A set of texture maps (presumed features) ready for real-time rendering, and the rendered RGB color and opacity for a given ray.

**Data Structures:**
The method replaces the hash table lookup used in standard INGP with explicit 3D texture grids.
*   **Texture Grids:** For each resolution level $L$, a 3D texture is stored.
*   **Presumed Features:** Instead of storing individual level features $g(v, l)$, the method stores presummed features $g'(v, l)$ defined as the cumulative sum of features from level 0 to $l$:
    $$ g'(v, l) = \sum_{k=0}^{l} g(v, k) $$
    This allows the retrieval of the total feature vector for any resolution level by interpolating between two adjacent presummed levels [S2].

**Numbered Steps:**
1.  **Training Phase:** Train the multi-resolution feature grids $g(v, l)$ using the standard INGP optimization process. The features are initialized in the range $[-10^{-4}, 10^{-4}]$ [S1].
2.  **Presumption Calculation:** After training, compute the presummed feature grids $g'(v, l)$ for each level $l$ by summing the features from the coarsest level (0) up to level $l$ [S2].
3.  **Texture Upload:** Upload the presummed feature grids $g'(v, l)$ to GPU texture memory. The number of levels $L$ and resolutions depend on the INGP configuration (e.g., $L=16$, $N_{min}=2$, $N_{max}=16$) [S1, S2].
4.  **Ray Sampling:** For a given ray, determine the sample points $x$.
5.  **Pixel Area Estimation:** Calculate the pixel area $p(x)$ at each sample point to determine the effective resolution required for anti-aliasing [S2].
6.  **Level Selection:** Identify the two adjacent resolution levels $l_1$ and $l_2$ implied by the pixel area $p(x)$ [S2].
7.  **Texture Fetching:** Perform texture lookups (interpolation) on the presummed feature grids $g'$ at levels $l_1$ and $l_2$ for the sample point $x$. This requires fetching features from 8 texture units (assuming 3D grids and 7 triplane levels in the specific HybridNeRF implementation cited, or generally fewer than the original 24 fetches) [S2].
8.  **Feature Interpolation:** Interpolate between the fetched presummed features to obtain the final feature vector $y$ for the sample point [S2].
9.  **MLP Evaluation:** Pass the feature vector $y$ (and potentially other inputs like view direction) into the density and color MLPs to compute volume density $\sigma$ and color $C$ [S1, S2].
10. **Volume Rendering:** Composite the colors and densities along the ray to produce the final pixel color [S2].

**Formulas:**
*   **Presumed Feature Storage:**
    $$ g'(v, l) = \sum_{k=0}^{l} g(v, k) $$
    Where $g(v, k)$ is the feature vector at level $k$ [S2].
*   **Feature Retrieval:** The feature vector $y$ at a sample point $x$ is obtained by interpolating between $g'(x, l_1)$ and $g'(x, l_2)$ based on the pixel area $p(x)$ [S2].
*   **Density Computation:**
    $$ \sigma = \text{MLP}_{\Phi_1}(y) $$
    Where $y$ is the interpolated feature vector [S1].

**Parameters:**
*   **Number of Levels ($L$):** Determines the number of resolution levels. In the cited INGP configuration, $L=16$ [S1].
*   **Coarsest Resolution ($N_{min}$):** The resolution of the coarsest grid. Typically $N_{min}=2$ [S1].
*   **Finest Resolution ($N_{max}$):** The resolution of the finest grid. Typically $N_{max}=16$ [S1].
*   **Texture Fetches:** The method reduces fetches to 8 per MLP evaluation, compared to 24 in the original hash-table approach (assuming 3D grids and 7 triplane levels) [S2].

**Edge Cases and Pitfalls:**
*   **Memory Locality:** The primary benefit relies on the GPU's texture cache. If the access pattern is not coherent, the performance gain may be reduced [S2].
*   **Anti-Aliasing:** The method assumes that interpolating between presummed levels provides a good approximation of the anti-aliased feature. Degenerate cases where pixel area changes rapidly may require careful level selection [S2].
*   **Hash Table Replacement:** This method is specific to explicit grid representations. It cannot be directly applied to hash-based encodings without converting them to grids, which may incur significant memory overhead for large scenes [S2].

## Operating characteristics
*   **Time Complexity:** The number of texture fetches is reduced by 3× (from 24 to 8) per MLP evaluation compared to the original INGP hash-table approach [S2]. This leads to faster rendering times, particularly on hardware with efficient texture interpolation units.
*   **Memory Complexity:** Storing explicit grids instead of hash tables may increase memory usage for very large scenes or high resolutions, as the grid size is fixed by $N_{max}$ and $L$. However, for typical INGP resolutions, the memory overhead is manageable [S1, S2].
*   **Performance:** The method is designed for real-time rendering. The reduction in fetches and improved memory locality significantly boost frame rates [S2].
*   **Strengths:** High rendering speed, hardware-accelerated interpolation, simplified anti-aliasing via level interpolation [S2].
*   **Limitations:** Fixed memory footprint based on grid resolution; less flexible than hash tables for highly sparse or irregular scenes [S2].
*   **When to use:** Real-time rendering applications where performance is critical and the scene can be represented efficiently with explicit grids [S2].
*   **When not to use:** Scenes with extremely high detail that would require prohibitively large grids, or when dynamic scene updates require frequent grid resizing [S2].

## Background
The concept of storing features as textures for real-time rendering was introduced in the context of HybridNeRF, which combines neural rendering with explicit volumetric surfaces [S2]. While Instant-NGP (INGP) popularized multi-resolution hash encoding for fast training and rendering, it relies on hash table lookups which can be less efficient on some hardware due to memory latency and lack of hardware interpolation [S2]. The texture storage method adapts the INGP feature representation to explicit grids to leverage GPU texture units [S2].

## Sources
[S1] CAwa-NeRF: Instant Learning of Compression-Aware NeRF Features, https://doi.org/10.1109/sds60720.2024.00015, fetched 2026-10-26
[S2] HybridNeRF: Efficient Neural Rendering via Adaptive Volumetric Surfaces, https://doi.org/10.1109/cvpr52733.2024.01858, fetched 2026-10-26
