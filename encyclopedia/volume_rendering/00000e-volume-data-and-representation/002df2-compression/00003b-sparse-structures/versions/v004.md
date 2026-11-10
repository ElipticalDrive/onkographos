# Sparse structures

Also called: Sparse volume representations, Adaptive voxel grids

## Overview
Sparse structures reduce memory usage by merging blocks of identical voxels into larger single voxels, recursively [S1, unofficial]. They solve the problem of storing empty space or homogeneous regions in volumetric data, which wastes memory in dense grids [S1, unofficial]. They differ from dense grids by storing only active cells and from tree-based structures by using flat linear index mapping (SparseGrid) or hash displacement (Hash Grids) rather than hierarchical pointers [S1, unofficial].

## Shared principles
All compression methods in this family share the following mechanism:

1.  **Redundancy Exploitation:** The core principle is to identify and eliminate redundancy in the volumetric data. This redundancy can be spatial (empty space, homogeneous regions), statistical (correlated values between neighbors), or structural (repetitive patterns in time-varying data) [G].
2.  **Transformation/Encoding:** Data is transformed into a domain where redundancy is more apparent or compact. This includes spatial subdivision (sparse structures), frequency domain transformation (wavelets), or dictionary-based representation (sparse coding, vector quantization) [G].
3.  **Quantization (Lossy):** For lossy compression, continuous or high-precision values are mapped to a discrete set of representatives. This step is critical for achieving high compression ratios but introduces reconstruction error, which must be managed via metrics like PSNR or perceptual quality [G].
4.  **Entropy Coding:** The transformed or quantized data is encoded into a bitstream using entropy coding techniques (e.g., Huffman, Arithmetic, rANS) to approach the Shannon entropy limit of the source [G].
5.  **Decompression/Reconstruction:** Rendering requires decoding the bitstream and reconstructing the volume data, either fully or partially (e.g., via LOD or adaptive loading), to provide input to the ray-marching or ray-casting pipeline [G].

## Subtopics

### [SparseGrid](904-sparsegrid.md)
SparseGrid is a sparse adaptive grid structure that allocates memory only for cells explicitly added to the domain, making it suitable for large volumetric spaces containing small iso-surface regions. It differs from dense grids by storing only active cell indices and their associated data on the device, and from tree-based structures by using a flat linear index mapping rather than hierarchical pointers. It serves as the foundational data container for GPU-accelerated marching tetrahedra and dual mesh generation pipelines.

### [Hash Grids](936-hash-grids.md)
Hash Grids are a sparse data structure used to map 3D spatial coordinates to data values without storing empty cells, primarily for ray tracing and volume rendering acceleration. Unlike dense voxel grids, they eliminate redundancy by using a perfect hash function to compress the spatial domain into a linear array, ensuring constant-time access to occupied cells while minimizing memory footprint. This method is distinct from tree-based structures (like Octrees) which use explicit pointers or bitmasks for hierarchy, and from brick maps which group voxels into fixed-size blocks; Hash Grids rely on mathematical displacement to resolve spatial collisions deterministically.

### [Stack-Based Representation (SBR)](1081-stack-based-representation-sbr.md)
Stack-Based Representation (SBR) is a sparse volumetric data structure that decomposes a voxel grid into vertical columns, or "stacks," where each stack is compacted using run-length encoding along the vertical axis. It solves the problem of storing layered volumetric data by exploiting vertical coherence, storing only the transitions in attribute values (such as density or color) rather than individual voxels. Unlike sibling structures like [SparseGrid](904-sparsegrid.md) or [Hash Grids](936-hash-grids.md) that rely on spatial hashing or uniform subdivision, SBR is inherently anisotropic, optimizing specifically for datasets with strong vertical layering, and serves as the foundational block for hierarchical variants like [QuadStack](1113-quadstack.md).

### [Pointer Compression](1770-pointer-compression.md)
Pointer Compression is a technique for compressing child pointers in sparse voxel structures, such as DAGs, by replacing absolute memory addresses with indices into a frequency-based lookup table. It sets itself apart from general entropy encoding by using a fixed-width, level-adaptive bit-packing scheme that explicitly handles both common relative pointers (via the table) and rare absolute pointers, thereby reducing memory usage at the cost of increased decoding complexity.

### [Brick-based Hierarchical Storage](2793-brick-based-hierarchical-storage.md)
Brick-based Hierarchical Storage organizes volumetric data into fixed-size, non-overlapping cubic blocks (bricks) to optimize memory locality, enable efficient sparse access, and support multi-resolution rendering. It solves the problem of inefficient memory access patterns and high memory overhead associated with dense voxel grids or deep tree traversals by decoupling logical spatial organization from physical storage. Unlike sibling sparse structures that rely on explicit pointers or hash tables for navigation, brick-based methods typically use spatial partitioning (ABRs, page tables) or compact encoding (CSV) to map logical coordinates to resident bricks, enabling $O(1)$ or $O(\log N)$ retrieval with minimal overhead.

### [Brickmap SDF Volume Rendering](3310-brickmap-sdf-volume-rendering.md)
Brickmap SDF Volume Rendering is a technique that represents 3D scenes as Constructive Solid Geometry (CSG) trees of Signed Distance Function (SDF) primitives, evaluates the resulting distance field on the CPU, compresses it into a sparse Brickmap structure, and raymarches through it on the GPU. It differs from direct GPU CSG evaluation by offloading the complex tree traversal and branching logic to the CPU, thereby avoiding high branching overhead on the GPU, and from dense volume rendering by using spatial compression to reduce memory footprint by 10-100x.

### [Compaction of Update Information](3364-compaction-of-update-information.md)
This method reduces the bandwidth of node update information sent from the GPU to the CPU during ray-guided volume rendering by eliminating duplicate node references using spatial coherence. It differs from sibling methods like [Stack-Based Representation](1081-stack-based-representation-sbr.md) by explicitly addressing the bandwidth bottleneck of feedback data through a GPU-optimized prefix-sum reduction (HistoPyramids) rather than relying solely on CPU-side data structures or simple list truncation.

### [Tree-based structures (Octree, Pointer Tree, Sparse Voxel DAG)](4139-tree-based-structures-octree-pointer-tree-sparse-voxel-dag.md)
Tree-based structures represent volumetric data by recursively subdividing the 3D domain into a hierarchy of nodes, where empty or homogeneous regions are pruned to achieve sparsity. This family solves the memory and computational inefficiency of dense voxel grids by allocating resources only to occupied space, enabling the storage and rendering of large, complex volumetric datasets on commodity hardware. It sets itself apart from sibling methods like Pointer Trees by standardizing the 8-way branching factor, and from Sparse Voxel DAGs by typically lacking the global deduplication of identical subtrees, though variants like Tree64 and OpenVDB introduce specific optimizations for traversal speed and random access respectively.

### [VDB-based Neural Rendering](8002-vdb-based-neural-rendering.md)
VDB-based Neural Rendering is a method that directly learns a VDB (hierarchical data structure for sparse volumes) from a set of posed images for real-time rendering. It sets itself apart from traditional NeRF by replacing the dense MLP with a sparse hierarchical structure inspired by classical computer graphics techniques such as Bounding Volume Hierarchy (BVH) and Sparse Voxel Octree (SVO) to model scenes in a sparse hierarchical form. This approach achieves a 10× to 20× speed-up over traditional NeRF, with rendering quality comparable to Neural Volumes and Scene Representation Networks.

### SILO Data Structure
The SILO data structure manages view-dependent subdivision of curvilinear grid cells into tetrahedra for rendering [S3, unofficial].
*   **Layout:** `NewCell` contains flags (`subdivided`, `oldsubdivided`), vertex pointers, and parent pointers. `NewFace` handles concave faces and subfaces. `Subface` stores plane equations ($Ax + By + Cz + D = 0$) and shared pointers to bounding cells. `BlockOfCells` manages pre-allocated blocks for pyramids, prisms, and hexahedra [S3, unofficial].
*   **Steps:** If a cell becomes subdivided (`subdivided=true`, `oldsubdivided=false`), a block is taken from pre-allocated arrays and shared pointers are revised. If it becomes un-subdivided, the parent pointer restores previous pointers and the block is freed. Exterior quadrilateral faces are divided into triangles for sorting [S3, unofficial].

### Block-based Sparse Buffer
A data structure that allocates memory more intelligently than dense buffers by subdividing the domain into coarse blocks, addressing the problem of unused voxels occupying memory [S4, unofficial].
*   **Layout:** Two-level-deep hierarchical structure. Domain is subdivided into coarse blocks (common size $8^3$ to $32^3$). Each block stores either a single value (if empty/unwritten) or an $N^3$ array of voxels [S4, unofficial].
*   **Steps:** A block remains unallocated until the first write-access to any of its voxels. Upon first write, all voxels in the block are allocated [S4, unofficial].
*   **Complexity:** Voxel read/write is $O(k)$ (constant time, but with larger $k$ than dense buffers). Allocation is amortized over subsequent accesses [S4, unofficial].
*   **Usage:** Implemented in Field3D's `SparseField` class [S4, unofficial].

### Sparse structure memory estimation
Memory usage follows surface area, not volume, for sparse structures [S5].
*   **Definition:** A sparse structure stores roughly the voxels near the surface; solid rock and empty sky collapse into cheap uniform nodes [S5].
*   **Parameters:** Roughness factor $k=2$ accounts for slopes, overhangs, and caves (real $k$ may be 1.5–4). Memory scales linearly with $k$ [S5].
*   **Cost:** Homogeneous interiors cost about the surface layer. Per-brick costs vary: raw bytes (~520 B) vs. compact (~210 B) [S5].

## Background
Sparse volumes were designed to address the waste of storing empty voxels or groups of identical density values in volumetric data [S1, unofficial]. The general idea involves creating a voxel the size of a $2 \times 2 \times 2$ block; if all voxels in the block have the same density, the block is deleted and the single value is stored in the larger voxel; otherwise, the larger voxel points to the original block [S1, unofficial]. This process is recursive, building lower levels from the highest resolution grid up [S1, unofficial]. OpenVDB uses an octree for upper levels but stores lower levels in $32 \times 32 \times 32$ blocks for cache coherency [S1, unofficial]. Block-based sparse buffers were introduced to avoid blind allocation of storage for every voxel, with Field3D providing an implementation [S4, unofficial]. Memory estimation for sparse structures indicates that for fully voxel planets, memory follows surface area, making large-scale voxel rendering affordable [S5].

## Sources
[S1] Volume Rendering Based On 3D Voxel Grids, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-voxel-grids.html, fetched 2026-10-24 (unofficial)
[S2] narnia-ai-mason/isomesh, https://github.com/narnia-ai-mason/isomesh, fetched 2026-10-27 (unofficial)
[S3] Cell projection of meshes with nonplanar faces, https://www.sciweavers.org/publications/cell-projection-meshes-non-planar-faces, fetched 2026-10-29 (unofficial)
[S4] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
[S5] https://github.com/theHimanshuShekhar/bhayanakvoxel/blob/research/voxel-data-structures/docs/research/voxel-data-structures.md, https://github.com/theHimanshuShekhar/bhayanakvoxel/blob/research/voxel-data-structures/docs/research/voxel-data-structures.md, fetched 2026-10-28
