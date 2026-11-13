# Voxel Grid Optimization

Also called: Sparse Voxel Grid Optimization, Voxel Data Optimization

## Overview
Voxel Grid Optimization encompasses techniques to reduce the computational and memory overhead of voxel-based volume rendering by optimizing data storage, grid resolution, spatial culling, and construction pipelines. It solves the problem of inefficient traversal and rendering of sparse or high-resolution volumetric data by restricting operations to active regions and minimizing redundant calculations. It differs from [Sparse and Hierarchical Structures](4391-sparse-and-hierarchical-structures.md) by focusing on grid-level optimizations (such as resolution calculation, smoothing, and bounding volume culling) rather than hierarchical tree structures like octrees, and from [Solid Voxelization via Octree](6830-solid-voxelization-via-octree.md) by prioritizing pipeline efficiency and spatial culling over hierarchical solid representation.

## Shared principles
The core mechanism involves:
1.  **Spatial Culling**: Restricting ray marching or voxel traversal to the minimal axis-aligned bounding box (AABB) that encloses active geometry, eliminating evaluations of non-contributing empty space [S1, unofficial][S2, unofficial].
2.  **Resolution Control**: Calculating integer grid dimensions based on target voxel counts and scene aspect ratios to balance memory usage and traversal cost [S1, unofficial].
3.  **Data Compression and Encoding**: Using methods like Run-Length Encoding (RLE) or occupancy block detection to reduce the storage footprint of binary or scalar voxel fields [S1, unofficial][S2, unofficial].
4.  **Pipeline Optimization**: Leveraging GPU features such as instancing, stream output, and layered rendering to accelerate the voxelization pass for dynamic scenes [S1, unofficial][S2, unofficial].

## Subtopics

### [3D Voxel Grid Smoothing](833-3d-voxel-grid-smoothing.md)
A smoothing operation applied to a 3D voxel grid to reduce noise or aliasing artifacts before isosurface extraction. It uses a separable 3D convolution with a 1D kernel of size 3. It differs from [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md) by operating on continuous values rather than binary occupancy, and from [Coordinate Grid Generation](1273-coordinate-grid-generation.md) by modifying the scalar field values rather than generating spatial coordinates.

### [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md)
Binary Voxel RLE Encoding compresses a 3D voxel grid into a binary stream using run-length encoding (RLE) to store occupancy data efficiently. It differs from sibling methods like [Voxel Indexing and Bit Compression](1707-voxel-indexing-and-bit-compression.md) by storing explicit occupancy bits for every possible voxel, whereas this method stores only runs of identical values, making it suitable for sparse data or when minimal bounding box optimization is applied. It differs from [RLE4 Compression](4441-rle4-compression.md) by storing only binary occupancy (0 or 1) rather than per-voxel material or intensity information.

### [Coordinate Grid Generation](1273-coordinate-grid-generation.md)
Coordinate Grid Generation produces a flattened tensor of spatial coordinates spanning a normalized domain (typically $[-1, 1]$) that matches the layout of a flattened image or voxel grid, serving as the primary input for neural field training and Signed Distance Function (SDF) evaluation. It differs from its parent [Voxel Grid Optimization](4131-voxel-grid-optimization.md) by focusing on the explicit creation of the coordinate tensor itself rather than optimizing the storage or culling of the resulting data, and it targets 2D image-plane layouts (or specific 3D variants) rather than the general optimization strategies like spatial culling or adaptive partitioning.

### [Occupancy Voxel Creation](2628-occupancy-voxel-creation.md)
Occupancy Voxel Creation converts a continuous density matrix into a discrete boolean occupancy mask and identifies fully occupied 2x2x2 blocks using convolution. It serves as a preprocessing step for sparse volume rendering by reducing the active voxel domain to dense clusters, differing from siblings like [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md) by focusing on block-level density detection rather than compression.

### [Grid Resolution Calculation](2908-grid-resolution-calculation.md)
Calculates the integer dimensions of a 3D voxel grid based on a target voxel count along one axis (typically Y) and the aspect ratio of the scene's bounding box. It ensures the grid maintains the geometric proportions of the scene while fixing the resolution in one dimension to control memory or traversal cost. It differs from [Grid Resolution Selection via Lambda](3270-grid-resolution-selection-via-lambda.md) by using direct aspect-ratio scaling rather than a complexity-based lambda function, and from [Voxel Grid Resolution Calculation](13832-voxel-grid-resolution-calculation.md) by deriving dimensions from a target count rather than a fixed voxel size.

### [Voxelization Optimization](3191-voxelization-optimization.md)
Voxelization Optimization reduces the computational and memory overhead of converting geometric primitives into voxel grids, a prerequisite for volume rendering and voxel-based global illumination. It distinguishes itself from general sparse voxelization by focusing on pipeline efficiency (reducing draw calls via instancing and stream output) and spatial culling (skipping empty or fully occupied columns) to maintain real-time performance on dynamic scenes.

### [Bounding Volume Optimization](3583-bounding-volume-optimization.md)
Bounding Volume Optimization reduces the computational cost of volume rendering by restricting ray marching or voxel traversal to the minimal axis-aligned bounding box (AABB) that encloses the active geometry, thereby eliminating false positives and skipping evaluations of non-contributing objects. Unlike sibling methods that focus on data compression or grid smoothing, this method operates on the spatial domain limits, ensuring that expensive Signed Distance Field (SDF) evaluations or ray steps are only performed where geometry exists.

### [Bucket-based LBVH construction](5163-bucket-based-lbvh-construction.md)
Bucket-based LBVH construction initializes a Bounding Volume Hierarchy (BVH) from a voxel model by grouping voxels into buckets rather than using a uniform grid with a fixed brick size. This approach, attributed to Fernandes and Walter, serves as a GPU-accelerated alternative to fixed-grid methods like those by Zellmann et al., aiming to improve performance for sparse volumes. It sets itself apart from its siblings by replacing static spatial partitioning with dynamic bucketing to handle varying voxel densities more efficiently.

### Volume Buffer
#### definition
- [S1, unofficial] A volume data caching structure that encapsulates functionality, storage, and access similar to a Frame Buffer Object (FBO), but for 3D scalar data. It rasterizes primitives in 3D space and accumulates up to 1024 bits of arbitrary data per voxel.
#### step
- [S1, unofficial] Generate the final volume by fusing three intermediate passes (one for each perpendicular sweep axis) into a single multi-buffer using the MAX blending operation.
- [S1, unofficial] Disable face culling and z-test, and do not attach a z-buffer to the frame buffer object.
- [S1, unofficial] Use the OpenGL extension for layered rendering, writing to gl_Layer in the geometry shader to render primitives to arbitrary volume texture layers.
#### parameter
- [S1, unofficial] Max bits per voxel: 1024 bits.
- [S1, unofficial] Max render targets: 8 floating point render targets (MRTs), as 8 is the current maximum available number of MRTs.
#### comparison
- [S1, unofficial] Uses MAX blending for scalar data fusion instead of the OR operation used in binary voxelization.

### Voxelization Projection Culling
#### definition
- [S2, unofficial] Optimizes multi-pass voxelization by skipping primitives whose precomputed dominant axis does not match the current projection direction, preventing redundant voxel writes.
#### step
- [S2, unofficial] In the vertex shader, compute world position. If multi-pass mode is active, read the primitive's dominant axis from triDominantAxis[vid / 3]. Compare with projDir.direction. If they mismatch, skip the primitive (early return/continue implied).
#### parameter
- [S2, unofficial] kVoxelizationMultiPass = !kVoxelizationSinglePass (compile-time flag controlling pass strategy)
#### layout
- [S2, unofficial] VoxelProjectionDir struct: uint direction; (bound to buffer 2). triDominantAxis buffer: bound to buffer index 10 (TRI_DOMINANT_BUFFER_BINDING_IDX).
#### pitfall
- [S2, unofficial] Vertex ID to triangle ID mapping uses integer division vid / 3, assuming contiguous triangle strips/arrays.

### Density Voxel Grid Reconstruction
#### definition
- [S3] A scene representation method using a density voxel grid to generate high-quality 3D reconstructed scenes and images once camera poses are optimized.
#### comparison
- [S3] Unlike NeRF which uses an MLP, or Plenoxels which uses sparse voxels with spherical harmonics, CBARF uses a density voxel grid for the final reconstruction stage after pose refinement.

### Significant Voxel Extraction
#### step
- [S4, unofficial] Extract and store only the most visually significant voxels to accelerate rendering by preventing traversal of the entire volume raster.

### Rate-distortion training objective with spatial predictive coding
#### step
- [S5] Incorporate the bitrate of prediction residuals into the loss function and jointly train for rate and distortion to enhance spatial correlation of the voxel grid.
#### parameter
- [S5] Trade-off factor λ controls the emphasis on bitrate during training; a larger λ emphasizes bitrate more, while λ=0 removes rate loss entirely.
#### number
- [S5] Ablation study shows disabling rate loss increases model size by +0.633 MB while improving PSNR by +0.13 dB and SSIM by +0.002.
#### comparison
- [S5] Training without rate loss (λ=0) yields better rendering quality but causes a rapid increase in bitrate, harming overall RD performance.

## Background
Voxel grid optimization techniques have evolved alongside real-time rendering requirements. Early methods focused on simple occupancy grids, while modern approaches integrate GPU-specific optimizations like stream output and layered rendering for dynamic scenes [S1, unofficial]. The introduction of neural radiance fields (NeRF) and voxel-based neural representations (like DVGO and Plenoxels) has driven new optimization strategies, including rate-distortion training and spatial predictive coding to compress voxel grids efficiently [S3, S5]. Bounding volume optimization remains a fundamental technique for culling empty space in both traditional and neural volume rendering [S2, unofficial].

## Sources
[S1] Two Simple Single-pass GPU methods for Multi-channel Surface Voxelization of Dynamic Scenes, https://doi.org/10.2312/pe/pg/pg2011short/031-036, fetched 2026-10-26 (unofficial)
[S2] kakashidinho/VoxelConeTracingMetal, https://github.com/kakashidinho/VoxelConeTracingMetal, fetched 2026-10-26 (unofficial)
[S3] CBARF: Cascaded Bundle-Adjusting Neural Radiance Fields From Imperfect Camera Poses, https://doi.org/10.1109/tmm.2024.3388929, fetched 2026-10-26
[S4] DTIC ADA606342: An Anti-Aliasing Technique for Splatting, https://archive.org/details/DTIC_ADA606342, fetched 2026-10-26 (unofficial)
[S5] SPC-NeRF: Spatial Predictive Compression for Voxel-Based Radiance Field, https://doi.org/10.1145/3795528, fetched 2026-10-26
