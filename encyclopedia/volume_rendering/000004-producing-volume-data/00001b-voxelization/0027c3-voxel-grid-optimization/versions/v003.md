# Voxel Grid Optimization


## Overview
Voxel Grid Optimization encompasses post-generation techniques applied to discrete 3D voxel grids to improve data quality, reduce storage requirements, or accelerate downstream rendering and processing pipelines. Unlike sibling methods in [Voxelization](12-voxelization.md) that focus on the geometric conversion of primitives into voxels, this family addresses the properties of the resulting grid itself. It distinguishes itself by operating on the *data* rather than the *geometry*, offering solutions for noise reduction ([3D Voxel Grid Smoothing](833-3d-voxel-grid-smoothing.md)), compression ([Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md)), structural simplification ([Occupancy Voxel Creation](2628-occupancy-voxel-creation.md)), and access pattern efficiency ([Bounding Volume Optimization](3583-bounding-volume-optimization.md)).

## Shared principles
The methods in this family share the assumption that a discrete 3D voxel grid has already been generated via [Voxelization](12-voxelization.md). They operate on the scalar, vector, or binary values stored within the grid cells defined by indices $(i, j, k)$. Common mechanisms include:
1.  **Spatial Filtering**: Applying convolution or averaging operations over local neighborhoods to modify voxel values [S1, unofficial].
2.  **Run-Length Encoding**: Compressing sequences of identical values by storing value-count pairs, often restricted to the minimal bounding box of the data [S2, unofficial].
3.  **Block Aggregation**: Analyzing groups of voxels (e.g., 2x2x2 blocks) to determine structural occupancy or simplify the representation [S1, unofficial].
4.  **Access Optimization**: Reducing the computational scope of raymarching or voxel traversal by culling empty regions or resizing arrays to fit data extents [S1, unofficial][S2, unofficial].

## Subtopics

### [3D Voxel Grid Smoothing](833-3d-voxel-grid-smoothing.md)
A post-processing step that reduces noise or aliasing artifacts in a discrete 3D voxel grid before isosurface extraction. It applies a separable 3D convolution using a 1D kernel of size 3 (typically $\{0.5, 1.0, 0.5\}$) to average voxel values over a local 3x3x3 neighborhood. Unlike [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md), which focuses on storage compression, this method operates directly on scalar values to produce a smoother field. Boundary handling excludes neighbors outside the grid, normalizing by the sum of valid weights to handle edges and corners.

### [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md)
A compression method that encodes a 3D voxel grid into a binary stream using run-length encoding (RLE). It stores only voxels within the minimal bounding box to reduce file size. The method packs runs of identical values (0 for empty, 1 for solid) into pairs of (value, count). The count is stored in a single byte (0-255), representing run lengths of 1-256. If a run exceeds 256 voxels, the encoder forces a new run at 255 to prevent overflow. The output includes an ASCII header with the voxel spacing.

### [Occupancy Voxel Creation](2628-occupancy-voxel-creation.md)
A technique that converts a continuous density matrix into a discrete boolean occupancy mask and identifies fully occupied 2x2x2 blocks. It differs from [3D Voxel Grid Smoothing](833-3d-voxel-grid-smoothing.md) by focusing on block-level density aggregation rather than spatial averaging. It uses a 2x2x2 kernel of ones in a 3D convolution to sum the values of 8 constituent voxels; a sum of 8 indicates a fully occupied block. This produces a structural mask rather than a compressed bitstream like [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md).

### [Voxelization Optimization](3191-voxelization-optimization.md)
A collection of techniques designed to reduce the computational cost and memory bandwidth requirements of generating voxel data from 3D geometry, particularly in real-time applications. It addresses bottlenecks in high draw call counts and redundant geometric transformations. It is distinguished from its siblings by focusing on the *generation pipeline* efficiency (e.g., stream output, instancing, LOD selection) rather than post-generation data compression or smoothing. Key strategies include using low-LOD meshes for static obstacles, caching transformed vertices for deforming meshes via GPU stream output, and using instanced rendering for slice-based voxelization.

### [Bounding Volume Optimization](3583-bounding-volume-optimization.md)
A method that reduces the computational cost of volume rendering by eliminating unnecessary evaluations of expensive primitives (such as Signed Distance Fields) for regions that do not contribute to the final image. It achieves this through two primary mechanisms: culling primitive evaluations using Axis-Aligned Bounding Boxes (AABBs) and resizing the voxel array to match the actual data extent. Unlike sibling methods that focus on data compression, this method optimizes the *access pattern* and *evaluation scope* during the raymarching or voxelization pipeline.

### Volume Buffer
- **Definition**: A volume data caching structure that encapsulates functionality, storage, and access similar to a Frame Buffer Object (FBO), but for 3D scalar data. It rasterizes primitives in 3D space and accumulates up to 1024 bits of arbitrary data per voxel [S1, unofficial].
- **Step**: Generate the final volume by fusing three intermediate passes (one for each perpendicular sweep axis) into a single multi-buffer using the MAX blending operation [S1, unofficial]. Disable face culling and z-test, and do not attach a z-buffer to the frame buffer object [S1, unofficial]. Use the OpenGL extension for layered rendering, writing to `gl_Layer` in the geometry shader to render primitives to arbitrary volume texture layers [S1, unofficial].
- **Parameter**: Max bits per voxel: 1024 bits [S1, unofficial]. Max render targets: 8 floating point render targets (MRTs), as 8 is the current maximum available number of MRTs [S1, unofficial].
- **Comparison**: Uses MAX blending for scalar data fusion instead of the OR operation used in binary voxelization [S1, unofficial].

### Voxelization Projection Culling
- **Definition**: Optimizes multi-pass voxelization by skipping primitives whose precomputed dominant axis does not match the current projection direction, preventing redundant voxel writes [S2, unofficial].
- **Step**: In the vertex shader, compute world position. If multi-pass mode is active, read the primitive's dominant axis from `triDominantAxis[vid / 3]`. Compare with `projDir.direction`. If they mismatch, skip the primitive [S2, unofficial].
- **Parameter**: `kVoxelizationMultiPass = !kVoxelizationSinglePass` (compile-time flag controlling pass strategy) [S2, unofficial].
- **Layout**: `VoxelProjectionDir` struct: `uint direction`; (bound to buffer 2). `triDominantAxis` buffer: bound to buffer index 10 (`TRI_DOMINANT_BUFFER_BINDING_IDX`) [S2, unofficial].
- **Pitfall**: Vertex ID to triangle ID mapping uses integer division `vid / 3`, assuming contiguous triangle strips/arrays [S2, unofficial].

### Bucket-based LBVH construction
- **Step**: Groups voxels into buckets instead of using a uniform grid with fixed brick size [S3, unofficial].

## Background
Voxel grid optimization techniques have evolved alongside real-time rendering and volume visualization requirements. Early methods focused on binary occupancy and simple compression. The introduction of GPU-based multi-channel voxelization allowed for the accumulation of complex scalar attributes, leading to structures like the Volume Buffer [S1, unofficial]. Optimization for rendering efficiency, such as bounding volume culling and bucket-based hierarchical bounding volume hierarchies (LBVH), has been driven by the need to handle large-scale biomedicinal and dynamic scenes efficiently [S2, unofficial][S3, unofficial].

## Sources
[S1] Two Simple Single-pass GPU methods for Multi-channel Surface Voxelization of Dynamic Scenes, https://doi.org/10.2312/pe/pg/pg2011short/031-036, fetched 2026-10-26 (unofficial)
[S2] kakashidinho/VoxelConeTracingMetal, https://github.com/kakashidinho/VoxelConeTracingMetal, fetched 2026-10-26 (unofficial)
[S3] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
