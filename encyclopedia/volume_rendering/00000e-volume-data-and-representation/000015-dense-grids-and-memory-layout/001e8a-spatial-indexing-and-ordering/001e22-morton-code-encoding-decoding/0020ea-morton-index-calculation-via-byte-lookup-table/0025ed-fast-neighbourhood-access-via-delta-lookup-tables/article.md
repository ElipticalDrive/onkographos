# Fast Neighbourhood Access via Delta Lookup Tables

Also called: Morton neighbour peeking, deltaX/Y/Z lookup

## Overview
A method to access immediate neighbours of a voxel in Morton-ordered memory without re-encoding coordinates, using precomputed offset tables that map a coordinate component to the memory stride required to reach its neighbour [S1, unofficial]. It solves the performance bottleneck of decoding a Morton index to $(x,y,z)$, modifying coordinates, and re-encoding, by exploiting the property that the Morton index delta $\Delta p$ for a single-axis step depends only on that axis's coordinate value [S1, unofficial]. This distinguishes it from full re-encoding methods, which require decoding and re-encoding coordinates, and from bitwise manipulation approaches by relying on precomputed static lookup tables [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Current voxel pointer `p` (memory address or linear index), current chunk-local coordinates $(x, y, z)$, and three lookup tables $\Delta X, \Delta Y, \Delta Z$ [S1, unofficial].
- Output: Pointer or linear index to the target neighbour [S1, unofficial].

**Data Structures:**
- `deltaX`, `deltaY`, `deltaZ`: Arrays of size 256 (for 8-bit coordinates), storing `int32_t` offsets [S1, unofficial]. Each entry stores the integer offset to add to the current Morton index to reach the neighbour in the respective axis [S1, unofficial].
- Boundary check macros (e.g., `CAN_GO_POS_X`) verify that the target coordinate remains within the current chunk before applying the delta [S1, unofficial].

**Steps:**
1. **Precompute Tables:** Generate $\Delta X, \Delta Y, \Delta Z$ by computing the Morton index for all $i \in [0, 255]$ and storing the offset to reach $i+1$ [S1, unofficial].
2. **Access Single-Axis Neighbour:** To access $(x+1, y, z)$, check the boundary macro for $x$ [S1, unofficial]. If valid, return `p + deltaX[x]`. Otherwise, fallback to a global coordinate lookup [S1, unofficial].
3. **Access Multi-Axis Neighbour:** To access $(x+1, y+1, z+1)$, check boundaries for all axes [S1, unofficial]. If all valid, return `p + deltaX[x] + deltaY[y] + deltaZ[z]` [S1, unofficial]. Otherwise, fallback [S1, unofficial].

**Formulas:**
Let $M(x,y,z)$ be the Morton index. The delta for the x-neighbour is:
$$ \Delta X[x] = M(x+1, y, z) - M(x, y, z) $$
The sources note that $\Delta X[x]$ is independent of $y$ and $z$ [S1, unofficial]. The accessed index is:
$$ p_{\text{neighbour}} = p_{\text{current}} + \sum_{axis \in \{x,y,z\}} \Delta_{axis}[coord_{axis}] $$

**Parameters:**
- Table size: 256 elements, supporting chunks up to $256^3$ voxels [S1, unofficial].
- Data type: `int32_t` for offsets [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Check:** If the neighbour lies outside the current chunk, the delta lookup is invalid for the global volume. The implementation must check chunk boundaries using macros before using the delta [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Precomputed lookup tables of size 256, storing int32_t offsets [S1, unofficial]
# First 16 elements of deltaX as given in the source; remaining 240 follow the same generation pattern
deltaX = [1, 7, 1, 55, 1, 7, 1, 439, 1, 7, 1, 55, 1, 7, 1, 3511]
deltaY = [0] * 256  # Generated similarly
deltaZ = [0] * 256  # Generated similarly

def peek_voxel_x_plus_one(current_voxel_ptr, x_pos_in_chunk):
    """Access voxel at (x+1, y, z)"""
    CHUNK_SIZE = 256
    if x_pos_in_chunk < CHUNK_SIZE - 1:
        return current_voxel_ptr + deltaX[x_pos_in_chunk]
    else:
        # Fallback to global coordinate lookup when crossing chunk boundary [S1, unofficial]
        return current_voxel_ptr

def peek_voxel_all_plus_one(current_voxel_ptr, x, y, z):
    """Access voxel at (x+1, y+1, z+1)"""
    CHUNK_SIZE = 256
    if x < CHUNK_SIZE - 1 and y < CHUNK_SIZE - 1 and z < CHUNK_SIZE - 1:
        return current_voxel_ptr + deltaX[x] + deltaY[y] + deltaZ[z]
    else:
        # Fallback to global coordinate lookup when crossing chunk boundary [S1, unofficial]
        return current_voxel_ptr
```

## Subtopics

### [Morton Code Incremental Traversal](6987-morton-code-incremental-traversal.md)
An algorithm for iterating through a 3D voxel volume in Morton (Z-order) curve order using precomputed delta tables to update the linear index incrementally, avoiding expensive coordinate-to-index conversions except at chunk boundaries. It builds on [Fast Neighbourhood Access via Delta Lookup Tables](3652-fast-neighbourhood-access-via-delta-lookup-tables.md) to provide $O(1)$ neighbour access within a fixed-size spatial chunk, distinguishing itself from full re-encoding methods by maintaining state across traversal steps.

## Operating characteristics
- **Time Complexity:** $O(1)$ for neighbour access within a chunk, assuming table lookup and addition are constant time [S1, unofficial].
- **Memory Complexity:** $3 \times 256 \times 4$ bytes = 3 KB for the lookup tables [S1, unofficial].
- **Measured Performance:** Significantly faster than the naive decode-modify-encode approach and competitive with bitwise manipulation methods like Fabian Giesen's [S1, unofficial].
- **Strengths:** Simple implementation using precomputed tables [S1, unofficial].
- **Limitations:** Requires boundary checks; table size is fixed to 256 elements per axis in the reference implementation [S1, unofficial].
- **When to Use:** Frequent neighbour access in Morton-ordered chunked volumes [S1, unofficial].
- **When Not to Use:** Not given in the sources [S1, unofficial].

## Background
Originated in the PolyVox library for chunked voxel data storage [S1, unofficial]. The method was developed to efficiently handle neighbour lookups in Morton-ordered volumes, addressing the lack of existing information on correct implementation [S1, unofficial]. It leverages the observation that Morton index deltas are axis-dependent only [S1, unofficial].

## Sources
[S1] Implementing Morton ordering for chunked voxel data, http://www.volumesoffun.com/implementing-morton-ordering-for-chunked-voxel-data/index.html, fetched 2026-10-30 (unofficial)
