# Sky Light Estimation with MIS

Also called: MIS for Environment Sky Light

## Overview
Sky Light Estimation with MIS uses an environment map to simulate lighting from the outer environment (sky, sunlight) which is hard to model as solid objects, avoiding the task of modeling all objects from the border of the bounding environment to the shading point and accumulating indirect lighting between any two objects [S1, unofficial]. It applies Multiple Importance Sampling (MIS) for both light and phase functions to handle the continuous nature of the environment map, distinguishing it from delta-light sampling in [Direct Lighting and Shading](3953-direct-lighting-and-shading.md).

## How it works
**Inputs and Outputs:**
*   **Input:** Shading point position, outgoing ray direction, environment map (spherical projection map), phase function parameters.
*   **Output:** Radiance contribution from the sky, sampled direction, and associated PDFs for MIS weighting.

**Data Structures:**
*   **Environment Map:** Stored as a spherical projection map (cubemaps are discouraged due to artifacts) [S1, unofficial].
*   **Spherical Coordinates:** $\theta$ (polar angle) and $\phi$ (azimuthal angle) derived from the cartesian direction.
*   **Grid Indices:** Row and column values calculated from $\theta$ and $\phi$ to index the map.

**Steps:**
1.  **Ray Intersection Check:** When shading a point, cast a ray in the reflected direction. If nothing in the local setting (models of interest) is hit, proceed to sample the environment map [S1, unofficial].
2.  **Coordinate Conversion:** Convert the cartesian direction of the ray to spherical coordinates $(\theta, \phi)$ [S2, unofficial].
3.  **Grid Indexing:** Calculate the row and column values from the spherical coordinates to determine the texel location in the spherical projection map [S2, unofficial].
4.  **PDF Calculation:** Calculate the Probability Density Function (PDF) for the sampled direction. This requires a specific method described in "Sampling A Procedural Sky" [S2, unofficial]. The PDF is derived based on the direction using the calculated row and column values [S2, unofficial].
5.  **MIS Weighting:** Apply MIS for both the light sampling distribution and the phase function distribution. The light PDF is often treated as uniform ($1.0$) for distant lights, but for environment maps, the directional PDF is critical [S2, unofficial].
6.  **Radiance Accumulation:** Sample the texel value from the environment map at the calculated indices. Multiply by the phase function value, transmittance, and MIS weights to get the final contribution [S2, unofficial].

**Formulas:**
*   **Spherical Conversion:**
    $$ \theta = \arccos(z) $$
    $$ \phi = \arctan2(y, x) $$
    Where $(x, y, z)$ is the normalized cartesian direction vector.
*   **Grid Indices:**
    $$ row = \text{floor}\left(\frac{\theta}{\pi} \times H\right) $$
    $$ col = \text{floor}\left(\frac{\phi + \pi}{2\pi} \times W\right) $$
    Where $H$ is the height and $W$ is the width of the spherical projection map. (Note: Exact mapping may vary by implementation, but the principle of mapping angles to grid indices is required [S2, unofficial]).
*   **MIS Weight:**
    $$ w = \frac{p_{light}(x) p_{phase}(x)}{p_{light}(x) + p_{phase}(x)} $$
    Where $p_{light}$ is the PDF of the sampling strategy for the environment direction, and $p_{phase}$ is the PDF of the phase function (e.g., Henyey-Greenstein) for the scattering direction [S2, unofficial].

**Parameters:**
*   **Environment Map Format:** Spherical projection map is preferred over cubemap for easier computation and fewer artifacts [S1, unofficial].
*   **Phase Function:** Henyey-Greenstein or Double-Henyey-Greenstein, with anisotropy parameter $g$ ($-1.0 < g < 1.0$) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Cubemap Artifacts:** Cubemaps have a higher chance of artifacts compared to spherical projection maps [S1, unofficial].
*   **PDF Calculation:** Calculating the PDF for environment sky light is more involved than for delta lights because it involves multiple directions and requires MIS for both light and phase functions [S2, unofficial].
*   **Boundary Conditions:** Ensure $\phi$ is wrapped correctly to $[-\pi, \pi]$ or $[0, 2\pi]$ to avoid indexing errors at the seam of the spherical projection map.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample for environment map lookup and PDF calculation, assuming constant-time spherical-to-grid conversion.
*   **Memory Complexity:** $O(W \times H)$ for the environment map storage.
*   **Strengths:** Efficiently simulates complex indirect lighting from the environment without modeling distant geometry [S1, unofficial]. Spherical projection maps reduce artifacts and simplify computation compared to cubemaps [S1, unofficial].
*   **Limitations:** Requires accurate PDF calculation for the environment map sampling strategy to ensure unbiased results with MIS [S2, unofficial]. Performance depends on the complexity of the phase function PDF calculation.

## Background
The use of environment maps for image-based lighting was introduced to handle lighting from the outer environment like sky and sunlight, which are hard to model as solid objects [S1, unofficial]. The application of Multiple Importance Sampling (MIS) to environment sky light in volumetric rendering was implemented based on pbrt, requiring a new function to draw a PDF value based on the techniques provided in "Sampling A Procedural Sky" [S2, unofficial]. The phase functions used, such as Henyey-Greenstein, were first introduced in 1941 to estimate light direction in participating media [S2, unofficial].

## Sources
[S1] GAPT Series - 10] Rendering Effects - Daqi's Blog, https://dqlin.xyz/tech/2017/06/30/10_fx/, fetched 2026-10-29 (unofficial)
[S2] Lighting And Transmittance, https://sergeneren.com/2019/04/26/lighting-and-transmittance/, fetched 2026-10-30 (unofficial)
