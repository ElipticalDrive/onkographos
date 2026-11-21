# Voxel Face Packing

Also called: Packed Voxel Cell Encoding with Face/Type Info

## Overview
Voxel Face Packing is a bitstream encoding method that compresses voxel spatial coordinates, material identity, and face orientation into a single 32-bit integer. It solves the problem of high vertex buffer overhead and frequent state changes in large voxel grid rendering by allowing all necessary geometry data to be packed into a compact, cache-friendly format. It differs from standard instanced rendering by avoiding per-voxel vertex buffers and instead projecting a fixed set of vertices, discarding empty space via near-plane clipping or geometry culling, leveraging vertex texture fetch capabilities [S1, unofficial].

## How it works
**Inputs:**
- `x`: Local x coordinate of the voxel center, integer range [0..15] (4 bits).
- `y`: Local y coordinate of the voxel center, integer range [0..15] (4 bits).
- `z`: Local z coordinate of the voxel center, integer range [0..15] (4 bits).
- `type`: Voxel type identifier, integer range [0..255] (8 bits).
- `direction`: Face direction index, integer range [0..5] (3 bits).

**Outputs:**
- A single 32-bit unsigned integer (`uint32`).

**Data Structure (Bit Layout):**
The 32-bit integer is laid out as follows:
- Bits 31–28: `x` (4 bits)
- Bits 27–24: `y` (4 bits)
- Bits 23–20: `z` (4 bits)
- Bits 19–12: `type` (8 bits)
- Bits 11–9: `direction` (3 bits)
- Bits 8–0: Unused (9 bits)

**Packing Algorithm:**
The values are packed into a single 32-bit integer using bitwise shifts and logical OR operations. The formula is:
$$ \text{packed} = (x \ll 20) \mid (y \ll 16) \mid (z \ll 12) \mid (\text{type} \ll 4) \mid (\text{direction} \ \& \ 0b111) $$

**Unpacking Algorithm:**
Values are recovered by shifting right and masking with bitwise AND. The formulas are:
$$ x = (\text{packed} \gg 20) \ \& \ 0b1111 $$
$$ y = (\text{packed} \gg 16) \ \& \ 0b1111 $$
$$ z = (\text{packed} \gg 12) \ \& \ 0b1111 $$
$$ \text{type} = (\text{packed} \gg 4) \ \& \ 0xFF $$
$$ \text{direction} = \text{packed} \ \& \ 0b111 $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack_voxel_face(x: int, y: int, z: int, voxel_type: int, direction: int) -> int:
    """
    Packs voxel coordinates, type, and face direction into a single 32-bit integer.
    
    Args:
        x: Local x coordinate (0-15)
        y: Local y coordinate (0-15)
        z: Local z coordinate (0-15)
        voxel_type: Voxel type identifier (0-255)
        direction: Face direction index (0-5)
        
    Returns:
        Packed 32-bit integer
    """
    # Validate inputs to ensure they fit in allocated bits
    if not (0 <= x <= 15):
        raise ValueError("x must be in range [0, 15]")
    if not (0 <= y <= 15):
        raise ValueError("y must be in range [0, 15]")
    if not (0 <= z <= 15):
        raise ValueError("z must be in range [0, 15]")
    if not (0 <= voxel_type <= 255):
        raise ValueError("voxel_type must be in range [0, 255]")
    if not (0 <= direction <= 5):
        raise ValueError("direction must be in range [0, 5]")
        
    packed = (x << 20) | (y << 16) | (z << 12) | (voxel_type << 4) | (direction & 0b111)
    return packed

def unpack_voxel_face(packed: int) -> dict:
    """
    Unpacks a 32-bit integer into its constituent voxel face components.
    
    Args:
        packed: The packed 32-bit integer
        
    Returns:
        Dictionary containing x, y, z, type, and direction
    """
    x = (packed >> 20) & 0b1111
    y = (packed >> 16) & 0b1111
    z = (packed >> 12) & 0b1111
    voxel_type = (packed >> 4) & 0xFF
    direction = packed & 0b111
    
    return {
        "x": x,
        "y": y,
        "z": z,
        "type": voxel_type,
        "direction": direction
    }
```

**Edge Cases and Pitfalls:**
- **Precision:** The method relies on integer arithmetic. Ensure inputs are integers before packing.
- **Boundaries:** Inputs must strictly adhere to the defined ranges. Overflow or underflow will corrupt the bit layout.
- **Unused Bits:** Bits 8–0 are unused. Their values in the packed integer are undefined and should be ignored during unpacking.

## Subtopics

## Operating characteristics
- **Time Complexity:** Packing and unpacking are $O(1)$ operations involving a constant number of bitwise shifts and logical operations.
- **Memory Complexity:** Reduces memory footprint per vertex from multiple variables (e.g., 4 floats or ints) to a single 32-bit integer.
- **Performance:** Enables efficient vertex buffer storage and cache-friendly access patterns in GPU shaders.
- **Strengths:** Compact representation, fast bitwise operations, reduced state changes in rendering pipelines.
- **Limitations:** Limited to 16x16x16 local coordinate space per voxel cell, 256 voxel types, and 6 face directions. Not suitable for high-precision coordinates or large numbers of material types without extension.

## Background
This method is described in the context of web-based voxel terrain rendering, specifically in the `web_voxel_terrain` project. It is designed to optimize rendering performance by minimizing the data transferred to the GPU and simplifying vertex shader logic.

## Sources
[S1] andrewbloese-00/web_voxel_terrain, https://github.com/andrewbloese-00/web_voxel_terrain, fetched 2026-10-27 (unofficial)
