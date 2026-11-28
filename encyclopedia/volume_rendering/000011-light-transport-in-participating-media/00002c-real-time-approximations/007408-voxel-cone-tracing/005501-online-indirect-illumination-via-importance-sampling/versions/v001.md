# Online Indirect Illumination via Importance Sampling

Also called: Online Indirect Illumination, Importance Sampling Inter-reflection

## Overview
This method computes indirect illumination (inter-reflections) in real-time by importance sampling rays near the reflective direction of the view vector. It addresses the limitations of caching-based global illumination in dynamic scenes, particularly those with directional lighting such as flashlight illumination, by calculating inter-reflections on the fly rather than storing them in static caches. It differs from standard voxel cone tracing by focusing on specular surfaces where the primary source of indirect light originates from the reflection of the view angle around the surface normal, allowing for efficient approximation with fewer samples.

## How it works
**Inputs:**
*   Surface point $x$ with normal $n$.
*   View direction $w_o$.
*   Reflective direction $w_r$ (reflection of $w_o$ around $n$).
*   Scene geometry and lighting (specifically flashlight intensity).
*   Material properties: BRDF $f_r$, roughness.
*   Learnable scalar $\gamma$.

**Outputs:**
*   Indirect illumination component $f_{ir}(x)$.

**Data Structures:**
*   No persistent data structures are required for the sampling process itself; it is an online computation per surface point.
*   The method relies on ray-surface intersection queries to find secondary intersection points.

**Steps:**
1.  **Determine Reflective Direction:** Calculate the reflective direction $w_r$ for the surface point $x$ based on the view direction $w_o$ and surface normal $n$.
2.  **Sample Rays:** Cast multiple rays near $w_r$ to sample the incoming radiance. These rays represent potential paths for inter-reflections.
3.  **Find Secondary Intersection:** For each sampled ray, identify the secondary intersection point $x'$ where the ray hits another surface in the scene.
4.  **Compute Incoming Radiance:** Calculate the incoming radiance $L_{ind}(w_i; x)$ for each sample. This is defined as the outgoing radiance from the secondary intersection point $x'$.
    *   **Lighting Constraint:** The radiance rendering at $x'$ considers only the intense lighting from the flashlight. Points occluded from the flashlight are excluded from this calculation.
5.  **Integrate:** Integrate the incoming radiances over the upper hemisphere around the surface normal $n$ at point $x$.
6.  **Blend with Learnable Scalar:** Apply the learnable scalar $\gamma$ to blend the radiance from the incoming direction $w_i$ around the reflective view to mitigate sampling artifacts.

**Formula:**
The indirect illumination component $f_{ir}(x)$ is computed as:

$$
f_{ir}(x) = \gamma \cdot \sum L_{ind}(w_i; x) f_r(w_i, w_o; x) (w_i \cdot n)
$$

Where:
*   $\gamma$: A learnable scalar blending factor used to mitigate artifacts from insufficient sampling.
*   $L_{ind}(w_i; x)$: The incoming radiance from direction $w_i$ at point $x$.
*   $f_r(w_i, w_o; x)$: The Bidirectional Reflectance Distribution Function (BRDF) at point $x$ for incoming direction $w_i$ and outgoing direction $w_o$.
*   $n$: The surface normal at point $x$.
*   $w_i$: The incoming direction of the sampled ray.

**Parameters:**
*   $\gamma$: Learnable scalar. Typical values are not specified in the sources, but it is optimized to blend radiance. It is deduced from common physical properties: distance, view direction, and roughness, rather than point coordinates or local geometry, to better represent varying inter-reflections in concave areas [S1].

**Edge Cases and Pitfalls:**
*   **Concave Areas:** Relying solely on point position and local geometry for $\gamma$ restricts the representation of varying inter-reflections, especially in concave areas. Using physical properties (distance, view direction, roughness) is recommended [S1].
*   **Occlusion:** The method explicitly excludes points occluded from the flashlight when computing radiance at $x'$, which may lead to inaccuracies if indirect light comes from non-flashlight sources or indirect bounces not captured by the primary flashlight illumination model.
*   **Sampling Artifacts:** Insufficient sampling near $w_r$ can cause artifacts, mitigated by the learnable scalar $\gamma$ [S1].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Depends on the number of samples cast near $w_r$. Since it is an online computation, it scales with the number of surface points and samples per point.
*   **Memory Complexity:** Low, as it does not require storing large radiance caches or voxel structures for indirect illumination.
*   **Strengths:** Efficient for dynamic scenes with directional lighting (e.g., flashlight); captures specular inter-reflections effectively; avoids the computational load of full path tracing or large cache updates.
*   **Limitations:** Approximate; relies on the assumption that indirect light primarily comes from the reflective direction; may miss complex indirect bounces not aligned with $w_r$; accuracy depends on the quality of the $\gamma$ parameterization.
*   **When to Use:** Real-time applications with dynamic lighting and specular surfaces where full global illumination is too expensive.
*   **When Not to Use:** Scenes with diffuse-dominated indirect lighting or complex multi-bounce inter-reflections not aligned with the view reflection.

## Background
This method was proposed to address the issues of missing indirect illumination details and high computational load in dynamic scenes, particularly those captured under directional lighting like a flashlight. It is part of a pipeline for Photometric Inverse Rendering with Shading Cues Modeling and Surface Reflectance Regularization (PIR) [S1]. The approach leverages the observation that specular surfaces exhibit more pronounced inter-reflections, with the main source of indirect illumination coming from the reflective direction of the view angle.

## Sources
[S1] PIR: Photometric Inverse Rendering with Shading Cues Modeling and Surface Reflectance Regularization, https://doi.org/10.1109/3dv66043.2025.00055, fetched 2026-10-26
