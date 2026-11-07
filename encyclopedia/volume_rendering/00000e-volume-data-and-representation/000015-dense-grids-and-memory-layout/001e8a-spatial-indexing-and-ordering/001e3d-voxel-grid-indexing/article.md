# Voxel Grid Indexing

Also called: Dense Voxel Grid, Voxel Grid Coordinate Mapping, Voxel Grids, Voxel Indexing, VoxelGridDesc

## Overview
Voxel Grid Indexing maps 3D voxel coordinates $(i, j, k)$ to a 1D linear memory index and vice versa, enabling efficient random access to dense volumetric data stored in contiguous arrays [S1, S2, S4, S8, unofficial]. It solves the problem of locating specific voxels in a 3D grid without traversing hierarchical structures, relying on arithmetic strides for $O(1)$ lookup [S8, unofficial]. Unlike [Morton Code Encoding/Decoding](1700-morton-code-encoding-decoding.md), which interleaves bits to preserve spatial locality for sparse structures, Voxel Grid Indexing uses simple row-major (or column-major) strides, offering the fastest possible query time at the cost of cubic memory complexity and poor locality for sparse data [S7, S8, unofficial].

## How it works
**Inputs:**
- 3D integer coordinates $(i, j, k)$ or a linear index $I$.
- Grid dimensions: $X, Y, Z$ (number of voxels along each axis) [S2, S4, unofficial].
- Optional padding parameters: `sectionSide` (base dimension) or `kPaddedSize` (dimension including border) [S1, S3, unofficial].

**Outputs:**
- Linear 1D index $I$ (integer) or 3D coordinates $(i, j, k)$.

**Data Structures:**
- **Dense Array:** A 1D array (e.g., `TArray<BlockType>`, `float[]`) of size $X \times Y \times Z$ (or padded equivalent) [S1, S8, unofficial].
- **Grid Descriptor:** Metadata containing dimensions ($X, Y, Z$), voxel size, and padding offsets [S5, unofficial][S11].

**Formulas:**
The standard row-major linearization maps $(i, j, k)$ to $I$ such that $i$ is the slowest-varying index and $k$ is the fastest [S5, unofficial]:
$$ I = i \cdot (Y \cdot Z) + j \cdot Z + k $$
where $Y \cdot Z$ is the stride for the $i$-axis and $Z$ is the stride for the $j$-axis [S2, S4, S5, unofficial].

For padded grids (e.g., to handle boundary checks without wrapping), the effective dimensions are increased by 1 (or more) [S1, S3, unofficial]:
$$ I_{padded} = i \cdot (Y_{pad} \cdot Z_{pad}) + j \cdot Z_{pad} + k $$
where $Y_{pad} = Y + 1$ and $Z_{pad} = Z + 1$ [S1, S3, unofficial].

The inverse mapping computes coordinates from $I$:
$$ i = \lfloor I / (Y \cdot Z) \rfloor $$
$$ j = \lfloor (I \pmod{Y \cdot Z}) / Z \rfloor $$
$$ k = I \pmod Z $$
[S1, S5, unofficial].

**Steps:**
1. **Forward Mapping (3D to 1D):**
   - Retrieve grid dimensions $X, Y, Z$.
   - If padding is required, calculate padded strides $S_y = Y_{pad} \cdot Z_{pad}$ and $S_z = Z_{pad}$.
   - Compute $I = i \cdot S_y + j \cdot S_z + k$.
   - Return $I$.

2. **Inverse Mapping (1D to 3D):**
   - Retrieve grid dimensions $X, Y, Z$.
   - Calculate strides $S_y = Y \cdot Z$ and $S_z = Z$.
   - Compute $i = I // S_y$.
   - Compute remainder $R = I \% S_y$.
   - Compute $j = R // S_z$.
   - Compute $k = R \% S_z$.
   - Return $(i, j, k)$.

3. **World-to-Index Conversion (Optional):**
   - Convert world position $\mathbf{p}$ to voxel coordinates using grid origin and cell size:
     $$ i = \lfloor (p_x - origin_x) / size_x \rfloor $$
     $$ j = \lfloor (p_y - origin_y) / size_y \rfloor $$
     $$ k = \lfloor (p_z - origin_z) / size_z \rfloor $$
   - Clamp indices to $[0, X-1], [0, Y-1], [0, Z-1]$ [S5, S6, unofficial].
   - Apply forward mapping to get $I$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linearize_index(i: int, j: int, k: int, size_x: int, size_y: int, size_z: int) -> int:
    """
    Maps 3D voxel coordinates to a 1D linear index using row-major layout.
    i: x-coordinate (slowest varying)
    j: y-coordinate
    k: z-coordinate (fastest varying)
    """
    return i * (size_y * size_z) + j * size_z + k

def unlinearize_index(index: int, size_x: int, size_y: int, size_z: int) -> tuple:
    """
    Maps a 1D linear index back to 3D voxel coordinates.
    """
    stride_yz = size_y * size_z
    stride_z = size_z
    
    i = index // stride_yz
    remainder = index % stride_yz
    j = remainder // stride_z
    k = remainder % stride_z
    
    return (i, j, k)

def world_to_voxel_index(x: float, y: float, z: float, 
                         origin_x: float, origin_y: float, origin_z: float,
                         voxel_size: float, 
                         size_x: int, size_y: int, size_z: int) -> int:
    """
    Converts world coordinates to a linear voxel index, with clamping.
    """
    # Convert world pos to voxel grid coordinates
    ix = int((x - origin_x) / voxel_size)
    iy = int((y - origin_y) / voxel_size)
    iz = int((z - origin_z) / voxel_size)
    
    # Clamp to grid bounds
    ix = max(0, min(ix, size_x - 1))
    iy = max(0, min(iy, size_y - 1))
    iz = max(0, min(iz, size_z - 1))
    
    return linearize_index(ix, iy, iz, size_x, size_y, size_z)
```

**Edge Cases and Pitfalls:**
- **Boundary Access:** Direct neighbor access (e.g., $i+1$) can wrap to the next row if not padded. Padding the grid dimensions by +1 in each axis prevents this [S1, S3, unofficial].
- **Integer Overflow:** For high-resolution grids, $X \cdot Y \cdot Z$ may exceed 32-bit integer limits. Use 64-bit integers for indices in large grids [S8, unofficial].
- **Coordinate System:** Ensure consistency between the indexing order (x-fastest vs z-fastest) and the memory layout of the backend (e.g., OpenGL textures vs CUDA arrays) [S5, unofficial].

## Subtopics

### [Coordinate Space Conversion](1785-coordinate-space-conversion.md)
Coordinate Space Conversion defines the deterministic mappings between world-space floating-point positions, continuous voxel coordinates, integer voxel indices, and block indices in a sparse voxel grid. It solves the problem of accurately locating which voxel contains a given point and translating between the discrete indexing required for memory access and the continuous coordinates required for interpolation or rendering. It differs from [World to UVW Normalization](2229-world-to-uvw-normalization.md) by operating on physical units and integer grid topology rather than normalized texture coordinates, and from [Bitstream Voxel Position Encoding](2371-bitstream-voxel-position-encoding.md) by providing the arithmetic transforms that precede or replace bit-packing.

### [Voxel Grid Initialization](2168-voxel-grid-initialization.md)
Voxel Grid Initialization is the process of constructing the active set of a sparse voxel grid from raw data sources, such as dense arrays, level-set functions, or file-based representations. It solves the problem of mapping continuous or dense discrete inputs into the sparse hierarchical structure required for efficient ray marching and volume rendering. It differs from [Voxel Grid Construction](3085-voxel-grid-construction.md) by focusing on the activation logic and memory allocation of the voxel table rather than the geometric vertex deduplication, and from [Grid Background Change](3210-grid-background-change.md) by establishing the initial active state rather than modifying existing values.

### [World to UVW Normalization](2229-world-to-uvw-normalization.md)
World to UVW Normalization maps arbitrary 3D world coordinates into the normalized $[0, 1]$ range of a voxel grid, enabling consistent indexing and texture sampling regardless of the scene's physical scale or origin. It differs from [Coordinate Space Conversion](1785-coordinate-space-conversion.md) by explicitly handling discrete voxel grid boundaries and dimensionality, and from [Texture Coordinate Conversion](3581-texture-coordinate-conversion.md) by operating at the grid indexing level rather than applying shader-specific offsets like Y-flips or centering adjustments [G].

### [Bitstream Voxel Position Encoding](2371-bitstream-voxel-position-encoding.md)
Bitstream Voxel Position Encoding is a method to encode the 3D position of a voxel into a linear bitstream for tree traversal, where each child index is determined by 3 bits. It solves the problem of mapping spatial coordinates to a hierarchical structure (such as an Octree) by creating a deterministic bit sequence that guides navigation from the root to the leaf node representing the voxel.

### [3D Texture Representation (CPU/GPU Indexing)](2393-3d-texture-representation-cpu-gpu-indexing.md)
3D Texture Representation stores voxel data in a dense 3D texture (or equivalent linear array) with dimensions matching the voxel grid, providing the simplest storage mechanism for volume rendering. Unlike tree-based structures like octrees, it fully leverages hardware acceleration for 3D textures, including caching and filtering, offering high runtime performance but low memory efficiency due to its inability to address sparse memory.

### [Uniform Grid Adjacent Cell Indexing](2992-uniform-grid-adjacent-cell-indexing.md)
Uniform Grid Adjacent Cell Indexing computes the linear memory indices of the four cells sharing a specific grid edge in a uniform voxel grid. It solves the problem of efficiently locating neighboring cells for operations such as multi-material interface definition without traversing a graph structure. It differs from its parent, [Voxel Grid Indexing](1719-voxel-grid-indexing.md), by specifically resolving the topology of edge-adjacent neighbors using arithmetic offsets based on edge orientation. It differs from [Multi-Material Cell Map Indexing & Access](4886-multi-material-cell-map-indexing-access.md) by computing raw linear indices rather than material-specific pointers.

### [Spatial Hashing](5002-spatial-hashing.md)
Spatial Hashing maps 3D world positions to hierarchical string-based spatial hashes (keys) and vice versa, enabling efficient traversal of a voxel hierarchy without storing explicit grid structures. It differs from flat spatial hashing by using a fixed-depth 12-level octal encoding where each character represents a 2x2x2 voxel block, allowing direct conversion between coordinates and region boundaries. Unlike [Bitstream Voxel Position Encoding](2371-bitstream-voxel-position-encoding.md) which stores raw bits, this method compresses position into a human-readable string for hierarchical queries.

### [Uniform Grid Indexing](6941-uniform-grid-indexing.md)
Uniform Grid Indexing is the standard row-major or column-major linearization of a dense 3D grid, serving as the baseline for all voxel indexing methods. It assumes a fixed, contiguous memory layout and provides $O(1)$ access time. It is the foundation for [Voxel Grid Indexing](1719-voxel-grid-indexing.md) but lacks the padding or coordinate transformation logic often required for robust boundary handling in rendering pipelines.

### [Sparse Voxel Grid](8021-sparse-voxel-grid.md)
A sparser version of a dense grid that only saves non-zero elements as a list of coordinates and corresponding data. It requires additional indexing logic (e.g., hash maps or trees) to map coordinates to data, contrasting with the direct arithmetic indexing of dense grids.

### [Memory Layout and Data Mapping](8133-memory-layout-and-data-mapping.md)
Memory Layout and Data Mapping defines how voxel data is arranged in memory (e.g., row-major, column-major, or structure-of-arrays) to optimize cache coherence and GPU memory coalescing. It determines the stride values used in the linearization formula and is critical for performance in high-throughput rendering [G].

### [Sparse Voxel Chunk Indexing](8606-sparse-voxel-chunk-indexing.md)
Sparse Voxel Chunk Indexing manages memory by dividing the voxel space into fixed-size chunks (e.g., 16x16x16), each stored in a separate contiguous block. It uses a global index to locate the chunk and a local index to locate the voxel within the chunk, combining the efficiency of dense arrays with the sparsity support of hierarchical structures.

### Data Collection from Indices
Data Collection from Indices extracts data from a tensor given a list of indices, handling both 2D and 3D data tensors. It reshapes data to 2D (flattening the first two dimensions) if necessary, calculates a flattened point index by combining index and count indices, and gathers the corresponding values. This is often used in differentiable rendering pipelines to sample voxel features [S9, unofficial].

### Grid Embedding
Grid Embedding is a learnable 4D tensor lookup table that stores shape-specific latent codes indexed by model ID, structured as a dense grid. The forward pass gathers embeddings via `self.embeddings[indices]` and reshapes the result to `(n_models, grid_size+1, grid_size+1, grid_size+1, -1)` to preserve the 4D grid structure across the batch dimension. The tensor shape is `(n, grid_size+1, grid_size+1, grid_size+1, latent_dim)` and is stored as an `nn.Parameter` for gradient tracking [S10, unofficial].

### Coarse Geometry Searching
Coarse Geometry Searching is a two-stage optimization process where the first stage efficiently finds the coarse 3D areas of interest (geometry) using a low-resolution voxel grid with view-invariant color, reducing queried points for the fine stage [S11]. It initializes the coarse density voxel grid $V^{(density)(c)}$ to 0 and sets a bias $b$ to ensure initial transmittance is close to 1:
$$ b = \log \left( (1 - \alpha^{(init)(c)})^{-\frac{1}{s^{(c)}}} - 1 \right) $$
where $\alpha^{(init)(c)}$ is a hyperparameter and $s^{(c)}$ is the voxel size [S11]. It also applies a view-count-based learning rate scaling, where the learning rate for each grid point is scaled by $n_j / n_{max}$, with $n_j$ being the number of training views visible to point $j$ [S11]. The coarse density grid is stored in $\mathbb{R}^{1 \times N_x^{(c)} \times N_y^{(c)} \times N_z^{(c)}}$ and color in $\mathbb{R}^{3 \times N_x^{(c)} \times N_y^{(c)} \times N_z^{(c)}}$ [S11].

## Operating characteristics
- **Time Complexity:** $O(1)$ for forward and inverse mapping [S8, unofficial].
- **Memory Complexity:** $O(N^3)$ where $N$ is the resolution along one axis, as the entire volume is allocated regardless of occupancy [S7, S8, unofficial].
- **Performance:** Provides the fastest possible query time for dense data due to simple arithmetic and cache-friendly linear access [S8, unofficial].
- **Strengths:** Simple implementation, no overhead for tree traversal or hashing, optimal for dense volumes or when memory is not a constraint [S8, unofficial].
- **Limitations:** Prohibitive memory usage for high-resolution or sparse scenes; poor spatial locality for sparse data compared to Morton codes or octrees [S7, S8, unofficial].
- **When to Use:** Small to medium dense volumes, dense fields in neural rendering (e.g., Direct Voxel Grid Optimization), or when hardware 3D texture sampling is leveraged [S1, unofficial][S11].
- **When Not to Use:** Large-scale sparse scenes, outdoor environments, or any scenario where memory footprint is critical [S7, S8, unofficial].

## Background
Voxel grids are one of the earliest methods for volume representation, predating modern neural rendering [S7, unofficial]. The naive approach of explicit discretization into cubes storing metadata in an array has been widely used in computer graphics for collision detection, voxelization of meshes, and early volume rendering [S2, S7, unofficial]. The linear indexing method described here is fundamental to GPU texture sampling and dense field representations in modern neural radiance fields (NeRF) variants, such as Direct Voxel Grid Optimization, which leverages the $O(1)$ access speed for fast convergence [S11]. While NeRF and other implicit methods exploit symmetries and continuous functions, voxel grids remain relevant for their explicit, queryable nature and hardware acceleration support [S7, unofficial].

## Sources
[S1] Manolo94/UE5MinecraftClone, https://github.com/Manolo94/UE5MinecraftClone, fetched 2026-10-26 (unofficial)
[S2] topskychen/voxelizer, https://github.com/topskychen/voxelizer, fetched 2026-10-27 (unofficial)
[S3] dmadevice/streaming-binary-greedy-meshing, https://github.com/dmadevice/streaming-binary-greedy-meshing, fetched 2026-10-27 (unofficial)
[S4] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
[S5] wlim33/vox-ocean, https://github.com/wlim33/vox-ocean, fetched 2026-10-27 (unofficial)
[S6] https://github.com/google-research/google-research/blob/master/tf3d/utils/voxel_utils.py#L305, https://github.com/google-research/google-research/blob/master/tf3d/utils/voxel_utils.py, fetched 2026-10-27 (unofficial)
[S7] A Visual Exploration of Neural Radiance Fields, https://djrhails.github.io/post--radiance-fields/, fetched 2026-10-29 (unofficial)
[S8] Faster Neural Radiance Fields Inference — Chris Choy, https://chrischoy.org/posts/nerfs/, fetched 2026-10-29 (unofficial)
[S9] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
[S10] TurtleJayWang/VC-SDFGen, https://github.com/TurtleJayWang/VC-SDFGen, fetched 2026-10-28 (unofficial)
[S11] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
