# Photon Beam Tracing

Also called: Photon beams

## Overview
Photon Beam Tracing is a volumetric radiance estimation method where photon paths are stored as beam segments rather than discrete points, allowing for density estimation via beam intersection [S1]. It differs from standard volumetric photon tracing by storing the photon energy up to the point of intersection with solid geometry, rather than discarding the photon upon collision [S1]. This approach provides a superset of the information contained in a standard photon map, enabling more efficient handling of boundaries between participating media and opaque surfaces [S1].

## How it works
**Inputs:** Light source positions and emission properties; scene geometry (solid objects); participating media properties (absorption and scattering coefficients); total number of photons to shoot.
**Outputs:** A photon beams map containing stored beam segments and their associated radiance/energy data.

**Data Structures:**
The photon beams map stores segments defined by:
1.  **Start Point ($P_{start}$):** A 3D vector. For initial beams, this is the light source position. For subsequent bounces, this is the previous interaction point.
2.  **End Point ($P_{end}$):** A 3D vector. This is either the next interaction point within the volume or the intersection point with a solid object.
3.  **Direction ($\vec{d}$):** A normalized 3D vector from $P_{start}$ to $P_{end}$.
4.  **Radiance ($L$):** The radiance carried by the beam.
5.  **Beam Radius ($r$):** The width of the beam used for density estimation (see Adaptive Beam Width).

**Steps:**
1.  **Photon Emission:** Shoot photon paths from light sources into the scene.
2.  **Propagation:** March the photon through the participating media.
3.  **Storage:**
    *   If the photon interacts with another volume element (scattering event), store a beam segment from the previous position to the new interaction point. Update the photon's position and direction based on the scattering phase function.
    *   If the photon path intersects a solid object before reaching its next intended scattering event or maximum propagation distance, store a beam segment from the previous position to the intersection point. **Do not discard the photon.** The beam ends at the object surface [S1].
4.  **Termination:** If the photon escapes the scene or is terminated via Russian roulette, stop processing.

**Density Estimation (Beam × Point 3D):**
To estimate radiance at a query point $Q$, intersect the eye ray (or query ray) with the stored beams. The contribution is derived from the Beam Radiance Estimate (BRE) adapted for 3D volumetric queries [S1]. The local density of beams is computed by treating each beam as a cylinder of radius $r$. The radiance estimate $L(Q)$ is proportional to the number of beams intersecting a small volume around $Q$, weighted by their radiance and the beam geometry [S1].

**Parameters:**
*   **Beam Radius ($r$):** Can be fixed or adaptive. A fixed width is simple but may not suit all domains [S1]. Adaptive methods adjust $r$ based on local photon density [S1].

**Edge Cases:**
*   **Precision:** Intersection tests between rays and beam cylinders must be robust to avoid gaps or double-counting.
*   **Boundaries:** The method explicitly handles the boundary between volume and solid geometry by terminating beams at intersections, preserving energy that would otherwise be lost [S1].

## Subtopics

### Volume Photon Map Lookup
This is the process of estimating inscattering contribution during eye ray traversal using the pre-computed volume photon map [S2, unofficial]. For each step during eye ray traversal, a lookup is performed in the volume photon map [S2, unofficial]. Unlike surface photon map lookups which always use final gathering, volume photon map lookups are performed without a final gather step [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of photons and the efficiency of the beam intersection queries. Beam intersection can be more complex than point distance queries but reduces the number of discarded photons.
*   **Memory Complexity:** Stores beam segments instead of points. Each beam requires storing start/end points, direction, and radiance. This may use more memory per photon than point-based methods but captures more geometric information.
*   **Strengths:** Efficiently handles light transport near boundaries between participating media and solids. Prevents energy loss at object intersections.
*   **Limitations:** Density estimation is more complex than point-based methods. Requires careful handling of beam geometry for accurate radiance estimation.
*   **When to use:** Scenes with significant light transport involving participating media and solid geometry boundaries.
*   **When not to use:** Simple homogeneous volumes without solid boundaries, where standard photon mapping may be sufficient and simpler.

## Background
Photon Beam Tracing was introduced by [S1] as an extension of volumetric photon mapping. It builds on the theory of beam radiance estimates (BRE) [S1]. The method allows for the integration of photon beams into existing volumetric photon mapping implementations with minor modifications to the photon shooting preprocess [S1].

## Sources
[S1] A comprehensive theory of volumetric radiance estimation using photon points and beams, https://doi.org/10.1145/1899404.1899409, fetched 2026-10-26
[S2] CS348b Final Project - Rendering Underwater Scenes, http://www.graphics.stanford.edu/courses/cs348b-competition/cs348b-04/underwater/index.html, fetched 2026-10-30 (unofficial)
