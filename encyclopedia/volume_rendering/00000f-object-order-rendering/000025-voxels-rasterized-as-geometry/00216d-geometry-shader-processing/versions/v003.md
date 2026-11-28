# Geometry Shader Processing

Also called: Geometry Shader Volume Rendering, GS Voxelization

## Overview
This family of techniques utilizes the GPU Geometry Shader (GS) to generate geometric primitives (cubes, quads, lines, or strips) directly from point or triangle inputs, enabling the rasterization of volumetric data without pre-computed mesh buffers. It solves the problem of dynamic geometry generation for voxel-based rendering by offloading primitive assembly to the GPU, allowing for real-time updates and adaptive detail. It differs from [Voxels rasterized as geometry](27-voxels-rasterized-as-geometry.md) by explicitly leveraging the GS stage for primitive expansion rather than relying solely on vertex buffers or fixed-function instancing.

## Shared principles
The family shares the fundamental mechanism of converting scalar field data into geometric primitives that are submitted to the graphics pipeline.

1.  **Primitive Generation**: Each occupied voxel is represented by one or more geometric primitives. The most common representation is a cube (6 faces, 24 vertices for flat shading, or 8 vertices for shared shading) or a set of quads representing visible faces [S1, S3, S4, unofficial].
2.  **Coordinate Transformation**: Voxel indices $(i, j, k)$ are transformed into world space. For a dense grid with bounds $P_{min}$ and $P_{max}$ and dimension $N_{dim}$, the world position $P_{world}$ of a voxel center is calculated as [S4, unofficial]:
    $$
    P_{world} = P_{min} + \frac{I_{voxel}}{N_{dim}} \times (P_{max} - P_{min})
    $$
    For unit voxels, this simplifies to $P_{world} = I_{voxel} \times V_{size}$, where $V_{size}$ is the voxel dimension [S2, S4, unofficial].
3.  **Rasterization**: The generated primitives are transformed into clip space using the Model-View-Projection (MVP) matrix and rasterized. Visibility is determined by the depth buffer (z-buffer) or a back-to-front painter's algorithm if transparency is required [S2, S3, unofficial].
4.  **Culling**: To manage performance, frustum culling is applied to exclude voxels outside the camera view [S2, S4, unofficial]. Back-face culling is often enabled to discard faces oriented away from the camera, calculated by the dot product of the view-space normal and the view vector [S2, unofficial].

## Subtopics

### [Geometry Shader Cube Expansion](1074-geometry-shader-cube-expansion.md)
Converts each input point (representing a voxel) into a full cube shell of up to six faces. It optimizes memory by storing only center position, color, and a 1-byte face bitmask per voxel, emitting up to 12 vertices per primitive. Choose this for dense, static voxel grids where memory bandwidth for vertex buffers is a constraint.

### [Velocity Visualization via Geometry Shader Extrusion](1581-velocity-visualization-via-geometry-shader-extrusion.md)
Visualizes velocity fields by extruding points into line segments based on sampled 3D texture data. It renders alpha-gradient lines where length and direction reflect the vector. Choose this for flow field visualization where the underlying data is stored in a 3D texture.

### [Geometry Shader Triangle Slicing (Z-Pass)](1681-geometry-shader-triangle-slicing-z-pass.md)
Performs multi-channel surface voxelization by clipping input triangles against volume slice boundaries within the GS. It emits triangle strips to specific layers (`gl_Layer`). Choose this for voxelizing complex meshes into volume layers, particularly on architectures with limited pixel shader bandwidth.

### [Geometry Shader (Quads)](4369-geometry-shader-quads.md)
Generates one quad per visible voxel face, where each input point represents a single face. It outputs only 4 vertices per primitive, reducing buffer space and improving GPU compatibility compared to cube expansion. Choose this when face-level control is needed and input data can be pre-processed to one point per face.

### [GPU Particle Rendering via Geometry Shader](4444-gpu-particle-rendering-via-geometry-shader.md)
Renders point-based particle systems as 2D sprites (billboards) by expanding points into quads in screen space. It solves the CPU-side mesh generation problem for dynamic particles. Choose this for large-scale particle effects requiring billboarding and texture atlas support.

### Recursive Geometry Shader Beam Generation
A method for generating base caustic triangles and performing adaptive subdivision using a geometry shader executed recursively via stream-out feedback [S1].
- **Definition**: It packs the vertices of each beam triangle into a single point primitive instead of processing triangle primitives, allowing the vertex shader to handle part of the processing and hinting to the pipeline that specific attributes belong together [S1]. The geometry shader outputs 0-4 vertices instead of 0-12 [S1].
- **Steps**: The method performs beam extrusion separately for reflection and refraction in the beam-rasterization pass using the same adaptive base grid [S1]. If multiple bounces are enabled, an additional stream-out pass determines intersections with the environment, after which adaptive beam refinement restarts [S1].
- **Parameters**: Initial coarse grid resolution is $16 \times 16$, with 3-4 iterations of adaptive subdivision producing pleasing results [S1].

## Background
The use of Geometry Shaders for volume rendering emerged as a method to avoid the memory overhead of storing pre-computed vertex buffers for every face of a voxel grid. Early implementations focused on expanding points into cubes or quads to render volumetric data directly [S2, S3, unofficial]. The technique of triangle slicing for voxelization was introduced to handle complex mesh inputs efficiently [S1]. Recursive beam generation for caustics was later developed to handle adaptive subdivision of light transport paths in real-time [S1].

## Sources
[S1] Real-time volume caustics with adaptive beam tracing, https://doi.org/10.1145/1944745.1944753, fetched 2026-10-28
