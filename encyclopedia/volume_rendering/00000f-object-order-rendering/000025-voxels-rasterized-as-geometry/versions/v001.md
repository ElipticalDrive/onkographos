# Voxels rasterized as geometry

Also called: Voxel rasterization, Geometry-based volume rendering, Voxel mesh rendering

## Overview
Voxels rasterized as geometry is an object-order volume rendering technique that treats volumetric data as a collection of 3D primitives (typically cubes or quads) and renders them using standard rasterization pipelines [S1, S3, S4, unofficial]. It solves the problem of visualizing volumetric data by explicitly projecting voxel geometry onto the 2D image plane, leveraging hardware-accelerated depth testing and blending [S2, S4, unofficial]. It differs from its siblings, such as [Splatting](23-splatting.md) (which projects soft blobs) or [Texture slicing](25-texture-slicing.md) (which samples 3D textures), by generating explicit geometric primitives for the volume elements, allowing for precise occlusion culling and standard shading models at the cost of high primitive counts [S1, S3, unofficial].

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

### [Binary Volume Rendering (Opacity Thresholding)](1015-binary-volume-rendering-opacity-thresholding.md)
A technique where voxels are treated as either completely opaque or transparent based on a scalar threshold $K$. It differs from direct volume rendering by ignoring interior voxels hidden by the opaque surface, optimizing for surface visualization. Choose this when visualizing distinct boundaries or surfaces rather than translucent volumes.

### [Naive Voxel Rendering](1103-naive-voxel-rendering.md)
Rasterizes every occupied voxel as a full geometric cube without LOD or spatial optimization. It differs from siblings by explicitly generating geometry for every voxel, resulting in extreme simplicity but massive primitive counts. Choose this for debugging or small datasets where implementation complexity must be minimized.

### [Intensity-based Sparse Voxel Storage](1775-intensity-based-sparse-voxel-storage.md)
Represents volumetric data by sorting voxels by intensity and using a histogram to index contiguous runs, treating intensity as run-length encoding. It differs from [Binary Volume Rendering](1015-binary-volume-rendering-opacity-thresholding.md) by supporting multi-level intensity and from [Naive Voxel Rendering](1103-naive-voxel-rendering.md) by eliminating per-voxel intensity storage. Choose this when rendering large volumes with many identical intensity values to reduce memory bandwidth.

### [OpenGL Display List Voxel Rendering](2220-opengl-display-list-voxel-rendering.md)
Pre-compiles voxel geometry into OpenGL 1.0 fixed-pipeline display lists for fast repeated rendering. It differs from modern VAO methods by relying on legacy API commands (`glNewList`/`glEndList`) and trading higher memory usage for superior per-frame render times. Choose this for legacy systems or static scenes requiring minimal CPU overhead per frame.

### [Vertex Array Object Voxel Rendering](2226-vertex-array-object-voxel-rendering.md)
Stores explicit position and color data in interleaved Vertex Buffer Objects (VBOs) managed by a Vertex Array Object (VAO), rendered via `glDrawArrays`. It differs from [Geometry Shader Processing](2524-geometry-shader-processing.md) by pre-computing geometry in memory rather than generating it on the GPU. Choose this for modern OpenGL pipelines requiring explicit control over vertex attributes.

### [ClearView Vertex Shader](2468-clearview-vertex-shader.md)
Reconstructs face geometry and attributes from texture buffers using `gl_VertexID`, avoiding vertex attribute buffer overhead. It sets itself apart by decoupling geometry data from the vertex stream, storing it in dedicated texture buffers for efficient random access. Choose this for rendering massive voxel counts where vertex buffer bandwidth is a bottleneck.

### [Geometry Shader Processing](2524-geometry-shader-processing.md)
Generates voxel geometry dynamically from primitive inputs (points/triangles) within the GPU pipeline. Unlike methods that use instancing or display lists, this relies on the Geometry Shader (GS) to expand input primitives into voxel shells or billboards at runtime. Choose this when geometry generation needs to be dynamic or view-dependent on the GPU.

### [Voxel Billboard Rendering](2554-voxel-billboard-rendering.md)
Represents a 3D scalar field as a set of 2D quads (billboards) positioned at voxel locations, using instanced rendering. Unlike [Binary Volume Rendering](1015-binary-volume-rendering-opacity-thresholding.md) which discards data, or [Naive Voxel Rendering](1103-naive-voxel-rendering.md) which renders cubes, this method generates billboard instances with per-instance position and scale. Choose this for volumetric effects like clouds or smoke where full 3D geometry is unnecessary.

### [Instancing with Procedural/Shader Generation](2794-instancing-with-procedural-shader-generation.md)
Renders volumetric data by submitting minimal base geometry and using GPU instancing to replicate it, with per-instance attributes processed in shaders. It differs from [Binary Volume Rendering](1015-binary-volume-rendering-opacity-thresholding.md) by using explicit primitives and from [Naive Voxel Rendering](1103-naive-voxel-rendering.md) by leveraging hardware instancing to reduce draw call overhead. Choose this for rendering millions of voxels efficiently.

### [SVO LOD Management](2990-svo-lod-management.md)
Not given in the sources.

### [Sparse Voxel Rasterization](3032-sparse-voxel-rasterization.md)
A real-time high-fidelity radiance field rendering system that integrates an efficient rasterizer with a multi-level sparse voxel scene representation. It adaptsively fits sparse voxels into different levels-of-detail. Choose this for high-quality rendering of complex scenes with varying density.

### [Base Renderer Geometry](3902-base-renderer-geometry.md)
Not given in the sources.

### [Quad-to-Geometry Assembly](4604-quad-to-geometry-assembly.md)
Not given in the sources.

### [Debug and Visualization](4974-debug-and-visualization.md)
Provides geometric primitives and cross-sectional views to inspect the spatial extent and internal structure of voxel data. It solves the problem of spatial orientation by overlaying wireframes or slices, distinct from methods that visualize volume content itself. Choose this for development and debugging.

### [Splatting and Point-based Rendering](4975-splatting-and-point-based-rendering.md)
Not given in the sources.

### [Hexagonal Prism Voxel Rendering](5626-hexagonal-prism-voxel-rendering.md)
Represents voxels as hexagonal prisms to reduce aliasing and provide smoother surfaces compared to cubic voxels. Choose this when visual quality and reduced stair-stepping artifacts are prioritized over standard cubic voxel compatibility.

### [Voxel Fragment Shader Rendering](5849-voxel-fragment-shader-rendering.md)
A fragment shader that renders voxels as solid cubes by performing ray-AABB intersection in screen space, computing normals, and applying fog. Choose this for custom shading effects where per-fragment ray casting is feasible.

### [Voxelization via Rasterization](6352-voxelization-via-rasterization.md)
Generates a voxel representation of 3D geometry by leveraging standard 2D rasterization hardware. It distinguishes itself by relying on 2D rasterization pipelines rather than brute-force 3D traversal. Choose this for converting mesh data into voxel formats.

#### Geometry Loading (Non-Triangle)
Loads basic geometry parameters (size, type) for non-triangle volumes (cylindrical/spherical) [S2, unofficial]. It copies geometry size and type to device symbols and sets a flag indicating the geometry is not triangles [S2, unofficial].

#### TLAS Instance Traversal
Manages transformed voxel objects by storing local acceleration structures and applying inverse rotations during ray queries [S3, unofficial]. It transforms rays into instance local space via inverse rotation (transpose, since the matrix is orthogonal) [S3, unofficial]. The `VoxelInstance` contains a local `BrickPool` (BLAS), rotation matrix, and bounding sphere [S3, unofficial].

#### Rasterizing Proxy Geometry
Proposes rasterizing proxy geometry representing a view-dependent cut of octree nodes [S4, unofficial]. This approach optimizes the number of look-ups into empty-space skipping data structures by avoiding hierarchy traversal on the GPU [S4, unofficial].

#### Point-Based Chunk Rendering
Each chunk is rendered as a collection of points using `GL_POINTS`, with vertex attributes for position, normal, and texture coordinates [S5, unofficial]. It binds the chunk VAO, sets model/view/projection matrices, and calls `glDrawArrays(GL_POINTS, 0, num_vertices)` [S5, unofficial]. A pitfall is that meshes are always updated on the GPU even if empty to prevent floating quads when destroying blocks near chunk borders [S5, unofficial].

## Background
The technique of rasterizing voxels as geometry has evolved alongside GPU capabilities. Early implementations used OpenGL display lists for static volumes [S1, S2, unofficial]. With the advent of vertex shaders and VBOs, explicit vertex buffers became standard [S1, S3, unofficial]. More recent approaches leverage geometry shaders and instancing to handle larger datasets dynamically [S1, S2, S3, unofficial]. The method is widely used in scientific visualization, medical imaging, and real-time voxel-based games [S1, S3, S4, unofficial].

## Sources
[S1] upupming/volume-rendering, https://github.com/upupming/volume-rendering, fetched 2026-10-26 (unofficial)
[S2] ojaisnielsen/dtu-2010-cuda-volume-rendering, https://github.com/ojaisnielsen/dtu-2010-cuda-volume-rendering, fetched 2026-10-27 (unofficial)
[S3] NiekMSoftware/bad-apple-voxel-raytracer, https://github.com/NiekMSoftware/bad-apple-voxel-raytracer, fetched 2026-10-24 (unofficial)
[S4] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S5] EmaMaker/voxel-engine, https://github.com/EmaMaker/voxel-engine, fetched 2026-10-26 (unofficial)
