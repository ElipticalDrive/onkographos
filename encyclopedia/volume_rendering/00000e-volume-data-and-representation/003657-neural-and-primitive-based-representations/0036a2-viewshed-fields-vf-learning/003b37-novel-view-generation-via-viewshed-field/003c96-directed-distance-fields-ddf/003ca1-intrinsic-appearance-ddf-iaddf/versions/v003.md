# Intrinsic Appearance DDF (IADDF)
Also called: DDF Path Tracing, Path Tracing with Intrinsic Appearance DDFs (IADDF), Recursive DDF Light Transport

## Overview
Intrinsic Appearance DDF (IADDF) is a rendering technique that augments a Probabilistic Directed Distance Field (PDDF) with material and lighting models to enable path tracing by recursively querying the DDF for surface intersections [S1]. It solves the problem of disentangling geometry, reflectance, and illumination by treating the DDF as a visibility field that models all possible light paths through a scene, where each path segment corresponds to a DDF call [S1]. Unlike standard neural radiance fields (NeRF) which act as entangled "clouds of particles," IADDF explicitly factors the rendered image into intrinsic appearance and lighting fields, allowing for differentiable rendering of intrinsic images from any viewpoint [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A scene description defined by the IADDF tuple $(G, M, L)$, camera parameters (ray origin and direction), and a maximum bounce depth.
*   **Output:** The final pixel irradiance (radiance) value.

**Data Structures:**
The IADDF is a tuple $(G, M, L)$ consisting of:
1.  **Geometry Field $G$:**
    *   Distance field $d: \mathbb{R}^3 \times S^2 \rightarrow \mathbb{R}^+$
    *   Visibility field $\xi: \mathbb{R}^3 \times S^2 \rightarrow [0, 1]$
2.  **Material Appearance Model $M$:**
    *   BRDF $f_B: \mathbb{R}^3 \times S^2 \times S^2 \rightarrow \mathbb{R}^{|C|+}$
    *   Importance sampler $\Psi: \mathbb{R}^3 \times S^2 \rightarrow P[S^2]$
3.  **Lighting Model $L$:**
    *   Emission field $M_L: \mathbb{R}^3 \times S^2 \rightarrow C$
    *   Environment light $E_L: S^2 \rightarrow C$

Where $C$ denotes the radiance emitted from a position (commonly an RGB colour value) [S1]. $P[S^2]$ is the set of probability distributions on the unit sphere $S^2$ [S1].

**Numbered Steps:**
1.  **Ray Casting:** For each pixel, cast a primary ray into the scene with origin $o$ and direction $\omega_o$.
2.  **Surface Intersection:** Query the DDF to find the scene point $q$ and normal $n$. The DDF provides the distance $d(q, -\omega_o)$ and visibility $\xi(q, -\omega_o)$ for the incoming direction $-\omega_o$ [S1].
3.  **Termination Check:** If $\xi(q, -\omega_o) \le 0.5$ (or effectively 0, indicating no geometry), the ray misses the scene. Return the environment light $E_L(-\omega_o)$ [S1].
4.  **Emission Accumulation:** Add the local emission $M_L(q, -\omega_o)$ to the current path contribution [S1].
5.  **Direction Sampling:** Based on the importance sampler $\Psi$, sample a new outgoing direction $\omega_s$ from $q$. Note that $\Psi$ depends on the local surface normal $n$ at $q$ [S1].
6.  **Recursive Bounce:** Use the DDF to find the next surface intersection in direction $\omega_s$. This corresponds to querying $d(q, -\omega_s)$ and $\xi(q, -\omega_s)$ to find the next point $q'$ along the ray $q + t\omega_s$ [S1].
7.  **Attenuation:** Compute the reflectance contribution using the BRDF $f_B(q, -\omega_o, \omega_s)$ [S1].
8.  **Recursion:** Repeat steps 3-7 for the new ray segment $(q', \omega_s)$, accumulating the result attenuated by the BRDF values of previous bounces.
9.  **Final Irradiance:** The final pixel irradiance is the accumulated value, attenuated by the material BRDF $f_B$ through the bounces, plus any emissions $M_L$ encountered along the way [S1].

**Formulas:**
The incoming radiance $\mathcal{L}_i(q, \omega_i)$ at point $q$ for direction $\omega_i$ is computed recursively as [S1]:

$$
\mathcal{L}_i(q, \omega_i) = \begin{cases} 
\mathcal{L}(q - d(q, -\omega_i)\omega_i, -\omega_i) & \text{if } \xi(q, -\omega_i) > 0.5, \\ 
\mathcal{L}_E(q, -\omega_i) & \text{otherwise}
\end{cases}
$$

Where $\mathcal{L}_E$ represents the environment light contribution [S1].

**Parameters:**
*   $C$: Colour channel set (e.g., RGB) [S1].
*   Typical values for $\xi$ threshold: $0.5$ [S1].

**Pitfalls:**
*   **View-Inconsistency:** The DDF is 5D (position + direction). Depth-rendering the same field from different viewpoints can produce incompatible geometries if the field is not view-consistent [S1]. This is a critical issue when the DDF is learned or approximated, as it may not satisfy the View Consistent (VC) property required for a valid shape representation [S1, Parent].
*   **Precision:** Accurate depths, visibilities, and normals are necessary for visual plausibility, especially for glossy and mirror materials [S1].

## Subtopics

### [Neural Depth Renderer as DDF](9287-neural-depth-renderer-as-ddf.md)
Treats a neural depth renderer as a conditional DDF by mapping camera parameters and scene shapes to depth and visibility images. It differs from standard neural rendering by explicitly framing outputs as functions of ray origins and directions, inheriting DDF properties like the relationship between depth gradients and surface normals. Choose this when you need to leverage existing neural depth estimation architectures within a DDF-based path tracing framework.

## Operating characteristics
*   **Time Complexity:** Depends on the number of bounces and the cost of querying the DDF. Each bounce requires a DDF query for distance and visibility.
*   **Memory Complexity:** Depends on the storage of the DDF fields ($d, \xi$) and the INRs for appearance ($f_B, \Psi, M_L$) and lighting ($E_L$).
*   **Strengths:** Disentangles geometry, reflectance, and illumination; allows for color constancy as a side effect; naturally supports inter-surface light transport (inter-reflections) via recursive DDF calls [S1].
*   **Limitations:** Susceptible to view-inconsistency if the DDF is not properly constrained; requires accurate normals and distances for plausible rendering of glossy/mirror materials [S1].
*   **When to use:** When intrinsic image decomposition is desired, or when integrating DDFs into a path tracing framework for physically based rendering [S1].
*   **When not to use:** When view-consistency is critical and the DDF cannot be guaranteed to be view-consistent, or when computational resources are limited for recursive queries [S1].

## Background
The concept of Intrinsic Appearance DDFs was introduced in the paper "Probabilistic Directed Distance Fields for Ray-Based Shape Representations" [S1]. It builds on the interpretation of DDFs as fields of all possible depth images and extends this to inter-surface rays, modeling light transport via recursive DDF calls [S1]. The work connects DDFs to the classical notion of intrinsic images, which separate a 2D image into depth, orientation, reflectance, and illumination [S1]. The authors demonstrate a proof-of-concept with simplified materials and lighting, using analytic functions for appearance and INRs for geometry [S1].

## Sources
[S1] Probabilistic Directed Distance Fields for Ray-Based Shape Representations, https://doi.org/10.1109/tpami.2025.3594225, fetched 2026-10-29
