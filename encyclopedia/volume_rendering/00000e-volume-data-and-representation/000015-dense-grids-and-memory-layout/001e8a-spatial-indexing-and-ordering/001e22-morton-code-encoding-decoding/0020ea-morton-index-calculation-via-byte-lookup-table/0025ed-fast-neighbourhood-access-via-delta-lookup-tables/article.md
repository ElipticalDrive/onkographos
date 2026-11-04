# Fast Neighbourhood Access via Delta Lookup Tables

Also called: Morton neighbour peeking, deltaX/Y/Z lookup

## Overview
A method to access immediate neighbours of a voxel in Morton-ordered memory without re-encoding coordinates, using precomputed offset tables that map a coordinate component to the memory stride required to reach its neighbour [S1, unofficial]. It solves the performance bottleneck of decoding a Morton index to $(x,y,z)$, modifying coordinates, and re-encoding, which involves expensive bitwise operations [S1, unofficial]. It is significantly faster than the naive decode-modify-encode approach and offers a simpler implementation alternative to bitwise manipulation methods like Fabian Giesen’s [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Current voxel pointer in Morton-ordered memory, current chunk-local coordinates $(x, y, z)$, and precomputed delta lookup tables.
- Output: Pointer to the neighbour voxel (e.g., at $x+1, y, z$) or a fallback mechanism if the neighbour is outside the current chunk.

**Data Structures:**
Three lookup tables: `deltaX`, `deltaY`, and `deltaZ`.
- Size: 256 elements each (for 8-bit coordinate components, supporting chunks up to $256^3$ voxels) [S1, unofficial].
- Type: Integer offsets (e.g., `int32_t`) representing the stride in memory [S1, unofficial].
- Layout: The value at index $i$ is the Morton index difference between a voxel at coordinate $i$ and its neighbour at $i+1$ along that axis, assuming the other two coordinates are constant.

**Numbered Steps:**
1. Initialize three lookup tables (`deltaX`, `deltaY`, `deltaZ`) of size 256.
2. Populate `deltaX[i]` with the difference between the Morton index of $(i+1, 0, 0)$ and $(i, 0, 0)$ for $i \in [0, 255]$.
3. Populate `deltaY[i]` with the difference between the Morton index of $(0, i+1, 0)$ and $(0, i, 0)$ for $i \in [0, 255]$.
4. Populate `deltaZ[i]` with the difference between the Morton index of $(0, 0, i+1)$ and $(0, 0, i)$ for $i \in [0, 255]$.
5. To access the neighbour at $(x+1, y, z)$:
   - Check if $x+1$ is within the current chunk boundaries.
   - If yes, compute `new_ptr = current_ptr + deltaX[x]`.
   - If no, fallback to global coordinate calculation.
6. To access the neighbour at $(x+1, y+1, z+1)$:
   - Check if $x+1, y+1, z+1$ are within current chunk boundaries.
   - If yes, compute `new_ptr = current_ptr + deltaX[x] + deltaY[y] + deltaZ[z]`.
   - If no, fallback to global coordinate calculation.

**Formulas:**
Let $M(x, y, z)$ be the Morton index of voxel $(x, y, z)$.
The delta for the x-axis is defined as:
$$ \Delta X_x = M(x+1, 0, 0) - M(x, 0, 0) $$
The delta for the y-axis is defined as:
$$ \Delta Y_y = M(0, y+1, 0) - M(0, y, 0) $$
The delta for the z-axis is defined as:
$$ \Delta Z_z = M(0, 0, z+1) - M(0, 0, z) $$
The new pointer for a neighbour at $(x+\delta_x, y+\delta_y, z+\delta_z)$ is:
$$ P_{new} = P_{current} + \sum_{axis \in \{x,y,z\}, \delta_{axis}=1} \Delta_{axis}[coord_{axis}] $$

**Parameters:**
- Table size: 256 (supports chunks up to $256^3$) [S1, unofficial].
- Coordinate bit-width: 8 bits per axis (implied by table size) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Chunk Boundaries:** If the neighbour lies outside the current chunk, the delta lookup is invalid for the global volume. The implementation must check chunk boundaries using macros (e.g., `CAN_GO_POS_X`) before using the delta [S1, unofficial].
- **Fallback:** If the boundary check fails, the method must fall back to decoding the Morton index to global coordinates, modifying them, and re-encoding or using a global lookup [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Precomputed tables (size 256)
deltaX = [0] * 256
deltaY = [0] * 256
deltaZ = [0] * 256

def init_delta_tables():
    """
    Initialize delta lookup tables.
    Assumes Morton encoding function morton_encode(x, y, z) is available.
    """
    for i in range(256):
        # Calculate delta for x-axis
        # Note: y and z are 0 for the base calculation
        idx1 = morton_encode(i + 1, 0, 0)
        idx0 = morton_encode(i, 0, 0)
        deltaX[i] = idx1 - idx0
        
        # Calculate delta for y-axis
        idx1 = morton_encode(0, i + 1, 0)
        idx0 = morton_encode(0, i, 0)
        deltaY[i] = idx1 - idx0
        
        # Calculate delta for z-axis
        idx1 = morton_encode(0, 0, i + 1)
        idx0 = morton_encode(0, 0, i)
        deltaZ[i] = idx1 - idx0

def peek_voxel_x_plus_one(current_ptr, x_pos_in_chunk):
    """
    Access voxel at (x+1, y, z) relative to current voxel.
    """
    # Check boundary: if x_pos_in_chunk is 255, x+1 is out of chunk
    if x_pos_in_chunk < 255:
        return current_ptr + deltaX[x_pos_in_chunk]
    else:
        # Fallback: decode, modify, re-encode or global lookup
        # This part is not detailed in the source, so we mark it as a comment
        # return global_get_voxel(x_pos_in_chunk + 1, y_pos_in_chunk, z_pos_in_chunk)
        pass

def peek_voxel_all_plus_one(current_ptr, x_pos_in_chunk, y_pos_in_chunk, z_pos_in_chunk):
    """
    Access voxel at (x+1, y+1, z+1) relative to current voxel.
    """
    # Check boundaries for all axes
    if x_pos_in_chunk < 255 and y_pos_in_chunk < 255 and z_pos_in_chunk < 255:
        return current_ptr + deltaX[x_pos_in_chunk] + deltaY[y_pos_in_chunk] + deltaZ[z_pos_in_chunk]
    else:
        # Fallback: decode, modify, re-encode or global lookup
        # return global_get_voxel(x_pos_in_chunk + 1, y_pos_in_chunk + 1, z_pos_in_chunk + 1)
        pass
```

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(1)$ for neighbour access within the same chunk, as it involves only array lookups and addition.
- **Memory Complexity:** $O(1)$ extra memory for the three lookup tables (3 * 256 * 4 bytes = 3 KB for `int32_t`).
- **Measured Performance:** Significantly faster than the naive decode-modify-encode approach [S1, unofficial].
- **Scaling:** Supports chunks up to $256^3$ voxels with fixed table size [S1, unofficial].
- **Strengths:** Simple implementation, fast access, no bitwise operations during runtime.
- **Limitations:** Requires boundary checks; fallback to global lookup is needed for cross-chunk neighbours [S1, unofficial].

## Background
Originated in the PolyVox project, as described in a blog post by Matt and contributors [S1, unofficial]. The method was developed to efficiently handle voxel data stored in Morton-ordered chunks, addressing the complexity of neighbour access in such a layout [S1, unofficial].

## Sources
[S1] Implementing Morton ordering for chunked voxel data, http://www.volumesoffun.com/implementing-morton-ordering-for-chunked-voxel-data/index.html, fetched 2026-10-30 (unofficial)
