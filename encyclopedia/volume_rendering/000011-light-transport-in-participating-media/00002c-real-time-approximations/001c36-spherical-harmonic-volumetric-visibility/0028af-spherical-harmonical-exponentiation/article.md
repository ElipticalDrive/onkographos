# Spherical Harmonical Exponentiation

Also called: Ren et al. Spherical Harmonical Exponentiation, VXAO (Voxel Accelerated Ambient Occlusion)

## Overview
Spherical Harmonical Exponentiation is a real-time technique for encoding volumetric and ambient occlusion visibility using spherical harmonic (SH) basis functions [S1, unofficial]. It solves the problem of efficiently computing directional occlusion for dynamic scenes by approximating geometry as spheres and combining their visibility contributions via logarithmic summation [S1, unofficial]. It differs from [Spherical Harmonic Volumetric Visibility](1225-spherical-harmonic-volumetric-visibility.md) by specifically leveraging the exponential property of SH coefficients to aggregate occlusion from multiple spherical primitives, whereas its sibling [Real-Time Image-Based Volume Lighting](2202-real-time-image-based-volume-lighting.md) relies on image-space approximations [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of spherical primitives approximating the scene geometry; a query point $P$ and direction $\omega$.
*   **Output:** Spherical harmonic coefficients representing the aggregate visibility function at $P$.

**Data Structures:**
*   **SH Coefficients:** A vector of coefficients $c_{l,m}$ for degree $L$ and order $m$.
    *   $l=0, m=0$: Ambient occlusion factor (diffuse visibility).
    *   $l=1, m \in \{-1, 0, 1\}$: Bent normal (directional bias).
    *   $l > 1$: Higher-order environment shadowing details [S1, unofficial].

**Steps:**
1.  **Sphere Visibility Modeling:** For each sphere $i$ occluding the query point, compute the visibility function $V_i(\omega)$. Ren et al. note that the visibility of a point occluded by a single sphere can be modeled analytically using spherical harmonics [S1, unofficial]. Let the SH coefficients for sphere $i$ be $\mathbf{c}_i$.
2.  **Logarithmic Aggregation:** To combine the visibility of $N$ spheres, the method sums the logarithms of the visibility functions. In the SH domain, this corresponds to a specific aggregation of coefficients. The text states: "summing the logarithm of all visibility functions and exponentiating the result" [S1, unofficial].
    *   *Note:* The exact mathematical operation for "summing logarithms" in SH space is not explicitly detailed in the provided source text beyond the conceptual description [S1, unofficial]. Implementation requires deriving the SH representation of $\log(V_i)$ or an equivalent approximation that allows linear summation of transformed coefficients, followed by exponentiation [G].
3.  **Exponentiation:** The aggregate visibility is obtained by exponentiating the summed logarithmic terms, yielding the final SH coefficients $\mathbf{C}_{agg}$ [S1, unofficial].
4.  **Evaluation:** The visibility in direction $\omega$ is evaluated by projecting $\mathbf{C}_{agg}$ onto the SH basis functions $Y_{l,m}(\omega)$:
    $$ V(\omega) = \sum_{l=0}^{L} \sum_{m=-l}^{l} c_{l,m} Y_{l,m}(\omega) $$

**Parameters:**
*   **SH Band Count ($L$):** Determines precision. Higher $L$ provides more precise visibility encoding but increases computational cost [S1, unofficial]. Typical values are not specified in the sources [S1, unofficial].

**Pitfalls:**
*   **Geometric Approximation:** Since geometry is approximated by spheres, occlusion from small details, sharp creases, and non-spherical features is not accounted for [S1, unofficial].
*   **Precision:** The accuracy of the visibility encoding is directly tied to the number of spherical harmonic bands used [S1, unofficial].

## Subtopics

### Spherical Harmonics AO
A specific application of the method where the scene is approximated by spheres to compute ambient occlusion. It models the visibility function of a point occluded by a single sphere using SH, then aggregates these by summing logarithms and exponentiating [S1, unofficial]. First-order coefficients yield ambient occlusion factors, the next three yield bent normals, and higher orders enable environment map shadowing [S1, unofficial]. It is chosen when soft, spherical approximations of geometry are acceptable. It fails to capture occlusion from small details and creases due to the spherical approximation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The method involves per-sphere SH modeling and aggregation, which is likely linear in the number of spheres $N$ and polynomial in the SH degree $L$ [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of spheres and the size of the SH coefficient vector per sphere [S1, unofficial].
*   **Strengths:** Provides directional occlusion data (bent normals) and supports environment map shadowing; efficient aggregation via logarithmic summation [S1, unofficial]. Implemented in Nvidia's VXGI/VXAO package [S1, unofficial].
*   **Limitations:** Loses high-frequency geometric detail (creases, small objects) due to spherical approximation; accuracy depends on SH band count [S1, unofficial].
*   **When to use:** Real-time global illumination where soft shadows and directional occlusion are needed for dynamic scenes with approximated geometry [S1, unofficial].
*   **When not to use:** When high-fidelity geometric details (creases, thin structures) are critical for occlusion [S1, unofficial].

## Background
The method was presented by Ren et al. [49] and implemented by Nvidia in its Voxel Accelerated Global Illumination (VXGI) package as Voxel Accelerated Ambient Occlusion (VXAO) [S1, unofficial]. It is a special case of a full global illumination solution [S1, unofficial]. It is related to other directional occlusion methods like voxel cone tracing (Crassin et al. [7]) and distance mapping (Zhukov et al. [63]), but distinguishes itself by using SH exponentiation for aggregation [S1, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
