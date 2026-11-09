# Voxel Material Packing

Also called: Voxel Data Packing, Voxel uint32_t layout

## Overview
Voxel Material Packing is a data layout for storing material properties and color within a single 32-bit voxel value [S1, unofficial]. It allows up to 256 distinct materials to be managed by a MaterialManager [S1, unofficial]. It builds on [Voxel Data Layout](4076-voxel-data-layout.md) [S1, unofficial].

## How it works
**Inputs:** Material ID (8-bit integer), Red channel (8-bit integer), Green channel (8-bit integer), Blue channel (8-bit integer) [S1, unofficial].
**Outputs:** A single 32-bit unsigned integer (`uint32_t`) [S1, unofficial].

**Data Structure:**
The 32-bit integer is divided into four 8-bit fields [S1, unofficial]. The layout assigns significance from most to least significant byte:
- Bits [31..24]: Material ID (8 bits)
- Bits [23..16]: Red channel (8 bits)
- Bits [15..8]: Green channel (8 bits)
- Bits [7..0]: Blue channel (8 bits) [S1, unofficial].

**Steps:**
1. Validate that all input components are within the 0–255 range.
2. Shift the material ID left by 24 bits.
3. Shift the Red channel left by 16 bits.
4. Shift the Green channel left by 8 bits.
5. Combine the shifted values with the Blue channel using bitwise OR. [G]

**Formulas:**
Given `mat_id`, `r`, `g`, `b` as 8-bit integers:
$$
\text{packed} = (\text{mat\_id} \ll 24) \ | \ (r \ll 16) \ | \ (g \ll 8) \ | \ b
$$
Given `packed`:
$$
\text{mat\_id} = (\text{packed} \gg 24) \ \& \ 0xFF
$$
$$
r = (\text{packed} \gg 16) \ \& \ 0xFF
$$
$$
g = (\text{packed} \gg 8) \ \& \ 0xFF
$$
$$
b = \text{packed} \ \& \ 0xFF
$$ [S1, unofficial].

**Parameters:**
Maximum number of materials managed by the MaterialManager is 256 [S1, unofficial].

**Edge Cases and Pitfalls:**
Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack_voxel_material(mat_id: int, r: int, g: int, b: int) -> int:
    """Packs material ID and RGB color into a single 32-bit integer."""
    return (mat_id << 24) | (r << 16) | (g << 8) | b

def unpack_voxel_material(packed: int) -> tuple:
    """Unpacks a 32-bit integer into material ID and RGB color."""
    mat_id = (packed >> 24) & 0xFF
    r = (packed >> 16) & 0xFF
    g = (packed >> 8) & 0xFF
    b = packed & 0xFF
    return (mat_id, r, g, b)
```

## Subtopics

### Voxel Block Data Layout
A variant where the 32-bit integer stores voxel coordinates rather than material properties [S2, unofficial]. Bits 24–31 hold x, bits 16–23 hold y, and bits 8–15 hold z [S2, unofficial]. The color is stored separately as an 8-bit palette index in the Block object, not packed into the integer [S2, unofficial]. Coordinates are packed into a single integer using bitwise shifts and masks, while color is passed as a second argument [S2, unofficial]. A pitfall is that an inline comment lists byte order as color, x, y, z, but the implementation packs x, y, z into the integer with x as the most significant byte, and color is excluded from the integer [S2, unofficial].

## Operating characteristics
Memory usage is 4 bytes per voxel [S1, unofficial]. Maximum supported materials is 256 [S1, unofficial]. Time complexity, measured performance, scaling, strengths, limitations, and usage recommendations are Not given in the sources.

## Background
Origin, history, development, and specific usage contexts are Not given in the sources.

## Sources
[S1] NiekMSoftware/bad-apple-voxel-raytracer, https://github.com/NiekMSoftware/bad-apple-voxel-raytracer, fetched 2026-10-24 (unofficial)
[S2] Show HN: Basic voxel engine in WebGL/JavaScript with full source, https://github.com/Lallassu/vox2, fetched 2026-10-26 (unofficial)
