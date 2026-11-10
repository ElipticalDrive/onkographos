# AR mapping to 3D texture

Also called: 3D texture mapping, direct volume rendering via 3D textures

## Overview
AR mapping to 3D texture is a volume rendering technique that stores scalar field data (mapped to color and opacity) directly into a 3D texture object, allowing the graphics hardware to perform ray casting or slicing via texture lookups [S1, unofficial][S2]. It solves the problem of high computational cost in per-voxel integration by offloading the volume composition operator to hardware-accelerated texture interpolation [S2]. It differs from [3D Texture Generation via Slicing](4891-3d-texture-generation-via-slicing.md) by avoiding the explicit generation of 2D slice images and instead treating the volume as a continuous 3D resource accessed via 3D texture coordinates [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field dataset (volume data).
*   **Output:** A rendered 2D image.

**Data Structures:**
*   **3D Texture:** A volumetric texture object with dimensions $W \times H \times D$. Each texel stores RGBA values. The RGB components represent the color, and the A component represents the opacity (alpha) [S1, unofficial].
*   **Display List:** A cached set of OpenGL commands defining the geometry (parallel polygons) to optimize repeated rendering [S1, unofficial].

**Steps:**
1.  **Texture Coordinate Definition:** Define linear texture coordinates for the parallel polygons that represent the viewing rays or slices. Because the mapping is linear, texture coordinates need only be defined once for the entire set of parallel polygons [S1, unofficial].
2.  **Rotation Handling:** If the view direction changes, the parallel polygons are rotated. Unlike methods requiring non-linear formulae for texture coordinate recalculation at each step, this method allows the polygons to be "legally" rotated without recalculating the texture coordinates, as the linear mapping remains valid [S1, unofficial].
3.  **Display List Construction:** Place the set of rotated polygons into a display list. A display list is a group of precomputed and cached OpenGL commands for later execution, improving performance when redrawing the same geometry multiple times [S1, unofficial].
4.  **Rendering:** Render the polygons. The graphics hardware performs trilinear interpolation on the 3D texture to sample color and opacity values along the ray path.
5.  **Compositing:** Use linear alpha blending to composite the sampled values. The hardware handles the accumulation of color and opacity [S1, unofficial].

**Formulas:**
The method relies on linear mapping to place color and opacity in the 3D texture, avoiding non-linear formulae typically associated with depth integration [S1, unofficial]. The compositing follows standard linear alpha blending:
$$ C_{out} = C_{in} + C_{sample} \cdot \alpha_{sample} \cdot (1 - \alpha_{in}) $$
$$ \alpha_{out} = \alpha_{in} + \alpha_{sample} \cdot (1 - \alpha_{in}) $$
where $C$ is color and $\alpha$ is opacity [S1, unofficial].

**Parameters:**
*   **Framebuffer Resolution:** A color/RGBA resolution of at least 16 bits or more is desirable for good quality [S2].

**Edge Cases and Pitfalls:**
*   **Quality Limitation:** The quality of the display is limited due to the lack of bit accuracy inherent in the texture mapping approach [S2].
*   **Estimation Error:** The estimation method used for texture coordinate mapping is likely worse than Wilson's depth integration, but visualizations remain effective [S1, unofficial].
*   **Off-Axis Degradation:** Image quality degrades at off-axis views [S2].
*   **Interpolation Cost:** At high magnifications, the cost of increased trilinear interpolations may overtake footprint mappings, leading to frame rate drops [S2].

## Subtopics

### [Gradient-Isosurface Texture Packing](889-gradient-isosurface-texture-packing.md)
A data layout strategy storing pre-computed gradients in RGB and isovalue in Alpha. Chosen when shading normals are needed without runtime calculation.

### [Uniform Grid](920-uniform-grid.md)
A structured mesh with uniform vertex spacing. Chosen as the foundational data structure for efficient spatial indexing of the scalar field.

### [Röttger et al. Rendering](938-r-ttger-et-al-rendering.md)
A technique using a 3D lookup table for volume composition. Chosen to replace per-voxel integration with texture lookups for arbitrary transfer functions.

## Operating characteristics
*   **Time Complexity:** Sub-second rendering times for moderately-sized datasets [S2].
*   **Memory Complexity:** Requires storage for the 3D texture and display lists.
*   **Performance:** High performance due to hardware acceleration, but quality is limited by bit accuracy [S2].
*   **Strengths:** Fast rendering for moderate datasets; simple implementation using standard OpenGL features; display lists improve redraw performance [S1, unofficial].
*   **Limitations:** Lack of bit accuracy; degradation of image quality at off-axis views; high interpolation costs at high magnifications [S2].
*   **When to Use:** When sub-second rendering is required for moderately-sized datasets and high bit-accuracy is not critical [S2].
*   **When Not to Use:** When high-quality, bit-accurate results are required, or when off-axis views are common [S2].

## Background
The method was described in the context of Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration [S1, unofficial] and evaluated in A Practical Evaluation of Popular Volume Rendering Algorithms [S2]. It is part of the family of high-performance renderers including shear-warp, offering a trade-off between speed and quality [S2].

## Sources
[S1] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S2] A Practical Evaluation of Popular Volume Rendering ..., https://courses.cs.duke.edu/spring03/cps296.8/papers/PracticalEvaluationofVolumeRenderingAlgorithms.pdf, fetched 2026-10-29
