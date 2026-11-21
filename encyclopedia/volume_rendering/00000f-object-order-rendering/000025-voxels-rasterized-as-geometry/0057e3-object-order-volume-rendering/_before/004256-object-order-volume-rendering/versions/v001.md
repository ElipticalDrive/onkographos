# Object-Order Volume Rendering

Also called: Object space rendering

## Overview
Object-order volume rendering is a forward mapping technique that iterates through volumetric data samples and projects them directly onto the image plane [S1, unofficial][S2, book]. It solves the problem of rendering scalar fields by treating volume elements as geometric entities to be composited, distinct from image-order methods that cast rays per pixel [S1, unofficial]. It is set apart from sibling geometry-based methods (e.g., [Voxels rasterized as geometry](27-voxels-rasterized-as-geometry.md)) by operating directly on the data samples without generating intermediate geometric primitives like cubes or quads [S1, unofficial].

## Shared principles
The family shares the fundamental mechanism of converting scalar field data into geometric primitives that are submitted to the graphics pipeline [G].

1. **Primitive Generation**: Each occupied voxel is represented by one or more geometric primitives. The most common representation is a cube (6 faces, 8 vertices for shared shading, or 24 vertices for flat shading) or a set of quads representing visible faces [G].
2. **Coordinate Transformation**: Voxel indices $(i, j, k)$ are transformed into world space. For a dense grid with bounds $P_{min}$ and $P_{max}$ and dimension $N_{dim}$, the world position $P_{world}$ of a voxel center is calculated as:
$$
P_{world} = P_{min} + \frac{I_{voxel}}{N_{dim}} \times (P_{max} - P_{min})
$$
For unit voxels, this simplifies to $P_{world} = I_{voxel} \times V_{size}$, where $V_{size}$ is the voxel dimension [S4, unofficial].
3. **Rasterization**: The generated primitives are transformed into clip space using the Model-View-Projection (MVP) matrix and rasterized. Visibility is determined by the depth buffer (z-buffer) or a back-to-front painter's algorithm if transparency is required [G].
4. **Culling**: To manage performance, frustum culling is applied to exclude voxels outside the camera view. Back-face culling is often enabled to discard faces oriented away from the camera, calculated by the dot product of the view-space normal and the view vector [G].

## Subtopics

### [Binary Volume Rendering (Opacity Thresholding)](1015-binary-volume-rendering-opacity-thresholding.md)
Uses binary voxel values (0 for background, 1 for object) to determine projection. It is the simplest form of object-order rendering, suitable for solid objects where internal structure is irrelevant. Choose when data is binary and performance is critical.

### [OpenGL Display List Voxel Rendering](2220-opengl-display-list-voxel-rendering.md)
Pre-compiles voxel projection commands into OpenGL display lists. It sets itself apart by optimizing repeated rendering of static volumes. Choose for static binary volumes in legacy OpenGL pipelines.

### [Vertex Array Object Voxel Rendering](2226-vertex-array-object-voxel-rendering.md)
Uses modern VBOs/VAOs to manage voxel geometry data. It differs from display lists by allowing more flexible data updates and better performance on modern hardware. Choose for dynamic or frequently updated static volumes.

### [ClearView Vertex Shader](2468-clearview-vertex-shader.md)
Offloads voxel projection logic to the vertex shader. It sets itself apart by performing per-voxel transformations on the GPU, reducing CPU overhead. Choose for high-performance rendering of large binary volumes.

### [Geometry Shader Processing](2524-geometry-shader-processing.md)
Generates projection geometry (e.g., billboards or quads) from point primitives in the geometry shader. It differs from fixed-function approaches by allowing dynamic geometry generation. Choose when flexible projection shapes are needed.

### [Voxel Billboard Rendering](2554-voxel-billboard-rendering.md)
Projects voxels as 2D billboards facing the camera. It sets itself apart by simplifying 3D voxels to 2D sprites, reducing overdraw. Choose for large-scale volume visualization where exact depth precision is less critical than performance.

### [Instancing with Procedural/Shader Generation](2794-instancing-with-procedural-shader-generation.md)
Uses instancing to render many voxels efficiently, with procedural generation in shaders. It differs by minimizing draw calls. Choose for massive voxel datasets where memory bandwidth is a bottleneck.

### [Quad-to-Geometry Assembly](4604-quad-to-geometry-assembly.md)
Assembles quads from voxel data for rendering. It sets itself apart by optimizing the primitive stream for rasterization. Choose when quad-based rendering offers better performance than full cubes.

### [Debug and Visualization](4974-debug-and-visualization.md)
Specialized rendering for debugging voxel data. It differs by emphasizing data inspection over visual fidelity. Choose for development and validation of voxel algorithms.

### [Rasterizing Proxy Geometry](5164-rasterizing-proxy-geometry.md)
Renders simplified proxy geometry to approximate the volume. It sets itself apart by trading accuracy for speed. Choose for real-time applications requiring quick volume previews.

### [Point-Based Chunk Rendering](5555-point-based-chunk-rendering.md)
Renders chunks of voxels as points. It differs by using point sprites for extreme scalability. Choose for very large, sparse volumes.

### [Hexagonal Prism Voxel Rendering](5626-hexagonal-prism-voxel-rendering.md)
Uses hexagonal prisms instead of cubes. It sets itself apart by reducing aliasing and improving coverage. Choose when higher visual quality is needed for isotropic data.

### [Voxel Fragment Shader Rendering](5849-voxel-fragment-shader-rendering.md)
Performs compositing logic in the fragment shader. It differs by allowing complex per-fragment operations. Choose for advanced compositing effects like MIP or average.

### [Voxelization via Rasterization](6352-voxelization-via-rasterization.md)
Generates voxel data from rasterized geometry. It is an inverse process but shares the rasterization pipeline. Choose when converting mesh data to volume data.

### [Sparse Voxel Storage and Management](8435-sparse-voxel-storage-and-management.md)
Manages sparse voxel data structures for object-order rendering. It sets itself apart by optimizing memory usage. Choose for large, sparse volumes.

### [Naive Voxel Geometry](11683-naive-voxel-geometry.md)
Renders every voxel as a full cube. It is the baseline method. Choose for simplicity and correctness verification.

## Background
Object-order volume rendering was introduced as a fundamental technique in volume visualization, contrasting with image-order (ray casting) methods [S1, unofficial][S2, book]. The approach relies on forward mapping, where data samples are projected directly to the image plane [S1, unofficial]. Early implementations assumed uniform grid spacing and binary voxel values [S1, unofficial]. The technique has been widely adopted in scientific visualization and medical imaging, particularly in systems like VTK, where it serves as a basis for efficient rendering of large datasets [S2, book].

## Sources
[S1] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
[S2] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
