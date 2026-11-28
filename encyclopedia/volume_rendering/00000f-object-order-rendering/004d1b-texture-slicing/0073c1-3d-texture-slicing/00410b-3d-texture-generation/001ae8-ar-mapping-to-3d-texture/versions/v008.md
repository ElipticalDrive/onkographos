# AR mapping to 3D texture
## Overview
AR mapping to 3D texture is a volume rendering technique that stores scalar field data as color and opacity values within a 3D texture object to facilitate hardware-accelerated rendering [S1, unofficial]. It solves the problem of efficiently mapping volumetric data to screen space by using linear mapping to place color and opacity in the 3D texture, thereby avoiding the non-linear formulae typically associated with depth integration [S1, unofficial]. It is distinguished from its siblings by its reliance on pre-defined linear texture coordinates for parallel polygons, which allows the geometry to be cached in a display list, and from [Röttger et al. Rendering](938-r-ttger-et-al-rendering.md) by not using the texture to evaluate a pre-composed integration lookup table [S1, unofficial][S2].
## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume dataset, a set of parallel polygons representing the viewing direction, and a transfer function mapping scalar values to RGBA.
*   **Output:** A composited 2D image in the frame buffer.

**Data Structures:**
*   **3D Texture:** A 3D texture object with dimensions $W \times H \times D$. Each texel stores RGBA values, where RGB represents color and A represents opacity [S1, unofficial].
*   **Display List:** A group of OpenGL commands that have been precomputed and cached for later execution [S1, unofficial].

**Steps:**
1.  **Define Texture Coordinates:** Define the texture coordinates for the parallel polygons once. These coordinates map the 3D texture space to the polygon space [S1, unofficial].
2.  **Linear Mapping:** Apply a linear mapping to place color and opacity into the 3D texture. This avoids the need for non-linear formulae during rendering [S1, unofficial].
3.  **Rotate Polygons:** Rotate the set of parallel polygons to change the viewing angle. Because the texture coordinates are defined once and rely on linear mapping, they do not need to be recalculated for each rotation [S1, unofficial].
4.  **Cache Geometry:** Place the set of polygons in a display list. This improves performance when redrawing the same geometry multiple times, which is ideal for interaction with the rectangular solid [S1, unofficial].
5.  **Render and Composite:** Render the polygons using the 3D texture. The graphics hardware performs trilinear interpolation on the 3D texture to sample color and opacity values [S1, unofficial]. Fragments are composited into the frame buffer using hardware-supported alpha blending [S1, unofficial].

**Formulas:**
The standard blending equation used for compositing is:
$$ C_{out} = C_{in} + C_{sample} \cdot \alpha_{sample} \cdot (1 - \alpha_{in}) $$
$$ \alpha_{out} = \alpha_{in} + \alpha_{sample} \cdot (1 - \alpha_{in}) $$
where $C$ is color and $\alpha$ is opacity [S1, unofficial].

**Parameters:**
*   **Framebuffer Resolution:** A color/RGBA resolution of at least 16 bits or more is desirable for good quality [S2].

**Edge Cases and Pitfalls:**
*   **Precision:** The quality of the display is limited due to the lack of bit accuracy [S2].
*   **Accuracy:** The estimation method (linear mapping) is likely worse than Wilson's depth integration, but visualizations remain effective [S1, unofficial].
*   **Performance:** There is a critical screen size for each dataset where the cost for the increased number of trilinear interpolations overtakes the cost for footprint mappings [S2].

## Subtopics

### [Gradient-Isosurface Texture Packing](889-gradient-isosurface-texture-packing.md)
Stores pre-computed gradient vectors in RGB and scalar isovalue in Alpha. Sets itself apart by embedding shading normals directly into the volume representation, eliminating the need for a second texture or runtime gradient calculation. Choose when shading normals are required without extra texture lookups.

### [Uniform Grid](920-uniform-grid.md)
A structured mesh with uniformly spaced vertices. Sets itself apart by defining geometry implicitly via bounding box limits, eliminating explicit connectivity storage. Choose as the foundational data structure for efficient spatial indexing.

### [Röttger et al. Rendering](938-r-ttger-et-al-rendering.md)
Approximates the rendering integral via a pre-computed 3D lookup table. Sets itself apart by evaluating the volume composition operator directly via texture interpolation rather than static data storage. Choose for high-performance rendering where integration approximation is acceptable.

## Operating characteristics
*   **Time Complexity:** Sub-second rendering times for moderately-sized datasets [S2].
*   **Memory Complexity:** Requires storage for the 3D texture and the display list [S1, unofficial].
*   **Measured Performance:** High-performance renderer, but quality is limited by lack of bit accuracy [S2].
*   **Scaling:** Performance degrades at off-axis views [S2]. There is a critical screen size where trilinear interpolation costs outweigh footprint mapping costs [S2].
*   **Strengths:** Fast rendering for moderate datasets; efficient geometry caching via display lists; avoids non-linear formulae [S1, unofficial].
*   **Limitations:** Lower quality due to bit accuracy limitations; estimation method is less accurate than Wilson's depth integration; off-axis view degradation [S1, unofficial][S2].
*   **When to use:** When sub-second rendering of moderately-sized datasets is required and high bit-accuracy is not critical [S2].
*   **When not to use:** When high-quality, bit-accurate rendering is required, or when off-axis views are frequent [S2].

## Background
The method is described in the context of Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration [S1, unofficial]. It is also evaluated in A Practical Evaluation of Popular Volume Rendering Algorithms [S2].
## Sources
[S1] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S2] A Practical Evaluation of Popular Volume Rendering ..., https://courses.cs.duke.edu/spring03/cps296.8/papers/PracticalEvaluationofVolumeRenderingAlgorithms.pdf, fetched 2026-10-29
