# Binary Volume Ray Casting with Lookup Table

Also called: Depth cueing with binary mask, LUT-based ray tracing

## Overview
Binary Volume Ray Casting with Lookup Table is a rendering technique that accelerates the determination of the first intersection point with a volume by using a binary mask to skip empty regions [S1, unofficial]. It finds the first non-zero byte in a ray's path via sequential checking and uses a 256-entry lookup table (LUT) to determine the exact bit position within that byte, thereby avoiding the memory overhead of a $2^{32}$-byte direct addressing table [S1, unofficial]. This method builds on [Impostor Depth Optimization](4434-impostor-depth-optimization.md) by providing a concrete mechanism for empty space skipping that allows for fast previewer applications where the original volume data does not need to reside in main memory [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A binary volume $B$ representing voxel transparency status, where non-zero values indicate occupied space. The volume is stored in a 3D integer array `mask[z][y][x]`. The dimensions are $X \times Y \times Z$.
*   **Output:** The depth $z_{hit}$ of the first non-zero voxel along the ray for a given pixel $(x, y)$, or $Z-1$ if the ray passes through empty space.

**Data Structures:**
*   **Binary Mask (`mask`):** An integer array of shape $(Z_{max}, Y, X)$, where $Z_{max} = \lceil Z / 32 \rceil$. Each integer stores 32 bits of voxel status along the z-axis. The most significant bit (MSB) of each integer corresponds to the voxel nearest to the viewer (smallest z-index within that 32-voxel segment) [S1, unofficial].
*   **Lookup Table (`lut`):** A 1D array of 256 entries. Each entry stores the offset (0–7) of the first non-zero bit within a single byte [S1, unofficial].

**Parameters:**
*   **Integer Size:** 32 bits [S1, unofficial].
*   **LUT Size:** 256 entries [S1, unofficial].
*   **Bit Order:** Most significant bit is nearest to the viewer [S1, unofficial].

**Algorithm Steps:**
1.  Calculate the number of 32-voxel segments along the z-axis: $Z_{max} = \lceil Z / 32 \rceil$. In integer arithmetic, this is computed as `(Z + 31) / 32` [S1, unofficial].
2.  For a given pixel $(x, y)$, iterate through the segments $z$ from $0$ to $Z_{max} - 1$.
3.  Retrieve the segment value `segment` from `mask[z][y][x]`.
4.  Check if `segment` is non-zero.
    *   If `segment` is zero, continue to the next segment.
    *   If `segment` is non-zero, this is the first occupied segment. Calculate the exact depth using the `Depth` function and return $z \times 32 + \text{Depth}(\text{segment})$ [S1, unofficial].
5.  If the loop completes without finding a non-zero segment, return $Z - 1$ (indicating no intersection) [S1, unofficial].

**Depth Calculation:**
The `Depth` function determines the offset of the first non-zero bit within the 32-bit integer. It uses binary search logic to identify which byte contains the first non-zero bit, then indexes the LUT for the precise bit offset [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Precomputed LUT: lut[byte] contains the index (0-7) of the first non-zero bit in that byte.
# If the byte is 0, the value is undefined or handled as a fallback, though the logic below
# ensures lut is only accessed for non-zero bytes.

def Depth(segment):
    """
    Determines the offset of the first non-zero bit in a 32-bit integer.
    Assumes MSB is nearest to viewer.
    """
    # Check the lower 16 bits (bits 0-15)
    if segment < 0x00010000:
        # Check the lower 8 bits (bits 0-7)
        if segment < 0x00000100:
            # First non-zero bit is in the lowest byte (bits 0-7)
            # Offset is 24 + lut[byte_value] because MSB is bit 31 (offset 0)
            # Bit 7 is offset 24.
            return 24 + lut[segment]
        else:
            # First non-zero bit is in the second lowest byte (bits 8-15)
            # Shift right by 8 to get the byte value.
            # Bit 15 is offset 16.
            return 16 + lut[segment >> 8]
    else:
        # Check the upper 16 bits (bits 16-31)
        if segment < 0x01000000:
            # First non-zero bit is in the third byte (bits 16-23)
            # Shift right by 16.
            # Bit 23 is offset 8.
            return 8 + lut[segment >> 16]
        else:
            # First non-zero bit is in the highest byte (bits 24-31)
            # Shift right by 24.
            # Bit 31 is offset 0.
            return lut[segment >> 24]

def Trace(x, y, Z, mask, lut):
    """
    Finds the first intersection depth for ray (x, y).
    """
    # Calculate number of 32-voxel segments
    z_max = (Z + 31) // 32
    
    for z in range(z_max):
        segment = mask[z][y][x]
        
        # Check if segment is non-zero
        if segment != 0:
            return z * 32 + Depth(segment)
            
    # No intersection found
    return Z - 1
```

**Edge Cases and Pitfalls:**
*   **Integer Insufficiency:** A single integer is insufficient to store a complete row of the binary volume if $Z > 32$. The algorithm must check segments sequentially [S1, unofficial].
*   **LUT Access:** The LUT is only accessed for non-zero bytes. The binary search logic ensures that `segment >> shift` results in a value $< 256$ before indexing [S1, unofficial].
*   **Bit Order Assumption:** The correctness of the offsets (24, 16, 8, 0) depends on the assumption that the most significant bit is nearest to the viewer. If the bit order is reversed, the offsets must be adjusted accordingly [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The time to find the first intersection is proportional to the number of empty 32-voxel segments along the ray. In the worst case (empty volume or intersection at the end), it checks all $Z_{max}$ segments. In the best case (intersection in the first segment), it performs constant-time LUT lookups.
*   **Memory Complexity:** The binary mask requires $Z_{max} \times Y \times X$ integers. The LUT requires 256 integers. This is significantly less than the original volume if the volume is stored as floats or doubles, and avoids the $2^{32}$ byte array required for direct addressing [S1, unofficial].
*   **Strengths:** Fast previewing; does not require storing the original volume in main memory; simple 2D image-based shading can be applied to the resulting depth image [S1, unofficial].
*   **Limitations:** Only provides the first intersection depth; does not perform full volume integration (transparency/opacity accumulation) in this step. It is a preprocessing step for depth cueing or shading [S1, unofficial].

## Background
This technique was described in the context of interactive volume rendering for medical data visualization [S1, unofficial]. It addresses the computational cost of ray casting by skipping empty space using a binary mask and a compact lookup table. The method is particularly suited for fast previewer applications where the original volume data is not kept in main memory [S1, unofficial]. The approach is related to depth cueing and depth-gradient shading, which can be applied to the resulting distance image [S1, unofficial].

## Sources
[S1] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
