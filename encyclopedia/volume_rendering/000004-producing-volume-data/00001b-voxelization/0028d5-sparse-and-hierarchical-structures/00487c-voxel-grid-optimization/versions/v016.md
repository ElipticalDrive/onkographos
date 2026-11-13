# Voxel Grid Optimization

Also called: Sparse Voxel Grid Optimization, Voxel Data Management

## Overview
Voxel Grid Optimization encompasses the techniques used to reduce the computational and memory overhead of converting geometry into voxel grids, addressing the high cost of numerous draw calls and redundant vertex transformations. It distinguishes itself from general grid management methods by focusing on the rendering pipeline efficiency (instancing, stream output) and spatial culling during the conversion phase, rather than post-processing the resulting data structure. It differs from its siblings like [3D Voxel Grid Smoothing](833-3d-voxel-grid-smoothing.md) by operating on the generation pipeline rather than the scalar field values, and from [Bounding Volume Optimization](3583-bounding-volume-optimization.md) by targeting the voxelization step rather than the raymarching evaluation step.

## Shared principles
The core mechanism involves:
1.  **Pipeline Efficiency**: Reducing the number of draw calls and vertex transformations by leveraging GPU features such as instanced rendering and stream output caching [S1, unofficial][S3].
2.  **Spatial Culling**: Skipping primitives or voxels that do not contribute to the final volume, either by precomputing dominant axes [S2, unofficial] or by analyzing bounding box aspect ratios [S1, unofficial].
3.  **Data Reduction**: Minimizing the active voxel domain through techniques like resizing to minimal bounding boxes [S1, unofficial] or extracting only significant voxels [S4, unofficial].
4.  **Adaptive Resolution**: Determining grid dimensions based on scene aspect ratios or target voxel counts to maintain geometric proportions while controlling memory usage [S1, unofficial].

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
This method determines the integer dimensions of a 3D voxel grid by fixing the resolution along one axis (typically the Y-axis) and calculating the other axes (X and Z) based on the aspect ratios of the scene's bounding box. It solves the problem of maintaining geometric proportions in the voxel grid while allowing the user to control the total voxel count via a single parameter. It is distinguished from [Grid Resolution Selection via Lambda](3270-grid-resolution-selection-via-lambda.md) by its reliance on bounding box aspect ratios rather than primitive counts, and from [Voxel Grid Resolution Calculation](13832-voxel-grid-resolution-calculation.md) by its specific use of a target Y-resolution as the primary driver.

### [Voxelization Optimization](3191-voxelization-optimization.md)
Voxelization Optimization encompasses techniques to reduce the computational and memory overhead of converting geometry into voxel grids, addressing the high cost of numerous draw calls and redundant vertex transformations. It distinguishes itself from general grid management methods by focusing on the rendering pipeline efficiency (instancing, stream output) and spatial culling during the conversion phase, rather than post-processing the resulting data structure.

### [Bounding Volume Optimization](3583-bounding-volume-optimization.md)
Bounding Volume Optimization reduces the computational cost of volume rendering by restricting Signed Distance Field (SDF) evaluations to regions where geometry actually exists, thereby eliminating false positives and redundant calculations. It achieves this by either resizing the voxel array to the minimal bounding box of the data or by using bounding volumes to cull expensive primitive evaluations during raymarching. Unlike sibling methods that focus on data compression or grid generation, this method specifically targets the traversal and evaluation phase by adapting the active computation domain to the scene's spatial extent.

### [Bucket-based LBVH construction](5163-bucket-based-lbvh-construction.md)
Bucket-based LBVH construction builds a Bounding Volume Hierarchy (BVH) from a voxel model by grouping voxels into dynamic buckets rather than using a uniform grid with a fixed brick size. This approach sets itself apart from siblings like [3D Voxel Grid Smoothing](833-3d-voxel-grid-smoothing.md) by enabling efficient empty-space skipping for sparse volumes, potentially yielding two to three times higher frame rates than methods without such skipping. It differs from [Bounding Volume Optimization](3583-bounding-volume-optimization.md) by utilizing a bucket-based spatial partitioning strategy specifically optimized for GPU-based parallel construction.

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
Voxelization has long been a fundamental operation in computer graphics for substituting polygonal models with volume representations to simplify storage, visibility calculations, and global illumination effects [S1, unofficial]. Early methods focused on binary occupancy, but modern techniques handle multi-channel scalar data and sparse structures. The introduction of GPU-based real-time voxelization allowed for dynamic scene processing, leading to optimizations like stream output caching and instanced slicing to handle deforming meshes [S1, unofficial][S3]. More recently, neural rendering methods like NeRF and its voxel-based variants (DVGO, Plenoxels) have driven new optimization strategies, such as spatial predictive coding and rate-distortion training, to compress dense voxel grids efficiently [S5].

## Sources
[S1] Two Simple Single-pass GPU methods for Multi-channel Surface Voxelization of Dynamic Scenes, https://doi.org/10.2312/pe/pg/pg2011short/031-036, fetched 2026-10-26 (unofficial)
[S2] kakashidinho/VoxelConeTracingMetal, https://github.com/kakashidinho/VoxelConeTracingMetal, fetched 2026-10-26 (unofficial)
[S3] CBARF: Cascaded Bundle-Adjusting Neural Radiance Fields From Imperfect Camera Poses, https://doi.org/10.1109/tmm.2024.3388929, fetched 2026-10-26
[S4] DTIC ADA606342: An Anti-Aliasing Technique for Splatting, https://archive.org/details/DTIC_ADA606342, fetched 2026-10-26 (unofficial)
[S5] SPC-NeRF: Spatial Predictive Compression for Voxel-Based Radiance Field, https://doi.org/10.1145/3795528, fetched 2026-10-26
