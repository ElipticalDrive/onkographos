# 3D Coordinate Packing

Also called: Voxel Key Generation, getKey

## Overview
This method converts three 3D voxel coordinates $(x, y, z)$ into a unique single integer key for use as a hash map index [S1, unofficial]. It is designed for sparse voxel grids where dimensions fit within specific bit limits, providing a compact key for hash map indexing [S1, unofficial]. Unlike general space-filling curves (e.g., Z-order), this method relies on fixed bit-width allocations, making it faster but dimensionally constrained [S1, unofficial].

## How it works
**Inputs:** Three integer coordinates $x, y, z$.
**Outputs:** A single 64-bit integer key.

**Data Structures:**
The method relies on a memory-overlap structure (union) containing a 64-bit integer and a struct with three 16-bit signed integer fields. The layout assumes little-endian byte order and no padding between fields.

**Steps:**
1.  Define a union type that contains a 64-bit integer (`__int64` or `uint64_t`) and a struct containing three 16-bit signed integers (`short` or `int16_t`).
2.  Assign the input coordinates $x, y, z$ to the struct fields. The specific mapping depends on the struct definition order and endianness.
3.  Read the 64-bit integer from the union to obtain the packed key.

**Formula:**
Based on the source derivation, assuming a struct layout where the first field corresponds to the least significant bits and the last to the most significant, and assuming little-endian architecture:
$$ \text{key} = (z \ll 32) \mid (y \ll 16) \mid x $$
Where:
*   $x$ occupies bits 0–15.
*   $y$ occupies bits 16–31.
*   $z$ occupies bits 32–47.
*   Bits 48–63 are zero (assuming 16-bit signed inputs).

**Parameters:**
*   Coordinate width: 16 bits per axis.
*   Key width: 64 bits.

**Edge Cases and Pitfalls:**
*   **Endianness:** The method relies on little-endian architecture. On big-endian systems, the byte order of the struct fields relative to the 64-bit integer will differ, producing incorrect keys [S1, unofficial].
*   **Struct Alignment:** The method assumes no padding between the `short` fields. Compiler-specific packing directives (e.g., `#pragma pack`) may be required to ensure this layout [S1, unofficial].
*   **Range Limitation:** Coordinates are limited to the 16-bit signed range (-32768 to 32767). Values outside this range will overflow or wrap [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Note: Python does not have fixed-width signed integers or memory-overlap unions.
# This pseudocode illustrates the bit-manipulation logic derived from the C/C++ union approach.

def pack_3d_coordinate(x: int, y: int, z: int) -> int:
    """
    Packs three 16-bit signed coordinates into a 64-bit integer key.
    Assumes little-endian logic: x is LSB, z is MSB of the used bits.
    """
    # Validate range for 16-bit signed integers
    if not (-32768 <= x <= 32767):
        raise ValueError("x out of 16-bit signed range")
    if not (-32768 <= y <= 32767):
        raise ValueError("y out of 16-bit signed range")
    if not (-32768 <= z <= 32767):
        raise ValueError("z out of 16-bit signed range")

    # Mask to ensure 16-bit width (handles potential sign extension in Python)
    x_masked = x & 0xFFFF
    y_masked = y & 0xFFFF
    z_masked = z & 0xFFFF

    # Combine using bit shifts
    # x -> bits 0-15
    # y -> bits 16-31
    # z -> bits 32-47
    key = (z_masked << 32) | (y_masked << 16) | x_masked

    return key

def unpack_3d_coordinate(key: int) -> tuple:
    """
    Unpacks a 64-bit integer key back into three 16-bit signed coordinates.
    """
    x = key & 0xFFFF
    y = (key >> 16) & 0xFFFF
    z = (key >> 32) & 0xFFFF

    # Convert to signed 16-bit integers if necessary
    if x >= 32768:
        x -= 65536
    if y >= 32768:
        y -= 65536
    if z >= 32768:
        z -= 65536

    return (x, y, z)
```

## Subtopics

### [Grid Index Packing & Unpacking](12913-grid-index-packing-unpacking.md)
This method converts three 3D voxel coordinates $(x, y, z)$ into a unique single integer ID and reverses the operation using bit shifting and addition. It is designed for sparse voxel grids where dimensions fit within specific bit limits, providing a compact key for hash map indexing. Unlike general space-filling curves (e.g., Z-order), this method relies on fixed bit-width allocations, making it faster but dimensionally constrained.

## Operating characteristics
*   **Time Complexity:** $O(1)$ for packing and unpacking, involving only bitwise operations and shifts.
*   **Memory Complexity:** $O(1)$, storing three coordinates in a single 64-bit integer.
*   **Strengths:** Extremely fast; minimal memory overhead compared to storing three separate integers; simple implementation.
*   **Limitations:** Strictly limited to 16-bit coordinate ranges; dependent on endianness and struct alignment; does not provide spatial locality (unlike Z-order or Hilbert curves), which may lead to poor cache performance for spatially adjacent voxels if used as a direct hash key without further hashing [S1, unofficial].
*   **When to use:** When coordinate ranges are small and known in advance, and maximum packing/unpacking speed is required.
*   **When not to use:** When coordinates exceed 16 bits, when spatial locality is critical for cache performance, or on big-endian architectures without adaptation.

## Background
The method is derived from implementations found in unofficial source code repositories, specifically utilizing C/C++ union structures for type-punning to pack coordinates [S1, unofficial]. It is a common technique in graphics programming for generating hash keys from 3D positions.

## Sources
[S1] nustxujun/AfterHumanDeclined, https://github.com/nustxujun/AfterHumanDeclined, fetched 2026-10-26 (unofficial)
