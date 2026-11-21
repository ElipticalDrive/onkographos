# Voxel Buffer Memory Layout and Loop Optimization

Also called: 3D-to-1D Voxel Indexing, Array-Order Layout, Cache-Optimized Voxel Access, Dense Field Implementation, Index Computation Optimization, Z-Order Layout

## Overview
This method defines the memory organization of a dense 3D voxel grid as a contiguous 1D array and the corresponding loop structures required to access it efficiently on modern hardware. It solves the problem of cache thrashing and memory bandwidth bottlenecks that occur when 3D coordinates are mapped to linear memory without regard for spatial locality or CPU/GPU cache line alignment [S2, unofficial][S3, unofficial]. It differs from [Voxel Memory Bank Allocation and Miniblock Organization](1638-voxel-memory-bank-allocation-and-miniblock-organization.md) by focusing on the global array indexing scheme rather than hardware-specific bank conflicts, and from [Volume Interleaving](2174-volume-interleaving.md) by keeping data for a single voxel type contiguous rather than interleaving multiple attributes [S2, unofficial][S3, unofficial]. The primary trade-off is between simple row-major indexing (fast index calculation, poor spatial locality for non-sequential access) and Z-order indexing (excellent spatial locality, high index calculation cost) [S3, unofficial].

## How it works
### Inputs and Outputs
- **Input**: 3D voxel grid dimensions $(x_{size}, y_{size}, z_{size})$ and a coordinate triplet $(i, j, k)$ representing the voxel to access.
- **Output**: A 1D linear index $I$ into a contiguous memory buffer, and the associated voxel data.

### Data Structures
The voxel buffer is a contiguous 1D array. In C++, this is typically implemented as `std::vector<T>` or a raw pointer allocated with `malloc`/`new` [S2, unofficial].
- **Layout**: Row-major order. The x-axis is the innermost dimension (varies fastest), followed by y, then z [S2, unofficial][S3, unofficial].
- **Total Size**: $x_{size} \times y_{size} \times z_{size}$ elements of type $T$ [S1, unofficial].
- **Lookup Tables (Optional)**: For optimized index computation, lookup tables are stored in GPU constant memory [S3, unofficial].

### Indexing Formulas
Two primary layout schemes are defined in the sources:

1.  **Array-Order (Row-Major)**:
    The index $I$ for coordinate $(i, j, k)$ is:
    $$ I = i + j \cdot x_{size} + k \cdot x_{size} \cdot y_{size} $$
    where $i \in [0, x_{size})$, $j \in [0, y_{size})$, $k \in [0, z_{size})$ [S2, unofficial][S3, unofficial].
    For cubic grids where $x_{size} = y_{size} = z_{size} = size$:
    $$ idx(x, y, z) = x + size \times (y + size \times z) $$
    [S1, unofficial]

2.  **Z-Order (Morton Order)**:
    A method where points nearby in index space are nearby in physical memory [S3, unofficial]. The index is computed via bitwise interleaving of the coordinates. The formulation by Pascucci and Frank is cited [S3, unofficial].

### Loop Optimization
To maximize cache hit rates, the iteration order must match the memory layout. The innermost loop must vary the fastest-changing dimension (x) [S2, unofficial].

**Naive (Incorrect) Loop:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
// x is outermost, z is innermost
for (int i = 0; i < xSize; ++i) {
  for (int j = 0; j < ySize; ++j) {
    for (int k = 0; k < zSize; ++k) {
      buffer.lvalue(i, j, k) = 0.0f;
    }
  }
}
```
This causes a memory stride of $x_{size} \cdot y_{size}$ between consecutive accesses in the inner loop, leading to a cache miss at every step [S2, unofficial].

**Optimized Loop:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
// x is innermost
for (int k = 0; k < zSize; ++k) {
  for (int j = 0; j < ySize; ++j) {
    for (int i = 0; i < xSize; ++i) {
      buffer.lvalue(i, j, k) = 0.0f;
    }
  }
}
```
This ensures sequential memory access [S2, unofficial].

### Index Computation Optimization
Direct multiplication for array-order indexing requires three multiplications and two additions per access [S3, unofficial]. Z-order calculation on the fly requires $12n$ bitwise operations (where $n$ is bits per dimension) [S3, unofficial]. To optimize:
1.  **Array-Order**: Use two table lookups and three additions [S3, unofficial].
2.  **Z-Order**: Use three table lookups and two logical OR operations [S3, unofficial].
These lookup tables are stored in GPU constant memory for fast access [S3, unofficial].

### Pseudocode
The following Python-like pseudocode demonstrates the optimized iteration and index calculation. It assumes a 1D list `buffer` and helper functions for the specific indexing schemes.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_voxel_buffer_array_order(buffer, x_size, y_size, z_size):
    """
    Processes a voxel buffer using array-order indexing with cache-optimized loops.
    """
    # Precompute strides to avoid repeated multiplication
    stride_y = x_size
    stride_z = x_size * y_size
    
    # Optimized loop: x is innermost
    for k in range(z_size):
        offset_z = k * stride_z
        for j in range(y_size):
            offset_y = offset_z + j * stride_y
            for i in range(x_size):
                # Index calculation: i + j*x_size + k*x_size*y_size
                idx = offset_y + i
                # Access buffer
                buffer[idx] = 0.0

def process_voxel_buffer_z_order(buffer, x_size, y_size, z_size, lookup_tables):
    """
    Processes a voxel buffer using Z-order indexing with lookup tables.
    lookup_tables: dict containing precomputed Morton codes for each dimension.
    """
    # Note: Z-order iteration is complex because linear iteration does not 
    # correspond to spatial traversal. This example assumes we iterate 
    # through the pre-sorted Z-order index list or use a specific traversal.
    # For simple linear iteration over the buffer (which destroys spatial locality 
    # benefits of Z-order unless the buffer itself is stored in Z-order), 
    # we just access by index.
    
    # If the buffer is stored in Z-order, the linear index IS the Morton code.
    # If we need to compute Morton code from (i,j,k) for random access:
    for k in range(z_size):
        for j in range(y_size):
            for i in range(x_size):
                # Compute Morton index using lookup tables
                # Morton(i, j, k) = morton_i | (morton_j << 1) | (morton_k << 2)
                # Using the optimized method: 3 lookups, 2 ORs
                morton_i = lookup_tables['x'][i]
                morton_j = lookup_tables['y'][j]
                morton_k = lookup_tables['z'][k]
                
                idx = morton_i | (morton_j << 1) | (morton_k << 2)
                
                # Access buffer
                buffer[idx] = 0.0
```

### Edge Cases and Pitfalls
- **Cache Thrashing**: Naive loops with the x-dimension outermost cause cache misses because the inner loop steps along the z-axis, resulting in a memory stride of $x_{size} \cdot y_{size}$ [S2, unofficial].
- **Index Calculation Cost**: Calculating Z-order index on the fly is computationally expensive ($12n$ bitwise operations) compared to lookup tables [S3, unofficial].
- **Bounds Checking**: Explicit loops require manual bounds checks. Iterators or `std::fill` are preferred to avoid explicit bounds checks and multiplication overhead [S2, unofficial].
- **Non-Cubic Grids**: The formulas must account for different $x_{size}$ and $y_{size}$; assuming cubic grids ($size^3$) leads to incorrect indexing if dimensions differ [S2, unofficial][S3, unofficial].

## Subtopics

### [Voxel Data Structure Layout](3584-voxel-data-structure-layout.md)
Voxel Data Structure Layout defines the memory organization of individual voxel elements within a dense grid, solving the problem of efficient storage and retrieval of geometric and material properties (such as density, color, normals, and occupancy). It differs from [Voxel Data Layout](4076-voxel-data-layout.md) by focusing on the payload content and memory footprint of a single voxel element rather than the array indexing scheme, and from [Paged Block Array](12102-paged-block-array.md) by defining the element structure rather than chunk allocation. It serves as the base layer for optimizations like [Voxel Material Packing](4907-voxel-material-packing.md) and [Voxel Renderer with Top-Level Grid and Bricks](8114-voxel-renderer-with-top-level-bricks.md) [G].

### Shared Position Buffer with Per-Chunk Color
An optimization where vertex positions for all voxels in all chunks are stored in a single global structured buffer (`m_shared_chunk_position_buffer`), while colors are stored per-chunk. This reduces memory bandwidth for positions if many chunks share similar geometry structures [S4, unofficial].
- **Steps**: Pre-calculate all vertex positions for all voxels in a chunk. For each voxel `i`, get its 3D index, calculate its offset in world space (`index_3d * EDGE_LENGTH`), and add the 8 local vertex offsets to create 8 global positions. Store these in `chunk_position_data` [S4, unofficial].
- **Layout**: 
  - Shared Position Buffer: `StructuredBuffer<DirectX::XMFLOAT3>`. Size is `Chunk::NUMBER_OF_VOXELS * 8` per chunk. Each voxel contributes 8 vertices [S4, unofficial].
  - Chunk Constant Buffer (`ChunkConstantBuffer`): Contains `translation_vector` (XMVECTOR4: chunk world offset), `position_buffer_index` (u32: SRV index of shared position buffer), `color_buffer_index` (u32: SRV index of this chunk's color buffer) [S4, unofficial].

## Operating characteristics
- **Time Complexity**: Index calculation is $O(1)$ for array-order with precomputed strides, or $O(1)$ with lookup tables for Z-order. On-the-fly Z-order calculation is $O(n)$ where $n$ is bit width [S3, unofficial].
- **Memory Complexity**: $O(x_{size} \cdot y_{size} \cdot z_{size})$ for the dense buffer [S1, unofficial].
- **Performance**: Cache-optimized loops (x-innermost) significantly outperform naive loops by eliminating cache misses [S2, unofficial]. Lookup tables for index computation are significantly faster than bitwise operations or multiplications [S3, unofficial].
- **Strengths**: Simple implementation, high throughput for sequential access, leverages hardware cache lines effectively [S2, unofficial].
- **Limitations**: Poor spatial locality for random access patterns compared to Z-order; high memory usage for very high resolutions ($>1000^3$) without sparsity [S2, unofficial].
- **When to Use**: When access patterns are sequential (e.g., ray marching, volume rendering) and cache efficiency is critical [S2, unofficial]. When random access to nearby voxels is frequent, Z-order may be preferred despite higher index cost [S3, unofficial].

## Background
The concept of mapping 3D grids to 1D arrays is fundamental to dense field implementations [S2, unofficial]. The specific optimization of loop ordering for cache performance is a standard technique in high-performance computing [S2, unofficial]. The Z-order (Morton) curve layout for volume rendering was explored in the context of parallel raycasting optimization, with formulations by Pascucci and Frank cited [S3, unofficial]. The shared position buffer optimization is a more recent technique in voxel engine implementations for reducing GPU memory bandwidth [S4, unofficial].

## Sources
[S1] Arablyna/CUDA-kernels-for-3D-SDF-voxel-sampling-and-2D-projection-visualization., https://github.com/Arablyna/CUDA-kernels-for-3D-SDF-voxel-sampling-and-2D-projection-visualization., fetched 2026-10-26 (unofficial)
[S2] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
[S3] Multi-core and Many-core Shared-memory Parallel Raycasting Volume Rendering Optimization and Tuning, https://digital.library.unt.edu/ark:/67531/metadc829701/?q=empty space skipping volume rendering, fetched 2026-10-28 (unofficial)
[S4] rtarun9/voxel-engine, https://github.com/rtarun9/voxel-engine, fetched 2026-10-27 (unofficial)
