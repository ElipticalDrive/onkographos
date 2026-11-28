# Voxel Data Layout

Also called: Input Data Format, Node Data Layout, Voxel Cube, Voxel Data Structure (Naive), World Tensor

## Overview
Voxel Data Layout defines the fundamental memory organization for dense voxel grids, mapping 3D spatial coordinates to a linear or tensor-based storage structure. It solves the problem of efficient spatial indexing and boundary validation in GPU or CPU-based volume rendering by replacing multi-dimensional array overhead with linear memory access [S2, S4, unofficial]. Unlike hierarchical approaches like [Paged Block Array](12102-paged-block-array.md) which optimize for sparse data, or [Voxel Data Planar De-interleaving](6794-voxel-data-planar-de-interleaving.md) which optimizes for cache locality, this method focuses on the raw data representation and coordinate mapping [S4, unofficial]. It serves as the base layer for [Voxel Material Packing](4907-voxel-material-packing.md) and [Voxel Representation with Gaussian Lobes](4128-voxel-representation-with-gaussian-lobes.md) [G].

## How it works
**Inputs and Outputs:**
- Input: 3D world coordinates $(x, y, z)$ and grid dimensions $(W, H, D)$.
- Output: A pointer or index to the voxel data in memory, or the voxel attributes (type, color, opacity, etc.).

**Data Structures:**
The layout varies by implementation, but generally falls into two categories:

1.  **Naive Tensor/Cube Layout [S4, unofficial]:**
    -   Structure: A 3D tensor indexed by `world[x, y, z]`.
    -   Voxel Content: Each entry is a tensor containing 4 numbers: opacity $\sigma$ followed by an RGB triplet $(R, G, B)$.
    -   Out-of-bounds: Addressing a voxel position not in the world returns $[0, 0, 0, 0]$ [S4, unofficial].
    -   Scaling: Voxels are scaled such that edges are $1/8$th unit in length [S1, unofficial].

2.  **Linear Buffer Layout [S1, S2, S3, S6, unofficial]:**
    -   Structure: A contiguous 1D array (storage buffer) mapped from 3D coordinates.
    -   Voxel Size: Typically 32 bits per voxel [S1, unofficial].
    -   Bit Packing (32-bit example [S1, unofficial]):
        -   Bits 0-7: Voxel Type (8 bits).
        -   Bits 8-23: Color (16 bits).
        -   Bits 24-31: Auxiliary Data (8 bits).
    -   Indexing Formula [S6, unofficial]:
        $$index = x + W \times (y + H \times z)$$
        Where $W$ is ChunkWidth, $H$ is ChunkHeight, and $z$ is the depth index.
    -   Binary Format Order: Left-to-right, bottom-to-top, back-to-front [S2, unofficial].

**Numbered Steps for Access:**
1.  Validate coordinates $(x, y, z)$ against grid bounds $(W, H, D)$.
2.  If out of bounds, return default value (e.g., $[0,0,0,0]$ or empty voxel) [S4, unofficial].
3.  Calculate linear index using the layout-specific formula (e.g., $x + W(y + Hz)$) [S6, unofficial].
4.  Retrieve raw bytes from the buffer at `index * bytes_per_voxel`.
5.  Decode fields based on the bit-packing scheme (e.g., shift and mask for type, color, auxiliary) [S1, unofficial].

**Parameters:**
-   `INVALID_NODE`: Often defined as `NodeRef::MAX` (0xFFFFFFFF) for sparse variants [S3, unofficial].
-   `ROOT_NODE`: 0 [S3, unofficial].
-   `TextureAtlasSizeInBlocks`: 16 [S6, unofficial].
-   `NormalizedBlockTextureSize`: $1/16$ [S6, unofficial].
-   `TexturePadding`: 0.002f [S6, unofficial].

**Edge Cases and Pitfalls:**
-   **Precision:** Floating-point coordinate conversion to integer indices must handle negative coordinates correctly if the grid origin is not $(0,0,0)$.
-   **Boundary Checks:** Failure to check bounds leads to out-of-bounds memory access or wrapping errors in linear indexing [S4, unofficial].
-   **Endianness:** Binary formats stored in "left-to-right, bottom-to-top, back-to-front" order must be interpreted consistently across platforms [S2, unofficial].

## Subtopics

### [Voxel Representation with Gaussian Lobes](4128-voxel-representation-with-gaussian-lobes.md)
Voxel Representation with Gaussian Lobes is a method for compactly encoding directional scattering properties and occlusion within dense voxel grids, replacing single-value averages with isotropic Gaussian distributions to improve rendering accuracy. It solves the problem of directional aliasing and loss of anisotropic detail in standard voxel cone tracing by storing the mean direction and spread of light transport or surface normals. Unlike [Voxel Data Volume Storage and Access](4453-voxel-data-volume-storage-and-access.md) which typically stores scalar density or color, this method stores vector-valued directional statistics, and unlike [Voxel Renderer with Top-Level Grid and Bricks](8114-voxel-renderer-with-top-level-grid-and-bricks.md) which focuses on hierarchical traversal, this method defines the fundamental data layout for the leaf-level directional information.

### [Voxel Data Volume Storage and Access](4453-voxel-data-volume-storage-and-access.md)
This method defines the memory layout and access patterns for dense voxel volumes, storing data in a single contiguous 1D array mapped from 3D coordinates. It solves the problem of efficient spatial indexing and boundary validation in GPU or CPU-based volume rendering by replacing multi-dimensional array overhead with linear memory access. It differs from brick-based approaches [8114-voxel-renderer-with-top-level-grid-and-bricks.md] by using a flat grid, and from material packing [4907-voxel-material-packing.md] by focusing on raw data storage rather than attribute compression.

### [Voxel Material Packing](4907-voxel-material-packing.md)
Voxel Material Packing is a data layout for storing material properties and color within a single 32-bit voxel value. It allows up to 256 distinct materials to be managed by a MaterialManager. It builds on [Voxel Data Layout](4076-voxel-data-layout.md) [G].

### [Voxel Renderer with Top-Level Grid and Bricks](8114-voxel-renderer-with-top-level-grid-and-bricks.md)
This method renders voxel data using a two-level grid structure (top-level grid containing 8x8x8 voxel bricks) to optimize memory transfer and cache locality. It solves the bottleneck of transferring large voxel datasets to the GPU by synchronizing only changed bricks and leveraging high internal GPU bandwidth for staging. It differs from flat dense grid renderers by introducing spatial tiling for cache efficiency and dynamic update budgets.

### [Compact Voxel Mode](16284-compact-voxel-mode.md)
Compact Voxel Mode is a compile-time optional storage format that reduces voxel memory usage by 4x compared to the default mode by packing voxel data into a single byte per voxel. It solves the problem of excessive memory consumption in large-scale voxel scenes, enabling the loading of models that would otherwise exceed available RAM or hit volume limits. Unlike the default mode, which preserves full feature support, this mode sacrifices color depth, transparency, normals, and bone indices to achieve its memory efficiency.

### Volume Result Data Structure
**Layout:**
-   Structure: `VolumeResult` is a frozen dataclass with fields `density` (np.ndarray), `disturbance` (np.ndarray), and `metadata` (dict) [S5, unofficial].
-   Axis Order: Voxel axis order is Z, Y, X [S5, unofficial].
-   Data Type: Channel values are 8-bit linear data [S5, unofficial].

### Voxel Mesh Topology and Indexing
**Formula:**
-   Voxel index in a 1D array is calculated as: $index = x + ChunkWidth \times (y + ChunkHeight \times z)$ [S6, unofficial].

**Parameters:**
-   `TextureAtlasSizeInBlocks`: 16 [S6, unofficial].
-   `NormalizedBlockTextureSize`: $1/16$ [S6, unofficial].
-   `TexturePadding`: 0.002f [S6, unofficial].

**Layout:**
-   `VoxelData.VoxelVertices`: A `Vector3[]` of 8 unit cube corners: $(0,0,0), (1,0,0), (1,1,0), (0,1,0), (0,0,1), (1,0,1), (1,1,1), (0,1,1)$ [S6, unofficial].
-   `VoxelData.FaceChecks`: A `Vector3Int[]` of 6 neighbor offsets for face culling: $(0,0,-1), (0,0,1), (0,1,0), (0,-1,0), (-1,0,0), (1,0,0)$ [S6, unofficial].
-   `VoxelData.VoxelTriangles`: An `int[6,4]` array defining 6 faces with 4 vertex indices each:
    -   Face 0 (Z-): 0, 3, 1, 2
    -   Face 1 (Z+): 5, 6, 4, 7
    -   Face 2 (Y+): 3, 7, 2, 6
    -   Face 3 (Y-): 1, 5, 0, 4
    -   Face 4 (X-): 4, 7, 0, 3
    -   Face 5 (X+): 1, 2, 5, 6 [S6, unofficial].

### Anisotropic Voxel Layout
**Definition:**
-   Voxel storage format that stores six directional radiance values per voxel instead of one, preventing light from leaking through walls by ensuring surfaces only receive light from the front [S7, unofficial].

**Code:**
-   `VoxelLayout = Anisotropic ? new VoxelLayoutAnisotropic() : new VoxelLayoutIsotropic()` [S7, unofficial].

**Comparison:**
-   Differs from isotropic layout (1 value/voxel) by using ~6x memory and slower voxelization, but eliminates 'light leaks through the wall' artifacts [S7, unofficial].

## Operating characteristics
-   **Time Complexity:** Access is $O(1)$ given valid coordinates and precomputed dimensions [S4, S6, unofficial].
-   **Memory Complexity:** $O(W \times H \times D)$ for dense grids [S4, unofficial]. The naive tensor layout uses 4 floats (16 bytes) or 32 bits depending on implementation [S1, S4, unofficial]. Anisotropic layouts increase this by ~6x [S7, unofficial].
-   **Measured Performance:** Not given in the sources.
-   **Scaling:** Linear scaling with volume size. Cache locality is poor for large volumes without tiling (bricks) or de-interleaving [S4, S6, unofficial].
-   **Strengths:** Simple implementation, fast random access, low overhead for dense data [S4, unofficial].
-   **Limitations:** High memory usage for sparse data, poor cache locality for large volumes, no inherent support for hierarchical detail [S4, S7, unofficial].
-   **When to Use:** Small to medium dense volumes, debugging, or as a base for more complex layouts [S4, unofficial].
-   **When Not to Use:** Large sparse scenes, hierarchical rendering, or when memory bandwidth is a critical bottleneck without tiling [S4, S7, unofficial].

## Background
The naive voxel data structure originated as a simple tensor-based representation for volumetric rendering, often referred to as a "world tensor" [S4, unofficial]. It was used in early experiments with neural radiance fields (NeRF) and spherical harmonic representations before being replaced by more efficient structures like 27 spherical harmonic coefficients per voxel [S4, unofficial]. The linear buffer approach with bit-packing is common in real-time voxel engines like VoxelPlay [S1, unofficial]. The concept of anisotropic voxel layouts to prevent light leaks was developed in the context of voxel-based global illumination (GI) [S7, unofficial].

## Sources
[S1] Ray-Marched Voxel Simulation - Joris Rijsdijk, https://www.jorisar.nl/blog/VoxelPlayground/, fetched 2026-10-25 (unofficial)
[S2] SurfaceNets for Multi-Label Segmentations with Preservation of Sharp Boundaries., https://europepmc.org/article/MED/36325473, fetched 2026-10-26 (unofficial)
[S3] marceline-cramer/svo-cpu, https://github.com/marceline-cramer/svo-cpu, fetched 2026-10-27 (unofficial)
[S4] Total Internal Reflection, https://avishek.net/2022/12/05/pytorch-guide-plenoxels-nerf-part-2.html, fetched 2026-10-29 (unofficial)
[S5] mingtu149411/Noise-Cloud, https://github.com/mingtu149411/Noise-Cloud, fetched 2026-10-27 (unofficial)
[S6] philip2012/voxel-forge, https://github.com/philip2012/voxel-forge, fetched 2026-10-27 (unofficial)
[S7] Nicogo1705/StrideVoxelGI, https://github.com/Nicogo1705/StrideVoxelGI, fetched 2026-10-26 (unofficial)
