# Ray Tracing
Also called: Classic ray tracing, Whitted-style Ray Tracing

## Overview
Ray tracing is a rendering technique that forms an image by tracing the path of light rays through an image plane and calculating the color of each pixel based on the objects the rays intersect [S1, S2, unofficial]. It solves the problem of accurately simulating global illumination effects such as shadows, reflections, and refractions, which simpler rasterization or basic ray casting methods cannot natively support [S1, S2, unofficial]. It is distinguished from its siblings (e.g., [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md)) by its explicit recursive handling of secondary rays for specular interactions rather than implicit surface extraction or fixed-step traversal [S1, S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**: A scene description containing geometric primitives (surfaces), material properties (reflectance, refractive indices), light sources, and a camera model defining the image plane and primary ray directions.
*   **Outputs**: A 2D image where each pixel value represents the accumulated radiance.

**Data Structures**
*   **Ray**: Defined by an origin $\mathbf{o}$ and a direction $\mathbf{d}$, typically normalized [G].
*   **Intersection Record**: Contains the hit point $\mathbf{p}$, the surface normal $\mathbf{n}$, the material properties at $\mathbf{p}$, and the distance $t$ from the ray origin.

**Algorithm Steps**
1.  **Primary Ray Generation**: For each pixel in the image plane, generate a primary ray $\mathbf{r}_0$ originating from the camera eye position and passing through the pixel center [S1, unofficial].
2.  **Intersection Test**: Find the closest intersection between $\mathbf{r}_0$ and any scene primitive. If no intersection occurs, return the background color [S1, unofficial].
3.  **Shading Calculation**: At the intersection point $\mathbf{p}$, calculate the diffuse shading component based on the surface normal and light sources [S1, unofficial].
4.  **Shadow Ray Casting**: For each light source, cast a shadow ray from $\mathbf{p}$ toward the light. If any object intersects this ray before reaching the light, the point is in shadow and the direct illumination from that light is zero [S1, unofficial].
5.  **Secondary Ray Generation**:
    *   **Reflection**: If the material is reflective, generate a reflection ray $\mathbf{r}_{refl}$ using the incident direction and surface normal.
    *   **Refraction**: If the material is transparent/translucent, generate a refraction ray $\mathbf{r}_{refr}$ using Snell's Law and the surface normal [S1, unofficial].
6.  **Recursive Tracing**: Recursively trace $\mathbf{r}_{refl}$ and $\mathbf{r}_{refr}$ to determine their contributions. The final color is a weighted sum of the diffuse component, the reflected radiance, and the refracted radiance [S1, unofficial].
7.  **Termination**: Recursion terminates when a maximum depth is reached or the ray escapes the scene.

**Formulas**
*   **Reflection Direction**: $\mathbf{r}_{refl} = \mathbf{d} - 2(\mathbf{d} \cdot \mathbf{n})\mathbf{n}$, where $\mathbf{d}$ is the incident ray direction and $\mathbf{n}$ is the surface normal [G].
*   **Refraction Direction**: $\mathbf{r}_{refr} = \eta \mathbf{d} + (\eta (\mathbf{d} \cdot \mathbf{n}) - \sqrt{1 - \eta^2 (1 - (\mathbf{d} \cdot \mathbf{n})^2)}) \mathbf{n}$, where $\eta$ is the relative index of refraction [G].

**Parameters**
*   **Max Depth**: The maximum number of recursive bounces. Typical values range from 3 to 10 depending on desired accuracy [G].
*   **Threshold**: A reflectance/transmittance threshold below which recursion stops to save computation [G].

**Edge Cases and Pitfalls**
*   **Precision Errors**: Intersection tests may fail due to floating-point inaccuracies, causing "shadow acne" or missed intersections. A small epsilon offset is often applied to the shadow ray origin [G].
*   **Total Internal Reflection**: Refraction calculations must handle cases where the discriminant in Snell's Law is negative, switching to total internal reflection [G].

## Subtopics

### Binary Search IsoSurface Extraction
Extracts isosurfaces via binary search along rays. Chosen when implicit surfaces are defined by scalar fields and exact boundary location is required without explicit geometry. [Link](1316-binary-search-isosurface-extraction.md)

### Trilinear Isosurface Intersection
Intersects rays with trilinearly interpolated volumes. Chosen for dense volumetric data where linear interpolation suffices for speed. [Link](2213-trilinear-isosurface-intersection.md)

### Hybrid Implicit Isosurface Ray Tracing
Combines ray tracing with implicit surface acceleration. Chosen for complex implicit geometries requiring efficient traversal. [Link](3832-hybrid-implicit-isosurface-ray-tracing.md)

### Neubauer Iso-surface Ray Casting
A specific ray casting approach for isosurfaces. Chosen for simpler shading requirements without recursive reflections. [Link](7024-neubauer-iso-surface-ray-casting.md)

### Hybrid Packetization Strategy
Groups rays for batched processing. Chosen for performance optimization on parallel hardware. [Link](7606-hybrid-packetization-strategy.md)

## Operating characteristics
*   **Time Complexity**: $O(N \cdot D \cdot K)$, where $N$ is the number of pixels, $D$ is the maximum recursion depth, and $K$ is the cost of intersection testing per ray. Performance is highly dependent on the efficiency of the acceleration structure used for intersection tests.
*   **Memory Complexity**: $O(P + R)$, where $P$ is the size of the scene primitives and $R$ is the stack depth for recursion.
*   **Strengths**: Physically accurate simulation of light transport, including complex effects like caustics and soft shadows (with area lights).
*   **Limitations**: High computational cost compared to rasterization; difficult to implement efficiently for dynamic scenes without robust acceleration structures.
*   **When to Use**: Offline rendering, visual effects, and scenarios requiring high-fidelity lighting.
*   **When Not to Use**: Real-time applications with strict latency constraints and simple lighting requirements.

## Background
Ray tracing was first proposed by Turner Whitted in 1980 [S2, unofficial]. It extended the earlier concept of ray casting (introduced by Appel) by introducing the recursive tracing of secondary rays for reflections and refractions [S1, S2, unofficial]. This method became the standard for high-quality offline rendering and is often referred to as "Whitted-style" ray tracing [S1, S2, unofficial].

## Sources
[S1] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S2] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
