# Brick-based Hierarchical Storage

Also called: Brick Maps, ExaBricks, Sparse Page Table and Brick Pool, Compressed Segmentation Volume (CSV)

## Overview
Brick-based Hierarchical Storage organizes volumetric data into fixed-size, non-overlapping cubic blocks (bricks) to optimize memory locality, enable efficient sparse access, and support multi-resolution rendering. It solves the problem of inefficient memory access patterns and high memory overhead associated with dense voxel grids or deep tree traversals by decoupling logical spatial organization from physical storage [S1, S3, unofficial]. Unlike sibling sparse structures that rely on explicit pointers or hash tables for navigation, brick-based methods typically use spatial partitioning (ABRs, page tables) or compact encoding (CSV) to map logical coordinates to resident bricks, enabling $O(1)$ or $O(\log N)$ retrieval with minimal overhead [S1, S3, unofficial].

## Shared principles
All compression methods in this family share the following mechanism:

1.  **Redundancy Exploitation:** The core principle is to identify and eliminate redundancy in the volumetric data. This redundancy can be spatial (empty space, homogeneous regions), statistical (correlated values between neighbors), or structural (repetitive patterns in time-varying data) [G].
2.  **Transformation/Encoding:** Data is transformed into a domain where redundancy is more apparent or compact. This includes spatial subdivision (sparse structures), frequency domain transformation (wavelets), or dictionary-based representation (sparse coding, vector quantization) [G].
3.  **Quantization (Lossy):** For lossy compression, continuous or high-precision values are mapped to a discrete set of representatives. This step is critical for achieving high compression ratios but introduces reconstruction error, which must be managed via metrics like PSNR or perceptual quality [G].
4.  **Entropy Coding:** The transformed or quantized data is encoded into a bitstream using entropy coding techniques (e.g., Huffman, Arithmetic, rANS) to approach the Shannon entropy limit of the source [G].
5.  **Decompression/Reconstruction:** Rendering requires decoding the bitstream and reconstructing the volume data, either fully or partially (e.g., via LOD or adaptive loading), to provide input to the ray-marching or ray-casting pipeline [G].

## Subtopics

### [ExaBricks Hierarchy](2182-exabricks-hierarchy.md)
ExaBricks is a rendering method for Block-Structured Adaptive Mesh Refinement (BS-AMR) data that reorganizes input cells into compact, non-overlapping bricks to enable high-quality smooth interpolation across level boundaries and efficient hardware-accelerated ray tracing [2182-exabricks-hierarchy.md]. It solves the problem of costly cell location kernels and discontinuous interpolation at AMR refinement boundaries by decoupling storage (bricks) from spatial traversal (Active Brick Regions) [2182-exabricks-hierarchy.md]. Unlike prior methods that require deep tree traversals or stitching unstructured meshes, ExaBricks uses a spatial partitioning of brick support regions to provide $O(1)$ cell retrieval for reconstruction and supports adaptive sampling via an RTX BVH [2182-exabricks-hierarchy.md].

### [Sparse Page Table and Brick Pool](2550-sparse-page-table-and-brick-pool.md)
Sparse Page Table and Brick Pool is a sparse volume representation that avoids materializing a dense 3D grid by using a two-level structure: a page table mapping logical $8^3$-texel regions to resident bricks, and a pool of apron-padded bricks [2550-sparse-page-table-and-brick-pool.md]. It allows full source resolution (up to 2048 cells per axis) while bounding memory usage and enabling empty-space skipping [2550-sparse-page-table-and-brick-pool.md]. It is implemented in Unreal Engine as "Heterogeneous Volumes" and "Sparse Volume Textures" (SVT) [2550-sparse-page-table-and-brick-pool.md]. Unlike dense voxel grids, it significantly reduces memory footprint by storing only non-empty tiles [2550-sparse-page-table-and-brick-pool.md].

### [Brick ID Page Table Layout](3192-brick-id-page-table-layout.md)
Not given in the sources.

### [Compressed Segmentation Volume](4985-compressed-segmentation-volume.md)
Compressed Segmentation Volume (CSV) is a lossless compression technique for voxel-based segmentation volumes that encodes individual bricks using a multi-resolution label pyramid and entropy-coded operation sequences [4985-compressed-segmentation-volume.md]. It solves the storage and transfer bottlenecks of large segmentation datasets by achieving compression ratios of 1–3% of the original data size while supporting fast, parallel decompression and GPU-based level-of-detail (LoD) rendering [4985-compressed-segmentation-volume.md]. Unlike general-purpose compressors (e.g., HDF5, PNG) or specialized segmentation tools (e.g., Compresso), CSV exploits the local homogeneity of label regions through parent/neighbor reuse and palette back-references, enabling efficient on-the-fly decompression without full volume reconstruction [4985-compressed-segmentation-volume.md].

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
Brick-based storage concepts originated in the context of 3D texture caching and volume rendering optimization. The term "brick map" was coined by the Pixar Renderman team, adapting the 2D mipmap concept to 3D voxel grids to handle aliasing and memory bandwidth constraints [S1, unofficial]. The approach has since been generalized to handle adaptive mesh refinement (ExaBricks), sparse voxel datasets (Sparse Page Table), and semantic segmentation volumes (CSV), reflecting a broader industry shift towards block-based data structures for efficient GPU rendering [2182-exabricks-hierarchy.md, 2550-sparse-page-table-and-brick-pool.md, 4985-compressed-segmentation-volume.md].

## Sources
[S1] Volume Rendering Based On 3D Voxel Grids, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-voxel-grids.html, fetched 2026-10-24 (unofficial)
[S2] Fast Voxelization and Level of Detail for Microgeometry Rendering, https://doi.org/10.48550/arxiv.2604.13191, fetched 2026-10-26
