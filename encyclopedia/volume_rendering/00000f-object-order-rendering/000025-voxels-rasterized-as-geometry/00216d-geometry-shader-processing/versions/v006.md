# Geometry Shader Processing


## Overview
Geometry Shader Processing generates geometric primitives (cubes, quads, lines, or strips) directly on the GPU from scalar or vector field data to solve dynamic volumetric visualization without CPU-side mesh generation [G]. Unlike [Voxels rasterized as geometry](27-voxels-rasterized-as-geometry.md) which relies on static buffers or CPU culling, this family performs visibility culling and primitive generation in the geometry shader stage, optimizing memory usage by storing only compact voxel attributes (position, color, face flags) [G].

## Shared principles
The family shares the fundamental mechanism of converting scalar field data into geometric primitives that are submitted to the graphics pipeline [G]. For coordinate transformation, rasterization, and culling details, see [Voxels rasterized as geometry](27-voxels-rasterized-as-geometry.md).

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

### Geometry Shader Voxel Rendering
A technique where each vertex represents a voxel, and a geometry shader spawns individual cubic geometry for each voxel [S1, unofficial]. It packs data into voxel vertices using bitflags to maximize space efficiency, using the 5 least significant bits for 32 voxel types and the 3 most significant bits for other data [S1, unofficial]. Its viability depends heavily on voxel size and intended viewing distance, as it generates large amounts of geometry [S1, unofficial]. Extracting packed bit data on the GPU requires newer versions of GLSL and HLSL that support bitwise operators [S1, unofficial].

## Background
Originating from the introduction of the Geometry Shader in OpenGL 3.0, these techniques leverage the GPU's ability to generate and transform primitives on the fly [G]. They are widely used in scientific visualization for real-time volumetric data exploration, particle systems, and caustic rendering, where dynamic geometry generation is preferred over static mesh storage [G].

## Sources
[S1] Rendering different materials in a voxel terrain, https://gamedev.stackexchange.com/questions/45022/rendering-different-materials-in-a-voxel-terrain, fetched 2026-10-26 (unofficial)
