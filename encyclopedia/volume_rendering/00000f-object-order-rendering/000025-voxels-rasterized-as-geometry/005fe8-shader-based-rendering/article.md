# Shader-based Rendering


## Overview
Shader-based Rendering is a family of techniques that generate voxel geometry and compute shading directly on the GPU using vertex and fragment shaders, avoiding CPU-side mesh generation and vertex buffer uploads [G]. It differs from sibling methods like [Instancing with Procedural/Shader Generation](2794-instancing-with-procedural-shader-generation.md) by explicitly reconstructing topology and computing per-fragment intersections or attribute lookups within the shader stages rather than relying on pre-baked instancing or explicit geometry shaders [G].

## Shared principles
Builds on the parent family's mechanism of converting scalar field data into geometric primitives that are submitted to the graphics pipeline [G]. See [Voxels rasterized as geometry](27-voxels-rasterized-as-geometry.md) for the full shared details.

## Subtopics

### [ClearView Vertex Shader](2468-clearview-vertex-shader.md)
The ClearView Vertex Shader is a geometry reconstruction technique that generates face primitives from sparse texture buffers using `gl_VertexID`, eliminating the need for vertex attribute buffers [G]. It solves the problem of rendering complex volumetric or mesh data with minimal GPU memory bandwidth by fetching vertex and edge attributes on-the-fly [G]. It differs from sibling methods like [Voxel Fragment Shader Rendering](5849-voxel-fragment-shader-rendering.md) by deferring attribute lookup to the vertex stage and relying on explicit face indexing rather than implicit voxel topology [G]. Choose this method when explicit face indexing and vertex-stage reconstruction are required [G].

### [Voxel Fragment Shader Rendering](5849-voxel-fragment-shader-rendering.md)
Voxel Fragment Shader Rendering is a fragment-shader-based technique that renders voxels as solid cubes by performing ray-AABB intersection in screen space [G]. It computes surface normals, samples colors, and applies fog effects entirely within the fragment shader, bypassing the need for explicit geometry generation for every voxel face [G]. This method distinguishes itself from sibling approaches like ClearView Vertex Shader by deferring the geometric intersection logic to the fragment stage, allowing for per-fragment precision in ray-casting against voxel bounds [G]. Choose this method when per-fragment precision and bypassing explicit geometry generation are required [G].

### Static Shader with Attribute-Based Rendering
#### definition
A GLSL shader program that renders voxel terrain using packed integer attributes for position, texture ID, and normal direction, with uniform matrices for transformation and lighting [S1, unofficial].
#### parameter
Vertex Attribute 0: 'attrib1' (int), containing packed position, texture ID, and dimensions [S1, unofficial]. Vertex Attribute 1: 'attrib2' (int), containing packed normal direction and side information [S1, unofficial]. Uniform 'transformationMatrix': 4x4 matrix for model transformation [S1, unofficial]. Uniform 'projectionMatrix': 4x4 matrix for perspective projection [S1, unofficial]. Uniform 'viewMatrix': 4x4 matrix for camera view transformation [S1, unofficial]. Uniform 'sunColor': 3D vector for directional light color [S1, unofficial]. Uniform 'sunPosition': 3D vector for directional light position [S1, unofficial]. Uniform 'uv_scale': 2D vector for UV scaling [S1, unofficial].

## Background
Not given in the sources.

## Sources
[S1] knaxel/Voxel-Engine-Example, https://github.com/knaxel/Voxel-Engine-Example, fetched 2026-10-26 (unofficial)
