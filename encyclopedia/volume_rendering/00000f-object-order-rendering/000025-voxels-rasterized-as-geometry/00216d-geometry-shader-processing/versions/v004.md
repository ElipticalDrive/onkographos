# Geometry Shader Processing

Also called: Geometry Shader Volume Rendering, GS Voxelization, GS Extrusion

## Overview
Geometry Shader Processing is a family of GPU-based techniques that generate geometric primitives (cubes, quads, lines, or strips) directly from scalar field data or point streams within the graphics pipeline's geometry shader stage [S1, S2, S3, unofficial]. It solves the problem of rendering volumetric data or vector fields by offloading primitive generation to the GPU, avoiding the memory overhead of storing pre-computed vertex buffers for every face or the CPU cost of dynamic mesh generation [S2, S3, unofficial]. Unlike [Voxels rasterized as geometry](27-voxels-rasterized-as-geometry.md) which may use fixed-function or vertex-shader-driven instancing, this family explicitly manipulates primitive topology (e.g., expanding points to cubes or slicing triangles) to optimize for specific rendering goals like transparency, velocity visualization, or multi-channel voxelization [S1, S2, S3, unofficial].

## Shared principles
The family shares the fundamental mechanism of converting scalar field data into geometric primitives that are submitted to the graphics pipeline [G].

1.  **Primitive Generation**: Each occupied voxel or data point is represented by one or more geometric primitives. The most common representation is a cube (6 faces, 8 vertices for shared shading, or 24 vertices for flat shading) or a set of quads representing visible faces [G].
2.  **Coordinate Transformation**: Voxel indices $(i, j, k)$ are transformed into world space. For a dense grid with bounds $P_{min}$ and $P_{max}$ and dimension $N_{dim}$, the world position $P_{world}$ of a voxel center is calculated as:
    $$
    P_{world} = P_{min} + \frac{I_{voxel}}{N_{dim}} \times (P_{max} - P_{min})
    $$
    For unit voxels, this simplifies to $P_{world} = I_{voxel} \times V_{size}$, where $V_{size}$ is the voxel dimension [G].
3.  **Rasterization**: The generated primitives are transformed into clip space using the Model-View-Projection (MVP) matrix and rasterized. Visibility is determined by the depth buffer (z-buffer) or a back-to-front painter's algorithm if transparency is required [G].
4.  **Culling**: To manage performance, frustum culling is applied to exclude voxels outside the camera view. Back-face culling is often enabled to discard faces oriented away from the camera, calculated by the dot product of the view-space normal and the view vector [G].

## Subtopics

### [Geometry Shader Cube Expansion](1074-geometry-shader-cube-expansion.md)
Converts each input voxel (represented as a single point) into a full cube shell composed of up to six faces. It optimizes memory usage by storing only center position, color, and face enable flags, generating up to 12 vertices per input point. Choose this when rendering opaque or semi-opaque volumes where minimizing input buffer size is critical.

### [Velocity Visualization via Geometry Shader Extrusion](1581-velocity-visualization-via-geometry-shader-extrusion.md)
Visualizes grid velocities by extruding points into line segments with alpha gradients. It samples a 3D volume texture to determine vector direction and length. Choose this for flow field visualization where direct line rendering is preferred over particle systems.

### [Geometry Shader Triangle Slicing (Z-Pass)](1681-geometry-shader-triangle-slicing-z-pass.md)
A single-pass method for multi-channel surface voxelization that clips input triangles against volume slice boundaries within the geometry shader. It emits precise triangle strips for specific volume layers. Choose this on architectures with small bandwidth or non-unified pipelines where pixel shader cores are intensively used, as it avoids generating unnecessary fragments.

### [Geometry Shader (Quads)](4369-geometry-shader-quads.md)
Generates one quad per visible voxel face, where each input point represents a single face. It outputs only 4 vertices per primitive, reducing buffer space and improving GPU architecture compatibility compared to cube expansion. Choose this when vertex throughput is a bottleneck and face-level control is required.

### [GPU Particle Rendering via Geometry Shader](4444-gpu-particle-rendering-via-geometry-shader.md)
Renders point-based particle systems as 2D sprites (billboards) by expanding single point primitives into quads. It solves the problem of CPU-side mesh generation for dynamic particles. Choose this for large numbers of small particles where 3D volumetric detail is unnecessary.

### [Recursive Geometry Shader Beam Generation](8395-recursive-geometry-shader-beam-generation.md)
Generates base caustic triangles and performs adaptive subdivision using a geometry shader executed recursively via stream-out feedback. It treats beam geometry as a dynamic mesh refined through GPU feedback loops. Choose this for realistic caustic rendering where light transport paths require adaptive refinement.

## Background
The use of geometry shaders for volume and voxel rendering emerged as a method to bypass the limitations of fixed-function rasterization and the memory costs of storing explicit meshes for dense scalar fields. Early implementations focused on expanding point clouds into cubes to visualize sparse data [S2, S3, unofficial]. The technique evolved to include specialized cases like velocity vector extrusion [S1, unofficial] and multi-channel voxelization via triangle slicing [S1, unofficial], addressing specific performance bottlenecks in pixel shader-heavy pipelines. Recursive beam generation represents a more advanced application, leveraging stream-out feedback for adaptive light transport simulation [S1].

## Sources
[S1] Geometry Shader Techniques for Volume and Vector Visualization, https://example.com/gs-volume, 2023-10-01
[S2] GPU-Based Voxel Rendering with Geometry Shaders, https://example.com/gpu-voxel-gs, 2022-05-15
[S3] Efficient Voxel Shell Generation via Geometry Shader, https://example.com/voxel-shell, 2021-11-20
