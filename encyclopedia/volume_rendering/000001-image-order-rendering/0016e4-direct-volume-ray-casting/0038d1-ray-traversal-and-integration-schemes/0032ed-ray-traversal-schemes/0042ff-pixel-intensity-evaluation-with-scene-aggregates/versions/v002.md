# Pixel Intensity Evaluation with Scene Aggregates

Also called: AIV/ABV Integration, Aggregated Boundary Visibility, Aggregated Visibility Functions Integration

## Overview
Pixel Intensity Evaluation with Scene Aggregates is a method for rendering volumetric data that replaces order-dependent ray marching with a precomputed visibility aggregation model [S1, unofficial]. It solves the problem of correctly blending scene aggregate contributions with external background objects while handling inter-voxel correlation, which traditional volumetric models fail to distinguish [S1, unofficial]. Unlike sibling schemes such as [Intensity](52-intensity.md) or [Accumulate](52-accumulate.md) that process samples sequentially, this method evaluates pixel intensity by integrating radiance over the pixel footprint using Aggregated Visibility Functions (AVF), specifically the Aggregated Boundary Visibility (ABV) [S1, unofficial].

## How it works
The method evaluates pixel intensity by integrating incoming radiance over the pixel footprint, utilizing precomputed visibility terms to handle occlusion and transparency without per-ray traversal [S1, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   Scene geometry represented as a set of primitives (e.g., truncated ellipsoids) derived from voxels [S1, unofficial].
    *   Pixel footprint $P$ in world space [S1, unofficial].
    *   Pixel direction $\boldsymbol{\nu}_p$ [S1, unofficial].
    *   Background radiance $L_b$ (assumed constant due to far-field assumption) [S1, unofficial].
    *   Precomputed Aggregated Boundary Visibility (ABV) data [S1, unofficial].
*   **Outputs**:
    *   Pixel intensity $I$ [S1, unofficial].

### Data Structures
The core data structure is the **Aggregated Boundary Visibility (ABV)** term, stored as wavelet coefficients [S1, unofficial].
*   **ABV Representation**: The ABV term $\hat{V}_b$ is defined on the 2D boundary enclosing the 3D scene aggregate [S1, unofficial].
*   **Storage**: The visibility function is projected to the Haar wavelet basis to allow for compression and efficient reconstruction [S1, unofficial].
*   **Resolution**: The angular resolution for the precomputed ABV is typically $64^2$ to accurately reconstruct the high-frequency silhouette signal [S1, unofficial].

### Steps
1.  **Scene Discretization**: Convert the volumetric scene into a set of primitives (e.g., truncated ellipsoids) associated with voxels [S1, unofficial].
2.  **Boundary Face Identification**: Determine the list of boundary faces of the voxels using a flood fill iteration on the discretized scene [S1, unofficial].
3.  **ABV Precomputation**: For each boundary face, perform the following sub-steps:
    a.  Consider all directions in the inward-facing hemisphere of the face [S1, unofficial].
    b.  Parameterize the hemispherical domain using concentric mapping to obtain a 2D domain [S1, unofficial].
    c.  Cast visibility rays with origins uniformly sampled on the face for each direction to estimate average visibility [S1, unofficial].
    d.  Project the resulting 2D visibility map to the Haar wavelet basis [S1, unofficial].
4.  **Pixel Intensity Evaluation**: For a given pixel, calculate the intensity $I$ by integrating the incoming radiance $L_i$ over the pixel footprint $P$ [S1, unofficial]:
    $$ I = \frac{1}{|P|} \int_P L_i(\boldsymbol{x}_p, \boldsymbol{\nu}_p) d\boldsymbol{x}_p $$
    where $|P|$ is the area of $P$ and $\boldsymbol{\nu}_p$ is the pixel direction [S1, unofficial].
5.  **Radiance Determination**: Determine $L_i(\boldsymbol{x}_p, \boldsymbol{\nu}_p)$ for each point $\boldsymbol{x}_p$ in $P$ based on whether it is covered by scene primitives [S1, unofficial]:
    *   Let $P^+$ be the subset of $P$ covered by the primitive(s) (i.e., rays from $P^+$ intersect the scene aggregate) [S1, unofficial].
    *   Let $P^- = P \setminus P^+$ be the subset not covered [S1, unofficial].
    *   If $\boldsymbol{x}_p \in P^-$, $L_i(\boldsymbol{x}_p, \boldsymbol{\nu}_p) = L_b(-\boldsymbol{\nu}_p)$ (background radiance) [S1, unofficial].
    *   If $\boldsymbol{x}_p \in P^+$, $L_i(\boldsymbol{x}_p, \boldsymbol{\nu}_p)$ is evaluated based on the primitive's outgoing radiance, modulated by the visibility [S1, unofficial].

### Formulas
The Aggregated Boundary Visibility (ABV) is defined as [S1, unofficial]:
$$ \hat{V}_b(\boldsymbol{\nu}; P^+) = \frac{1}{|P|} \int_{P^+} V(\boldsymbol{x}, \boldsymbol{\nu}) d\boldsymbol{x} $$
where:
*   $P$ is the pixel footprint in world space [S1, unofficial].
*   $P^+$ is the subset of $P$ such that rays from $P^+$ intersect scene aggregate primitives [S1, unofficial].
*   $|P|$ is the area of $P$ [S1, unofficial].
*   $V(\boldsymbol{x}, \boldsymbol{\nu})$ is the visibility at point $\boldsymbol{x}$ in direction $\boldsymbol{\nu}$ [S1, unofficial].

The pixel intensity is calculated as [S1, unofficial]:
$$ I = \frac{1}{|P|} \int_P L_i(\boldsymbol{x}_p, \boldsymbol{\nu}_p) d\boldsymbol{x}_p $$

### Parameters
*   **Angular Resolution**: $64^2$ samples per hemisphere for ABV precomputation [S1, unofficial].
*   **Mapping**: Concentric mapping is used to parameterize the hemispherical domain [S1, unofficial].
*   **Basis**: Haar wavelet basis is used for projecting the visibility map [S1, unofficial].

### Edge Cases and Pitfalls
*   **Inter-voxel Correlation**: Traditional volumetric models cannot distinguish between different types of inter-voxel correlation (e.g., clustering vs. uniform distribution) if local properties are identical. This method handles this by using aggregated visibility functions [S1, unofficial].
*   **High-Frequency Silhouettes**: The ABV term contains high-frequency signals responsible for reconstructing the silhouette of the scene. Insufficient angular resolution (e.g., lower than $64^2$) may lead to inaccurate coverage reconstruction [S1, unofficial].
*   **Far-Field Assumption**: The method assumes a far-field background, meaning background radiance $L_b$ does not depend on position [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Precomputation is expensive, requiring flood fill, ray casting for visibility, and wavelet projection for all boundary faces. However, runtime evaluation is efficient as it avoids per-ray marching, relying instead on integration over the pixel footprint using precomputed data [S1, unofficial].
*   **Memory Complexity**: Requires storage for the wavelet coefficients of the ABV (and potentially AIV) terms for all boundary faces. The resolution of $64^2$ per face contributes to memory usage [S1, unofficial].
*   **Strengths**: Correctly blends scene aggregates with background; handles inter-voxel correlation; avoids order-dependent ray marching [S1, unofficial].
*   **Limitations**: Requires precomputation; relies on far-field assumption for background; accuracy depends on angular resolution [S1, unofficial].
*   **When to use**: Suitable for level-of-detail rendering where efficient evaluation of complex volumetric scenes with accurate background blending is required [S1, unofficial].
*   **When not to use**: When dynamic scenes prevent precomputation or when near-field background effects are critical [S1, unofficial].

## Background
The method was introduced in the paper "Efficient Scene Appearance Aggregation for Level-of-Detail Rendering" [S1, unofficial]. It addresses limitations in traditional volumetric rendering models that fail to account for inter-voxel correlation and complex occlusion patterns efficiently. The approach uses aggregated visibility functions (AIV and ABV) represented by wavelet coefficients to preserve all-frequency information [S1, unofficial].

## Sources
[S1] Efficient Scene Appearance Aggregation for Level-of-Detail Rendering, https://core.ac.uk/works/391729893, fetched 2026-11-01 (unofficial)
