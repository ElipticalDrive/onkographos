# Emissive Clip Planes

Also called: emissive illumination for clip planes

## Overview
Emissive Clip Planes is a volume rendering technique that eliminates visual artifacts at the boundary of clipped volume data by overriding standard Phong illumination with purely emissive coloring [S1, unofficial]. It solves two specific problems inherent in gradient-based lighting: random, noisy lighting in homogeneous materials caused by estimated gradient noise, and dark, unlit back-facing surfaces revealed when the volume is cut [S1, unofficial]. Unlike methods that modify gradient normals or ignore clipped voxels, this approach preserves the classified material appearance directly at the cut plane while maintaining specular reflections on the rest of the volume [S1, unofficial].

## How it works
**Inputs:**
*   Volume dataset (scalar values). [S1, unofficial]
*   Clip plane definition (position and normal). [S1, unofficial]
*   Classification map (transfer function mapping scalar values to RGBA). [S1, unofficial]
*   Lighting parameters (light positions, material properties for Phong model). [S1, unofficial]

**Outputs:**
*   Final image with emissive illumination at clip boundaries and standard illumination elsewhere. [S1, unofficial]

**Data Structures:**
*   **Volume Data:** 3D grid of scalar values. Exact layouts, sizes, and bit packing are not given in the sources. [G]
*   **Classification Map:** Function mapping scalar values to RGBA. Exact layouts, sizes, and bit packing are not given in the sources. [G]
*   **Clip Plane:** Defined by a point and a normal vector. Exact layouts, sizes, and bit packing are not given in the sources. [S1, unofficial]

**Steps:**
1.  Identify the clip plane surface, which defines the boundary where the volume is cut. [S1, unofficial]
2.  Cast rays through the volume. [G]
3.  For each sample along a ray, determine if it lies on the clipped side. Samples on the clipped side are treated as transparent. [S1, unofficial]
4.  For samples immediately adjacent to the clip plane, override the standard Phong illumination calculation with purely emissive illumination. [S1, unofficial]
5.  For samples not adjacent to the clip plane, compute standard Phong illumination using gradients estimated from neighboring voxels. [S1, unofficial]
6.  Composite the color and opacity along the ray to produce the final pixel color. [G]

**Formulas:**
*   **Emissive Color:** $C_{emissive} = C(v)$, where $C(v)$ is the color output from the classification map for scalar value $v$. [S1, unofficial]
*   **Gradient Estimation:** $\mathbf{g}$ is estimated using central differences on neighboring voxels. [S1, unofficial]
*   **Phong Illumination:** $I = k_a + k_d (\mathbf{N} \cdot \mathbf{L}) + k_s (\mathbf{R} \cdot \mathbf{V})^n$, where $\mathbf{N} = \mathbf{g} / \|\mathbf{g}\|$ is the normalized gradient, $\mathbf{L}$ is the light vector, $\mathbf{V}$ is the view vector, $\mathbf{R}$ is the reflection vector, and $k_a, k_d, k_s, n$ are Phong coefficients. [G]
*   Note: The exact mathematical formula for the clip plane test and the distance threshold for adjacency are not given in the sources.

**Parameters:**
*   Adjacency threshold: Not given in the sources.
*   Phong coefficients ($k_a, k_d, k_s, n$): Standard lighting parameters. Typical values depend on the renderer implementation. [G]

**Edge Cases and Pitfalls:**
*   **Random Lighting:** In homogeneous materials, estimated gradients $\mathbf{g}$ have random orientations due to noise in neighboring voxels, causing random lighting artifacts when used as normals in the Phong model. [S1, unofficial]
*   **Dark Back-Facing Surfaces:** Clipping reveals inner structures that may be back-facing relative to light sources, appearing as dark, unlit regions. [S1, unofficial]
*   **Incorrect Materials via Interpolation:** If clipping is done by setting voxel values to zero before classification, interpolation of zero values leads to incorrect material appearance. [S1, unofficial]
*   **Specular Reflections on Cut Plane:** Customers disliked specular reflections on the cut plane itself, even though they wanted them on the volume. [S1, unofficial]

## Operating characteristics
*   **Time Complexity:** Same as standard volume rendering with gradient-based lighting, plus a simple adjacency check per sample. [G]
*   **Memory Complexity:** No additional memory required beyond standard volume rendering buffers. [G]
*   **Performance:** Implemented in hardware (TeraRecon VolumePro 1000), suggesting efficient execution. [S1, unofficial]
*   **Strengths:** Eliminates noisy lighting artifacts at clip boundaries; preserves material appearance; avoids dark back-facing surfaces; maintains specular highlights on the volume body. [S1, unofficial]
*   **Limitations:** Does not provide gradient-based shading on the cut plane itself; may appear less "realistic" if specular highlights are desired on the cut surface (though this was a customer preference). [S1, unofficial]
*   **When to Use:** When clipping volumes and visual artifacts (noise, darkness) at the cut plane are unacceptable, and specular highlights on the cut plane are undesirable. [S1, unofficial]
*   **When Not to Use:** When gradient-based shading (e.g., specular highlights) is required on the cut plane surface itself. [S1, unofficial]

## Background
Emissive Clip Planes were developed to address illumination issues in volume rendering, specifically in the context of the TeraRecon VolumePro 1000 hardware renderer. [S1, unofficial] The technique was documented in an unofficial supplement to a paper on volume rendering, highlighting common problems in gradient-based lighting when clipping is applied. [S1, unofficial] The approach leverages the concept of emissive illumination from OpenGL, where the color is taken directly from the classification without lighting modification. [S1, unofficial] The problem of random lighting in homogeneous materials was also discussed by Kniss et al. in 2002, who proposed using gradient magnitude to interpolate between lit and unlit results, but this was deemed too expensive and visually unsatisfactory for cut planes due to unwanted specular reflections. [S1, unofficial]

## Sources
[S1] Emissive Clip Planes for Volume Rendering - Supplement., http://www.hardenbergh.org/jch/volumes/emsuppl.pdf, fetched 2026-10-29 (unofficial)
