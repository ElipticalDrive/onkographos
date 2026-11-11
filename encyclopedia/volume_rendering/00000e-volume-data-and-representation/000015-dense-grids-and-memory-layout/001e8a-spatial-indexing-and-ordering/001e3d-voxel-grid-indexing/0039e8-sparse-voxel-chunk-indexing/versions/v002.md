# Sparse Voxel Chunk Indexing

Also called: Bitmask Chunk Indexing, Chunk-based Voxel Indexing

## Overview
Sparse Voxel Chunk Indexing maps a linear map index to a specific voxel within a sparse 8x8x8 chunk structure using bitmask arithmetic and cumulative counts, enabling efficient traversal of sparse volumetric data [S1, unofficial]. It distinguishes itself from hash-based spatial indexing by preserving strict spatial coherence and from octrees by using a fixed-depth shallow tree structure that avoids logarithmic access overhead [S1, unofficial].

## How it works
**Inputs:**
- `mapIndex`: The linear index in the global sparse voxel map.
- `map`: An array of chunk handles, where each handle contains `voxelIndex` (offset into the linear voxel buffer) and a reference to the chunk's bitmask/count data.
- `chunks`: An array of chunk structures containing `bitMask[16]` and `partialCounts[3]`.

**Outputs:**
- `voxelIndex`: The absolute linear index of the voxel in the global voxel buffer.
- `chunkPos`: The 3D integer coordinates $(x, y, z)$ of the chunk in the global grid.

**Data Structures:**
- **Chunk**:
  - `bitMask[16]`: Array of 16 unsigned integers (64 bytes total). Represents an 8x8x8 voxel grid (512 bits). Each bit corresponds to a voxel; a set bit indicates the voxel is present. The bits are packed in row-major order within the 3D chunk.
  - `partialCounts[3]`: Array of 3 integers. Stores cumulative voxel counts at the 1/4, 1/2, and 3/4 points of the chunk to accelerate binary search for bit positions.
- **ChunkHandle**:
  - `voxelIndex`: Integer offset pointing to the start of this chunk's valid voxels in the linear voxel buffer.

**Algorithm: Map Index to Voxel Index**
1. Retrieve the `ChunkHandle` at `mapIndex`.
2. Calculate `localIndex` within the chunk (0 to 511).
3. Determine `bitMaskIndex` by shifting `localIndex` right by 5 (`localIndex >> 5`).
4. Initialize `voxNum` (the count of set bits before the target) using `partialCounts` if `bitMaskIndex > 3`.
5. Loop from `bitMaskIndex & ~3` to `bitMaskIndex` (inclusive).
6. For each mask in the loop:
   - If it is the final index (the mask containing the target bit), mask out bits at and after the target voxel using `(1 << (localIndex & 31)) - 1`.
   - Add the population count (`bitCount`) of the masked value to `voxNum`.
7. Return `map[mapIndex].voxelIndex + voxNum`.

**Algorithm: Chunk Index and Voxel Number to Chunk Position**
1. Initialize `count = 0`, `pos = 0`, `startIndex = 0`.
2. Loop `i` from 0 to 2:
   - If `chunks[chunk].partialCounts[i] < voxNum`:
     - Set `count` to `chunks[chunk].partialCounts[i]`.
     - Add 128 to `pos`.
     - Add 4 to `startIndex`.
   - Else: Break.
3. Increment `voxNum` by 1.
4. Loop `i` from `startIndex` to 15:
   - If `count == voxNum`, break.
   - Get `bitCount` of `bitMask[i]`.
   - If `count + bitCount >= voxNum`:
     - Step bit-by-bit:
       - Initialize `bitNum = 0`.
       - Loop:
         - Add `(bitMask[i] >> bitNum) & 1` to `count`.
         - Increment `bitNum` and `pos`.
         - If `count == voxNum`, break inner loop.
   - Else:
     - Add `bitCount` to `count`.
     - Add 32 to `pos`.
5. Decrement `pos` by 1.
6. If `count < voxNum`, return `(-1, -1, -1)`.
7. Else, return `ivec3(pos % 8, (pos / 8) % 8, pos / 64)`.

**Pitfalls:**
- **Total Internal Reflection (TIR)**: In refraction calculations, TIR can cause infinite loops or significant lag. Handle by checking if `rayDir == vec3(0.0)` after refraction; if so, revert to `oldRayDir` [S1, unofficial].

## Subtopics

### [GPU Accelerated Sparse Grid Construction](9812-gpu-accelerated-sparse-grid-construction.md)
This method constructs the **IndexGrid** acceleration structure on the GPU from a set of signed voxel coordinates, enabling the handling of large-scale inputs with a competitive memory footprint. Unlike standard NanoVDB which assumes a fixed topology, this algorithm supports dynamic sparsity patterns, such as those required for Level-of-Detail (LOD) hierarchies. It serves as the foundational construction step for the IndexGrid data structure used in the fVDB framework.

### [IndexGrid](9819-indexgrid.md)
IndexGrid is a sparse data structure derived from NanoVDB, designed to aggressively accelerate stencil-centric operations (e.g., convolution) while serving as an effective acceleration structure for ray marching and other spatial operations. It solves the problem of memory inefficiency and code complexity in standard VDB implementations by separating topology (sparsity pattern) from values (features). Unlike hash tables, which lack spatial coherence, or octrees, which suffer from $O(\log n)$ access times, IndexGrid uses a fixed-depth shallow tree to maintain $O(1)$ amortized access while preserving spatial coherence.

### Slice-Based Binary Shell (SBS) Conversion
#### Definition
Converts a dense 3D volume into a sparse Slice-Based Binary Shell structure containing only non-empty voxels, organized by slice with cumulative offsets for fast GPU traversal [S2, unofficial].

#### Steps
1. Initialize `current_offset = 0` and iterate over each z-slice [S2, unofficial].
2. For each slice, find non-empty voxel coordinates where `slice_data > 0` [S2, unofficial].
3. Stack 2D indices into `[y, x]` format and append to `index_array` [S2, unofficial].
4. Extract corresponding voxel values and append to `attribute_list` [S2, unofficial].
5. Update `slice_offsets[z] = current_offset` and increment `current_offset += len(indices)` [S2, unofficial].
6. Convert lists to numpy arrays: `index_array` to int32, `attribute_list` to uint8 [S2, unofficial].

#### Layout
- Dictionary `sbs_data` with three fields:
  - `index_array`: N×2 int32, stores [y, x] coordinates [S2, unofficial].
  - `attribute_list`: N uint8, stores voxel values [S2, unofficial].
  - `slice_offsets`: Z int32, stores cumulative start index per z-slice [S2, unofficial].

## Operating characteristics
**Time Complexity:**
- Lookup is $O(1)$ amortized for chunk access, with bitmask traversal being proportional to the number of bits in the relevant mask words (constant for 8x8x8 chunks) [S1, unofficial].
- Construction depends on the underlying method (e.g., GPU construction is parallelized) [S1, unofficial].

**Memory Complexity:**
- Highly efficient for sparse data. Only stores bitmasks for occupied chunks and linear buffers for values [S1, unofficial].
- The 8x8x8 chunk size (512 bits) balances granularity with overhead [S1, unofficial].

**Strengths:**
- Preserves spatial coherence, beneficial for cache locality and stencil operations [S1, unofficial].
- Fixed-depth structure avoids pointer chasing and recursion overhead of octrees [S1, unofficial].
- Supports dynamic sparsity and LOD hierarchies [S1, unofficial].

**Limitations:**
- Fixed chunk size may lead to fragmentation or wasted space if sparsity patterns are highly irregular at the 8x8x8 scale [S1, unofficial].
- Bitmask arithmetic can be complex to implement correctly, especially regarding bit ordering and population counts [S1, unofficial].

**When to use:**
- When spatial coherence is critical (e.g., convolution, ray marching) and data is sparse but clustered [S1, unofficial].
- When dynamic sparsity or LOD is required [S1, unofficial].

**When not to use:**
- When data is dense, as the overhead of the index structure outweighs the benefits [S1, unofficial].
- When extreme irregularity at the chunk level leads to many empty or nearly empty chunks [S1, unofficial].

## Background
Sparse Voxel Chunk Indexing is part of the evolution of sparse voxel representations, building on [Voxel Grid Indexing](1719-voxel-grid-indexing.md). The specific implementation details described here are derived from the DoonEngine repository, which utilizes an IndexGrid structure inspired by NanoVDB [S1, unofficial]. The IndexGrid approach addresses limitations in standard VDB implementations by decoupling topology from values [S1, unofficial].

## Sources
[S1] frozein/DoonEngine, https://github.com/frozein/DoonEngine, fetched 2026-10-26 (unofficial)
[S2] GitHub - priscilla100/medical_volume_renderer, https://github.com/priscilla100/medical_volume_renderer/, fetched 2026-10-29 (unofficial)
