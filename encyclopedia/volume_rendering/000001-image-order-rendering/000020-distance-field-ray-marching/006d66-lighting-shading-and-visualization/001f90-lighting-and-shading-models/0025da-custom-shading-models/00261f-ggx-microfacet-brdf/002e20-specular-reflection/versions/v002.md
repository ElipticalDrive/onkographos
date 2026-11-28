# Specular Reflection

Also called: Perfect mirror reflection

## Overview
Specular reflection models perfectly smooth surfaces where outgoing radiance is not scattered but mirrors the incident trajectory across the surface normal [S1, unofficial]. The method concentrates all reflected energy into a single geometric direction, requiring exact alignment between the incident light direction and the view vector relative to the surface normal [G].

## How it works
**Inputs and Outputs:** Incident radiance $L_i$, incident direction $\vec{\omega}_i$, reflected direction $\vec{\omega}_r$, and surface normal $\vec{n}$ are provided as inputs [S1, unofficial]. The method outputs the reflected radiance $L_r$ [S1, unofficial].

**Data Structures:** Not given in the sources. Standard vector and spherical coordinate representations are sufficient [G].

**Steps:**
1. Compute the geometric reflection direction $\vec{\omega}_{refl}$ for the view direction $\vec{\omega}_r$ using the vector reflection formula $\vec{\omega}_{refl} = 2(\vec{\omega}_r \cdot \vec{n})\vec{n} - \vec{\omega}_r$ [G].
2. Convert $\vec{\omega}_i$ and $\vec{\omega}_{refl}$ to spherical coordinates $(\theta_i, \phi_i)$ and $(\theta_r, \phi_r)$ relative to the surface normal $\vec{n}$ [G].
3. Evaluate the angular conditions $\theta_r = \theta_i$ and $\phi_r = \phi_i \pm \pi$ [S1, unofficial].
4. If both conditions hold, assign $L_r(\theta_r, \phi_r) = L_i(\theta_r, \phi_i \pm \pi)$; otherwise, assign $L_r = 0$ [S1, unofficial].

**Formula:**
The Bidirectional Reflectance Distribution Function (BRDF) is defined using the Dirac delta function $\delta$:
$$ f_{r,m} = \frac{\delta(\cos \theta_i - \cos \theta_r)}{\cos \theta_i} \delta(\phi_i - (\phi_r \pm \pi)) $$
where:
*   $\delta$ is the Dirac delta function [S1, unofficial].
*   $\theta_i, \theta_r$ are the incident and reflected polar angles [S1, unofficial].
*   $\phi_i, \phi_r$ are the incident and reflected azimuthal angles [S1, unofficial].

**Parameters:**
*   Roughness: $0$ [S2].
*   Typical values for other parameters are not given in the sources.

**Edge Cases and Pitfalls:**
*   **Delta Function Zeroes:** The Dirac delta function yields zero for any non-zero parameter mismatch, meaning no light is reflected if the geometric reflection conditions are not perfectly satisfied [S1, unofficial].
*   **Numerical Precision:** Floating-point comparisons for angle equality ($\theta_r = \theta_i$) are prone to error; implementers must use a small epsilon tolerance or vector dot-product checks instead of direct angle equality [G].
*   **Discrete Sampling:** In rasterization or discrete ray tracing, finding the exact incident direction is difficult; this model is best suited for analytical ray tracing or environment map lookups rather than light sampling in path tracers without importance sampling techniques [G].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per pixel if the incident radiance is available via direct ray intersection or environment map lookup [G].
*   **Memory Complexity:** $O(1)$ additional memory [G].
*   **Strengths:** Computationally cheap per sample and physically accurate for polished metals and mirrors [G].
*   **Limitations:** Cannot model rough surfaces, requires exact geometric alignment, and is difficult to sample stochastically without specialized techniques [G].
*   **When to use:** Rendering perfect mirrors, polished chrome, or the zero-roughness limit of microfacet models [G].
*   **When not to use:** For rough surfaces, diffuse materials, or when approximate lighting is acceptable [G].

## Background
The mathematical formulation using Dirac delta functions for specular reflection originates from radiometry and computer graphics theory, specifically detailed in works on real-time global illumination [S1, unofficial]. The concept of "perfect mirror reflection" with zero roughness is also implemented in voxel-based rendering tools like MagicaVoxel, which added a perfect mirror reflection material with support for over eight light bounces in version 0.96.2 [S2].

## Sources
[S1] Real-Time Global Illumination Using OpenGL And Voxel Cone Tracing, https://doi.org/10.48550/arxiv.2104.00618, fetched 2026-10-26 (unofficial)
[S2] MagicaVoxel .vox, https://ephtracy.github.io/, fetched 2026-10-26
