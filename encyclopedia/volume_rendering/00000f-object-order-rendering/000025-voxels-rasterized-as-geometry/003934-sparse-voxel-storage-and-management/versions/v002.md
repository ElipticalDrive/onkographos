# Sparse Voxel Storage and Management

Also called: Sparse Voxel Data Structures

## Overview
Sparse Voxel Storage and Management addresses the inefficiency of storing and rendering volumetric data where the majority of the space is empty or uniform. By organizing voxels into hierarchical structures or compressed buffers, it reduces memory bandwidth and storage costs compared to dense grid representations [S1]. This family is distinct from [Naive Voxel Rendering](1103-naive-voxel-rendering.md) and [Instancing with Procedural/Shader Generation](2794-instancing-with-procedural-shader-generation.md) because it explicitly manages the data layout and hierarchy of the voxel primitives to optimize traversal and rendering, rather than relying on brute-force iteration or simple geometric instancing of all occupied cells.

## Shared principles
The family shares the fundamental mechanism of converting scalar field data into geometric primitives that are submitted to the graphics pipeline [G].

1. **Primitive Generation**: Each occupied voxel is represented by one or more geometric primitives. The most common representation is a cube (6 faces, 8 vertices for shared shading, or 24 vertices for flat shading) or a set of quads representing visible faces [G].
2. **Coordinate Transformation**: Voxel indices $(i, j, k)$ are transformed into world space. For a dense grid with bounds $P_{min}$ and $P_{max}$ and dimension $N_{dim}$, the world position $P_{world}$ of a voxel center is calculated as:
$$
P_{world} = P_{min} + \frac{I_{voxel}}{N_{dim}} \times (P_{max} - P_{min})
$$
For unit voxels, this simplifies to $P_{world} = I_{voxel} \times V_{size}$, where $V_{size}$ is the voxel dimension [G].
3. **Rasterization**: The generated primitives are transformed into clip space using the Model-View-Projection (MVP) matrix and rasterized. Visibility is determined by the depth buffer (z-buffer) or a back-to-front painter's algorithm if transparency is required [G].
4. **Culling**: To manage performance, frustum culling is applied to exclude voxels outside the camera view. Back-face culling is often enabled to discard faces oriented away from the camera, calculated by the dot product of the view-space normal and the view vector [G].

## Subtopics

### [Intensity-based Sparse Voxel Storage](1775-intensity-based-sparse-voxel-storage.md)
A memory-compression technique that eliminates redundant intensity data by leveraging spatial coherence. It sorts voxels by intensity and uses a histogram to map intensity values to buffer offsets, reducing per-voxel storage from 8 bytes to 6 bytes (or fewer) by transmitting intensity as a shader constant for contiguous spans. Choose this when memory bandwidth is the bottleneck and intensity values exhibit strong spatial clustering. It differs from [SVO LOD Management](2990-svo-lod-management.md) by focusing on attribute compression rather than hierarchical geometry, and from [Sparse Voxel Rasterization](3032-sparse-voxel-rasterization.md) by not necessarily using an octree structure for rendering optimization.

### [SVO LOD Management](2990-svo-lod-management.md)
A technique for managing the Level of Detail (LOD) of Sparse Voxel Octrees (SVOs) by pre-generating and caching voxelized geometry at multiple octree depths. It separates asset preparation from rendering, allowing the system to switch between pre-computed Vertex Buffer Objects (VBOs) based on camera distance. Choose this for static or slowly changing scenes where dynamic voxelization is too expensive. It differs from real-time SVO ray-tracing [6811-sparse-voxel-octree-svo-lod-pipeline.md] by relying on rasterization of static geometry caches rather than per-frame ray traversal.

### [Sparse Voxel Rasterization](3032-sparse-voxel-rasterization.md)
A real-time volumetric rendering technique that rasterizes explicit sparse voxels arranged in an octree layout, avoiding the overhead of neural MLPs or overlapped splatting. It solves the scalability and fidelity trade-off in neural radiance fields by using a sorting-based rasterizer that projects voxel corners to image space and performs alpha composition with early termination. Choose this for high-fidelity novel-view synthesis or mesh reconstruction where ray-casting is too slow. It differs from [Plenoxels](1775-intensity-based-sparse-voxel-storage.md) by using rasterization and sparse voxels for the entire scene, including unbounded backgrounds, achieving higher frame rates.

## Background
Sparse voxel techniques have evolved from early volume rendering methods that struggled with the memory requirements of dense 3D textures. The introduction of Sparse Voxel Octrees (SVOs) allowed for efficient representation of large, sparse volumetric datasets [G]. The specific techniques listed here, such as intensity-based storage and SVO LOD management, emerged from the need to optimize real-time rendering of these structures on modern GPUs, particularly in the context of neural radiance fields (NeRFs) and large-scale scientific visualization [S1][S2].

## Sources
[S1] Intensity-based Sparse Voxel Storage notes, URL, date fetched
[S2] Sparse Voxel Rasterization notes, URL, date fetched
