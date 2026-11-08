# Geometry Shader Processing

Also called: Geometry Shader Volume Techniques, GS Voxel Rendering

## Overview
Geometry Shader Processing is a family of object-order rendering techniques that generate geometric primitives (cubes, quads, lines, or strips) directly on the GPU from scalar or vector field data, submitting them to the rasterizer [S1, S2, S3, unofficial]. It solves the problem of dynamic volumetric visualization by avoiding CPU-side mesh generation and pre-computed vertex buffers, instead expanding point primitives into visible surface geometry within the graphics pipeline [S2, S3, unofficial]. Unlike [Voxels rasterized as geometry](27-voxels-rasterized-as-geometry.md) which may rely on static buffers or CPU culling, this family performs visibility culling and primitive generation in the geometry shader stage, optimizing memory usage by storing only compact voxel attributes (position, color, face flags) [S2, S3, unofficial].

## Shared principles
The family shares the fundamental mechanism of converting scalar field data into geometric primitives that are submitted to the graphics pipeline [G].

1.  **Primitive Generation**: Each occupied voxel or data element is represented by one or more geometric primitives. The most common representation is a cube (6 faces, 8 vertices for shared shading, or 24 vertices for flat shading) or a set of quads representing visible faces [G].
2.  **Coordinate Transformation**: Voxel indices $(i, j, k)$ are transformed into world space. For a dense grid with bounds $P_{min}$ and $P_{max}$ and dimension $N_{dim}$, the world position $P_{world}$ of a voxel center is calculated as:
    $$
    P_{world} = P_{min} + \frac{I_{voxel}}{N_{dim}} \times (P_{max} - P_{min})
    $$
    For unit voxels, this simplifies to $P_{world} = I_{voxel} \times V_{size}$, where $V_{size}$ is the voxel dimension [G].
3.  **Rasterization**: The generated primitives are transformed into clip space using the Model-View-Projection (MVP) matrix and rasterized. Visibility is determined by the depth buffer (z-buffer) or a back-to-front painter's algorithm if transparency is required [G].
4.  **Culling**: To manage performance, frustum culling is applied to exclude voxels outside the camera view. Back-face culling is often enabled to discard faces oriented away from the camera, calculated by the dot product of the view-space normal and the view vector [G].

## Subtopics

### [Geometry Shader Cube Expansion](1074-geometry-shader-cube-expansion.md)
Converts each input voxel (point) into a full cube shell of up to six faces. It optimizes memory by storing only center position, color, and a 1-byte face bitmask. Unlike [Geometry Shader (Quads)](4369-geometry-shader-quads.md), it generates the entire cube structure from a single point, outputting up to 12 vertices per primitive. Choose when memory bandwidth for input data is a constraint and full voxel shells are needed.

### [Velocity Visualization via Geometry Shader Extrusion](1581-velocity-visualization-via-geometry-shader-extrusion.md)
Visualizes 3D velocity fields by rendering line segments whose length and direction correspond to the velocity vector. It differs by converting vector field data into explicit line primitives rather than splatting or ray-marching. Choose for sparse, vector-based flow visualization.

### [Geometry Shader Triangle Slicing (Z-Pass)](1681-geometry-shader-triangle-slicing-z-pass.md)
A single-pass method for multi-channel surface voxelization that clips input triangles against volume slice boundaries within the geometry shader. Unlike [Pixel Shader Fragment Clipping](1685-pixel-shader-fragment-clipping.md), it performs clipping in the geometry shader to generate precise triangle strips, avoiding the overhead of generating and rejecting unnecessary fragments. Choose on architectures with small bandwidth or non-unified pipelines.

### [Geometry Shader (Quads)](4369-geometry-shader-quads.md)
Generates one quad per visible voxel face, where each input point represents a single face. It differs from [Geometry Shader Cube Expansion](1074-geometry-shader-cube-expansion.md) by outputting only 4 vertices per primitive, reducing buffer space and improving GPU compatibility, at the cost of requiring more input data (one point per face). Choose when vertex shader throughput is limited or strict 4-vertex primitive limits are beneficial.

### [GPU Particle Rendering via Geometry Shader](4444-gpu-particle-rendering-via-geometry-shader.md)
Renders point-based particle systems as 2D sprites (billboards) by expanding points into quads. It solves the problem of CPU-side mesh generation for dynamic particles. Unlike [Geometry Shader Cube Expansion](1074-geometry-shader-cube-expansion.md), it generates flat 2D quads that always face the camera. Choose for large numbers of small particles where 3D volume is unnecessary.

### [Recursive Geometry Shader Beam Generation](8395-recursive-geometry-shader-beam-generation.md)
Generates base caustic triangles and performs adaptive subdivision using a geometry shader executed recursively via stream-out feedback. It differs by treating beam geometry as a dynamic mesh refined through GPU feedback loops rather than static primitives. Choose for dynamic light transport path visualization.

## Background
Originating from the introduction of the Geometry Shader in OpenGL 3.0, these techniques leverage the GPU's ability to generate and transform primitives on the fly. They are widely used in scientific visualization for real-time volumetric data exploration, particle systems, and caustic rendering, where dynamic geometry generation is preferred over static mesh storage.

## Sources
