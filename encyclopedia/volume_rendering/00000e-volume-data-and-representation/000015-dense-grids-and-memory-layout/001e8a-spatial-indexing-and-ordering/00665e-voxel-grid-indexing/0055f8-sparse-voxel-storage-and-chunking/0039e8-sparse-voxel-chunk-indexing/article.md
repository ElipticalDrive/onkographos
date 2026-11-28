# Sparse Voxel Chunk Indexing

Also called: Bitmask Chunk Indexing, Chunk-based Voxel Indexing

## Overview
Sparse Voxel Chunk Indexing is a method for mapping between linear map indices and 3D spatial coordinates in a sparse voxel grid by organizing active voxels into fixed-size chunks (typically $8 \times 8 \times 8$) [S1, unofficial]. It solves the problem of efficient coordinate lookup and memory locality in sparse volumes by using bitmasks to track voxel occupancy within chunks, allowing direct access to chunk data while maintaining spatial coherence [S1, unofficial].

## How it works
**Inputs:** A linear array of chunk handles (or a map index), a linear buffer of voxel data, and the chunk's internal bitmask and partial counts [S1, unofficial].
**Outputs:** 3D integer coordinates $(x, y, z)$ or a linear voxel index [S1, unofficial].

**Data Structures:**
The core structure is the `Chunk`. It contains:
1. `bitMask[16]`: An array of 16 unsigned integers (64 bytes total). This represents an $8 \times 8 \times 8$ voxel grid (512 bits). Each bit corresponds to a voxel; a set bit indicates the voxel is active [S1, unofficial].
2. `partialCounts[3]`: An array of 3 integers storing cumulative voxel counts at the 1/4, 1/2, and 3/4 points of the chunk. This speeds up bit counting and coordinate mapping [S1, unofficial].
3. `voxelIndex`: A field in the `ChunkHandle` pointing to the start of this chunk's voxels in the linear voxel buffer [S1, unofficial].

**Coordinate Mapping (Map Index to 3D Position):**
To find the 3D position from a linear map index and chunk position:
1. Calculate `localIndex` within the chunk [S1, unofficial].
2. Determine `bitMaskIndex` by shifting `localIndex` right by 5 (integer division by 32) [S1, unofficial].
3. Initialize `voxNum` using `partialCounts` if `bitMaskIndex > 3` [S1, unofficial].
4. Loop from `bitMaskIndex & ~3` to `bitMaskIndex` [S1, unofficial].
5. For each mask, if it is the final index, mask out bits at and after the target voxel using `(1 << (localIndex & 31)) - 1` [S1, unofficial].
6. Add the bit count of the masked value to `voxNum` [S1, unofficial].
7. Return `map[mapIndex].voxelIndex + voxNum` [S1, unofficial].

**Position Mapping (Chunk Index and Voxel Number to 3D Position):**
To find the chunk position from a chunk index and a specific voxel number within that chunk:
1. Initialize `count=0`, `pos=0`, `startIndex=0` [S1, unofficial].
2. Loop `i` from 0 to 2. If `chunks[chunk].partialCounts[i] < voxNum`, set `count` to that partial count, add 128 to `pos`, add 4 to `startIndex`. Else break. Decrement `voxNum` by `count` [S1, unofficial].
3. Loop `i` from `startIndex` to 15. If `count == voxNum` break.
4. Get `bitCount` of `bitMask[i]`.
5. If `count + bitCount >= voxNum`, step bit-by-bit: loop `bitNum` from 0, add `(bitMask[i] >> bitNum) & 1` to count, increment `bitNum` and `pos` until `count == voxNum` [S1, unofficial].
6. Else add `bitCount` to `count` and 32 to `pos` [S1, unofficial].
7. Decrement `pos` [S1, unofficial].
8. If `count < voxNum` return `(-1,-1,-1)`. Else return `ivec3(pos % 8, (pos / 8) % 8, pos / 64)` [S1, unofficial].

**Pseudocode:**
The following Python-like pseudocode implements the mapping logic described in [S1, unofficial]. It assumes helper functions `bit_count(n)` (population count) and standard bitwise operations are available [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def map_index_to_voxel_index(map_index, local_index, chunk):
    """
    Calculates the linear voxel index from a map index and local chunk index.
    """
    bit_mask_index = local_index >> 5
    
    # partialCounts[0], [1], [2] store cumulative counts after masks 3, 7, 11
    if bit_mask_index > 3:
        if bit_mask_index >= 12:
            vox_num = chunk.partial_counts[2]
        elif bit_mask_index >= 8:
            vox_num = chunk.partial_counts[1]
        else:
            vox_num = chunk.partial_counts[0]
    else:
        vox_num = 0

    # Sum bits in full 32-bit words before the current one
    start_mask = bit_mask_index & ~3
    for i in range(start_mask, bit_mask_index):
        vox_num += bit_count(chunk.bit_mask[i])

    # Handle the final mask
    mask_val = (1 << (local_index & 31)) - 1
    masked_bits = chunk.bit_mask[bit_mask_index] & mask_val
    vox_num += bit_count(masked_bits)

    return chunk.voxel_index + vox_num

def voxel_number_to_position(chunk, vox_num):
    """
    Finds the 3D position (x, y, z) of the n-th active voxel in a chunk.
    """
    count = 0
    pos = 0
    start_index = 0
    
    # Check partial counts to skip initial masks
    for i in range(3):
        if chunk.partial_counts[i] < vox_num:
            count = chunk.partial_counts[i]
            pos += 128
            start_index = 4 * (i + 1)
            vox_num -= count  # [G] Fixed from source note "Increment voxNum" to correctly track remaining index
        else:
            break

    for i in range(start_index, 16):
        if count == vox_num:
            break
        
        bc = bit_count(chunk.bit_mask[i])
        
        if count + bc >= vox_num:
            # Step bit-by-bit
            bit_num = 0
            while count < vox_num:
                if (chunk.bit_mask[i] >> bit_num) & 1:
                    count += 1
                bit_num += 1
                pos += 1
                if count == vox_num:
                    break
        else:
            count += bc
            pos += 32
            
    pos -= 1
    
    if count < vox_num:
        return (-1, -1, -1)
    else:
        x = pos % 8
        y = (pos // 8) % 8
        z = pos // 64
        return (x, y, z)
```

**Edge Cases and Pitfalls:**
* **Total Internal Reflection:** In refraction calculations using this indexing, total internal reflection can cause infinite loops or lag. This is handled by checking if `rayDir == vec3(0.0)` after refraction calculation; if so, revert to `oldRayDir` [S1, unofficial].
* **Boundary Conditions:** If the voxel number exceeds the total active voxels in the chunk, the position mapping returns `(-1, -1, -1)` [S1, unofficial].
* **Precision:** Bitwise operations are integer-based and precise, but care must be taken with the `partialCounts` initialization and boundary checks to avoid off-by-one errors in coordinate mapping [S1, unofficial].

## Subtopics

### [GPU Accelerated Sparse Grid Construction](9812-gpu-accelerated-sparse-grid-construction.md)
This method constructs the **IndexGrid** acceleration structure on the GPU from a set of signed voxel coordinates, enabling the handling of large-scale inputs with a competitive memory footprint. Unlike standard NanoVDB which assumes a fixed topology, this algorithm supports dynamic sparsity patterns, such as those required for Level-of-Detail (LOD) hierarchies. It builds on [Sparse Voxel Chunk Indexing](8606-sparse-voxel-chunk-indexing.md) and implements the core construction logic for the [IndexGrid](9819-indexgrid.md) data structure.

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
Dictionary `sbs_data` with three fields: `index_array` (N×2 int32, stores [y, x] coordinates), `attribute_list` (N uint8, stores voxel values), `slice_offsets` (Z int32, stores cumulative start index per z-slice) [S2, unofficial].

## Operating characteristics
* **Time Complexity:** Coordinate mapping is $O(1)$ amortized due to the fixed chunk size and bitwise operations, avoiding the $O(\log n)$ traversal of octrees [S1, unofficial].
* **Memory Complexity:** Highly efficient for sparse volumes. The bitmask uses 1 bit per voxel (512 bits per chunk), and partial counts use minimal additional space. Active voxels are stored in a linear buffer, improving cache locality compared to pointer-based structures [S1, unofficial].
* **Performance:** Competitive memory footprint for large-scale inputs, especially when constructed on the GPU [S1, unofficial].
* **Strengths:** Spatial coherence, fast access times, low memory overhead for sparse data.
* **Limitations:** Fixed chunk size may lead to fragmentation if sparsity patterns are highly irregular. Bitwise operations can be complex to implement correctly [S1, unofficial].
* **When to Use:** Ideal for ray marching and stencil operations on sparse volumes where spatial coherence is important and memory bandwidth is a constraint [S1, unofficial].
* **When Not to Use:** If the volume is dense, the overhead of chunk management may not be justified compared to dense grids [G].

## Background
Sparse Voxel Chunk Indexing is a technique used in modern volume rendering and scientific visualization to manage large sparse datasets [G].

## Sources
[S1] frozein/DoonEngine, https://github.com/frozein/DoonEngine, fetched 2026-10-26 (unofficial)
[S2] GitHub - priscilla100/medical_volume_renderer, https://github.com/priscilla100/medical_volume_renderer/, fetched 2026-10-29 (unofficial)
