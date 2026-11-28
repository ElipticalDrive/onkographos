# Photon Mapping

Also called: Photon Maps, フォトンマッピング

## Overview
Photon mapping is a stochastic method that samples interaction points from a function of the extinction coefficient $\kappa_t$ [S1]. It solves the problem of noisy indirect lighting and caustics by decoupling light transport from image formation, storing photon data in a spatial structure for density estimation [S2, unofficial]. It is distinguished from its siblings, such as [Path Integral Framework](873-path-integral-framework.md) and [Volumetric Radiance Estimators](818-volumetric-radiance-estimators.md), by decoupling light transport from image formation, storing photons for later density estimation, which introduces bias through density estimation parameters [S2, unofficial].

## How it works
**Inputs:** Scene geometry, material properties (albedo, scattering coefficients), light source definitions, and a photon count $N$ [G].
**Outputs:** A photon map data structure and, optionally, a final rendered image [G].

**Data Structures:**
The core data structure is the **Photon Map**, typically implemented as a k-d tree or similar spatial index [G].
*   **Photon Node:** Stores the 3D position $\mathbf{x}$, the incoming radiance direction $\omega_i$, the wavelength or RGB color $\mathbf{c}$, and the throughput weight $w$ [G].
*   **k-d Tree:** A binary space partitioning tree where each node represents a region in space. Internal nodes store a splitting plane (normal and distance); leaf nodes store a list of photons [G].

**Steps:**
1.  **Photon Emission:** For each of the $N$ photons, sample an initial position and direction from the light source distribution [G].
2.  **Photon Tracing:** Trace the photon through the scene.
    *   Sample the distance to the next interaction from a distribution governed by the extinction coefficient $\kappa_t$ [S1].
    *   If the photon hits a surface, apply the material's scattering phase function and albedo to determine the new direction and throughput [G].
    *   Store the photon's position, direction, and throughput in the photon map at the interaction point [G].
    *   Repeat until the photon is absorbed or escapes the scene [G].
3.  **Density Estimation (Rendering Pass):** For each pixel, cast a ray and collect photons within a search radius $r$ around the intersection point [G].
    *   Estimate the radiance $L$ using a kernel density estimator:
        $$ L(\mathbf{x}, \omega_o) = \int_{\Omega} f_r(\mathbf{x}, \omega_i, \omega_o) \hat{L}_i(\mathbf{x}, \omega_i) d\omega_i $$
        where $\hat{L}_i$ is the estimated incident radiance derived from the photon density [G].
    *   The incident radiance is approximated by summing the contributions of $k$ nearest photons:
        $$ \hat{L}_i(\mathbf{x}, \omega_i) \approx \frac{1}{\pi r^2} \sum_{j=1}^{k} \frac{w_j}{\kappa_t(\mathbf{x}_j)} $$
        [G: This formula assumes a flat kernel and uniform distribution within the radius $r$. The term $\kappa_t(\mathbf{x}_j)$ accounts for the probability of the photon reaching $\mathbf{x}$ from $\mathbf{x}_j$.]

**Parameters:**
*   $N$: Total number of photons. Not given in the sources.
*   $k$: Number of nearest neighbors for density estimation. Typical values are 50–100 [G].
*   $r$: Search radius. Often adaptive based on $k$ and local photon density [G].

**Edge Cases and Pitfalls:**
*   **Bias:** The method is biased because the density estimation depends on arbitrary parameters $k$ and $r$ [S2, unofficial].
*   **Artifacts:** Uneven photon distribution can cause visible artifacts such as noise or blotchiness [S2, unofficial].
*   **Memory:** Storing all photons requires significant memory for large scenes [G].

## Subtopics

### Caustic Photon Mapping
Stores photons that have undergone specular interactions to efficiently render caustics. Chosen when specular highlights and caustics are prominent.

### Diffuse Photon Mapping
Stores photons that have undergone diffuse interactions for indirect lighting. Chosen for general global illumination.

## Operating characteristics
*   **Time Complexity:** $O(N \log N)$ for building the k-d tree and $O(\log N)$ for querying nearest neighbors [G].
*   **Memory Complexity:** $O(N)$ to store all photons [G].
*   **Strengths:** Efficiently handles caustics and complex indirect lighting with less noise than pure path tracing for the same computational budget [S2, unofficial].
*   **Limitations:** Biased result due to density estimation parameters; artifacts from uneven photon distribution [S2, unofficial]. High memory usage and preprocessing time [G].
*   **When to use:** Offline rendering where high-quality caustics and indirect lighting are required, and noise is unacceptable.
*   **When not to use:** Real-time rendering due to memory and preprocessing time; scenes requiring strict physical accuracy without bias.

## Background
Photon mapping was developed in 1995 by Henrik Wann Jensen, a Danish researcher [S2, unofficial]. It was created to address the limitations of ray tracing in efficiently handling indirect lighting and caustics [S2, unofficial]. The method gained significant importance in global illumination research during the 2000s and was adopted by major renderers such as Maya Mental Ray [S2, unofficial].

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S2] 3Dレンダリング方式についてchatGPTに聞いてみた｜ほねぷー, https://note.com/honepoo/n/nc610518091f7, fetched 2026-10-30 (unofficial)
