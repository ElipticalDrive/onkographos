# Ray Tracing
Also called: Classic ray tracing, Whitted-style Ray Tracing

## Overview
Ray tracing is a rendering technique that forms an image by tracing the path of light rays through an image plane and computing their interactions with scene geometry to determine color, shadows, reflections, and refractions [S1, unofficial]. It is distinguished from simple ray casting by its native support for secondary rays, specifically those required for shadow determination and specular reflections/refractions, thereby capturing global illumination effects beyond direct lighting [S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**: A scene description containing geometric primitives (surfaces), material properties (reflectivity, refractivity, diffuse color), light source positions, and a camera model defining the image plane and primary ray directions.
*   **Outputs**: A 2D image where each pixel contains a color value resulting from the accumulated light transport along the traced ray paths.

**Data Structures**
The method relies on a geometric representation of the scene. While the specific acceleration structures (e.g., BVH, grid) are implementation details of the intersection engine, the core data required for the shading step includes:
1.  **Intersection Point**: The 3D coordinate $\mathbf{p}$ where the ray intersects the closest primitive.
2.  **Surface Normal**: The unit normal vector $\mathbf{n}$ at $\mathbf{p}$, required for shading calculations.
3.  **Material Properties**: Parameters defining how the surface interacts with light (e.g., diffuse albedo, specular coefficient, index of refraction).

**Numbered Steps**
1.  **Primary Ray Generation**: For each pixel in the image plane, generate a primary viewing ray $\mathbf{r}_0$ originating from the camera eye position and passing through the pixel center [S1, unofficial].
2.  **Intersection Test**: Find the closest intersection point $\mathbf{p}$ between the ray and any scene primitive. If no intersection occurs, assign a background color [S2, unofficial].
3.  **Shading Calculation**: Compute the color contribution at $\mathbf{p}$ based on the material properties and lighting conditions.
4.  **Shadow Ray Tracing**: From $\mathbf{p}$, cast a shadow ray toward each light source. If the ray intersects an occluding object before reaching the light, the point is in shadow; otherwise, it receives direct illumination [S1, unofficial].
5.  **Secondary Ray Generation**: If the material is reflective or refractive, spawn secondary rays:
    *   **Reflection Ray**: Spawned in the direction of perfect reflection relative to $\mathbf{n}$ [S1, unofficial].
    *   **Refraction Ray**: Spawned according to Snell's Law, passing through the object [S1, unofficial].
6.  **Recursive Tracing**: Recursively trace the secondary rays (steps 2-5) up to a maximum depth. The final color is the sum of the direct illumination and the weighted contributions from reflected and refracted rays [S1, unofficial].

**Formulas**
*   **Reflection Direction**: Given incident ray direction $\mathbf{d}$ and surface normal $\mathbf{n}$, the reflected direction $\mathbf{r}$ is:
    $$ \mathbf{r} = \mathbf{d} - 2(\mathbf{d} \cdot \mathbf{n})\mathbf{n} $$
*   **Refraction Direction**: Given incident direction $\mathbf{d}$, normal $\mathbf{n}$, and indices of refraction $n_1$ (incident medium) and $n_2$ (transmitted medium), the transmitted direction $\mathbf{t}$ is derived from Snell's Law. Let $\eta = n_1/n_2$.
    $$ \mathbf{t} = \eta \mathbf{d} + (\eta (\mathbf{d} \cdot \mathbf{n}) - \sqrt{1 - \eta^2 (1 - (\mathbf{d} \cdot \mathbf{n})^2)}) \mathbf{n} $$
    [S1, unofficial]

**Parameters**
*   **Max Depth**: The maximum number of recursive bounces. Typical values range from 3 to 8 depending on desired accuracy and performance constraints. Not given in the sources.
*   **Ray Bias**: A small epsilon added to the intersection point to prevent self-intersection artifacts in recursive steps. Not given in the sources.

**Edge Cases and Pitfalls**
*   **Self-Intersection**: Recursive rays may intersect the surface they originated from due to floating-point precision errors. A bias offset is required [G].
*   **Total Internal Reflection**: When the argument to the square root in the refraction formula is negative, refraction is impossible, and the ray should be treated as fully reflected.
*   **Infinite Recursion**: Caused by perfect mirrors or lossless dielectrics without absorption. Must be capped by max depth or energy attenuation.

## Subtopics

### Binary Search IsoSurface Extraction
A method for finding intersections with implicit surfaces using binary search. Chosen when primitives are defined by scalar fields rather than explicit geometry. [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md)

### Trilinear Isosurface Intersection
Intersects rays with trilinearly interpolated implicit surfaces. Chosen for smooth, organic shapes defined by voxel data. [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md)

### Hybrid Implicit Isosurface Ray Tracing
Combines explicit and implicit representation strategies. Chosen for scenes with mixed explicit geometry and implicit volumes. [Hybrid Implicit Isosurface Ray Tracing](3832-hybrid-implicit-isosurface-ray-tracing.md)

### Neubauer Iso-surface Ray Casting
A specific ray casting algorithm for isosurfaces. Chosen for simpler direct volume rendering or isosurface visualization without complex reflection/refraction. [Neubauer Iso-surface Ray Casting](7024-neubauer-iso-surface-ray-casting.md)

### Hybrid Packetization Strategy
Optimizes ray traversal by grouping rays into packets. Chosen for improving cache coherence and parallel efficiency on GPUs. [Hybrid Packetization Strategy](7606-hybrid-packetization-strategy.md)

## Operating characteristics
*   **Time Complexity**: $O(N \cdot D \cdot I)$, where $N$ is the number of pixels, $D$ is the maximum recursion depth, and $I$ is the cost of intersection testing per ray. With acceleration structures, $I$ is reduced from $O(P)$ to $O(\log P)$ or better, where $P$ is the number of primitives.
*   **Memory Complexity**: $O(P)$ for storing scene primitives and acceleration structures. Recursive depth adds stack space proportional to $D$.
*   **Strengths**: Physically plausible rendering of shadows, reflections, and refractions; handles complex geometry naturally.
*   **Limitations**: Computationally expensive due to recursive ray tracing and numerous intersection tests; prone to noise in stochastic variants (though classic Whitted-style is deterministic).
*   **When to use**: Offline rendering, visual effects, and applications requiring high-fidelity specular and shadow effects.
*   **When not to use**: Real-time applications with strict latency constraints and simple diffuse-only scenes where rasterization is more efficient.

## Background
Ray tracing was first proposed by Turner Whitted in 1980 [S2, unofficial]. It builds upon the earlier concept of ray casting, introduced by Appel, by adding the capability to trace secondary rays for shadows and refractions [S2, unofficial]. The method is named "Whitted-style" to distinguish it from later global illumination techniques that use stochastic sampling (e.g., Metropolis light transport) or path tracing with diffuse interreflection. It remains the foundational algorithm for deterministic ray-traced rendering [S1, unofficial].

## Sources
[S1] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S2] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
