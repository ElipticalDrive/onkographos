# Voxel Face Packing

Also called: Packed Voxel Cell Encoding with Face/Type Info

## Overview
Voxel Face Packing encodes a voxel's local spatial coordinates, material type, and face direction index into a single 32-bit unsigned integer [S1, unofficial]. It solves the problem of efficiently transmitting or storing the result of a ray-marching step in voxel terrain rendering, where the hit voxel's position, material identity, and surface orientation must be recovered with minimal memory bandwidth [S1, unofficial]. It differs from its parent [Packed Voxel Cell Encoding](3281-packed-voxel-cell-encoding.md) by including explicit face direction and material type fields rather than just the coordinate and crossing axis, enabling direct geometry reconstruction [S1, unofficial].

## How it works
**Inputs:**
- `x`: Local x-coordinate of the voxel center, integer in range $[0, 15]$ [S1, unofficial].
- `y`: Local y-coordinate of the voxel center, integer in range $[0, 15]$ [S1, unofficial].
- `z`: Local z-coordinate of the voxel center, integer in range $[0, 15]$ [S1, unofficial].
- `type`: Voxel type identifier, integer in range $[0, 255]$ [S1, unofficial].
- `direction`: Face direction index, integer in range $[0, 5]$ [S1, unofficial].

**Outputs:**
- A single 32-bit unsigned integer (`uint32`) [S1, unofficial].

**Data Structure Layout:**
The 32-bit integer is packed as follows [S1, unofficial]:
- Bits 31–28 (4 bits): `x`
- Bits 27–24 (4 bits): `y`
- Bits 23–20 (4 bits): `z`
- Bits 19–12 (8 bits): `type`
- Bits 11–9 (3 bits): `direction`
- Bits 8–0 (9 bits): Unused

**Packing Algorithm:**
The values are combined using bitwise shifts and logical OR operations [S1, unofficial]:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack_voxel_face(x: int, y: int, z: int, type: int, direction: int) -> int:
    # Align shifts to match the explicit bit layout (31-28 for x, etc.)
    packed = (x << 28) | (y << 24) | (z << 20) | (type << 12) | (direction << 9)
    return packed
```

**Unpacking Algorithm:**
Values are recovered by shifting right and masking with bitwise AND [S1, unofficial]:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def unpack_voxel_face(packed: int) -> dict:
    x = (packed >> 28) & 0xF
    y = (packed >> 24) & 0xF
    z = (packed >> 20) & 0xF
    type = (packed >> 12) & 0xFF
    direction = (packed >> 9) & 0b111
    return {"x": x, "y": y, "z": z, "type": type, "direction": direction}
```

**Parameters and Constraints:**
- The method supports a local grid resolution of $16 \times 16 \times 16$ voxels per chunk due to the 4-bit coordinate fields [S1, unofficial].
- The `type` field supports 256 distinct material or block types [S1, unofficial].
- The `direction` field supports 6 face orientations (typically corresponding to the 6 axes of a cube: $\pm x, \pm y, \pm z$) [S1, unofficial].
- **Edge Cases:** The 9 lower bits (0–8) are unused [S1, unofficial]. If the input `direction` exceeds 5, it is masked with `0b111` (7), potentially causing ambiguity if values 6 or 7 are passed, though the source specifies the valid range is 0–5 [S1, unofficial]. Precision is limited to integer coordinates; sub-voxel precision is not encoded in this specific packing format [G].

## Subtopics

### [Packed Voxel Vertex Unpacking & Rendering](4805-packed-voxel-vertex-unpacking-rendering.md)
Unpacks a single 32-bit integer per vertex into spatial coordinates, material ID, face ID, ambient occlusion ID, and flip ID to render voxel geometry efficiently. It sets itself apart from sibling methods by packing all necessary rendering attributes into one integer, allowing the GPU to reconstruct the full vertex state and apply per-face shading and texture lookups without additional vertex buffer attributes. Choose this method when rendering voxel terrain on the GPU with strict vertex attribute limits [G].

## Operating characteristics
- **Time Complexity:** Packing and unpacking are $O(1)$ bitwise operations [G].
- **Memory Complexity:** Uses exactly 32 bits (4 bytes) per voxel face record, significantly reducing memory footprint compared to storing separate integers for coordinates, type, and direction [G].
- **Performance:** Bitwise operations are extremely fast on modern CPUs and GPUs. The compact size improves cache efficiency and reduces memory bandwidth during ray marching and geometry generation [G].
- **Strengths:** Minimal memory usage, fast serialization/deserialization, supports local chunk-based coordinate systems [G].
- **Limitations:** Limited to $16^3$ local resolution per packed unit; requires global coordinate offsetting if the voxel world exceeds $16^3$ voxels in any dimension; unused bits waste potential capacity [S1, unofficial].

## Background
Voxel Face Packing is used in voxel-based terrain rendering systems, particularly in web-based or lightweight 3D engines where memory bandwidth and rendering performance are critical [S1, unofficial]. It allows the ray marcher to return a compact result that can be directly consumed by geometry generators to create mesh vertices for the visible voxel faces [S1, unofficial].

## Sources
[S1] andrewbloese-00/web_voxel_terrain, https://github.com/andrewbloese-00/web_voxel_terrain, fetched 2026-10-27 (unofficial)
