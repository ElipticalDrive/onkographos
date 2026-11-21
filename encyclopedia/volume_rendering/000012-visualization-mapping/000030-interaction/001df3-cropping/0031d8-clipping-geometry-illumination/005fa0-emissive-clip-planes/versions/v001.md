# Emissive Clip Planes

Also called: emissive illumination for clip planes

## Overview
Emissive Clip Planes is a volume rendering technique that eliminates visual artifacts at the boundary of clipped volume data by overriding standard Phong illumination with purely emissive coloring [S1, unofficial]. It solves two specific problems inherent in gradient-based lighting: random, noisy lighting in homogeneous materials caused by estimated gradient noise, and dark, unlit back-facing surfaces revealed when the volume is cut [S1, unofficial]. Unlike methods that modify gradient normals or ignore clipped voxels, this approach preserves the classified material appearance directly at the cut plane while maintaining specular reflections on the rest of the volume [S1, unofficial].

## How it works
**Inputs:**
*   Volume dataset (scalar values).
*   Clip plane definition (position and normal).
*   Classification map (transfer function mapping scalar values to RGBA).
*   Lighting parameters (light positions, material properties for Phong model).

**Outputs:**
*   Final image with emissive illumination at clip boundaries and standard illumination elsewhere.

**Data Structures:**
*   **Volume Data:** 3D grid of scalar values.
*   **Classification Map:** Function $C(v) \rightarrow (R, G, B, A)$ mapping scalar value $v$ to color and opacity.
*   **Clip Plane:** Defined by a point $P_0$ and a normal vector $\mathbf{n}$. A sample at position $\mathbf{x}$ is considered "adjacent" to the clip plane if it lies on the visible side of the plane and is within a small distance threshold $\epsilon$ from the plane, or if it is the first sample encountered along a ray after crossing the plane boundary [S1, unofficial].

**Steps:**
1.  **Ray Casting:** For each pixel, cast a ray through the volume.
2.  **Clipping Check:** Determine if the ray intersects the clip plane. If it does, samples on the "cut" side (where $(\mathbf{x} - P_0) \cdot \mathbf{n} < 0$) are treated as transparent/ignored [S1, unofficial].
3.  **Sample Processing:** For each sample $\mathbf{x}$ along the ray:
    a.  **Classification:** Retrieve the scalar value $v$ at $\mathbf{x}$ and compute the base color $C_{base} = C(v)$ [S1, unofficial].
    b.  **Adjacency Test:** Determine if $\mathbf{x}$ is immediately adjacent to the clip plane boundary. This is typically the first sample on the visible side of the plane [S1, unofficial].
    c.  **Illumination Decision:**
        *   If $\mathbf{x}$ is adjacent to the clip plane: Apply **Emissive Illumination**. The final color contribution is $C_{final} = C_{base} \cdot \alpha$. No lighting model is applied.
        *   If $\mathbf{x}$ is not adjacent: Apply **Standard Phong Illumination**.
            i.  Estimate gradient $\mathbf{g}$ at $\mathbf{x}$ using central differences or similar method [S1, unofficial].
            ii. Compute normal $\mathbf{N} = \mathbf{g} / \|\mathbf{g}\|$.
            iii. Compute Phong lighting model $L(\mathbf{N}, \mathbf{V}, \mathbf{L})$ using view vector $\mathbf{V}$ and light vector $\mathbf{L}$ [S1, unofficial].
            iv. $C_{final} = C_{base} \cdot L(\mathbf{N}, \mathbf{V}, \mathbf{L}) \cdot \alpha$.
4.  **Compositing:** Accumulate $C_{final}$ and $\alpha$ along the ray using standard front-to-back or back-to-front compositing [S1, unofficial].

**Formulas:**
*   **Emissive Color:** $C_{emissive} = C(v)$ [S1, unofficial].
*   **Phong Illumination:** $I = k_a + k_d (\mathbf{N} \cdot \mathbf{L}) + k_s (\mathbf{R} \cdot \mathbf{V})^n$, where $\mathbf{R}$ is the reflection vector [S1, unofficial].
*   **Clip Plane Test:** Sample $\mathbf{x}$ is clipped if $(\mathbf{x} - P_0) \cdot \mathbf{n} < 0$ [S1, unofficial].

**Parameters:**
*   $\epsilon$: Distance threshold for "adjacent" samples. Not given in the sources.
*   Phong coefficients ($k_a, k_d, k_s, n$): Standard lighting parameters.

**Edge Cases and Pitfalls:**
*   **Random Lighting:** In homogeneous materials, estimated gradients $\mathbf{g}$ have random orientations due to noise in neighboring voxels, causing random lighting artifacts when used as normals in the Phong model [S1, unofficial]. Emissive clip planes solve this by bypassing lighting at the cut plane where such noise is most visible [S1, unofficial].
*   **Dark Back-Facing Surfaces:** Clipping reveals inner structures that may be back-facing relative to light sources, appearing as dark, unlit regions. Emissive illumination ensures these surfaces are visible with their classified color [S1, unofficial].
*   **Incorrect Materials via Interpolation:** If clipping is done by setting voxel values to zero before classification, interpolation of zero values leads to incorrect material appearance. Emissive clip planes avoid this by classifying before clipping or by using emissive coloring directly on the classified result [S1, unofficial].
*   **Specular Reflections on Cut Plane:** Customers disliked specular reflections on the cut plane itself, even though they wanted them on the volume. Emissive clip planes remove specular highlights at the boundary while preserving them elsewhere [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Same as standard volume rendering with gradient-based lighting, plus a simple adjacency check per sample.
*   **Memory Complexity:** No additional memory required beyond standard volume rendering buffers.
*   **Performance:** Implemented in hardware (TeraRecon VolumePro 1000), suggesting efficient execution [S1, unofficial].
*   **Strengths:** Eliminates noisy lighting artifacts at clip boundaries; preserves material appearance; avoids dark back-facing surfaces; maintains specular highlights on the volume body.
*   **Limitations:** Does not provide gradient-based shading on the cut plane itself; may appear less "realistic" if specular highlights are desired on the cut surface (though this was a customer preference) [S1, unofficial].
*   **When to Use:** When clipping volumes and visual artifacts (noise, darkness) at the cut plane are unacceptable, and specular highlights on the cut plane are undesirable.
*   **When Not to Use:** When gradient-based shading (e.g., specular highlights) is required on the cut plane surface itself.

## Background
Emissive Clip Planes were developed to address illumination issues in volume rendering, specifically in the context of the TeraRecon VolumePro 1000 hardware renderer [S1, unofficial]. The technique was documented in an unofficial supplement to a paper on volume rendering, highlighting common problems in gradient-based lighting when clipping is applied [S1, unofficial]. The approach leverages the concept of emissive illumination from OpenGL, where the color is taken directly from the classification without lighting modification [S1, unofficial]. The problem of random lighting in homogeneous materials was also discussed by Kniss et al. in 2002, who proposed using gradient magnitude to interpolate between lit and unlit results, but this was deemed too expensive and visually unsatisfactory for cut planes due to unwanted specular reflections [S1, unofficial].

## Sources
[S1] Emissive Clip Planes for Volume Rendering - Supplement., http://www.hardenbergh.org/jch/volumes/emsuppl.pdf, fetched 2026-10-29 (unofficial)
