# Debug and Visualization


## Overview
Debug and Visualization methods render explicit geometric primitives (boxes, lines, or spheres) derived from voxel data to inspect spatial distribution, verify data loading, and provide visual context for volume extents [G]. It sets itself apart from sibling methods like [Binary Volume Rendering](1015-binary-volume-rendering.md) or [Naive Voxel Rendering](1103-naive-voxel-rendering.md) by focusing on auxiliary debugging aids (wireframes, occupancy spheres) rather than the final volumetric appearance [G].

## Shared principles
This family shares the fundamental mechanism of converting scalar field data into geometric primitives that are submitted to the graphics pipeline [G].

## Subtopics

### [Axis-Aligned Bounding Box Geometry](1317-axis-aligned-bounding-box-geometry.md)
This method renders the clipped bounding box geometry of a volume extent to the framebuffer, optionally including texture coordinates. It serves as a debug and visualization primitive in the Debug and Visualization family, converting scalar field data bounds into explicit geometric primitives for the graphics pipeline. Unlike [Debug Voxel Visualization](7095-debug-voxel-visualization.md), which renders individual occupied voxels, this method renders a single box representing the entire volume extent. It differs from [Bounding Box Drawing](2733-bounding-box-drawing.md) by specifically handling the update of vertex positions based on dynamic volume extents and supporting optional texture coordinate generation for mapping volume data onto the box faces.

### [Bounding Box Drawing](2733-bounding-box-drawing.md)
Bounding Box Drawing renders the wireframe outline of an Axis-Aligned Bounding Box (AABB) to provide visual context for voxel data, such as highlighting a selected voxel or defining the volume's spatial extent. It differs from [Axis-Aligned Bounding Box Geometry](1317-axis-aligned-bounding-box-geometry.md) by explicitly using line primitives to draw edges rather than filled geometry, and serves as a debug or visualization aid rather than contributing to the final volumetric appearance.

### [Debug Voxel Visualization](7095-debug-voxel-visualization.md)
Debug Voxel Visualization renders occupied voxels as geometric primitives to inspect the spatial distribution and density of scalar field data. It solves the problem of visualizing sparse or dense voxel grids by converting discrete volume data into visible geometry, allowing developers to verify data loading, normalization, and occupancy. It differs from ray-marched volume rendering by using explicit geometry submission to the graphics pipeline, prioritizing immediate visual feedback over photorealistic integration.

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
Not given in the sources.

## Sources
[S1] Friduric/voxel-cone-tracing, https://github.com/Friduric/voxel-cone-tracing, fetched 2026-10-26 (unofficial)
