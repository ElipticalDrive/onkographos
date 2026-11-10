# Colored Stochastic Shadow Mapping for Volume Rendering
Also called: CSSM for DVR, Colored Stochastic Shadow Mapping (CSSM), Colored Stochastic Shadow Mapping (CSSM) for Direct Volume Rendering, Stochastic Shadow Mapping for Volume Rendering

## Overview
Colored Stochastic Shadow Mapping (CSSM) is a two-step rendering technique for Direct Volume Rendering (DVR) that generates colored shadows using stochastic shadow maps [S1]. It adapts the CSSM concept, originally developed for surface rendering, to volumetric data by modifying probabilistic events to represent photon traversal through a volume with changing opacities [S1]. Unlike traditional shadow maps that use binary occlusion or deep shadow maps that store piecewise linear approximations, CSSM uses stochastic sampling to compute partial, colored shadows for translucent objects while using significantly less texture memory [S1].

## How it works
CSSM operates in two distinct phases: CSSM Generation (building the shadow map) and DVR Rendering (compositing the volume with shadow factors) [S1].

### Inputs and Outputs
**Inputs:**
*   Volumetric dataset $V$ with opacity $\alpha$ and color/transmission properties.
*   Light source position and direction (point light assumed).
*   Camera parameters for DVR.
*   CSSM resolution (e.g., $1024^2$, $2048^2$, $4096^2$) [S1].
*   Number of offset queries $m$ (e.g., 7, 13, 21) [S1].

**Outputs:**
*   CSSM Texture: An RGB texture where each fragment stores three depth values $(c_R, c_G, c_B)$ representing the stochastic shadow depth for each color channel [S1].
*   Final Volume Rendered Image: Computed by DVR with shadow-aware illumination [S1].

### Data Structures
**CSSM Texture:**
*   Format: RGB texture.
*   Layout: Each fragment stores three floating-point depth values: $c_R, c_G, c_B$.
*   Range: Normalized depth $[0, 1]$ [S1].
*   Memory: 3 values per fragment [S1].

### CSSM Generation Steps
This phase builds the shadow map by simulating photon traversal through the volume from the light source [S1].

1.  **Initialize:** For all fragments of the CSSM texture, initialize depth accumulators for RGB channels: $(c_R, c_G, c_B) := (1, 1, 1)$ [S1].
2.  **Ray Casting:** Cast and discretize a ray from the light source through the current fragment into the volume [S1].
3.  **Segment Iteration:** Iterate through each segment $i$ of the discretized ray from $i = 1$ to $n$ [S1].
4.  **Depth Computation:** Compute the depth $z_i$ of segment $i$, normalized to the range $[0, 1]$ [S1].
5.  **Random Threshold:** Generate a random number $r_i$ uniformly distributed in $[0, 1]$ [S1].
6.  **Probability Calculation:** For each wavelength $\lambda \in \{R, G, B\}$, compute the probability $\rho_{i,\lambda}$ that the photon is not transmitted through segment $i$ [S1].
    *   Formula:
        $$p_{i,\lambda} = \alpha_i \cdot (1 - t_{i,\lambda})$$
        Where:
        *   $\alpha_i$ is the opacity of the volume in segment $i$ [S1].
        *   $t_{i,\lambda}$ is the conditional probability that the photon is transmitted given it interacts with the volume in segment $i$ [S1].
7.  **Stochastic Test:** If the computed probability $\rho_{i,\lambda}$ exceeds the random threshold $r_i$, update the depth accumulator for channel $\lambda$:
    $$c_\lambda := \min(c_\lambda, z_i)$$
    This records the closest depth where absorption occurred for that color [S1].
8.  **Early Termination:** Terminate the segment loop early if all color accumulators $(c_R, c_G, c_B)$ are smaller than 1, indicating the photon was fully absorbed or blocked in previous segments [S1].
9.  **Store:** Store the final accumulated depths $(c_R, c_G, c_B)$ in the CSSM fragment [S1].

### DVR Rendering Steps
This phase computes shadow factors for volumetric samples using the generated CSSM [S1].

1.  **Initialize:** For all samples of the DVR approach, initialize shadow factors for RGB channels: $(l_R, l_G, l_B) := (0, 0, 0)$ [S1].
2.  **Light Coordinates:** Compute the sample position $(s_x, s_y, s_z)$ in the light coordinate system [S1].
3.  **Offset Iteration:** Iterate through $m$ offset queries, for $j = 1$ to $m$ [S1].
4.  **CSSM Lookup:** Read the CSSM entries $(c_R, c_G, c_B)$ from the fragment located at $(s_x, s_y) + \Delta_j$, where $\Delta_j$ is the $j$-th offset [S1].
5.  **Shadow Factor Update:** For each $\lambda \in \{R, G, B\}$:
    *   If $c_{\lambda,j} > s_z$, then the sample is in shadow for that color.
    *   Update shadow factor:
        $$l_\lambda := l_\lambda + \frac{1}{m}$$
    *   Formula for final shadow factor:
        $$l_\lambda = \frac{1}{m} \sum_{j=1}^{m} \mathbb{I}(c_{\lambda,j} > s_z)$$
        Where $\mathbb{I}$ is the indicator function [S1].
6.  **Illumination:** Use the shadow factor $(l_R, l_G, l_B)$ to modulate the illumination of the volumetric sample [S1].

### Parameters
*   **$\alpha_i$**: Opacity of the volume in segment $i$. Derived from the volume's transfer function or data values [S1].
*   **$t_{i,\lambda}$**: Conditional probability of transmission for wavelength $\lambda$ in segment $i$. Depends on color properties [S1].
*   **$m$**: Number of offset queries for denoising/averaging. Typical value: 13 [S1].
*   **$\Delta_j$**: Spatial offsets for CSSM lookup. Example set for $m=13$: $\{(0,0), (\pm3,\pm3), (\pm4,0), (0,\pm4), (\pm7,0), (0,\pm7)\}$ [S1].
*   **CSSM Resolution**: Tested resolutions include $1024^2$, $2048^2$, and $4096^2$. Higher resolution improves shadow detail but increases memory [S1].
*   **Number of Offsets**: Typical values are 7 for pleasing results with no noticeable noise, or 21 for softer shadows. Depends on scene complexity [S1].

### Edge Cases and Pitfalls
*   **Noise:** Stochastic generation introduces noise. Mitigated by averaging over multiple CSSM lookups with offsets. Complex scenes may require more offsets to achieve pleasing results without noise [S1].
*   **Blurring:** Increasing the number of offsets ($m$) softens shadows but increases computation time and causes blurring [S1].
*   **Single Light Source:** Limited to a single point light source. Additional light sources require additional shadow maps, which is a general limitation of shadow mapping approaches [S1].
*   **Precision:** Depth values are normalized to $[0, 1]$. Precision issues may arise with very large depth ranges or very thin structures, though not explicitly detailed in sources [G].

## Operating characteristics
*   **Time Complexity:** CSSM generation is proportional to the number of fragments and segments per ray. Rendering time is proportional to the number of volumetric samples and offsets $m$.
*   **Memory Complexity:** CSSM stores 3 values per fragment. Deep Shadow Maps (DSM) with 3 nodes per channel store 12 values per fragment (4x storage). CSSM memory is constant regardless of offset count, while DSM memory scales linearly with nodes [S1].
*   **Measured Performance:** At $1024^2$ resolution, CSSM rendering time is 4.9 ms, higher than Traditional Shadow Mapping (TSM, 3.2 ms) and DSM (3.3 ms) due to increased texture fetches (13 per query for CSSM vs 1 for TSM and 3 for DSM) [S1].
*   **Strengths:** Produces higher-quality colored shadows than DSM (which appears grey); handles translucent objects correctly unlike TSM (which uses a threshold at opacity 0.5); uses less memory than DSM [S1].
*   **Limitations:** Limited to single point light source; stochastic noise requires averaging; blurring with high offset counts [S1].
*   **When to Use:** When colored shadows for translucent volumes are required and memory is constrained compared to DSM. When interactive rates are needed and single light source is sufficient [S1].
*   **When Not to Use:** When multiple light sources are required (without significant memory overhead for multiple maps); when sharp, non-stochastic shadows are preferred; when memory is extremely tight and TSM is sufficient (though TSM lacks color/translucency support) [S1].

## Background
CSSM was originally proposed for surface rendering [S1]. It was adapted for Direct Volume Rendering (DVR) by modifying the probabilistic events to represent photon traversal through a volume with changing opacities [S1]. The method was implemented in the Voreen framework using a GPU-based ray casting approach [S1]. It was tested on synthetic datasets (Spheres) and real-world data (Walnut CT dataset) [S1]. The approach is compatible with mixed scenes containing both volume and surface renderings, allowing CSSM to be generated for surfaces first and then DVR to append to the same buffer [S1].

## Sources
[S1] Colored Stochastic Shadow Mapping for Direct Volume Rendering, https://doi.org/10.2312/eurovisshort.20181072, fetched 2026-10-26
