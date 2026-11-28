# Voxel Face Packing

Also called: Packed Voxel Cell Encoding with Face/Type Info

## Overview
Voxel Face Packing encodes a voxel's local spatial coordinates, material type, and face direction into a single 32-bit integer [S1, unofficial]. It extends the parent method by appending face orientation and type identifiers to the coordinate encoding [S1, unofficial].

## How it works
**Inputs:**
- `x`: Local x coordinate of the voxel center, integer range [0..15] (4 bits) [S1, unofficial].
- `y`: Local y coordinate of the voxel center, integer range [0..15] (4 bits) [S1, unofficial].
- `z`: Local z coordinate of the voxel center, integer range [0..15] (4 bits) [S1, unofficial].
- `type`: Voxel type identifier, integer range [0..255] (8 bits) [S1, unofficial].
- `direction`: Face direction index, integer range [0..5] (3 bits) [S1, unofficial].

**Outputs:**
- A single 32-bit unsigned integer (`uint32`) [S1, unofficial].

**Data Structure (Bit Layout):**
The 32-bit integer layout is fixed [S1, unofficial]:
- Bits 31–28: `x` (4 bits)
- Bits 27–24: `y` (4 bits)
- Bits 23–20: `z` (4 bits)
- Bits 19–12: `type` (8 bits)
- Bits 11–9: `direction` (3 bits)
- Bits 8–0: Unused (9 bits) [S1, unofficial].

**Packing Algorithm:**
Values are packed into a single 32-bit integer using bitwise shifts and logical OR operations [S1, unofficial]:
$$ \text{packed} = (x \ll 20) \mid (y \ll 16) \mid (z \ll 12) \mid (\text{type} \ll 4) \mid (\text{direction} \ \& \ 0b111) $$

**Unpacking Algorithm:**
Values are recovered by shifting right and masking with bitwise AND [S1, unofficial]:
$$ x = (\text{packed} \gg 20) \ \& \ 0b1111 $$
$$ y = (\text{packed} \gg 16) \ \& \ 0b1111 $$
$$ z = (\text{packed} \gg 12) \ \& \ 0b1111 $$
$$ \text{type} = (\text{packed} \gg 4) \ \& \ 0xFF $$
$$ \text{direction} = \text{packed} \ \& \ 0b111 $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack_voxel_face(x: int, y: int, z: int, voxel_type: int, direction: int) -> int:
    """Packs voxel coordinates, type, and face direction into a single 32-bit integer."""
    packed = (x << 20) | (y << 16) | (z << 12) | (voxel_type << 4) | (direction & 0b111)
    return packed

def unpack_voxel_face(packed: int) -> dict:
    """Unpacks a 32-bit integer into its constituent components."""
    return {
        "x": (packed >> 20) & 0b1111,
        "y": (packed >> 16) & 0b1111,
        "z": (packed >> 12) & 0b1111,
        "type": (packed >> 4) & 0xFF,
        "direction": packed & 0b111
    }
```

**Edge Cases and Pitfalls:**
- Inputs must be integers within the specified ranges; values outside these ranges will corrupt the bit layout [G].
- Bits 8–0 are unused and should be ignored during unpacking [S1, unofficial].

## Subtopics

### [Packed Voxel Vertex Unpacking & Rendering](4805-packed-voxel-vertex-unpacking-rendering.md)
Renders voxel geometry by unpacking a single 32-bit integer per vertex into spatial coordinates, material ID, face ID, ambient occlusion ID, and flip ID. It sets itself apart by decoding the packed format directly into vertex attributes for efficient GPU rendering, reducing vertex buffer overhead. Choose it when rendering large voxel grids with minimal draw calls.

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel, requiring a constant number of bitwise shifts and logical operations [G].
- **Memory Complexity:** Reduces storage per vertex to a single 32-bit integer, replacing multiple floating-point or integer attributes [G].
- **Strengths:** Compact cache-friendly layout, extremely fast bitwise packing/unpacking, minimal pipeline state changes [G].
- **Limitations:** Restricted to a 16×16×16 local coordinate space, 256 material types, and 6 face directions; unsuitable for high-precision coordinates or larger type sets without format extension [S1, unofficial].

## Background
The method originates from the `web_voxel_terrain` project, which implements it for web-based voxel terrain rendering [S1, unofficial].

## Sources
[S1] andrewbloese-00/web_voxel_terrain, https://github.com/andrewbloese-00/web_voxel_terrain, fetched 2026-10-27 (unofficial)
