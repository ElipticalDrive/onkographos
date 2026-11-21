# AR mapping to 3D texture

Also called: 3D Texture Mapping

## Overview
AR mapping to 3D texture maps scalar volume data directly into a 3D texture map to avoid non-linear formulae during rendering [S1, unofficial]. It sets apart from its siblings by using a linear mapping for texture coordinates that allows parallel polygons to be legally rotated without recalculating coordinates, enabling the use of display lists for performance [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Scalar volume data, camera view parameters.
- Output: Rendered image with correct transparency compositing.

**Data Structures:**
- **3D Texture:** Stores RGBA values. The scalar data is classified into color and opacity.
- **Display List:** A group of precomputed OpenGL commands cached for later execution [S1, unofficial].

**Steps:**
1.  **Texture Population:** Place color and opacity into the 3D texture using a linear mapping [S1, unofficial].
2.  **Coordinate Definition:** Define texture coordinates once for the set of parallel polygons [S1, unofficial].
3.  **Geometry Rotation:** Rotate the set of parallel polygons legally. Because the linear mapping is used, texture coordinates do not need to be recalculated at each step [S1, unofficial].
4.  **Display List Creation:** Place the polygon set in a display list to improve performance when redrawing the same geometry multiple times [S1, unofficial].
5.  **Rasterization and Compositing:** Rasterize the polygons. Use linear alpha blending to composite fragments into the frame buffer [S1, unofficial].

**Formulas:**
Linear alpha blending equation:
$$ C_{dest} = C_{src} \cdot A_{src} + C_{dest} \cdot (1 - A_{src}) $$
Where $C_{src}$ and $A_{src}$ are the color and opacity from the texture sample, and $C_{dest}$ is the current frame buffer color [S1, unofficial].

**Parameters:**
- Framebuffer Resolution: A color/RGBA resolution of at least 16 bits or more is desirable for good quality [S2].

**Edge Cases and Pitfalls:**
- The estimation method is likely worse than Wilson's depth integration, but visualizations remain effective [S1, unofficial].
- Recalculating texture coordinates at each rotation is slow; the linear mapping avoids this [S1, unofficial].

## Subtopics

### [Gradient-Isosurface Texture Packing](889-gradient-isosurface-texture-packing.md)
Gradient-Isosurface Texture Packing is a data layout strategy for 3D volume textures that stores pre-computed gradient vectors in the RGB components and the original scalar isovalue in the Alpha component. This allows a single texture to provide both the data for transfer function indexing and the normals for shading, avoiding the need for a second texture copy or expensive runtime gradient calculation. It differs from AR mapping to 3D texture by embedding shading data directly into the volume representation, and from Röttger et al. Rendering by relying on hardware interpolation of pre-computed gradients rather than software-based ray casting optimizations.

### [Röttger et al. Rendering](938-r-ttger-et-al-rendering.md)
Röttger et al. Rendering is a technique that mitigates the high computational cost of volume integration by approximating the rendering integral and moving the computation to a pre-process. It computes a 3D lookup table of colors and opacities parameterized by front scalar value, back scalar value, and distance, which is stored in a 3D texture for hardware-accelerated lookups during rendering. This method builds on AR mapping to 3D texture by using the 3D texture not just for static data storage, but to evaluate the volume composition operator directly via texture interpolation. It is distinguished from standard ray casting by replacing per-voxel integration with texture lookups, and from pure pre-classification by supporting arbitrary transfer functions including isosurface effects through the parameterization of the lookup table.

### Uniform Grid
#### Step
- `set_values` accepts a (X, Y, Z) float32 tensor to update scalar values [S3, unofficial].

#### Parameter
- `aabb_min`: The minimum corner of the bounding box. Default is [-1, -1, -1] [S3, unofficial].
- `aabb_max`: The maximum corner of the bounding box. Default is [1, 1, 1] [S3, unofficial].
- `default_value`: Initial scalar value for all points. Default is float max [S3, unofficial].

#### Layout
- `get_points` returns a (X, Y, Z, 3) float32 tensor containing 3D coordinates of all grid points [S3, unofficial].
- `get_values` returns a (X, Y, Z) float32 tensor containing scalar values [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Sub-second rendering times for moderately-sized datasets [S2].
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** High-performance renderer with sub-second rendering times for moderately-sized datasets [S2].
- **Scaling:** Not given in the sources.
- **Strengths:** Avoids non-linear formulae; allows legal rotation of polygons without recalculating texture coordinates; enables use of display lists for performance [S1, unofficial].
- **Limitations:** Quality is limited due to lack of bit accuracy [S2]. The estimation method is likely worse than Wilson's depth integration [S1, unofficial].
- **When to use:** When sub-second rendering times are required for moderately-sized datasets and high bit accuracy is not critical [S2].
- **When not:** When high bit accuracy is required or when Wilson's depth integration is preferred for better estimation quality [S1, unofficial].

## Background
Origin: Not given in the sources.
History and development: The method is associated with SIV (Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration) [S1, unofficial].
Where it is used: Used in high-performance volume rendering systems [S2].

## Sources
[S1] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S2] A Practical Evaluation of Popular Volume Rendering ..., https://courses.cs.duke.edu/spring03/cps296.8/papers/PracticalEvaluationofVolumeRenderingAlgorithms.pdf, fetched 2026-10-29
[S3] GuangyanCai/isoext, https://github.com/GuangyanCai/isoext, fetched 2026-10-27 (unofficial)
