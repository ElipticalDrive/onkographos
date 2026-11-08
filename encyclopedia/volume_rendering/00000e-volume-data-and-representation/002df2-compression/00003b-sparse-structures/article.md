# Sparse structures

Also called: Sparse volume representations, Sparse voxel structures

## Overview
Sparse structures reduce memory usage and bandwidth by eliminating empty space and merging homogeneous regions in volumetric data, addressing the inefficiency of dense voxel grids where most voxels are empty or uniform [S1, S4, unofficial][S5]. They differ from sibling methods like [Hash Grids](936-hash-grids.md) or [Tree-based structures](4139-tree-based-structures-octree-pointer-tree-sparse-voxel-dag.md) by providing the foundational mechanisms (subdivision, hashing, or block allocation) that enable these savings, often serving as the storage backend for higher-level techniques like [Out-of-Core Rendering](5359-out-of-core-rendering-with-virtual-page-tables.md) or [Brick-based Hierarchical Storage](2793-brick-based-hierarchical-storage.md).

## Shared principles
All compression methods in this family share the following mechanism:

1.  **Redundancy Exploitation:** The core principle is to identify and eliminate redundancy in the volumetric data. This redundancy can be spatial (empty space, homogeneous regions), statistical (correlated values between neighbors), or structural (repetitive patterns in time-varying data) [G].
2.  **Transformation/Encoding:** Data is transformed into a domain where redundancy is more apparent or compact. This includes spatial subdivision (sparse structures), frequency domain transformation (wavelets), or dictionary-based representation (sparse coding, vector quantization) [G].
3.  **Quantization (Lossy):** For lossy compression, continuous or high-precision values are mapped to a discrete set of representatives. This step is critical for achieving high compression ratios but introduces reconstruction error, which must be managed via metrics like PSNR or perceptual quality [G].
4.  **Entropy Coding:** The transformed or quantized data is encoded into a bitstream using entropy coding techniques (e.g., Huffman, Arithmetic, rANS) to approach the Shannon entropy limit of the source [G].
5.  **Decompression/Reconstruction:** Rendering requires decoding the bitstream and reconstructing the volume data, either fully or partially (e.g., via LOD or adaptive loading), to provide input to the ray-marching or ray-casting pipeline [G].

## Subtopics

### [SparseGrid](904-sparsegrid.md)
A sparse adaptive grid that allocates memory only for cells explicitly added to the domain, making it suitable for large 3D spaces containing small iso-surface regions. It differs from hash grids by maintaining an explicit lattice structure defined by a shape and axis-aligned bounding box (AABB), rather than relying solely on hash collisions for spatial mapping.

### [Hash Grids](936-hash-grids.md)
Hash Grids compress a static sparse 3D matrix by mapping non-empty voxel coordinates to a compact linear array using a perfect hash function, eliminating the storage overhead of empty cells. Unlike sibling structures like [SparseGrid](904-sparsegrid.md) or [Tree-based structures](4139-tree-based-structures-octree-pointer-tree-sparse-voxel-dag.md) which rely on dynamic allocation or tree traversal, Hash Grids achieve $O(1)$ access time with a fixed memory footprint determined by the grid dimensions and the number of occupied cells, making them ideal for static scenes where the spatial distribution is known beforehand.

### [Stack-Based Representation (SBR)](1081-stack-based-representation-sbr.md)
Stack-Based Representation (SBR) decomposes a voxel grid $V$ of resolution $w \times h \times d$ into a set $S$ of vertical stacks, compressing each column via run-length encoding of attribute values along the $z$-axis. It solves the storage problem for layered volumetric data by exploiting vertical coherence, separating geometric layer boundaries from attribute values. Unlike sibling structures like [SparseGrid](904-sparsegrid.md) or [Hash Grids](936-hash-grids.md) that rely on spatial hashing or tree traversal for arbitrary sparsity, SBR is specialized for datasets with strong vertical anisotropy, achieving linear construction complexity.

### [Pointer Compression](1770-pointer-compression.md)
Pointer compression is a sparse structure optimization that reduces the memory footprint of child pointers in tree-based representations (such as Sparse Voxel DAGs) by replacing absolute addresses with indices into a frequency-ordered lookup table. It sets itself apart from sibling methods like [Offset Compression](1819-offset-compression.md) by targeting frequent subtree references across tree levels, though it yields less compression than offset encoding because the majority of pointers reside at levels that still require full four-byte storage.

### [Brick-based Hierarchical Storage](2793-brick-based-hierarchical-storage.md)
Brick-based Hierarchical Storage organizes volume data into fixed-size cubic blocks ("bricks") to optimize memory locality, enable adaptive level-of-detail (LoD), and facilitate out-of-core rendering. It sets itself apart from flat dense grids by collapsing empty space and from pure tree structures by decoupling spatial traversal (often via BVH or page tables) from the data storage, allowing hardware-accelerated ray marching and efficient entropy coding of segmented labels.

### [Brickmap SDF Volume Rendering](3310-brickmap-sdf-volume-rendering.md)
Brickmap SDF Volume Rendering is a hybrid technique that represents 3D scenes as Constructive Solid Geometry (CSG) trees of Signed Distance Function (SDF) primitives, evaluates the resulting distance field on the CPU, compresses it into a sparse Brickmap structure, and raymarches through this compressed data on the GPU. It solves the problem of high memory consumption and GPU branching overhead associated with direct GPU-based CSG evaluation or dense volume grids. It differs from sibling sparse structures by explicitly decoupling the geometric evaluation (CPU) from the rendering traversal (GPU) and using a fixed-size brick compression scheme tailored for SDF data, achieving significant memory reduction compared to dense arrays.

### [Compaction of Update Information](3364-compaction-of-update-information.md)
A process to reduce the bandwidth of node information sent to the CPU by removing duplicate node references using spatial coherence and HistoPyramids.

### [Tree-based structures (Octree, Pointer Tree, Sparse Voxel DAG)](4139-tree-based-structures-octree-pointer-tree-sparse-voxel-dag.md)
Tree-based structures represent volumetric data using a recursive spatial subdivision where empty or uniform regions are collapsed into single nodes, providing compression proportional to data sparsity [2792-octree-based-structures.md]. This family solves the memory and bandwidth bottlenecks of dense voxel grids by enabling efficient culling of empty space and adaptive resolution. Unlike sibling structures like [Hash Grids](936-hash-grids.md) or [SparseGrid](904-sparsegrid.md), which rely on flat hashing or fixed brick maps, tree-based structures prioritize hierarchical traversal and spatial locality, allowing for variable branching factors and explicit parent-child relationships [2792-octree-based-structures.md].

### [Out-of-Core Rendering with Virtual Page Tables](5359-out-of-core-rendering-with-virtual-page-tables.md)
Out-of-core rendering with virtual page tables is a technique for visualizing volume caches that exceed available memory by loading only visible data bricks on demand. It solves the memory bottleneck of large-scale simulations by mapping logical volume regions to physical GPU memory pages via a flat page table, enabling dynamic, asynchronous data transfers. Unlike dense grid rendering, which requires the entire dataset in memory, this method builds on [Sparse structures](5-sparse-structures.md) to exploit sparsity and hierarchical storage, allowing interactive visualization of datasets that would otherwise be impossible to load. It differs from pure hierarchical traversal methods by using a flat page table to reduce indirections and page misses, prioritizing rendering speed over complex tree navigation during the frame loop.

### [VDB-based Neural Rendering](8002-vdb-based-neural-rendering.md)
A method that directly learns a VDB (hierarchical data structure for sparse volumes) from a set of posed images for real-time rendering.

### SILO Data Structure
The SILO data structure manages view-dependent subdivision of curvilinear grids using a hierarchical cell representation. Each cell is defined by a `NewCell` struct containing flags for subdivision state (`subdivided`, `oldsubdivided`), type, vertex count, and pointers to parent cells and faces [S3, unofficial]. Faces are represented by `NewFace` structs, which may contain `Subface` structures defining planar equations ($Ax + By + Cz + D = 0$) and shared pointers to bounding cells [S3, unofficial]. Subdivision logic allocates blocks from pre-allocated arrays (pyramids, prisms, hexahedra) when a cell transitions from un-subdivided to subdivided, and restores pointers to the free list when transitioning back [S3, unofficial]. Exterior quadrilateral faces are divided into triangles for sorting algorithms like XMPVO or BSP-MPVO [S3, unofficial].

### Block-based Sparse Buffer
A block-based sparse buffer is a two-level-deep hierarchical data structure that subdivides the domain into coarse blocks to address unused voxels in dense buffers [S4, unofficial]. Each block remains unallocated (storing a single value, typically zero) until the first write-access to any of its voxels triggers the allocation of the full $N^3$ array for that block [S4, unofficial]. Common block sizes range from $8^3$ to $32^3$ [S4, unofficial]. This approach provides $O(k)$ constant-time access (with a larger constant than dense buffers) and amortizes the cost of block allocation over subsequent accesses [S4, unofficial]. It is implemented in Field3D's `SparseField` class [S4, unofficial].

### Sparse structure memory estimation
In sparse structures, memory usage follows surface area rather than volume, as homogeneous interiors and empty space collapse into cheap uniform nodes [S5]. The memory cost scales linearly with a roughness factor $k$ (typically 1.5–4, often estimated at 2) which accounts for terrain complexity like slopes and overhangs [S5]. For a given surface area, the number of bricks crossing the surface is proportional to the surface voxel count divided by the brick volume (e.g., 64 for $4^3$ bricks), leading to a cost per surface voxel of approximately 3.3–8 bytes depending on the compression scheme (compact vs. raw) [S5].

## Background
Sparse volume representations originated to address the significant waste of storing empty or homogeneous voxels in dense grids, particularly in fluid simulations and cloud rendering where density varies primarily at edges [S1, unofficial]. Early implementations used recursive $2 \times 2 \times 2$ block merging to form octrees, with systems like OpenVDB optimizing for cache coherency by storing lower levels in $32 \times 32 \times 32$ blocks [S1, unofficial]. The concept of block-based sparse buffers was formalized in production volume rendering contexts to provide intelligent memory allocation for voxel buffers, avoiding blind allocation of every voxel [S4, unofficial]. Modern applications include out-of-core rendering of planetary-scale volumes, where memory scales with surface area, and neural rendering techniques that learn sparse hierarchical structures directly from images [S5, 8002-vdb-based-neural-rendering.md].

## Sources
[S1] Volume Rendering Based On 3D Voxel Grids, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-voxel-grids.html, fetched 2026-10-24 (unofficial)
[S2] narnia-ai-mason/isomesh, https://github.com/narnia-ai-mason/isomesh, fetched 2026-10-27 (unofficial)
[S3] Cell projection of meshes with nonplanar faces, https://www.sciweavers.org/publications/cell-projection-meshes-non-planar-faces, fetched 2026-10-29 (unofficial)
[S4] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
[S5] https://github.com/theHimanshuShekhar/bhayanakvoxel/blob/research/voxel-data-structures/docs/research/voxel-data-structures.md, https://github.com/theHimanshuShekhar/bhayanakvoxel/blob/research/voxel-data-structures/docs/research/voxel-data-structures.md, fetched 2026-10-28
