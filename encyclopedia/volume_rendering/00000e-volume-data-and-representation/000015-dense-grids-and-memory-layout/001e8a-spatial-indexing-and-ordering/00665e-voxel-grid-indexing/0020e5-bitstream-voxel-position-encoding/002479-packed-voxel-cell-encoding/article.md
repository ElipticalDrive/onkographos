# Packed Voxel Cell Encoding
## Overview
Packed Voxel Cell Encoding is a bitstream method that compresses a voxel's 3D integer coordinate and the axis along which a ray exited that voxel into a single 32-bit unsigned integer [S1, unofficial]. It solves the problem of efficiently returning hit voxel data from a ray marcher while preserving the geometric context required for normal estimation and subsequent traversal [S1, unofficial]. It differs from its parent [Bitstream Voxel Position Encoding](2371-bitstream-voxel-position-encoding.md) by explicitly storing the crossing axis alongside the coordinate, enabling direct reconstruction of the intersection point and surface normal without additional ray-marching steps [S1, unofficial].

## How it works
**Inputs:** A voxel coordinate $(x, y, z)$ where each component is a signed 16-bit integer, and a crossing axis index $a \in \{0, 1, 2\}$ representing the X, Y, or Z axis respectively [S1, unofficial].
**Output:** A single 32-bit unsigned integer (`uint`) [S1, unofficial].

**Data Structure Layout:**
The 32-bit integer is divided into four fields: bits 0–9 (10 bits) store the X coordinate, bits 10–19 (10 bits) store the Y coordinate, bits 20–29 (10 bits) store the Z coordinate, and bits 30–31 (2 bits) store the crossing axis index [S1, unofficial].

**Constraints:**
The coordinate components must fit within 10 bits, limiting the representable range to $[-512, 511]$ for signed values [S1, unofficial]. The axis index is strictly limited to 0, 1, or 2 [S1, unofficial].

**Pseudocode:**
The following Python-like pseudocode demonstrates packing, unpacking, and the geometric reconstruction of the intersection point and surface normal as implemented in the source [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack_voxel_cell(coord: tuple[int, int, int], axis: int) -> int:
    x, y, z = coord
    packed = 0
    packed |= (x & 0x3FF) << 0
    packed |= (y & 0x3FF) << 10
    packed |= (z & 0x3FF) << 20
    packed |= (axis & 0x3) << 30
    return packed

def unpack_voxel_cell(packed_cell: int) -> tuple[tuple[int, int, int], int]:
    axis = (packed_cell >> 30) & 0x3
    x = (packed_cell >> 0) & 0x3FF
    y = (packed_cell >> 10) & 0x3FF
    z = (packed_cell >> 20) & 0x3FF
    if x & 0x200: x -= 0x400
    if y & 0x200: y -= 0x400
    if z & 0x200: z -= 0x400
    return ((x, y, z), axis)

def reconstruct_intersection(ray_origin: list[float], ray_dir: list[float], packed_cell: int) -> tuple[list[float], list[float]]:
    coord, axis = unpack_voxel_cell(packed_cell)
    cell = [float(c) for c in coord]
    axis_mask = [1.0 if axis == i else 0.0 for i in range(3)]
    sign_val = [-1.0 if ray_dir[i] > 0 else 1.0 if ray_dir[i] < 0 else 0.0 for i in range(3)]
    normal = [-sign_val[i] * axis_mask[i] for i in range(3)]
    entry_plane = [cell[i] + (1.0 if ray_dir[i] < 0.0 else 0.0) for i in range(3)]
    dot_entry = sum((entry_plane[i] - ray_origin[i]) * axis_mask[i] for i in range(3))
    dot_dir = sum(ray_dir[i] * axis_mask[i] for i in range(3))
    distance = dot_entry / dot_dir
    position = [ray_origin[i] + distance * ray_dir[i] for i in range(3)]
    return position, normal
```

**Edge Cases and Pitfalls:**
- **Coordinate Range:** The 10-bit storage limits the coordinate range to $[-512, 511]$ [S1, unofficial].
- **Axis Index:** The axis index must be strictly 0, 1, or 2 [S1, unofficial].
- **Ray Direction Zero:** If `ray_dir` has a zero component along the crossing axis, the distance calculation involves division by zero [S1, unofficial]. This should not happen in a valid ray-marching step where a voxel exit is detected [G].

## Subtopics

### [Voxel Face Packing](4166-voxel-face-packing.md)
Compresses a voxel's local spatial coordinates, material type, and the specific face direction of a ray intersection into a single 32-bit unsigned integer. It differs from Packed Voxel Cell Encoding by including explicit face direction and material type fields rather than just the coordinate and crossing axis, enabling direct geometry reconstruction. Choose it when transmitting or storing ray-marching results in voxel terrain rendering requires minimal memory bandwidth and direct recovery of surface normals and material identity.

## Operating characteristics
**Time Complexity:** Not given in the sources [S1, unofficial].
**Memory Complexity:** Not given in the sources [S1, unofficial].
**Performance:** Not given in the sources [S1, unofficial].
**Strengths:** Not given in the sources [S1, unofficial].
**Limitations:** Not given in the sources [S1, unofficial].

## Background
Packed Voxel Cell Encoding is used in the vxray renderer [S1, unofficial]. It is part of a family of bitstream voxel position encoding methods designed for efficient voxel terrain rendering [S1, unofficial]. The method allows the ray marcher to return a compact representation of the hit voxel, which can then be used for shading, normal estimation, and further traversal [S1, unofficial].

## Sources
[S1] Nelarius/vxray, https://github.com/Nelarius/vxray, fetched 2026-10-24 (unofficial)
