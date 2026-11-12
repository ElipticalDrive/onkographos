# Brick-based Hierarchical Storage


## Overview
Brick-based Hierarchical Storage organizes volumetric data into fixed-size, non-overlapping cubic blocks (bricks) to optimize memory locality, enable efficient sparse access, and support multi-resolution rendering [G]. It addresses the memory overhead and access inefficiencies of dense voxel grids by grouping data into spatially coherent units that can be loaded, compressed, or traversed independently. The family encompasses methods that use spatial partitioning for adaptive mesh refinement [2182-exabricks-hierarchy.md], sparse page tables for voxel datasets [2550-sparse-page-table-and-brick-pool.md], and compact encoding for segmentation labels [4985-compressed-segmentation-volume.md].

## Shared principles

## Subtopics

### [ExaBricks Hierarchy](2182-exabricks-hierarchy.md)
A rendering method for Block-Structured Adaptive Mesh Refinement (BS-AMR) data that reorganizes input cells into compact, non-overlapping bricks and builds a spatial index called Active Brick Regions (ABRs) to support smooth interpolation across level boundaries [2182-exabricks-hierarchy.md]. It is distinguished from siblings by its explicit construction of a secondary spatial index over brick support regions to enable hardware-accelerated traversal via an RTX BVH while maintaining $C^0$ continuity [2182-exabricks-hierarchy.md]. Choose this method when rendering AMR datasets that require high-quality interpolation across refinement boundaries without costly per-sample cell location kernels [2182-exabricks-hierarchy.md].

### [Sparse Page Table and Brick Pool](2550-sparse-page-table-and-brick-pool.md)
A sparse volume representation that avoids materializing a dense 3D grid by using a two-level structure: a page table mapping logical $8^3$-texel regions to resident bricks, and a pool of apron-padded bricks [2550-sparse-page-table-and-brick-pool.md]. It is distinguished from siblings by its explicit separation of spatial offset and page table ID within the brick ID structure and its use of a fixed-capacity brick pool with terrain occlusion handling [2550-sparse-page-table-and-brick-pool.md]. Choose this method when rendering large sparse voxel datasets (up to $2048^3$) in engines like Unreal Engine, where bounding memory usage and enabling empty-space skipping are priorities [2550-sparse-page-table-and-brick-pool.md].

### [Brick ID Page Table Layout](3192-brick-id-page-table-layout.md)
A memory-efficient indexing scheme used within brick-based hierarchical volume storage to map spatial brick locations to their corresponding page tables in a multi-channel, multi-resolution environment [3192-brick-id-page-table-layout.md]. It is distinguished from siblings by its specific bit-packing of spatial offsets and page table identifiers into fixed-width 32-bit integers, explicitly separating spatial offset and page table ID [3192-brick-id-page-table-layout.md]. Choose this method when implementing GPU-resident brick caches that require deterministic, cache-friendly lookups for multi-channel volumes with up to 256 page tables [3192-brick-id-page-table-layout.md].

### [Compressed Segmentation Volume](4985-compressed-segmentation-volume.md)
A lossless compression technique for voxel-based segmentation volumes that encodes individual bricks using a multi-resolution label pyramid and an iterative refinement scheme [4985-compressed-segmentation-volume.md]. It is distinguished from siblings by its explicit modeling of discrete, categorical labels through parent/neighbor reuse and palette back-references, rather than treating data as continuous fields [4985-compressed-segmentation-volume.md]. Choose this method when rendering large-scale segmented volumes with sharp, non-interpolating boundaries, as it significantly reduces memory bandwidth and storage requirements while supporting fast parallel decompression [4985-compressed-segmentation-volume.md].

### Brick Maps
#### definition
- [S1, unofficial] A multi-resolution storage structure for 3D volume caches, analogous to 2D mipmaps, used to reduce aliasing (jiggle/noise) when rendering distant objects.
#### step
- [S1, unofficial] Create downsized versions of the original grid data by averaging 8 voxels from the previous level to form one voxel in the next level.
- [S1, unofficial] Iterate through levels n=1 to numLevels-2, where baseResolution is halved each level.
#### formula
- [S1, unofficial] The value of a voxel at level n is the average of 8 voxels from level n-1: $V_{n}(x,y,z) = 0.125 \times \sum_{i=0}^{1}\sum_{j=0}^{1}\sum_{k=0}^{1} V_{n-1}(2x+i, 2y+j, 2z+k)$
#### parameter
- [S1, unofficial] numLevels: The number of resolution levels, calculated as $\log_2(baseResolution)$.
#### pitfall
- [S1, unofficial] The downsizing process is trivial only if the grid resolution is a power of 2.
#### usage
- [S1, unofficial] Supported by production renderers and formats such as OpenVDB.

### SGGX Storage Format
#### definition
- [S2] Stores the main indices of the SGGX matrix S in leaf nodes (LoD 0) using a compact linear space representation allowing 1 byte per parameter, totaling 6 bytes per distribution.

## Background
Brick-based storage concepts originated in the context of 3D texture caching and volume rendering optimization [G]. The term "brick map" was coined by the Pixar Renderman team, adapting the 2D mipmap concept to 3D voxel grids to handle aliasing, though the exact origin is uncertain [S1, unofficial]. The approach has since been generalized to handle adaptive mesh refinement [2182-exabricks-hierarchy.md], sparse voxel datasets [2550-sparse-page-table-and-brick-pool.md], and semantic segmentation volumes [4985-compressed-segmentation-volume.md].

## Sources
[S1] Volume Rendering Based On 3D Voxel Grids, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-voxel-grids.html, fetched 2026-10-24 (unofficial)
[S2] Fast Voxelization and Level of Detail for Microgeometry Rendering, https://doi.org/10.48550/arxiv.2604.13191, fetched 2026-10-26
