# Brick-based Hierarchical Storage

Also called: Brick Maps, Brick Pool, Multi-Resolution Brick Storage, ExaBricks, Compressed Segmentation Volume (CSV)

## Overview
Brick-based Hierarchical Storage organizes volume data into fixed-size cubic blocks ("bricks") to optimize memory locality, enable adaptive level-of-detail (LoD), and facilitate out-of-core rendering. It sets itself apart from flat dense grids by collapsing empty space and from pure tree structures by decoupling spatial traversal (often via BVH or page tables) from the data storage, allowing hardware-accelerated ray marching and efficient entropy coding of segmented labels.

## Shared principles
Brick-based Hierarchical Storage families share the following mechanism:

1.  **Brick Decomposition:** The 3D domain is partitioned into fixed-size cubic blocks (bricks). Each brick contains a dense sub-grid of voxels or labels. The brick size is typically a power of two (e.g., $8^3$, $16^3$, $32^3$) to facilitate efficient indexing and downsampling [S1, unofficial][S2].
2.  **Hierarchical or Sparse Indexing:** Bricks are managed via a higher-level structure that maps logical coordinates to resident bricks. This can be a sparse page table (mapping logical pages to a pool of resident bricks), a spatial partitioning tree (BVH, Octree), or a multi-resolution pyramid (MultiGrid) [S1, unofficial][S2].
3.  **Apdon Padding and Filtering:** To support hardware trilinear filtering across brick boundaries, bricks often include an "apron" or halo of voxels copied from neighboring logical cells. This ensures that sampling near the edge of a brick accesses valid data without requiring boundary checks or interpolation with external memory [S1, unofficial].
4.  **Adaptive Resolution and Compression:** The structure supports varying levels of detail. This is achieved either by storing multiple resolution levels per brick (brick maps/mipmaps) or by compressing the brick content using entropy coding (rANS) based on label frequency and spatial coherence (Compressed Segmentation Volume) [S1, unofficial][S2].
5.  **Empty Space Skipping:** The indexing structure allows the renderer to identify and skip regions of the volume that are empty or outside the transfer function range, significantly reducing ray marching steps [S1, unofficial][S2].

## Subtopics

### [ExaBricks Hierarchy](2182-exabricks-hierarchy.md)
ExaBricks is a method for rendering Block-Structured Adaptive Mesh Refinement (BS-AMR) data. It reorganizes irregular AMR cells into compact, non-overlapping bricks and uses Active Brick Regions (ABRs) within a BVH to enable smooth interpolation across level boundaries [2182-exabricks-hierarchy.md]. It sets itself apart by decoupling storage from spatial traversal, allowing RTX hardware acceleration while maintaining continuity at refinement interfaces. Choose this for AMR data where topological discontinuities must be handled smoothly without costly per-sample kernel lookups.

### [Sparse Page Table and Brick Pool](2550-sparse-page-table-and-brick-pool.md)
This representation avoids materializing a dense 3D grid by using a two-level structure: a page table mapping logical $8^3$-texel regions to resident bricks, and a pool of apron-padded bricks [2550-sparse-page-table-and-brick-pool.md]. It sets itself apart by bounding memory usage for large logical volumes (up to 2048 cells/axis) while enabling full source resolution and empty-space skipping. Choose this for large-scale, sparse volumes (e.g., weather data) where memory footprint must be strictly controlled and trilinear filtering is required.

### [Compressed Segmentation Volume](4985-compressed-segmentation-volume.md)
Compressed Segmentation Volume (CSV) is a lossless compression technique for segmented volumes. It encodes individual bricks using a multi-resolution label pyramid and entropy coding (rANS), enabling fast decompression and GPU-based rendering [4985-compressed-segmentation-volume.md]. It sets itself apart by separating label values from structural assignment logic, enabling efficient entropy coding and adaptive LoD access. Choose this for medical or geological segmentation data where label coherence is high and memory bandwidth is a bottleneck.

### Brick Maps
Brick Maps are a multi-resolution storage structure for 3D volume caches, analogous to 2D mipmaps, used to reduce aliasing (jiggle/noise) when rendering distant objects [S1, unofficial]. They are created by averaging 8 voxels from the previous level to form one voxel in the next level. The value of a voxel at level $n$ is calculated as:
$$ V_{n}(x,y,z) = 0.125 \times \sum_{i=0}^{1}\sum_{j=0}^{1}\sum_{k=0}^{1} V_{n-1}(2x+i, 2y+j, 2z+k) $$
The number of levels is $\log_2(baseResolution)$. This method is trivial only if the grid resolution is a power of two and is supported by formats like OpenVDB [S1, unofficial].

### Brick ID Page Table Layout
In multi-channel page table hierarchies, brick IDs are stored as unsigned 32-bit integers. The structure consists of 24 bits for the spatial offset (8 bits each for x, y, z coordinates relative to the origin) and remaining bits for a page table ID that uniquely identifies $(m \times k) \le 256$ page tables, where $m$ is the number of channels and $k$ is the number of resolution levels representable on the GPU [S2].

## Background
The concept of brick maps was coined by the Pixar Renderman team, adapting the 2D mipmap technique to 3D volume caches to handle aliasing in distant objects [S1, unofficial]. The idea of storing multiple resolutions is common in production renderers and formats like OpenVDB [S1, unofficial]. More recent developments include ExaBricks for AMR data [2182-exabricks-hierarchy.md], Sparse Page Tables for web-based volume rendering [2550-sparse-page-table-and-brick-pool.md], and Compressed Segmentation Volumes for efficient storage of labeled data [4985-compressed-segmentation-volume.md].

## Sources
[S1] Volume Rendering Based On 3D Voxel Grids, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-voxel-grids.html, fetched 2026-10-24 (unofficial)
[S2] Residency Octree: A Hybrid Approach for Scalable Web ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC10840607/, fetched 2026-10-29
