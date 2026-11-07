# Debug and Visualization

Also called: Debug Rendering, Debug Visualization, Bounding Box Debug, Voxel Debug

## Overview
This family provides auxiliary geometry to inspect voxel data spatially without contributing to the final volumetric appearance [S1, unofficial]. It differs from sibling methods like [Binary Volume Rendering](1015-binary-volume-rendering-opacity-thresholding.md) or [Naive Voxel Rendering](1103-naive-voxel-rendering.md) by rendering explicit debug primitives (boxes, lines, spheres) rather than the volume data itself, serving as a diagnostic tool for spatial extent and density [S1, S3, S4, unofficial].

## Shared principles
The family shares the fundamental mechanism of converting scalar field data into geometric primitives that are submitted to the graphics pipeline [G].

1. **Primitive Generation**: Each occupied voxel or volume extent is represented by one or more geometric primitives. The most common representation is a cube (6 faces, 8 vertices for shared shading, or 24 vertices for flat shading) or a set of quads representing visible faces [G].
2. **Coordinate Transformation**: Voxel indices $(i, j, k)$ are transformed into world space. For a dense grid with bounds $P_{min}$ and $P_{max}$ and dimension $N_{dim}$, the world position $P_{world}$ of a voxel center is calculated as:
$$
P_{world} = P_{min} + \frac{I_{voxel}}{N_{dim}} \times (P_{max} - P_{min})
$$
For unit voxels, this simplifies to $P_{world} = I_{voxel} \times V_{size}$, where $V_{size}$ is the voxel dimension [G].
3. **Rasterization**: The generated primitives are transformed into clip space using the Model-View-Projection (MVP) matrix and rasterized. Visibility is determined by the depth buffer (z-buffer) or a back-to-front painter's algorithm if transparency is required [G].
4. **Culling**: To manage performance, frustum culling is applied to exclude voxels outside the camera view. Back-face culling is often enabled to discard faces oriented away from the camera, calculated by the dot product of the view-space normal and the view vector [G].

## Subtopics

### [Axis-Aligned Bounding Box Geometry](1317-axis-aligned-bounding-box-geometry.md)
This method renders the clipped axis-aligned bounding box (AABB) geometry of a voxel volume to the framebuffer, providing visual feedback about the volume's spatial extent without contributing to the final volumetric appearance. It differs from [Bounding Box Drawing](2733-bounding-box-drawing.md) by explicitly generating the 36-vertex triangle mesh for the full box surface rather than just drawing wireframe lines, and from [Debug Voxel Visualization](7095-debug-voxel-visualization.md) by operating on the aggregate volume bounds rather than individual voxel cells.

### [Bounding Box Drawing](2733-bounding-box-drawing.md)
Bounding Box Drawing renders the wireframe outline of an Axis-Aligned Bounding Box (AABB) to provide visual context for voxel data, such as highlighting a selected voxel or defining the volume's spatial extent. It differs from [Axis-Aligned Bounding Box Geometry](1317-axis-aligned-bounding-box-geometry.md) by explicitly using line primitives (edges) rather than filled geometry, and serves as a debug/visualization aid rather than contributing to the final volumetric appearance.

### [Debug Voxel Visualization](7095-debug-voxel-visualization.md)
Debug Voxel Visualization renders auxiliary geometry (typically spheres) to provide visual feedback about the location and density of voxel data without contributing to the final volumetric appearance. It solves the problem of inspecting sparse or dense voxel grids by converting discrete volume data into recognizable 3D primitives. It differs from [Axis-Aligned Bounding Box Geometry](1317-axis-aligned-bounding-box-geometry.md) and [Bounding Box Drawing](2733-bounding-box-drawing.md) by representing individual occupied voxels rather than just the volume's extent.

### Voxel Visualization
#### definition
- Debug rendering technique that visualizes the 3D voxel texture by slicing it through a cube and displaying the resulting cross-sections on screen [S1, unofficial].

#### step
- Create two framebuffers (`vvfbo1`, `vvfbo2`) with dimensions `viewportHeight` x `viewportWidth` [S1, unofficial].
- Render the cube to `vvfbo1` with front-face culling enabled [S1, unofficial].
- Render the cube to `vvfbo2` with back-face culling enabled [S1, unofficial].
- Bind the default framebuffer, use the `voxel_visualization` shader, and upload camera matrices [S1, unofficial].
- Activate `vvfbo1` as `textureBack` at unit 0, `vvfbo2` as `textureFront` at unit 1, and the 3D voxel texture as `texture3D` at unit 2 [S1, unofficial].
- Set viewport to `viewportWidth` x `viewportHeight`, clear buffers, and render the quad [S1, unofficial].

## Background
Origin: The techniques described herein are primarily derived from implementation details found in the `Friduric/voxel-cone-tracing` repository, which provides unofficial examples of voxel debugging and visualization in OpenGL [S1, unofficial]. These methods are commonly used in voxel-based game engines and scientific visualization tools to verify spatial alignment, culling correctness, and data density before committing to expensive ray-marching or ray-casting passes.

## Sources
[S1] Friduric/voxel-cone-tracing, https://github.com/Friduric/voxel-cone-tracing, fetched 2026-10-26 (unofficial)
