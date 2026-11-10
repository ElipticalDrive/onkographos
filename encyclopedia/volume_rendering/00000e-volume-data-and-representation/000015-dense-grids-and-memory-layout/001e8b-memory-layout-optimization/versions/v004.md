# Memory Layout Optimization


## Overview
Memory Layout Optimization encompasses techniques for arranging volumetric data in contiguous memory buffers to maximize spatial locality, minimize cache misses, and eliminate hardware memory bank conflicts during rendering [S1, S2, S3, S4, unofficial]. It solves the problem where naive linear indexing causes cache thrashing or SDRAM bank conflicts, leading to performance degradation [S1, S1638]. It sets itself apart from sibling methods like [Evaluation Cache](874-evaluation-cache.md) (which caches computed values) and [Tile-based Volume Ray Marching](1678-tile-based-volume-ray-marching.md) (which optimizes traversal order) by focusing exclusively on the physical storage arrangement of the voxel data itself [S1, S1638, S2174].

## Shared principles
All methods in this family operate on the principle that memory access patterns during volume rendering (typically ray marching) must align with the physical layout of the data in RAM or VRAM to exploit hardware prefetching and burst capabilities [S1, S1638, S2174].

1.  **Spatial Locality:** Adjacent voxels in 3D space should be stored at adjacent memory addresses [S1, S2174]. This ensures that when a ray traverses the volume, consecutive samples hit the same or adjacent cache lines [S1, unofficial].
2.  **Stride Minimization:** The memory stride between consecutive accesses in the dominant traversal direction should be 1 (or the cache line size) [S1, unofficial]. Naive row-major layouts often result in large strides when traversing along the Z-axis if X is the innermost dimension [S1, unofficial].
3.  **Bank Conflict Avoidance:** For hardware implementations, memory addresses must be mapped to SDRAM banks such that concurrent requests from different threads/pipelines do not target the same bank simultaneously [S1638]. This is achieved through modulo arithmetic and skewing [S1638].
4.  **Block Decomposition:** Large volumes are often decomposed into fixed-size blocks (e.g., $32^3$ or $2^3$) to apply localized optimization rules, such as bit-interleaving or miniblock allocation [S1, S1638, S2174].

## Subtopics

### [Voxel Buffer Memory Layout and Loop Optimization](1598-voxel-buffer-memory-layout-and-loop-optimization.md)
This method optimizes the traversal of dense 3D voxel grids by aligning loop iteration order with the underlying 1D memory layout, thereby maximizing spatial locality and cache hit rates. It maps 3D coordinates to a contiguous array and reorders traversal loops to match the memory stride, differing from sibling methods that use bank-avoidance or bit-interleaving [G].

### [Voxel Memory Bank Allocation and Miniblock Organization](1638-voxel-memory-bank-allocation-and-miniblock-organization.md)
Voxel Memory Bank Allocation and Miniblock Organization is a hardware-oriented memory layout technique designed to maximize SDRAM burst read throughput and eliminate memory bank conflicts during dense volume rendering. It solves the problem of inefficient memory access by grouping voxels into small linear blocks (miniblocks) and applying modular arithmetic to distribute these blocks across memory modules and internal SDRAM banks. It sets itself apart from [Volume Interleaving](2174-volume-interleaving.md) by operating on fixed-size 2x2x2 spatial blocks rather than bit-interleaved linear arrays, and from [Memory-Insensitive Rendering](2774-memory-insensitive-rendering.md) by relying on strict spatial locality and skewing rather than algorithmic tolerance to random access.

### [Volume Interleaving](2174-volume-interleaving.md)
Volume Interleaving is a memory layout optimization technique designed to reduce cache misses in volume rendering by improving spatial locality. It achieves this by decomposing the volume into small, fixed-size blocks and rearranging the memory address of each voxel based on the interleaving of its coordinate bits. Unlike standard row-major or column-major layouts, which suffer from poor locality when traversing 3D space, Volume Interleaving ensures that spatially adjacent voxels are stored contiguously in memory. This method is distinct from [Voxel Memory Bank Allocation and Miniblock Organization](1638-voxel-memory-bank-allocation-and-miniblock-organization.md) in its specific use of bit-level interleaving for address generation rather than modulo-based bank distribution, and from [Memory-Insensitive Rendering](2774-memory-insensitive-rendering.md) which avoids layout optimization entirely by using streaming algorithms.

### [Memory-Insensitive Rendering](2774-memory-insensitive-rendering.md)
Memory-Insensitive Rendering (MIR) is an out-of-core technique for rendering arbitrarily large unstructured volumetric grids (specifically tetrahedral meshes) that requires only a constant, minimal amount of main memory. It solves the problem of rendering datasets exceeding available RAM by avoiding random access to the source geometry, relying instead on sequential disk scans and external sorting. Unlike sibling methods that optimize in-memory data layouts (e.g., [Voxel Buffer Memory Layout and Loop Optimization](1598-voxel-buffer-memory-layout-and-loop-optimization.md) or [Volume Interleaving](2174-volume-interleaving.md)), MIR prioritizes memory footprint over rendering speed, making it suitable for machines with severe memory constraints.

### DDS Channel Interleave/Deinterleave
- **Definition:** Converts between planar (deinterleaved) and interleaved byte layouts for multi-channel volume data, operating in blocks for memory efficiency [S1, unofficial].
- **Steps:**
    1. If `skip <= 1`, return immediately (no interleaving needed) [S1, unofficial].
    2. Allocate temporary buffer `data2` [S1, unofficial].
    3. For deinterleave (`restore=false`), scatter bytes: `for (ptr = data2, i = 0; i < skip; i++) for (j = i; j < bytes; j += skip) *ptr++ = data[j];` [S1, unofficial].
    4. For interleave (`restore=true`), gather bytes: `for (ptr = data, i = 0; i < skip; i++) for (j = i; j < bytes; j += skip) data2[j] = *ptr++;` [S1, unofficial].
    5. If `block > 0`, process data in chunks of `skip * block` bytes to limit peak memory usage [S1, unofficial].
- **Parameters:**
    - `skip`: Number of channels (stride) [S1, unofficial].
    - `block`: Chunk size for block-wise processing. 0 means full array [S1, unofficial].

### Canonical terms optimization
- **Definition:** An optimization for fields where every term is of the form $w$, $w \cdot a$, or $w \cdot a \cdot b$ (with $a$ even, $b=a+1$), storing them as three weights per field and column to avoid iterating over a term list [S2, unofficial]. Register-form kernels read every shell of a chunk from groupshared memory: centre, cutoff, $l$, AO count, primitives, and rows of $C$. Chunks packed to at most 256 primitives [S2, unofficial]. Canonical terms (`GTO_ISO_CANON_TERMS`, default 1): When every term of every field is $w \cdot a$, $w \cdot a^2$ or $w \cdot a \cdot b$ with $a$ even and $b = a + 1$ (orbitals, density matrices, products of pairs), the set keeps them as three weights per field and column [S2, unofficial].
- **Steps:**
    1. Stage a tile's weights in the epilogue [S2, unofficial].
    2. Each thread applies the staged weights to its own accumulators [S2, unofficial].
    3. Avoid rounds of 64 points over the term list [S2, unofficial].
    4. In the vertex kernel, put a vertex's four channels on neighbouring lanes [S2, unofficial].
    5. Shuffle the value channel to the gradient channels [S2, unofficial].
    6. Accumulate corner values in memory over tiles instead of in registers [S2, unofficial].
    7. Inner loop no longer waits on dependent global loads per shell. Each point skips shells out of its reach [S2, unofficial].
    8. Epilogues stage a tile's weights and every thread processes them [S2, unofficial].
- **Parameters:**
    - `GTO_ISO_CANON_TERMS`: Boolean flag enabling canonical terms; default value is 1 [S2, unofficial].

### Linear Span Arena
- **Definition:** A contiguous memory allocator for fixed-size spans (words) that tracks usage with a boolean array to enable fast allocation, deallocation, and reuse [S3, unofficial].
- **Steps:**
    1. `alloc` slides `sample_pos` backwards until `pos == 0` or `self.used[pos - 1]` is true to find `candidate_start` [S3, unofficial].
    2. Calculates `candidate_end = candidate_start + span_len` [S3, unofficial].
    3. If `candidate_end` exceeds data length, resets `next_alloc_start` to 0 and breaks [S3, unofficial].
    4. Samples `(sample_pos + 1..candidate_end)` in reverse to verify no used words exist [S3, unofficial].
    5. If free, sets range, updates `next_alloc_start`, and breaks [S3, unofficial].
    6. Marks `self.used[range].fill(true)` and copies `span_data` [S3, unofficial].
- **Parameters:**
    - `span_len`: length of the span to allocate in words [S3, unofficial].
    - `next_alloc_start`: tracks the next search position to avoid scanning from zero repeatedly; resets to 0 on wrap-around or after allocation [S3, unofficial].
- **Layout:**
    - `self.used`: boolean array tracking allocated spans [S3, unofficial].
    - `self.data`: underlying buffer for span data [S3, unofficial].
- **Code:**
    - `let candidate_start = { let mut pos = sample_pos; 'slide: loop { if pos == 0 || self.used[pos - 1] { break 'slide; } pos -= 1; } pos };` [S3, unofficial].
- **Pitfall:**
    - `populate_raw` rejects partial word data by panicking if `raw_data.len()` is not a multiple of `std::mem::size_of::<VxlData>()` [S3, unofficial].

### Avalanche Grid Memory Optimization
- **Definition:** A memory optimization technique for large volume grids (e.g., 1 mile x 1 mile x 1 mile at 6 inches resolution) by not allocating memory to empty voxels [S4, unofficial].
- **Parameters:**
    - Data type: 16-bit floats are used for density storage, as they are usually sufficient [S4, unofficial].
- **Numbers:**
    - For an Avalanche grid of 1 mile x 1 mile x 1 mile at 6 inches resolution (40,000 x 40,000 x 40,000), the implied memory is > 200 TB/frame, but actual memory usage is around 200 MB/frame due to sparse allocation [S4, unofficial].

## Background
Memory layout optimization in volume rendering originated from the need to overcome the limitations of CPU cache hierarchies and early GPU memory controllers. Early dense volume rendering suffered from severe cache thrashing due to the mismatch between 3D traversal patterns and 1D memory storage [S1, unofficial]. The concept of "Memory-Insensitive Rendering" was introduced to handle datasets larger than RAM by treating memory as a bottleneck to be bypassed via external sorting [S1, unofficial]. Hardware-specific optimizations, such as miniblock organization and bank skewing, were developed to exploit the burst capabilities of SDRAM in dedicated rendering hardware [S1638, S1, unofficial]. More recent techniques, such as Volume Interleaving, utilize bit-manipulation to improve spatial locality without changing the fundamental traversal logic [S1, unofficial]. In the context of GPU-based isosurface extraction, canonical term optimization reduces memory bandwidth pressure by restructuring how field terms are stored and accessed [S2, unofficial]. Sparse allocation techniques, as seen in the Avalanche grid example, demonstrate that for highly sparse volumes, avoiding allocation of empty space is the most effective memory optimization [S4, unofficial].

## Sources
[S1] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
[S2] md_gto_iso_gpu: optimized GPU isosurface extraction (non-blocking jobs, flood, one measured path), https://github.com/scanberg/mdlib/pull/15, fetched 2026-10-28 (unofficial)
[S3] MuffinTastic/rust-voxlap-vxl, https://github.com/MuffinTastic/rust-voxlap-vxl, fetched 2026-10-28 (unofficial)
[S4] PDF Production Volume Rendering - Clemson University, https://jtessen.people.clemson.edu/reports/papers_files/ProdVolRender.pdf, fetched 2026-10-27 (unofficial)
