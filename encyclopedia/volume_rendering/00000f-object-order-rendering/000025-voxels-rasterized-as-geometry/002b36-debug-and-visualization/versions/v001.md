# Debug and Visualization

Also called: Debug Rendering, Bounding Box Visualization, Voxel Visualization

## Overview
This family provides geometric primitives and cross-sectional views to inspect the spatial extent, selection state, and internal structure of voxel data during rendering. It solves the problem of spatial orientation and debugging by overlaying wireframes or slices onto the framebuffer, distinct from [Binary Volume Rendering (Opacity Thresholding)](1015-binary-volume-rendering-opacity-thresholding.md) which visualizes the volume content itself rather than its container or structure.

## Shared principles
All methods in this family operate on the principle of rendering auxiliary geometry or post-processing textures to provide visual feedback about the voxel data's location or bounds, without contributing to the final volumetric appearance. They typically rely on standard OpenGL rasterization states (depth testing, blending) and simple geometric primitives (lines, triangles, quads).

## Subtopics

### [Axis-Aligned Bounding Box Geometry](1317-axis-aligned-bounding-box-geometry.md)
Renders the clipped bounding box geometry of a volume extent to the framebuffer, optionally including texture coordinates. It defines the wireframe or solid outline of the volume using a fixed set of vertices representing the 6 faces of the box. Unlike sibling methods that may draw arbitrary debug lines or dynamic bounds, this method specifically targets the static volume extent defined by minimum and maximum corners.

### [Bounding Box Drawing](2733-bounding-box-drawing.md)
Renders the wireframe outline of an Axis-Aligned Bounding Box (AABB) to provide visual context for volume data or to highlight a specific selected voxel. It solves the problem of spatial orientation and selection feedback in voxel-based environments by drawing the 12 edges of the box as line primitives. It differs from [Axis-Aligned Bounding Box Geometry](1317-axis-aligned-bounding-box-geometry.md) by focusing on the rendering of the wireframe edges for debug/selection purposes rather than generating geometry for collision or physics.

### Voxel Visualization
#### definition
Debug rendering technique that visualizes the 3D voxel texture by slicing it through a cube and displaying the resulting cross-sections on screen [S1, unofficial].

#### step
1. Create two framebuffers (`vvfbo1`, `vvfbo2`) with dimensions `viewportHeight` x `viewportWidth` [S1, unofficial].
2. Render the cube to `vvfbo1` with front-face culling enabled [S1, unofficial].
3. Render the cube to `vvfbo2` with back-face culling enabled [S1, unofficial].
4. Bind the default framebuffer, use the `voxel_visualization` shader, and upload camera matrices [S1, unofficial].
5. Activate `vvfbo1` as `textureBack` at unit 0, `vvfbo2` as `textureFront` at unit 1, and the 3D voxel texture as `texture3D` at unit 2 [S1, unofficial].
6. Set viewport to `viewportWidth` x `viewportHeight`, clear buffers, and render the quad [S1, unofficial].

## Background
The techniques described herein are primarily derived from implementation details found in the `voxel-cone-tracing` repository [S1, unofficial]. The specific AABB geometry generation and Voxel Visualization post-processing steps are documented in the source code comments and implementation of that project. The Bounding Box Drawing method reflects common practices in OpenGL-based voxel debuggers for selection feedback.

## Sources
[S1] Friduric/voxel-cone-tracing, https://github.com/Friduric/voxel-cone-tracing, fetched 2026-10-26 (unofficial)
