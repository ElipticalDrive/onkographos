# Prefiltering

Also called: Adaptive Prefiltering, Anisotropy Prefiltering

## Overview
Prefiltering is a technique used in discrete ordinates rendering to remove singularity-like artifacts and avoid discretization errors, particularly in participating media with insufficient optical thickness to naturally blur sampled illumination [S1]. Unlike naive prefiltering of source illumination which blurs background features, this method is adaptive, allowing the perception of background illumination features while smoothing the initial scattering distribution [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An environment map or local light source distribution (e.g., VPLs) and a set of discrete ordinates (directions) for the radiative transfer equation [S1].
*   **Output:** A prefiltered initialization of the radiance or scattering coefficients for each ordinate, ready for ray marching [S1].

**Data Structures:**
*   **Ordinates:** A set of $N$ directions. For environment maps, these are often arranged in radial grids or spherical grids [S1]. For local lights, spherical grids with angular and radial coordinates aligned with the point source are used [S1].
*   **Importance Weights ($w_n$):** A scalar value for each ordinate $n$, representing its contribution to the image. This is mapped from the solid angle proportion of the direction [S1].

**Algorithm Steps:**
1.  **Calculate Importance:** Determine the importance $w_n$ for each ordinate $n$. The importance is proportional to the solid angle $\Omega_n$ subtended by the ordinate's cell in the directional domain [S1].
2.  **Map Importance to Anisotropy:** Compute the initial anisotropy $a_{n,i}$ for each ordinate $i$ using the heuristic formula:
    $$ a_{n,i} = \left(1 - \frac{w_n}{N}\right)^\beta $$
    Where:
    *   $N$ is the total number of ordinates [S1].
    *   $w_n$ is the importance of ordinate $n$ (mapped from solid angle proportion) [S1].
    *   $\beta > 0$ is a scalar factor defining proportionality [S1].
    *   Important ordinates (high $w_n$) are denser in the directional domain, have smaller solid angles, and result in higher initial anisotropy. Less important ordinates are sparser, have larger solid angles, and lower anisotropy [S1].
3.  **Clamp Singularities:** Apply clamping to the resulting values to reduce any remaining singularities [S1].
4.  **Propagation (Optional/Contextual):** In the context of Principal-Ordinates Propagation (POP), this prefiltered initialization is used before sampling the medium. For local lights, VPLs with large importance have high initial anisotropy, and vice versa [S1].

**Parameters:**
*   $\beta$: Scalar factor defining the proportionality in the anisotropy mapping heuristic. Value: Tuned empirically once for each environment map [S1].

**Edge Cases and Pitfalls:**
*   **Insufficient Optical Thickness:** Without prefiltering, singularity-like artifacts appear in thin, strongly-scattering media [S1].
*   **Grid Merging:** When combining grids from different light sources, geometric misalignments may occur, requiring careful merging strategies [S1].

## Subtopics

### Quasi-Interpolation Prefilter
A prefilter applied to reduce aliasing artifacts, specifically preventing 'valleys' of rims from becoming shallower due to oversmoothing [S2]. It addresses reconstruction quality issues in isosurface raycasting [S2].

## Operating characteristics
*   **Performance:** The prefiltering step itself is described as a "simple and quick task" [S1]. In the POP pipeline, prefiltering initialization is part of a sequence that includes propagation (e.g., 2 ms for residuum propagation) and ray marching (e.g., 5 ms) [S1].
*   **Strengths:** Removes singularity artifacts in thin media; preserves background illumination features due to adaptivity; handles high anisotropy (e.g., $g=0.98$) effectively [S1].
*   **Limitations:** Requires empirical tuning of $\beta$ for each environment map [S1].

## Background
Originated in the context of real-time rendering of participating media with high scattering anisotropy. The method was proposed to handle the shortcomings of current methods in handling highly anisotropic scattering, which is common in natural and artificial media [S1].

## Sources
[S1] Principal-Ordinates Propagation for Real-Time Rendering ..., https://cg.ivd.kit.edu/publications/2015/principal-ordinates/ElekCAG2014.pdf, fetched 2026-10-29
[S2] PDF GPU Isosurface Raycasting of FCC Datasets, https://minho-kim-uos.github.io/papers/kim12gpu.pdf, fetched 2026-10-29
