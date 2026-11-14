# Pixel Intensity Evaluation with Scene Aggregates

Also called: ABV, AIV/ABV Integration, Aggregated Boundary Visibility, Aggregated Visibility Functions Integration

## Overview
Pixel Intensity Evaluation with Scene Aggregates is a rendering method that integrates radiance over a pixel footprint using aggregated visibility functions to handle inter-voxel correlation and background blending without order-dependent ray marching [S1, unofficial]. It solves the problem of correctly blending scene aggregate contributions with external background objects, which standard volumetric models fail to distinguish when voxels have identical local properties but different spatial correlations [S1, unofficial]. It sets itself apart from sibling methods like [X-Ray Volume Rendering](11336-x-ray-volume-rendering.md) by explicitly modeling the semi-transparency of geometry primitives via boundary visibility, rather than relying solely on scalar accumulation or extremum selection [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A discretized scene represented as a set of voxels, each associated with a truncated ellipsoid primitive [S1, unofficial]. A pixel footprint $P$ in world space and the pixel direction $\boldsymbol{\nu}_p$ [S1, unofficial].
*   **Output:** Pixel intensity $I$ [S1, unofficial].

**Data Structures:**
*   **Boundary Faces:** A list of 2D faces enclosing the 3D scene aggregate, determined by a flood fill iteration on the discretized scene [S1, unofficial].
*   **ABV Map:** A 2D visibility map stored in the Haar wavelet basis, parameterizing the hemispherical domain for each boundary face [S1, unofficial].
*   **AIV Terms:** Visibility terms stored per voxel, also represented by wavelet coefficients for compression [S1, unofficial].

**Numbered Steps:**
1.  **Scene Discretization:** Represent the volume as a set of voxels, each abstracted as a truncated ellipsoid primitive [S1, unofficial].
2.  **Boundary Face Identification:** Perform a flood fill iteration on the discretized scene to determine the list of boundary faces [S1, unofficial].
3.  **ABV Precomputation:**
    a. For each boundary face, consider all directions in its inward-facing hemisphere [S1, unofficial].
    b. Use concentric mapping to parameterize the hemispherical domain [S1, unofficial].
    c. For each direction, cast visibility rays with origins uniformly sampled on the face to estimate average visibility [S1, unofficial].
    d. Project the resulting 2D visibility map to the Haar wavelet basis [S1, unofficial].
4.  **AIV Precomputation:** Compute Aggregated Interior Visibility terms for voxels and project them to the Haar wavelet basis for compression [S1, unofficial].
5.  **Pixel Intensity Evaluation:**
    a. Identify the subset $P^+$ of the pixel footprint covered by the union of primitives from relevant voxels [S1, unofficial].
    b. For points $\boldsymbol{x}_p \in P \setminus P^+$, evaluate incoming radiance as background radiance $L_b(-\boldsymbol{\nu}_p)$ [S1, unofficial].
    c. For points $\boldsymbol{x}_p \in P^+$, evaluate incoming radiance using the precomputed AIV and ABV terms to account for occlusion and scattering within the voxel [S1, unofficial].
    d. Integrate $L_i$ over $P$ to compute $I$ [S1, unofficial].

**Formulas:**
The pixel intensity $I$ is defined as the integration of incoming radiance $L_i$ over the pixel footprint $P$ [S1, unofficial]:
$$
I = \frac{1}{|P|} \int_P L_i(\boldsymbol{x}_p, \boldsymbol{\nu}_p) d\boldsymbol{x}_p
$$
where $|P|$ is the area of $P$ and $\boldsymbol{\nu}_p$ is the pixel direction [S1, unofficial].

The Aggregated Boundary Visibility (ABV) $\hat{V}_b$ is defined on the 2D boundary enclosing the scene aggregate [S1, unofficial]:
$$
\hat{V}_b(\boldsymbol{\nu}; P^+) = \frac{1}{|P|} \int_{P^+} V(\boldsymbol{x}, \boldsymbol{\nu}) d\boldsymbol{x}
$$
where $P^+$ is the subset of $P$ such that rays from $P^+$ intersect scene aggregate primitives, and $V(\boldsymbol{x}, \boldsymbol{\nu})$ is the visibility at point $\boldsymbol{x}$ in direction $\boldsymbol{\nu}$ [S1, unofficial].

**Parameters:**
*   **Angular Resolution:** The resolution for ABV precomputation is typically $64^2$, chosen for accurate reconstruction of the high-frequency silhouette signal [S1, unofficial].
*   **Wavelet Basis:** Haar wavelet basis is used for projecting visibility maps [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Inter-voxel Correlation:** Standard volumetric models cannot distinguish between voxels with identical local properties but different correlations [S1, unofficial].
*   **High-Frequency Silhouettes:** ABV contains high-frequency signals [S1, unofficial].
*   **Far-Field Assumption:** The method assumes a far-field background, meaning background radiance $L_b$ does not depend on position [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Precomputation of ABV and AIV involves ray casting and wavelet projection, which is offline [S1, unofficial]. Runtime evaluation involves integrating radiance using precomputed coefficients, avoiding order-dependent ray marching [S1, unofficial].
*   **Memory Complexity:** Storage of wavelet coefficients for ABV and AIV terms [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Correctly handles inter-voxel correlation and background blending; preserves all-frequency information through wavelet representation; efficient for level-of-detail rendering [S1, unofficial].
*   **Limitations:** Requires offline precomputation; relies on far-field assumption for background; complexity of implementing wavelet projection and concentric mapping [S1, unofficial].
*   **When to use:** When accurate blending of volumetric aggregates with background is required, and inter-voxel correlation is significant [S1, unofficial]. Suitable for level-of-detail rendering where visual fidelity of silhouettes is critical [S1, unofficial].
*   **When not to use:** When real-time performance is critical and precomputation is not feasible, or when the background is not far-field [S1, unofficial].

## Background
The method was introduced in the paper "Efficient Scene Appearance Aggregation for Level-of-Detail Rendering" [S1, unofficial]. It addresses limitations in traditional volumetric models that fail to distinguish different types of inter-voxel correlation [S1, unofficial]. The use of concentric mapping references Shirley and Chiu (1997) [S1, unofficial], and the wavelet representation references Clarberg (2008) [S1, unofficial]. It is used for level-of-detail rendering of volumetric scenes [S1, unofficial].

## Sources
[S1] Efficient Scene Appearance Aggregation for Level-of-Detail Rendering, https://core.ac.uk/works/391729893, fetched 2026-11-01 (unofficial)
