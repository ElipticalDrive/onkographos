# Voxel Block Data Layout

Also called: Block int32 packing, Coordinate bit-packing

## Overview
This layout stores 3D voxel coordinates within a single 32-bit integer to optimize memory access and indexing in voxel-based rendering engines. It differs from standard struct-of-arrays or array-of-structs by packing spatial position into a dense integer key, often used in conjunction with a separate palette for material/color data [S1, S2, unofficial].

## How it works
**Inputs:** Voxel coordinates $(x, y, z)$ and optional material/color index.
**Outputs:** A packed 32-bit integer representing the voxel position, and a separate storage for material properties.

**Data Structures:**
1.  **Packed Coordinate:** A 32-bit unsigned integer.
    *   Bits 24–31: $x$ coordinate (8 bits).
    *   Bits 16–23: $y$ coordinate (8 bits).
    *   Bits 8–15: $z$ coordinate (8 bits).
    *   Bits 0–7: Unused or reserved in the primary coordinate packing scheme [S1, unofficial].
2.  **Block Object:** Contains the packed coordinate and a separate 8-bit palette index for color/material [S1, unofficial].
3.  **Voxel Struct (Alternative):** In some implementations, voxels are stored as structs containing `flags` (uint8), `color` (uint32), and `blockType` (uint8), indexed by `V3_Vec3i16` coordinates [S2, unofficial].
4.  **Chunk Storage:** Chunks may store voxels in a map keyed by `V3_Vec3i16`, alongside a 2D heightmap `maxY` of type `[chunkSize][chunkSize]uint8` and a `flag` byte [S2, unofficial].

**Packing Formula:**
The coordinate $P$ is calculated as:
$$ P = (x \ll 24) \mid (y \ll 16) \mid (z \ll 8) $$
Where $\ll$ denotes bitwise left shift and $\mid$ denotes bitwise OR [S1, unofficial].

**Unpacking Formula:**
Given packed integer $P$:
$$ x = (P \gg 24) \ \& \ 0xFF $$
$$ y = (P \gg 16) \ \& \ 0xFF $$
$$ z = (P \gg 8) \ \& \ 0xFF $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack_voxel(x: int, y: int, z: int) -> int:
    """
    Packs x, y, z coordinates into a single 32-bit integer.
    x occupies bits 24-31, y occupies bits 16-23, z occupies bits 8-15.
    """
    # Ensure coordinates are within 8-bit range (0-255)
    if not (0 <= x <= 255 and 0 <= y <= 255 and 0 <= z <= 255):
        raise ValueError("Coordinates must fit in 8 bits")
    
    packed = (x << 24) | (y << 16) | (z << 8)
    return packed

def unpack_voxel(packed: int) -> tuple:
    """
    Unpacks x, y, z from a 32-bit integer.
    """
    x = (packed >> 24) & 0xFF
    y = (packed >> 16) & 0xFF
    z = (packed >> 8) & 0xFF
    return (x, y, z)

class Block:
    def __init__(self, x: int, y: int, z: int, color_index: int):
        self.packed_coord = pack_voxel(x, y, z)
        self.color = color_index # Stored separately as 8-bit palette index

class Chunk:
    def __init__(self, chunk_size: int):
        # Map keyed by V3_Vec3i16 coordinates
        self.voxels = {} 
        # 2D heightmap
        self.maxY = [[0 for _ in range(chunk_size)] for _ in range(chunk_size)]
        self.flag = 0
```

**Parameters:**
*   Coordinate precision: 8 bits per axis (range 0–255) [S1, unofficial].
*   Color/Material storage: Separate 8-bit palette index [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Byte Order Confusion:** Inline comments in some implementations suggest a byte order of color, x, y, z, but the actual implementation packs x, y, z into the integer with x as the most significant byte. Color is *not* packed into this integer [S1, unofficial].
*   **Coordinate Range:** The 8-bit limit restricts world coordinates to 256 units per axis per block/chunk context unless offset handling is implemented [S1, unofficial].
*   **Unused Bits:** Bits 0–7 are unused in the coordinate packing, potentially wasting space if not utilized for flags or other data [S1, unofficial].

## Operating characteristics
*   **Memory Efficiency:** High for coordinate storage; reduces overhead of storing three separate integers or floats per voxel.
*   **Access Speed:** Fast bitwise operations for packing/unpacking; efficient for hash map keys if coordinates are the primary lookup key.
*   **Limitations:** Limited coordinate range (8 bits per axis) without additional offset logic. Not suitable for high-precision scientific volume rendering where floating-point coordinates are required.
*   **Use Case:** Ideal for voxel engines with limited coordinate ranges (e.g., Minecraft-like worlds) where memory bandwidth and cache efficiency are critical [S1, S2, unofficial].

## Background
Originating from early WebGL voxel engine implementations, this layout was designed to minimize memory footprint and simplify spatial indexing in JavaScript-based 3D applications [S1, unofficial]. It has been adapted in various C++ voxel projects for similar performance benefits [S2, unofficial].

## Sources
[S1] Show HN: Basic voxel engine in WebGL/JavaScript with full source, https://github.com/Lallassu/vox2, fetched 2026-10-26 (unofficial)
[S2] myusernameisbetter/42-Vox, https://github.com/myusernameisbetter/42-Vox, fetched 2026-10-27 (unofficial)
