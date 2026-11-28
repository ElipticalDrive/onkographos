# Accurate single scattering under point light (Walter et al.)
Also called: Fermat's Principle based Single Scattering, Walter's Method for Accurate Single Scattering

## Overview
This method computes single scattering effects in participating media under a point light source by treating the computation as a deterministic search problem to find all points on the surface that connect the point light and the camera sample via Fermat’s principle [S1]. It is distinct from its siblings because it relies on interval Newton optimization to find all valid refraction points, whereas [Sidedness Test](4237-sidedness-test.md) uses geometric predicates and [Newton-Raphson for Refraction Point Search](4659-newton-raphson-for-refraction-point-search.md) uses standard iterative refinement without guaranteed global convergence. It solves the problem of accurate caustic rendering in media, which is impossible to render with volumetric path tracing or MEMLT [S1].

## How it works
**Inputs:**
*   Camera ray origin $O_c$ and direction $D_c$.
*   Point light position $L$.
*   Participating medium properties (extinction coefficient $\sigma_t$, phase function $p$).
*   Surface geometry (implicit or explicit function $S(x)=0$) and shading normal field $N_s(x)$.
*   Refractive indices $n_{in}$ (outside) and $n_{out}$ (inside).

**Outputs:**
*   Radiance contribution at the camera sample point $V$ along the ray.

**Data Structures:**
*   Interval Newton Solver state: current interval $[a, b]$ on the surface parameterization or ray parameter, function value $f$, derivative $f'$.
*   List of found entry points $P_i$.

**Steps:**
1.  **Formulate Fermat's Principle:** The path from light $L$ to surface point $P$ to volume sample $V$ must satisfy Snell's law at $P$. The optical path length is stationary. This defines a residual function $f(P)$ based on the misalignment of the refracted ray with the vector $PV$ [S1].
2.  **Interval Newton Search:** Use an interval Newton method to find all roots of $f(P)=0$ within the domain of the surface. This guarantees finding all entry points $P_i$ that connect $L$ and $V$ via refraction [S1].
3.  **Validate Points:** For each found point $P_i$, verify that the segment $L \to P_i$ and $P_i \to V$ are unobstructed by the surface (visibility test) and that the refraction is physically valid (no total internal reflection) [S1].
4.  **Compute Contribution:** For each valid $P_i$, compute the single scattering contribution:
    $$ L_{single} = \frac{1}{r_{LP}^2} T(L, P_i) T(P_i, V) \sigma_s p(\omega_i, \omega_o) $$
    where $r_{LP}$ is distance $L \to P_i$, $T$ is transmittance, $\sigma_s$ is scattering coefficient, and $p$ is the phase function [S1].
5.  **Accumulate:** Sum contributions from all valid $P_i$.

**Pitfalls:**
*   The presence of a shading normal at surface points makes the problem more complex than using geometric normals [S1].
*   To capture the sharpness of the caustics, a large number of camera samples are required along the camera ray [S1].
*   The method is computationally expensive due to the global search required for all solutions [S1].

## Subtopics

### [Distance Correction Factor for Refracted Paths](7604-distance-correction-factor-for-refracted-paths.md)
The Distance Correction Factor ($D$) accounts for geometric spreading and refraction effects in the contribution formula for light paths involving refractive boundaries in participating media. It ensures that the computed radiance matches reference solutions by correctly scaling the intensity based on the solid angle subtended the light source as seen from the scattering point, adjusted for the refractive index. It is distinct from simple inverse-square distance laws because it incorporates the Jacobian of the refraction mapping at the boundary.

## Operating characteristics
*   **Time Complexity:** High. Requires solving a non-linear system for every camera sample to find all roots. The interval Newton method ensures robustness but is slower than local methods.
*   **Memory Complexity:** Low. Only requires storing the list of found entry points per sample.
*   **Strengths:** Deterministic and accurate for single scattering under point lights. Captures sharp caustics correctly.
*   **Limitations:** Cannot handle multiple scattering. Requires many camera samples to avoid aliasing in caustics. Slower than Holzschuch's method [S1].
*   **When to use:** When accurate single scattering caustics are required under point lights in participating media.
*   **When not to use:** For multiple scattering, area lights, or when performance is critical and approximate caustics are acceptable.

## Background
Walter et al. [60] introduced this method for accurate computation of single scattering effects in participating media under a point light source. This configuration is impossible to render with volumetric path tracing or MEMLT [S1]. They treated the single scattering computation as a search problem: find all points on the surface that connect the point light and the camera sample via Fermat’s principle. The presence of a shading normal at surface points makes the problem more complex (see Fig. 14 in [S1]). They computed these entry points on the surface using Newton–Raphson optimization. Using an interval Newton method, all solutions can be found. However, to capture the sharpness of the caustics, a large number of camera samples are required along the camera ray. Wang et al. [62] can also be used for single scattering computation. It produces similar results to those of Walter et al. [60]. Holzschuch [61] improved it by computing the extent of the influence of each triangle on the camera ray, based on the observation that the radiance caused by an individual triangle on the surface varies smoothly and discontinuities correspond to triangle edges. This method is significantly faster than Walter et al. [60] while providing higher-quality results (see Fig. 15 in [S1]). Sun et al. [63] proposed an analytical solution for single scattering by p… [S1].

## Sources
[S1] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
