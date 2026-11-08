# Sparse Voxel Chunk Indexing

Also called: Bitmask Chunk Indexing, Chunk-based Voxel Indexing

## Overview
Sparse Voxel Chunk Indexing is a spatial indexing method that organizes a sparse 3D voxel grid into fixed-size $8 \times 8 \times 8$ chunks to optimize memory usage and traversal speed. It solves the problem of storing and accessing large, sparse volumetric datasets by only allocating memory for occupied voxels while maintaining fast coordinate-to-index mapping. It differs from uniform grid indexing by using a bitmask to track occupancy within chunks, allowing for significant memory savings in sparse scenes, and from pure sparse voxel grids by grouping voxels into fixed blocks to reduce pointer overhead and improve cache locality.

## How it works
**Inputs:** A map of chunk handles (containing `voxelIndex` offsets) and an array of chunk descriptors (containing `bitMask` and `partialCounts`).
**Outputs:** A linear voxel index for a given map/chunk position, or a 3D chunk position for a given chunk index and voxel number.

**Data Structures:**
The core structure is the `Chunk` descriptor, which represents an $8 \times 8 \times 8$ block (512 voxels).
*   `bitMask[16]`: An array of 16 unsigned integers (64 bits each). This represents the occupancy of the 512 voxels in the chunk. Each bit corresponds to one voxel. The layout is linear, typically following a specific traversal order (e.g., Z-order or linear row-major) within the chunk.
*   `partialCounts[3]`: An array of 3 integers storing cumulative voxel counts at the 1/4, 1/2, and 3/4 points of the chunk's bitstream. This accelerates rank queries (finding the $k$-th occupied voxel).
*   `voxelIndex`: Stored in the `ChunkHandle`, this is the starting offset (index) into the linear voxel data buffer where the values for this chunk's occupied voxels are stored.

**Algorithm: Map Index to Voxel Index**
Given a `mapIndex` (index into the chunk map) and a `localIndex` (the $k$-th occupied voxel within that chunk, $0 \le k < 512$):

1.  Calculate `bitMaskIndex` = `localIndex` >> 5 (integer division by 32). This identifies which of the 16 uints in `bitMask` contains the target bit.
2.  Initialize `voxNum` (the count of occupied voxels before the target) using `partialCounts`.
    *   If `bitMaskIndex > 3`, set `voxNum` = `partialCounts[bitMaskIndex & ~3]` (using the largest quarter-boundary index less than or equal to `bitMaskIndex`).
    *   Else, `voxNum` = 0.
3.  Loop `i` from `bitMaskIndex & ~3` (start of the current quarter) to `bitMaskIndex` (inclusive).
    *   If `i` is the final index (`i == bitMaskIndex`):
        *   Create a mask to isolate bits before the target: `targetMask` = `(1 << (localIndex & 31)) - 1`.
        *   Add `popcount(bitMask[i] & targetMask)` to `voxNum`.
    *   Else:
        *   Add `popcount(bitMask[i])` to `voxNum`.
4.  Return `map[mapIndex].voxelIndex + voxNum`.

**Algorithm: Chunk Position from Chunk Index and Voxel Number**
Given a `chunk` index (in the global chunk array) and a `voxNum` (the $k$-th occupied voxel globally within that chunk's context, or relative to the chunk start):

1.  Initialize `count = 0`, `pos = 0`, `startIndex = 0`.
2.  Loop `i` from 0 to 2:
    *   If `chunks[chunk].partialCounts[i] < voxNum`:
        *   `count` = `chunks[chunk].partialCounts[i]`
        *   `pos` += 128
        *   `startIndex` += 4
    *   Else:
        *   Break
3.  Increment `voxNum` by 1 (adjusting for 0-based vs 1-based counting if necessary, based on source logic).
4.  Loop `i` from `startIndex` to 15:
    *   If `count == voxNum`, break.
    *   Get `bitCount` = `popcount(chunks[chunk].bitMask[i])`.
    *   If `count + bitCount >= voxNum`:
        *   Step bit-by-bit within `chunks[chunk].bitMask[i]`:
            *   `bitNum` = 0
            *   While `count < voxNum`:
                *   `count` += `(chunks[chunk].bitMask[i] >> bitNum) & 1`
                *   `bitNum` += 1
                *   `pos` += 1
    *   Else:
        *   `count` += `bitCount`
        *   `pos` += 32
5.  Decrement `pos` by 1.
6.  If `count < voxNum`, return `(-1, -1, -1)` (error/out of bounds).
7.  Else, return `ivec3(pos % 8, (pos / 8) % 8, pos / 64)`.

**Pitfalls:**
*   **Total Internal Reflection (TIR):** In ray marching with refraction, TIR can cause the ray direction to become zero or oscillate, leading to infinite loops. Check if `rayDir == vec3(0.0)` after refraction; if so, revert to `oldRayDir` [S1, unofficial].
*   **Bitwise Precision:** Ensure `popcount` operations are efficient and correct for the target architecture.
*   **Boundary Conditions:** The chunk position calculation assumes valid `voxNum` ranges. Out-of-bounds access must be handled.

## Subtopics

### [GPU Accelerated Sparse Grid Construction](9812-gpu-accelerated-sparse-grid-construction.md)
A technique for constructing the IndexGrid acceleration structure on the GPU, enabling the framework to handle large-scale inputs with a competitive memory footprint.

### [IndexGrid](9819-indexgrid.md)
A novel sparse data structure derived from NanoVDB, designed with a re-imagined programming and execution model to aggressively accelerate stencil-centric operations (e.g., convolution) while serving as an effective acceleration structure for ray marching and other spatial operations.

## Operating characteristics
*   **Time Complexity:** Lookup is $O(1)$ for chunk access, but rank queries (finding the $k$-th voxel) require bit counting, which is $O(1)$ with hardware support or $O(N/32)$ with software popcount, where $N$ is the chunk size (512 bits).
*   **Memory Complexity:** $O(V)$ where $V$ is the number of occupied voxels, plus overhead for chunk descriptors. Each chunk uses 64 bytes for the bitmask plus 3 integers for partial counts, plus the handle. This is significantly more compact than pointer-based sparse grids for moderately dense chunks.
*   **Strengths:** High cache locality due to fixed-size chunks; efficient storage for sparse data; fast traversal for ray marching.
*   **Limitations:** Fixed chunk size may lead to fragmentation if occupancy is very low; bit manipulation can be complex to implement correctly on all hardware.
*   **When to use:** Large sparse volumetric datasets where memory bandwidth is a bottleneck and occupancy is not extremely sparse (empty chunks waste the bitmask overhead).
*   **When not to use:** Extremely sparse data where most chunks are empty (use a higher-level sparse grid or hash map); dense data where uniform grid is simpler and faster.

## Background
Sparse Voxel Chunk Indexing is used in modern volume rendering engines and scientific visualization tools to handle large datasets. It builds upon the concept of sparse voxel octrees but uses fixed-size blocks to simplify memory management and improve parallel processing efficiency. The specific implementation details, including the use of partial counts and bitmask traversal, are documented in the DoonEngine repository [S1, unofficial].

## Sources
[S1] frozein/DoonEngine, https://github.com/frozein/DoonEngine, fetched 2026-10-26 (unofficial)
