# Volume Interleaving

Also called: Bit Interleaving, Coordinate Bit Interleaving

## Overview
Volume Interleaving is a memory layout optimization designed to reduce cache misses during volume rendering by decomposing the volume into small blocks and interleaving the bits of voxel coordinates [S1]. It maps 3D spatial coordinates to a linear memory address such that spatially adjacent voxels are stored contiguously in memory [S1]. Unlike standard row-major or column-major layouts, which suffer from poor cache performance when traversing 3D volumes along arbitrary directions, interleaving ensures that consecutive samples along a traversal path are closer in memory [S1].

## How it works
**Inputs and Outputs:**
*   Input: 3D voxel coordinates $(X, Y, Z)$ within a block of size $32^3$ [S1].
*   Output: A linear memory address (index) in the interleaved buffer [S1].

**Data Structures:**
*   A 32-entry lookup table for bit manipulation [S1].
*   32-bit unsigned integers for coordinate storage and address generation [S1].

**Method:**
The volume is decomposed into small blocks containing $32^3$ samples (32x32x32 voxels) [S1]. Within each block, the bits of the coordinates are interleaved to generate the memory address [S1]. The mapping preserves the most significant bits (MSBs) grouped by dimension while interleaving the five least significant bits (LSBs) across dimensions [S1].

The bit interleaving mapping is defined as:
$$ (X_{n \dots 0}, Y_{n \dots 0}, Z_{n \dots 0}) \implies Z_{n \dots 5} Y_{n \dots 5} X_{n \dots 5} Z_4 Y_4 X_4 \dots Z_0 Y_0 X_0 $$ [S1].

Where:
*   $n$ is the index of the most significant bit of the coordinate [S1].
*   The bits from index $n$ down to 5 are grouped by dimension ($Z$, then $Y$, then $X$) [S1].
*   The bits from index 4 down to 0 are interleaved across dimensions ($Z_4 Y_4 X_4 \dots Z_0 Y_0 X_0$) [S1].

**Algorithm Steps:**
1.  Decompose the volume into $32 \times 32 \times 32$ blocks [S1].
2.  For a given voxel at coordinates $(X, Y, Z)$:
    a. Extract the five lower bits of $X$, $Y$, and $Z$ (bits 0–4) [S1].
    b. Use a 32-entry lookup table and binary operators to interleave these five lower bits [S1].
    c. Combine the interleaved lower bits with the upper bits ($n$ down to 5), which are kept grouped by dimension ($Z_{n \dots 5} Y_{n \dots 5} X_{n \dots 5}$) [S1].
3.  The resulting bit pattern forms the linear memory address [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_interleave_address(x, y, z, n):
    """
    Generates the linear memory address for a voxel at (x, y, z)
    using volume interleaving.
    
    Args:
        x, y, z: 32-bit unsigned integer coordinates
        n: Index of the most significant bit of the coordinates
    
    Returns:
        Linear memory address (32-bit unsigned integer)
    """
    # Extract upper bits (from n down to 5)
    if n < 5:
        x_upper = 0
        y_upper = 0
        z_upper = 0
    else:
        # Mask for bits 5 to n, then shift right by 5 to align to LSB
        mask = ((1 << (n + 1)) - 1) ^ ((1 << 5) - 1)
        x_upper = (x & mask) >> 5
        y_upper = (y & mask) >> 5
        z_upper = (z & mask) >> 5

    # Extract lower 5 bits
    x5 = x & 0x1F
    y5 = y & 0x1F
    z5 = z & 0x1F

    # Interleave lower 5 bits into 15 bits: Z4 Y4 X4 ... Z0 Y0 X0
    # Place X bits at positions 0, 3, 6, 9, 12
    x_spaced = (x5 << 0) | (x5 << 3) | (x5 << 6) | (x5 << 9) | (x5 << 12)
    # Place Y bits at positions 1, 4, 7, 10, 13
    y_spaced = (y5 << 1) | (y5 << 4) | (y5 << 7) | (y5 << 10) | (y5 << 13)
    # Place Z bits at positions 2, 5, 8, 11, 14
    z_spaced = (z5 << 2) | (z5 << 5) | (z5 << 8) | (z5 << 11) | (z5 << 14)

    lower_combined = x_spaced | y_spaced | z_spaced

    # Combine upper and lower parts
    address = (z_upper << 15) | (y_upper << 10) | (x_upper << 5) | lower_combined
    return address
```
*(Note: The 32-entry lookup table mentioned in the source implements this exact 5-bit to 15-bit permutation [G].)*

**Parameters:**
*   Block size: $32^3$ samples (32x32x32 voxels) [S1].

**Edge Cases and Pitfalls:**
*   The method assumes coordinates are within the defined block [G].
*   The source notes that code running on MMX-only processors may not allow line equation tests, leading to minor artifacts, but this relates to the rendering algorithm rather than the interleaving itself [S1].

## Subtopics

### Parallel Reordering of Interleaved Voxel Bits
A parallel technique to reorder two integers containing 4-bit groups representing voxels [S2, unofficial]. Specifically, it reorders interleaved groups of bits representing even voxels (0, 2, 4, 6) and odd voxels (1, 3, 5, 7) into a contiguous or specific layout [S2, unofficial]. The method shifts groups of information at the same time whenever possible to achieve the reordering in parallel [S2, unofficial]. This is used in solid voxelization algorithms to optimize memory access patterns for subsequent processing [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The address calculation involves bitwise operations and a table lookup, which is $O(1)$ per voxel [S1].
*   **Memory Complexity:** The layout is dense, storing $32^3$ samples per block [S1].
*   **Performance:** The method reduces cache misses by ensuring spatially adjacent voxels are stored contiguously in memory [S1].
*   **Strengths:** Improved cache locality for 3D volume traversal [S1].
*   **Limitations:** The method is specific to block sizes of $32^3$ [S1].

## Background
Volume Interleaving was introduced in the context of object-order ray-casting algorithms to optimize memory access patterns [S1]. The technique is described in "A new object-order ray-casting algorithm" [S1]. It is particularly useful for rendering large volumes on hardware where cache misses are a significant performance bottleneck [S1].

## Sources
[S1] A new object-order ray-casting algorithm, https://doi.org/10.5555/602099.602130, fetched 2026-10-26
[S2] Single-pass GPU Solid Voxelization and Applications, https://inria.hal.science/inria-00345291, fetched 2026-10-26 (unofficial)
