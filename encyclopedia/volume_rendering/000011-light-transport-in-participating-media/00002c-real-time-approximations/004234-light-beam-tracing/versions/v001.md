# Light Beam Tracing

Also called: Beam Tracing

## Overview
Light beam tracing is a real-time approximation method for light transport in participating media that aggregates rays into geometric beams to exploit spatial coherence, avoiding the explicit tracing of individual rays until a final sampling step [S2]. Unlike sibling methods such as [Voxel Cone Tracing](1454-voxel-cone-tracing.md) or [Froxel-based Volumetric Rendering](1919-froxel-based-volumetric-rendering.md) which maintain explicit volumetric data structures for scattering, light beam tracing relies on geometric primitives to approximate light shafts, making it distinct in its lack of intermediate volumetric caching for multiple scattering orders [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Scene geometry (typically triangle meshes or implicit surfaces), participating media density field, and light source positions.
*   **Output:** Rendered image with volumetric light shafts (god rays).

**Data Structures:**
The method does not use a fixed voxel grid for the beam representation itself. Instead, it relies on a spatial acceleration structure, typically a kd-tree, to manage the intersection of beam volumes with scene geometry [S2]. The beam itself is defined by a central ray and a solid angle or cross-sectional area that expands as it traverses the scene.

**Steps:**
1.  **Beam Generation:** For each pixel or screen-space block, generate a primary eye ray. Instead of tracing a single ray, define a beam volume originating from the camera frustum.
2.  **Beam Traversal:** Intersect the beam volume with the scene geometry (stored in a kd-tree) to determine the entry and exit points of the participating medium [S2].
3.  **Sampling:** At the final step, sample points along the beam's path within the medium to compute in-scattered radiance. This step aggregates the contribution of the entire beam cross-section [S2].
4.  **Accumulation:** Sum the surface radiance and the accumulated in-scattered radiance along the ray path to determine the final pixel color [S1].

**Formulas:**
The radiance $L(x_c \leftarrow \omega_c)$ at the camera is computed as:
$$ L(x_c \leftarrow \omega_c) = L(x_s \leftarrow \omega_c) + L_i(x_{tc} \leftarrow \omega_c) $$
Where:
*   $L(x_s \leftarrow \omega_c)$ is the surface radiance.
*   $L_i(x_{tc} \leftarrow \omega_c)$ is the accumulated in-scattered radiance along the ray [S1].

**Parameters:**
*   Beam expansion rate: Not given in the sources.
*   Sampling density: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Area Lights:** The method cannot easily handle area light sources, limiting its applicability to point or directional lights [S1].
*   **Artifacts:** The approximation nature of beam aggregation can lead to blurring or approximation artifacts, particularly in complex media [S1].
*   **Data Structure Limitation:** Existing implementations focus on kd-tree based structures; there are no known implementations of grid-based beam tracers, which limits integration with voxel-centric pipelines [S2].

## Subtopics

### Beam Tracing
A strategy where rays are considered only in a final sampling step, rather than being represented explicitly during traversal [S2]. It is competitive with frustum-based tracers for eye and shadow rays [S2]. However, it is limited by the lack of grid-based implementations, as existing work relies on kd-tree data structures [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the traversal algorithm's setup time ($T_{setup}$) and step time ($T_{step}$) [S2]. Performance is conflictual: low $T_{setup}$ favors low-resolution grids, while high $T_{step}$ degrades performance for large grids [S2].
*   **Memory Complexity:** Requires storage for the acceleration structure (e.g., kd-tree) [S2].
*   **Strengths:** Exploits ray coherence efficiently; competitive with frustum-based methods for primary and shadow rays [S2].
*   **Limitations:** Cannot handle area lights; suffers from blurring artifacts; no grid-based implementations exist [S1, S2].
*   **When to use:** When rendering simple light shafts in scenes with complex geometry where kd-tree traversal is efficient.
*   **When not to use:** When area lights are required or when grid-based integration is necessary.

## Background
Originating from the broader field of ray tracing optimization, beam tracing was explored by Overbeck et al. [ORM07] as a competitive alternative to frustum-based tracers. The survey by [S2] notes that while ray aggregation is common in interactive systems, beam tracing remains less prevalent due to its reliance on non-grid data structures. The specific application to volumetric radiance estimation using photon beams was further detailed by [S1], highlighting its ability to produce light shafts within complex refractive boundaries.

## Sources
[S1] A comprehensive theory of volumetric radiance estimation using photon points and beams, https://doi.org/10.1145/1899404.1899409, fetched 2026-10-26
[S2] A Survey on Exploiting Grids for Ray Tracing, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/089-096, fetched 2026-10-24
