# Next Event Estimation


## Overview
Next Event Estimation (NEE) is a Monte Carlo rendering technique used to reduce variance by estimating direct illumination via sampling a point on the emitter and testing its visibility by casting a shadow ray [S1]. It differs from standard light sampling by explicitly decoupling the direct light contribution estimation from the path traversal, allowing for targeted sampling of light sources rather than relying solely on the BSDF's natural scattering distribution.

## How it works
**Inputs and Outputs:**
*   **Input:** A surface interaction point $x$, the outgoing direction $\omega_o$, the emitter geometry (or a sampling distribution over emitters), and the medium properties along the path.
*   **Output:** An estimate of the direct illumination contribution $L_{dir}(x, \omega_o)$.

**Data Structures:**
*   **Emitter Sampling Distribution:** A probability density function (PDF) $p_e(y)$ over the surface area of light sources, where $y$ is a point on the emitter.
*   **Shadow Ray State:** A ray origin $x$, direction $\omega = \frac{y-x}{\|y-x\|}$, and distance $t = \|y-x\|$.

**Steps:**
1.  **Sample Light Source:** Generate a sample point $y$ on a light emitter using the emitter sampling distribution $p_e(y)$ [S1].
2.  **Compute Direction:** Calculate the vector from the surface point to the light sample: $\mathbf{d} = y - x$. Compute the distance $t = \|\mathbf{d}\|$ and the normalized direction $\omega = \mathbf{d} / t$ [S1].
3.  **Visibility Test:** Cast a shadow ray from $x$ in direction $\omega$ with length $t$.
    *   If the ray intersects an occluder before distance $t$, the direct illumination contribution is zero.
    *   If the ray reaches the light source (or passes through participating media without occlusion), proceed to step 4.
4.  **Evaluate Radiance:** Calculate the radiance $L_e(y, -\omega)$ emitted by the light at point $y$ in direction $-\omega$ [S1].
5.  **Compute Contribution:** Estimate the direct illumination using the Monte Carlo estimator:
    $$ L_{dir}(x, \omega_o) = \frac{L_e(y, -\omega) \cdot f_r(x, \omega, \omega_o) \cdot \cos(\theta_y)}{p_e(y)} $$
    Where:
    *   $f_r(x, \omega, \omega_o)$ is the Bidirectional Reflectance Distribution Function (BRDF) or Bidirectional Scattering Distribution Function (BSDF) at $x$.
    *   $\theta_y$ is the angle between the surface normal at $y$ and the direction $-\omega$.
    *   $p_e(y)$ is the PDF of sampling point $y$ on the emitter.
    *   Note: If participating media are present, the radiance must be attenuated by the transmittance $T(x, y)$ along the segment [S1].

**Parameters:**
*   **Number of NEE Samples:** Typically one sample per path vertex for direct lighting, though multiple samples can be used for variance reduction [S1].
*   **Emitter Sampling Strategy:** Uniform, proportional to area, or importance-sampled based on luminance [S1].

**Edge Cases and Pitfalls:**
*   **Refractive Boundaries:** NEE is not suitable for participating media with refractive boundaries because standard shadow rays do not account for refraction paths to the light source [S1]. Techniques like Manifold Next Event Estimation (MNEE) are required in such cases [S1].
*   **Geometric Derivatives:** Methods relying on geometric derivatives for manifold exploration do not handle detailed or displaced surfaces well [S1].
*   **Search Failure:** In complex manifolds, the search for a boundary vertex may not succeed, leading to missed connections [S1].

## Subtopics

### [Importance-Sampled Next Event Estimation](7684-importance-sampled-next-event-estimation.md)
Importance-Sampled Next Event Estimation (NEE) is a Monte Carlo technique that reduces variance in direct illumination estimation by decoupling the sample budget from ray casting and using pre-computed importance structures to guide sampling of light paths. It differs from standard NEE by explicitly sampling high-frequency boundary light paths using grid-based or proxy structures rather than uniform emitter sampling, and differs from [Random-Walk Subsurface Scattering](9028-random-walk-subsurface-scattering.md) by focusing on direct boundary interactions rather than volumetric scattering.

### Random-Walk Subsurface Scattering
A subsurface scattering approximation for translucent materials (wax, skin, marble) using a random-walk model inside the mesh geometry [S2, unofficial]. The Disney 'subsurface' lobe entry point initiates the walk inside the mesh [S2, unofficial]. The walk terminates at a white Lambertian vertex [S2, unofficial]. Full Next Event Estimation (NEE) is applied at the exit vertex [S2, unofficial]. Disney Principled BSDF parameters include subsurface weight, radius, scale, and anisotropy [S2, unofficial]. The random walk is not a bounce; it is a distinct scattering event within the material volume [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample for the visibility test, assuming efficient spatial acceleration structures (e.g., BVH) for intersection testing. The cost is dominated by the ray intersection query.
*   **Memory Complexity:** $O(N)$ for storing the scene geometry and emitter data structures, where $N$ is the number of primitives.
*   **Strengths:** Significantly reduces variance for direct illumination compared to pure path tracing with light sampling, especially for small or distant light sources.
*   **Limitations:** Does not handle refractive boundaries in participating media without additional techniques like MNEE [S1]. Performance can degrade with highly complex geometry or displaced surfaces if derivative-based methods are used [S1].
*   **When to Use:** Standard for direct lighting estimation in path tracers.
*   **When Not to Use:** When dealing with participating media surrounded by refractive boundaries, where standard NEE fails [S1].

## Background
Next Event Estimation was introduced as a standard technique in Monte Carlo rendering to improve the efficiency of direct light estimation [S1]. It is widely used in modern path tracers to reduce noise in scenes with direct illumination. The technique is particularly effective when combined with other variance reduction methods such as Multiple Importance Sampling (MIS) [S1].

## Sources
[S1] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
[S2] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
