# Soft Shadows via Area Lights

Also called: Area Light Shadow Sampling, Area Light Shadows, Partial Occlusion, Shadow Occlusion

## Overview
This method models shadows cast by light sources with spatial extent, allowing points in space to be partially occluded, unlike point sources which cast sharp shadows [S2, unofficial]. It sets itself apart from its siblings by replacing the single boolean shadow test of point sources with area light sampling to compute natural soft shadows with penumbra [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A ray origin $\mathbf{o}$ and direction $\boldsymbol{\omega}$, a distance field or volume representation of the scene, and an area light source definition (position, orientation, and extent).
*   **Output:** The transmittance $T$ or shadow factor representing the fraction of light from the area source that reaches the point along the ray.

**Data Structures**
*   **Area Light:** Defined by a center position, a normal vector (for directional area lights), and a geometric extent (e.g., radius for disks, width/height for rectangles).
*   **Sample Point:** A position on the light surface generated via uniform or stratified sampling.

**Steps**
1.  **Sample the Area Light:** Generate a sample point $\mathbf{l}$ on the surface of the area light source. This replaces the single direction vector used for point lights [S1, unofficial].
2.  **Compute Shadow Ray Direction:** Calculate the direction from the shading point $\mathbf{p}$ to the light sample: $\boldsymbol{\omega}_s = \frac{\mathbf{l} - \mathbf{p}}{\|\mathbf{l} - \mathbf{p}\|}$.
3.  **Cast Shadow Ray:** Perform a shadow ray march or trace along $\boldsymbol{\omega}_s$ from $\mathbf{p}$ to $\mathbf{l}$ [S1, unofficial].
4.  **Evaluate Occlusion:** Determine if the ray is blocked by geometry. In a distance field context, this involves checking if the distance to the surface is less than the distance along the ray [S2, unofficial].
5.  **Accumulate:** Repeat steps 1-4 for multiple samples and average the results to estimate the partial occlusion [S1, unofficial].

**Formulas**
The shadow factor $S$ is estimated by averaging $N$ samples:
$$ S = \frac{1}{N} \sum_{i=1}^{N} \text{ShadowTest}(\mathbf{p}, \mathbf{l}_i) $$
where $\text{ShadowTest}$ returns 1 if the path is clear and 0 if occluded [S2, unofficial].

**Parameters**
*   **Number of Samples ($N$):** Not given in the sources. Higher $N$ reduces noise but increases cost [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Aliasing:** Insufficient sampling leads to noisy shadows [S1, unofficial].
*   **Performance:** Shooting more rays per pixel increases computational cost exponentially [S1, unofficial].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Linear with the number of samples $N$ and the cost of the shadow ray traversal.
*   **Memory Complexity:** Low, requiring only storage for the scene representation and sample buffers.
*   **Strengths:** Produces realistic soft shadows with penumbra; works well with dynamic geometry in raster engines if using filtered shadow maps, or with distance fields in raymarching [S2, unofficial].
*   **Limitations:** Computationally expensive due to multiple ray casts; noise can be significant with few samples [S1, unofficial].
*   **When to use:** When realistic lighting with soft shadows is required and performance allows for multiple samples per pixel.

## Background
The concept of soft shadows arises from the physical reality that light sources have spatial extent, unlike the idealized point sources often used in early computer graphics [S2, unofficial]. The method of sampling area lights to approximate these shadows is a fundamental technique in global illumination and realistic rendering [S1, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] (PDF) Raymarching Distance Fields with CUDA - ResearchGate, https://www.researchgate.net/publication/356081826_Raymarching_Distance_Fields_with_CUDA, fetched 2026-10-27 (unofficial)
