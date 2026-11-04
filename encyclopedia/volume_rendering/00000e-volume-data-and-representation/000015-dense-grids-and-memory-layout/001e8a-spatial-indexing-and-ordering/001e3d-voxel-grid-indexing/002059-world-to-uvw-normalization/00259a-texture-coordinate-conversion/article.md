# Texture Coordinate Conversion

Also called: local2tex

## Overview
Texture Coordinate Conversion is a utility function that transforms local or world space coordinates ($P$) into normalized texture coordinates ($uv$) suitable for volume sampling [S1, unofficial]. It differs from its parent, [World to UVW Normalization](2229-world-to-uvw-normalization.md), by applying specific shader-space adjustments: an inverse volume size scaling, a 0.5 offset to center the sampling in the voxel, and a Y-axis flip to align with texture coordinate conventions [S1, unofficial].

## How it works
**Inputs:**
- $P$: Input position vector (float3), typically in local or world space [S1, unofficial].
- $g\_cf3InvVolSize$: Inverse dimensions of the volume, used to normalize coordinates to the $[0, 1]$ range [S1, unofficial].

**Outputs:**
- $uv$: Normalized texture coordinates (float3) in the range $[0, 1]$ [S1, unofficial].

**Steps:**
1. **Scale and Offset:** Multiply the input position $P$ by the inverse volume size $g\_cf3InvVolSize$ and add a constant offset of $0.5$ [S1, unofficial].
   $$ uv = P \times g\_cf3InvVolSize + 0.5 $$
2. **Flip Y-Axis:** Invert the Y component of the resulting vector [S1, unofficial].
   $$ uv.y = 1 - uv.y $$

**Formulas:**
$$ uv_x = P_x \cdot S_x + 0.5 $$
$$ uv_y = 1 - (P_y \cdot S_y + 0.5) $$
$$ uv_z = P_z \cdot S_z + 0.5 $$
where $S_x, S_y, S_z$ are the components of $g\_cf3InvVolSize$ [S1, unofficial].

**Parameters:**
- $g\_cf3InvVolSize$: Inverse dimensions of the volume. For a volume of size $N_x \times N_y \times N_z$, $S_x = 1/N_x$, $S_y = 1/N_y$, $S_z = 1/N_z$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Precision:** If $P$ is exactly at the boundary of the volume, the resulting $uv$ may be slightly outside $[0, 1]$ due to floating-point precision. Texture samplers usually clamp or wrap, but explicit clamping may be required for correctness [G].
- **Coordinate System Mismatch:** The Y-flip assumes a specific relationship between the voxel grid's Y-axis and the texture's Y-axis. If the volume data is stored with a different orientation, this flip may need to be removed or applied to a different axis [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def texture_coordinate_conversion(P, g_cf3InvVolSize):
    uv = P * g_cf3InvVolSize + 0.5
    uv.y = 1.0 - uv.y
    return uv
```

## Subtopics

### [Voxel Texture Atlas UV Mapping](4162-voxel-texture-atlas-uv-mapping.md)
Maps voxel face types and directions to a 2D texture atlas using calculated UV coordinates, incorporating safety margins to prevent texture bleeding. It differs from standard texture coordinate conversion by explicitly handling the discrete nature of voxel faces, remapping vertices to ensure unique UVs per face type, and applying per-voxel shading logic within the sampling pipeline. When to choose it is not given in the sources.

## Operating characteristics
Not given in the sources.

## Background
This method is derived from the Dynamic Volume Raycasting implementation by pengliu916, which uses a fixed-size voxel grid (e.g., $256 \times 256 \times 256$) and requires mapping ray intersection points to texture coordinates for trilinear interpolation [S1, unofficial].

## Sources
[S1] pengliu916/DynamicVolumeRayCasting, https://github.com/pengliu916/DynamicVolumeRayCasting, fetched 2026-10-26 (unofficial)
[S2] Voxel Texture Atlas UV Mapping, Not given in the sources.
