# Per-face Column-based Face Culling with Opacity Masks

Also called: Column-based Opacity Mask Culling

## Overview
A face culling technique that uses per-face column arrays and opaque column arrays to determine visibility based on solid and transparent block transitions, allowing full per-face control over opacity [S1, unofficial]. It extends [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by distinguishing between opaque and transparent surface contributions on partial shapes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `face_cols`: A 3D array of 64-bit integers representing general column data for each face [S1, unofficial].
    *   `face_cols_opaque`: A 3D array of 64-bit integers representing opaque column data for each face [S1, unofficial].
    *   `CHUNK_SIZE_P`: The dimension size of the chunk along each axis [S1, unofficial].
*   **Outputs:**
    *   `col_face_masks`: A 3D array of 64-bit integers storing the final visibility masks per face [S1, unofficial].

**Data Structures:**
The method uses three primary data structures indexed by `face_index`, and two perpendicular dimensions depending on the axis [S1, unofficial]. Each element is a 64-bit unsigned integer where each bit represents a voxel column along the depth axis [S1, unofficial].
1. `face_cols[face_index][dim1][dim2]`: General column mask [S1, unofficial].
2. `face_cols_opaque[face_index][dim1][dim2]`: Opaque column mask [S1, unofficial].
3. `col_face_masks[face_index][dim1][dim2]`: Resulting face mask [S1, unofficial].
The `face_index` ranges from 0 to 5, corresponding to the 6 faces of a voxel chunk [S1, unofficial]. The mapping of axes to face indices and array dimensions is as follows [S1, unofficial]:
*   **Axis 0 (Y-axis faces):** `face_pos = 0`, `face_neg = 1`; Dimensions: `[z][x]` [S1, unofficial].
*   **Axis 1 (Z-axis faces):** `face_pos = 2`, `face_neg = 3`; Dimensions: `[y][z]` [S1, unofficial].
*   **Axis 2 (X-axis faces):** `face_pos = 4`, `face_neg = 5`; Dimensions: `[y][x]` [S1, unofficial].

**Algorithm Steps:**
1. Iterate over each axis `axis` from 0 to 2 [S1, unofficial].
2. For each axis, iterate over the two dimensions perpendicular to the axis (e.g., `z` and `x` for axis 0) [S1, unofficial]. The loop bounds are `0` to `CHUNK_SIZE_P - 1` [S1, unofficial].
3. Calculate the positive and negative face indices: `face_pos = 2 * axis + 0` and `face_neg = 2 * axis + 1` [S1, unofficial].
4. Retrieve the general column masks for the current position: `col_pos = face_cols[face_pos][dim1][dim2]` and `col_neg = face_cols[face_neg][dim1][dim2]` [S1, unofficial].
5. Retrieve the opaque column masks for the current position: `col_opaque_pos = face_cols_opaque[face_pos][dim1][dim2]` and `col_opaque_neg = face_cols_opaque[face_neg][dim1][dim2]` [S1, unofficial].
6. Compute the effective opaque columns by intersecting general columns with opaque masks: `effective_opaque_pos = col_pos & col_opaque_pos` and `effective_opaque_neg = col_neg & col_opaque_neg` [S1, unofficial].
7. Detect solid transitions (solid to air) in the positive direction: `solid_descending = col_pos & ~(col_pos << 1)` [S1, unofficial].
8. Detect solid transitions (solid to air) in the negative direction: `solid_ascending = col_neg & ~(col_neg >> 1)` [S1, unofficial].
9. Detect opaque transitions (opaque to air) in the positive direction: `opaque_descending = effective_opaque_pos & ~(effective_opaque_pos << 1)` [S1, unofficial].
10. Detect opaque transitions (opaque to air) in the negative direction: `opaque_ascending = effective_opaque_neg & ~(effective_opaque_neg >> 1)` [S1, unofficial].
11. Combine solid and opaque transitions to form the final face masks: `col_face_masks[face_pos][dim1][dim2] = solid_descending | opaque_descending` and `col_face_masks[face_neg][dim1][dim2] = solid_ascending | opaque_ascending` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_face_masks(face_cols, face_cols_opaque, col_face_masks, CHUNK_SIZE_P):
    # Axis 0: Y faces, dimensions z and x
    for z in range(CHUNK_SIZE_P):
        for x in range(CHUNK_SIZE_P):
            face_pos, face_neg = 0, 1
            col_pos = face_cols[face_pos][z][x]
            col_neg = face_cols[face_neg][z][x]
            col_opaque_pos = face_cols_opaque[face_pos][z][x]
            col_opaque_neg = face_cols_opaque[face_neg][z][x]
            effective_opaque_pos = col_pos & col_opaque_pos
            effective_opaque_neg = col_neg & col_opaque_neg
            solid_descending = col_pos & ~(col_pos << 1)
            solid_ascending = col_neg & ~(col_neg >> 1)
            opaque_descending = effective_opaque_pos & ~(effective_opaque_pos << 1)
            opaque_ascending = effective_opaque_neg & ~(effective_opaque_neg >> 1)
            col_face_masks[face_pos][z][x] = solid_descending | opaque_descending
            col_face_masks[face_neg][z][x] = solid_ascending | opaque_ascending

    # Axis 1: Z faces, dimensions y and z
    for y in range(CHUNK_SIZE_P):
        for z in range(CHUNK_SIZE_P):
            face_pos, face_neg = 2, 3
            col_pos = face_cols[face_pos][y][z]
            col_neg = face_cols[face_neg][y][z]
            col_opaque_pos = face_cols_opaque[face_pos][y][z]
            col_opaque_neg = face_cols_opaque[face_neg][y][z]
            effective_opaque_pos = col_pos & col_opaque_pos
            effective_opaque_neg = col_neg & col_opaque_neg
            solid_descending = col_pos & ~(col_pos << 1)
            solid_ascending = col_neg & ~(col_neg >> 1)
            opaque_descending = effective_opaque_pos & ~(effective_opaque_pos << 1)
            opaque_ascending = effective_opaque_neg & ~(effective_opaque_neg >> 1)
            col_face_masks[face_pos][y][z] = solid_descending | opaque_descending
            col_face_masks[face_neg][y][z] = solid_ascending | opaque_ascending

    # Axis 2: X faces, dimensions y and x
    for y in range(CHUNK_SIZE_P):
        for x in range(CHUNK_SIZE_P):
            face_pos, face_neg = 4, 5
            col_pos = face_cols[face_pos][y][x]
            col_neg = face_cols[face_neg][y][x]
            col_opaque_pos = face_cols_opaque[face_pos][y][x]
            col_opaque_neg = face_cols_opaque[face_neg][y][x]
            effective_opaque_pos = col_pos & col_opaque_pos
            effective_opaque_neg = col_neg & col_opaque_neg
            solid_descending = col_pos & ~(col_pos << 1)
            solid_ascending = col_neg & ~(col_neg >> 1)
            opaque_descending = effective_opaque_pos & ~(effective_opaque_pos << 1)
            opaque_ascending = effective_opaque_neg & ~(effective_opaque_neg >> 1)
            col_face_masks[face_pos][y][x] = solid_descending | opaque_descending
            col_face_masks[face_neg][y][x] = solid_ascending | opaque_ascending
```

**Parameters and Edge Cases:**
*   `CHUNK_SIZE_P`: Must be $\le 64$ to fit within a 64-bit integer [G]. If the chunk size exceeds 64, this method requires modification to use wider integers or multiple words per column [G].
*   Bitwise operations assume standard two's complement representation. The `~` operator inverts all bits, including those beyond the valid voxel range. Implementers must ensure that bits beyond `CHUNK_SIZE_P` are masked out or ignored to prevent false positives from garbage bits [G].
*   The method assumes that `face_cols` and `face_cols_opaque` are correctly populated prior to this step [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(F \cdot D^2)$, where $F=6$ is the number of faces and $D$ is the chunk dimension size [S1, unofficial]. The complexity is linear with respect to the number of voxels on the surface of the chunk [S1, unofficial].
*   **Memory Complexity:** Requires storage for `face_cols`, `face_cols_opaque`, and `col_face_masks` [S1, unofficial]. Each is $6 \cdot D^2$ 64-bit integers [S1, unofficial].
*   **Performance:** Described as "definitely going to cost a bit more in terms of cycle time" compared to simpler culling, but "much faster than a non-binary meshing approach" [S1, unofficial]. It provides "very efficient" control over opacity [S1, unofficial].
*   **Strengths:** Full per-face control over opacity; distinguishes between opaque and transparent surfaces [S1, unofficial].
*   **Limitations:** Not given in the sources.

## Background
This method was introduced in the context of binary greedy meshing for voxel rendering, specifically to support transparent voxel textures [S1, unofficial]. It was discussed in a GitHub issue regarding a voxel meshing repository [S1, unofficial].

## Sources
[S1] Supporting Transparent Voxel Textures, https://github.com/TanTanDev/binary_greedy_mesher_demo/issues/5, fetched 2026-10-27 (unofficial)
