# Voxel UV Unwrapping for AO Interpolation

Also called: Voxel UV Unwrapping

## Overview
This method computes 2D texture coordinates from a world-space hit position and normal to interpolate Ambient Occlusion (AO) values across a voxel surface [S1, unofficial]. It projects voxel-space coordinates onto a 2D plane using the surface normal and wraps the result with modulo arithmetic to map onto a texture atlas [S1, unofficial].

## How it works
**Inputs:**
- `hit.pos`: World-space intersection position [S1, unofficial].
- `hit.normal`: World-space surface normal at the intersection [S1, unofficial].
- `VOXELS_PER_METER`: Voxel density constant [S1, unofficial].
- `texture_size`: Square texture atlas dimension [S1, unofficial].
- `ao`: Four pre-computed AO values (`ao.x`, `ao.y`, `ao.z`, `ao.w`) [S1, unofficial].

**Outputs:**
- `voxel_ao`: Final interpolated and gamma-corrected AO scalar [S1, unofficial].

**Data Structures:**
- `texture_coords`: 3D vector for scaled voxel-space position [S1, unofficial].
- `uv`: 2D vector for normalized texture coordinates in $[0, 1)$ [S1, unofficial].
- `interpolated_ao_pweig`: Scalar for bilinearly interpolated AO [S1, unofficial].

**Steps:**

1. **Calculate Voxel-Space Texture Coordinates:**
   $$ \text{texture\_coords} = \text{hit.pos} \times \text{VOXELS\_PER\_METER} + \frac{\text{texture\_size}}{2.0} $$
   [S1, unofficial]

2. **Calculate UV Coordinates:**
   $$ \text{uv} = \text{glmod}\left( \begin{bmatrix} \text{dot}(\text{hit.normal} \times \text{texture\_coords.yzx}, \text{vec3}(1.0)) \\ \text{dot}(\text{hit.normal} \times \text{texture\_coords.zxy}, \text{vec3}(1.0)) \end{bmatrix}, 1.0 \right) $$
   [S1, unofficial]

3. **Interpolate AO Values:**
   $$ \text{interpolated\_ao\_pweig} = \text{mix}\left( \text{mix}(\text{ao.z}, \text{ao.w}, \text{uv.x}), \text{mix}(\text{ao.y}, \text{ao.x}, \text{uv.x}), \text{uv.y} \right) $$
   [S1, unofficial]

4. **Apply Gamma Correction:**
   $$ \text{voxel\_ao} = \text{pow}(\text{interpolated\_ao\_pweig}, 1.0 / 3.0) $$
   [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_uv_unwrap_for_ao(hit_pos, hit_normal, voxels_per_meter, texture_size, ao):
    """Computes AO value for a voxel surface intersection [S1, unofficial]."""
    texture_coords = hit_pos * voxels_per_meter + texture_size / 2.0
    
    normal_yzx = hit_normal * texture_coords.yzx
    uv_x = (normal_yzx.x + normal_yzx.y + normal_yzx.z) % 1.0
    
    normal_zxy = hit_normal * texture_coords.zxy
    uv_y = (normal_zxy.x + normal_zxy.y + normal_zxy.z) % 1.0
    
    uv = (uv_x, uv_y)
    
    mix_zw = mix(ao.z, ao.w, uv.x)
    mix_yx = mix(ao.y, ao.x, uv.x)
    interpolated_ao_pweig = mix(mix_zw, mix_yx, uv.y)
    
    voxel_ao = pow(interpolated_ao_pweig, 1.0 / 3.0)
    return voxel_ao

def mix(a, b, t):
    return a * (1.0 - t) + b * t
```

**Parameters:**
- `VOXELS_PER_METER`: Scene-dependent density. Not given in the sources.
- `texture_size`: Atlas resolution. Not given in the sources.
- Gamma exponent: Fixed at $1/3.0$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Degenerate Input:** If `hit.normal` is zero, the dot products yield zero, resulting in `uv = (0, 0)` [G].
- **Precision & Boundaries:** Floating-point precision in `texture_coords` may cause UV jitter at large coordinates; the modulo operation wraps UVs to $[0, 1)$ but may cause discontinuities at integer boundaries without appropriate texture filtering [G].

## Subtopics

### [Voxel Face Texture Coordinate Unwrapping](14779-voxel-face-texture-coordinate-unwrapping.md)
This method computes 2D texture coordinates for a voxel/cube grid based on vertex position and normal, handling face selection and coordinate mapping.

## Operating characteristics
**Time Complexity:** $O(1)$ per fragment. The method performs a constant number of arithmetic operations [G].
**Memory Complexity:** $O(1)$ auxiliary memory. Requires storage for `texture_coords`, `uv`, and `interpolated_ao_pweig` [G].
**Measured Performance:** Not given in the sources.
**Strengths:** Deterministic projection based on surface normal and simple arithmetic operations suitable for GPU shaders [G].
**Limitations:** Relies on pre-computed AO values at voxel corners/faces [S1, unofficial].
**When to Use:** Rendering voxel-based scenes with pre-computed AO data where consistent texture mapping across boundaries is required [G].
**When Not to Use:** High-fidelity, non-deterministic UV parameterization or dynamic per-frame AO computation without pre-computation [G].

## Background
This method originates from the `bevy-voxel-engine` project, an unofficial voxel engine for the Bevy game engine [S1, unofficial]. It is used to render voxel data with ambient occlusion effects by leveraging the discrete nature of voxel grids to simplify UV unwrapping through normal-based projection [S1, unofficial].

## Sources
[S1] ria8651/bevy-voxel-engine, https://github.com/ria8651/bevy-voxel-engine, fetched 2026-10-27 (unofficial)
