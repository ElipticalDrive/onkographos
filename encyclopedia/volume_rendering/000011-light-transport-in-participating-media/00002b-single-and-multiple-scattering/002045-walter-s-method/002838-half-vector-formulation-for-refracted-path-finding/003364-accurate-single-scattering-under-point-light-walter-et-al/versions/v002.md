# Accurate single scattering under point light (Walter et al.)
Also called: Fermat's Principle based Single Scattering, Walter's Method for Accurate Single Scattering

## Overview
This method computes single scattering effects in participating media under a point light source by treating the computation as a search problem to find all points on the surface that connect the point light and the camera sample via Fermat’s principle [S1]. It is impossible to render this configuration with volumetric path tracing or MEMLT [S1].

## How it works
**Inputs:** Point light source, camera sample, surface geometry, shading normal field [S1].
**Outputs:** Single scattering radiance contribution at the camera sample [S1].
**Data Structures:** Interval Newton solver state tracking the current interval, function value, and derivative [S1]. List of found entry points on the surface [S1].
**Steps:**
1. Formulate the search problem to find all surface points $P$ connecting the point light and camera sample via Fermat’s principle [S1].
2. Compute entry points on the surface using Newton–Raphson optimization within an interval Newton method to find all solutions [S1].
3. Accumulate single scattering contributions from all valid entry points [S1].
**Formulas:** Not given in the sources.
**Parameters:** Not given in the sources.
**Edge cases and pitfalls:**
*   The presence of a shading normal at surface points makes the problem more complex than using geometric normals [S1].
*   To capture the sharpness of the caustics, a large number of camera samples are required along the camera ray [S1].

## Subtopics

### [Distance Correction Factor for Refracted Paths](7604-distance-correction-factor-for-refracted-paths.md)
The Distance Correction Factor ($D$) accounts for geometric spreading and refraction effects in the contribution formula for light paths involving refractive boundaries in participating media. It ensures that the computed radiance matches reference solutions by correctly scaling the intensity based on the solid angle subtended by the light source as seen from the scattering point, adjusted for the refractive index. It is distinct from simple inverse-square distance laws because it incorporates the Jacobian of the refraction mapping at the boundary.

### Accurate single scattering optimization (Holzschuch)
Holzschuch [61] improved Walter's method by computing the extent of the influence of each triangle on the camera ray, based on the observation that the radiance caused by an individual triangle on the surface varies smoothly and discontinuities correspond to triangle edges [S1]. This method is significantly faster than Walter et al. [60] while providing higher-quality results [S1].

## Operating characteristics
*   **Time Complexity:** Highly depends on scene complexity [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Deterministic and accurate for single scattering under point lights [S1].
*   **Limitations:** Limited to participating media under point lights [S1]. Requires many camera samples to avoid aliasing in caustics [S1]. Slower than Holzschuch's method [S1].
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
Walter et al. [60] introduced this method for accurate computation of single scattering effects in participating media under a point light source [S1]. This configuration is impossible to render with volumetric path tracing or MEMLT [S1]. They treated the single scattering computation as a search problem: find all points on the surface that connect the point light and the camera sample via Fermat’s principle [S1]. The presence of a shading normal at surface points makes the problem more complex [S1]. They computed these entry points on the surface using Newton–Raphson optimization [S1]. Using an interval Newton method, all solutions can be found [S1]. However, to capture the sharpness of the caustics, a large number of camera samples are required along the camera ray [S1]. Wang et al. [62] can also be used for single scattering computation and produces similar results to those of Walter et al. [60] [S1]. Holzschuch [61] improved it by computing the extent of the influence of each triangle on the camera ray, based on the observation that the radiance caused by an individual triangle on the surface varies smoothly and discontinuities correspond to triangle edges [S1]. This method is significantly faster than Walter et al. [60] while providing higher-quality results [S1]. Sun et al. [63] proposed an analytical solution for single scattering by participating media without boundaries under a point light, which can achieve a real-time frame rate [S1]. All of the above methods can compute accurate single scattering without noise, but the computation time highly depends on scene complexity [S1]. Moreover, these methods are limited to participating media under point lights [S1].

## Sources
[S1] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
