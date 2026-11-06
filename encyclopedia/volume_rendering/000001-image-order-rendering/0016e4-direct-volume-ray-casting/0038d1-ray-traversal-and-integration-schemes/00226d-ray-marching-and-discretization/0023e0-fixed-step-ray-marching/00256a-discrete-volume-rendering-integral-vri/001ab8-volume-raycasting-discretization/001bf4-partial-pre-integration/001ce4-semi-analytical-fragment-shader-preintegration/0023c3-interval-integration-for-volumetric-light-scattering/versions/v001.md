# Interval Integration for Volumetric Light Scattering
Also called: Analytic Interval Integration, Sum & Difference of Intervals

## Overview
Interval Integration computes in-scattered radiance by decomposing the ray path into intervals defined by the front and back faces of a light volume, allowing for the analytical evaluation of the scattering integral for directional lights and numerical or lookup-based evaluation for point and spot lights [S1, unofficial]. It sets itself apart from sibling methods by explicitly treating light volume boundaries as integration limits, enabling a "sum and difference" approach where contributions are added for front-face intersections and subtracted for back-face intersections [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** Ray origin, ray direction, medium definition (density, optical depth, phase function), light description (position, direction, falloff), and shadow map data.
*   **Outputs:** In-scattered radiance $L_S$ for the pixel.

**Data Structures**
*   **Light Volume Geometry:** Defined by front faces ($G_f$) and back faces ($G_b$). These define the integration intervals along the ray.
*   **Lookup Texture:** For omnidirectional lights, a texture mapping $\cos^{-1}(v \cdot l)$ to 2D space is used to approximate complex angular dependencies [S1, unofficial].

**Method Steps**
1.  **Interval Decomposition:** Identify the intersection points of the view ray with the light volume. Let $t_n$ be the distance to the $n$-th front face intersection and $t_m$ be the distance to the $m$-th back face intersection [S1, unofficial].
2.  **Integral Formulation:** The total in-scattered radiance is computed as the sum of contributions from front faces minus the sum of contributions from back faces [S1, unofficial]:
    $$ L_S = \sum_{n \in G_f} L'_S(t_n) - \sum_{m \in G_b} L'_S(t_m) $$
    Where $L'_S(t)$ is the indefinite integral of the scattering term [S1, unofficial].
3.  **Light-Type Specific Solvers:**
    *   **Directional Lights:** Assume constant direction and constant power (light distance $\gg$ medium depth). The integral simplifies to an analytic function evaluated in the pixel shader [S1, unofficial]:
        $$ L'_S(d) = L(d, l) \rho_m(x_l, \omega_x) \frac{1 - e^{-\tau_{ex} d}}{\tau_{ex}} $$
        Where $d$ is the distance along the ray, $L$ is light intensity, $\rho_m$ is the medium scattering property, and $\tau_{ex}$ is the extinction coefficient [S1, unofficial].
    *   **Omnidirectional Lights:** Due to complexity, terms are expressed with respect to direction and distance. A lookup texture mapping $\cos^{-1}(v \cdot l)$ is created. The differential is evaluated, and numerical integration is performed, potentially using a Compute Shader [S1, unofficial].
    *   **Spot Lights:** The angular falloff complicates the integral. The interval must be clamped to the cone intersection.
        *   *No Falloff:* Treated as a point light.
        *   *Fixed Falloff:* Use a modified point light LUT.
        *   *Variable Falloff:* Perform cone intersection, then numerically integrate using Newton-Cotes methods in the pixel shader [S1, unofficial].

**Pitfalls and Edge Cases**
*   **Shadow Map Inconsistencies:** If only front faces are rendered to the shadow map, or if distant occluders are excluded, the integration intervals may be incorrect, causing visible artifacts. Ensure shadow map contents are consistent with scene geometry (e.g., render both front and back faces if needed) [S1, unofficial].
*   **Temporal Jitter:** Temporal AA jitter can cause flickering in down-sampled buffers because a 1-pixel flicker at full resolution can become a 4x4 pixel artifact in a 1/4 resolution buffer. A separate TAA resolve must be applied to down-sampled buffers before compositing [S1, unofficial].
*   **High-Frequency Occluders:** Dense occluders at low viewing angles (e.g., sunset through woods) cause performance drops due to high pixel coverage. Adjust tessellation factors based on view angle and pre-filter shadow maps [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of light volume intersections and the complexity of the light solver. Directional lights are $O(1)$ per interval (analytic). Omnidirectional and spot lights require numerical integration or texture lookups, increasing cost [S1, unofficial].
*   **Memory Complexity:** Requires storage for the light volume geometry and, for non-directional lights, lookup textures [S1, unofficial].
*   **Strengths:** Physically plausible results; flexible enough for various engine requirements; compatible with physically-based engines [S1, unofficial].
*   **Limitations:** Performance can degrade with high-frequency occluders or complex spot light falloffs; requires careful handling of shadow maps and temporal filtering to avoid artifacts [S1, unofficial].
*   **When to Use:** When real-time volumetric scattering is required for directional, point, or spot lights in a physically-based rendering pipeline [S1, unofficial].
*   **When Not to Use:** When performance is critical and the scene contains many high-frequency occluders at grazing angles, or when analytical solutions are sufficient for all light types (e.g., simple directional lighting only) [S1, unofficial].

## Background
This method was developed by NVIDIA for use in *Fallout 4*, integrating with their Gameworks volumetric lighting system. It was presented in a technical talk regarding the implementation of fast, flexible, physically-based volumetric light scattering in the game [S1, unofficial]. The approach builds on earlier analytic single scattering models [S1, unofficial].

## Sources
[S1] Volumetric Light Scattering in Fallout 4, https://www.scribd.com/document/748135006/Fast-Flexible-Physically-Based-Volumetric-Light-Scattering, fetched 2026-10-29 (unofficial)
