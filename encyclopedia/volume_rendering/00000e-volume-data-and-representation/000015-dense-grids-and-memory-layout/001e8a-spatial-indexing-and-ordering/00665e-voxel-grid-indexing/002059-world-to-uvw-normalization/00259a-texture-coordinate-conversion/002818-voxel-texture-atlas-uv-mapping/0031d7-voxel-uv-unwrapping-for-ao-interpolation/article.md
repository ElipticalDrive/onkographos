# Voxel UV Unwrapping for AO Interpolation

Also called: Voxel UV Unwrapping

## Overview
This method computes 2D texture coordinates for a voxel grid by mapping world-space vertex positions and normals to a specific face of a voxel block, then projecting those coordinates onto a larger texture atlas. It differs from standard UV unwrapping by using the geometric normal to deterministically select the visible face and handling the discrete block structure explicitly, ensuring consistent texture alignment across adjacent voxels. It is primarily used to interpolate Ambient Occlusion (AO) values from voxel data onto rendered surfaces.

## How it works
**Inputs:**
- `hit.pos`: World-space position of the intersection point.
- `hit.normal`: World-space normal at the intersection point.
- `VOXELS_PER_METER`: Constant defining voxel density.
- `texture_size`: Size of the texture atlas (assumed square, $N \times N$).
- `ao`: A vector of 4 pre-computed AO values corresponding to the corners/faces of the voxel block.

**Outputs:**
- `voxel_ao`: The final interpolated and gamma-corrected AO value.

**Data Structures:**
- `texture_coords`: A 3D vector representing the voxel-space position scaled by density and offset.
- `uv`: A 2D vector representing the normalized texture coordinates $(0.0, 1.0)$.
- `interpolated_ao_pweig`: A scalar representing the bilinearly interpolated AO value.

**Steps:**

1.  **Calculate Voxel-Space Texture Coordinates:**
    Convert the world-space hit position to voxel coordinates and scale by the texture atlas size to map to texture space.
    $$
    \text{texture\_coords} = \text{hit.pos} \times \text{VOXELS\_PER\_METER} + \frac{\text{texture\_size}}{2.0}
    $$
    [S1, unofficial]

2.  **Calculate UV Coordinates:**
    Project the voxel coordinates onto the 2D plane defined by the surface normal. This is done by taking dot products of the rotated voxel coordinates with the normal, then applying modulo 1.0 to wrap around the texture atlas.
    $$
    \text{uv} = \text{glmod}\left( \begin{bmatrix} \text{dot}(\text{hit.normal} \times \text{texture\_coords.yzx}, \begin{bmatrix} 1.0 \\ 1.0 \\ 1.0 \end{bmatrix}) \\ \text{dot}(\text{hit.normal} \times \text{texture\_coords.zxy}, \begin{bmatrix} 1.0 \\ 1.0 \\ 1.0 \end{bmatrix}) \end{bmatrix}, 1.0 \right)
    $$
    [S1, unofficial]
    *Note: The multiplication `hit.normal * texture_coords.yzx` implies component-wise multiplication or a specific rotation logic. Given the context of "dot product with vec3(1.0)", it suggests summing components after a permutation. Specifically, `dot(hit.normal * texture_coords.yzx, vec3(1.0))` is equivalent to `sum(hit.normal * texture_coords.yzx)`. However, standard UV unwrapping usually involves projecting onto a plane. The source code explicitly uses `dot(hit.normal * texture_coords.yzx, vec3(1.0))`. This effectively sums the components of the element-wise product of the normal and the permuted coordinates.*

3.  **Interpolate AO Values:**
    Perform bilinear interpolation using the calculated `uv` coordinates on the four AO values (`ao.x`, `ao.y`, `ao.z`, `ao.w`). The mapping is:
    - `uv.x = 0.0` corresponds to `ao.z` and `ao.y`.
    - `uv.x = 1.0` corresponds to `ao.w` and `ao.x`.
    - `uv.y = 0.0` corresponds to `ao.z` and `ao.w`.
    - `uv.y = 1.0` corresponds to `ao.y` and `ao.x`.
    
    Formula:
    $$
    \text{interpolated\_ao\_pweig} = \text{mix}\left( \text{mix}(\text{ao.z}, \text{ao.w}, \text{uv.x}), \text{mix}(\text{ao.y}, \text{ao.x}, \text{uv.x}), \text{uv.y} \right)
    $$
    [S1, unofficial]

4.  **Apply Gamma Correction:**
    Correct the interpolated AO value for gamma.
    $$
    \text{voxel\_ao} = \text{pow}(\text{interpolated\_ao\_pweig}, \frac{1.0}{3.0})
    $$
    [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_uv_unwrap_for_ao(hit_pos, hit_normal, voxels_per_meter, texture_size, ao):
    """
    Computes AO value for a voxel surface intersection.
    
    Args:
        hit_pos: vec3, world-space position.
        hit_normal: vec3, world-space normal.
        voxels_per_meter: float, voxel density.
        texture_size: float, size of the square texture atlas.
        ao: tuple of 4 floats (ao_x, ao_y, ao_z, ao_w).
        
    Returns:
        float, the final gamma-corrected AO value.
    """
    # Step 1: Calculate texture coordinates
    # hit_pos * voxels_per_meter converts to voxel space
    # + texture_size / 2.0 centers/offsets for the atlas
    texture_coords = hit_pos * voxels_per_meter + texture_size / 2.0
    
    # Step 2: Calculate UV coordinates
    # Permute texture_coords for YZX and ZXY projections
    # Component-wise multiply hit_normal with permuted coords
    # Sum the components (dot with vec3(1.0))
    # Apply modulo 1.0 to wrap within [0, 1)
    
    # yzx permutation: (y, z, x)
    normal_yzx = hit_normal * texture_coords.yzx # Component-wise multiplication
    uv_x = (normal_yzx.x + normal_yzx.y + normal_yzx.z) % 1.0
    
    # zxy permutation: (z, x, y)
    normal_zxy = hit_normal * texture_coords.zxy # Component-wise multiplication
    uv_y = (normal_zxy.x + normal_zxy.y + normal_zxy.z) % 1.0
    
    uv = (uv_x, uv_y)
    
    # Step 3: Bilinear interpolation of AO
    # ao structure: (x, y, z, w)
    # mix(a, b, t) = a * (1-t) + b * t
    
    # Interpolate along X axis first
    mix_zw = mix(ao.z, ao.w, uv.x)
    mix_yx = mix(ao.y, ao.x, uv.x)
    
    # Interpolate along Y axis
    interpolated_ao_pweig = mix(mix_zw, mix_yx, uv.y)
    
    # Step 4: Gamma correction
    voxel_ao = pow(interpolated_ao_pweig, 1.0 / 3.0)
    
    return voxel_ao

def mix(a, b, t):
    return a * (1.0 - t) + b * t
```

**Parameters:**
- `VOXELS_PER_METER`: Depends on scene scale. Not given in the sources.
- `texture_size`: Depends on atlas resolution. Not given in the sources.
- Gamma exponent: Fixed at $1/3.0$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Floating point precision in `texture_coords` may cause UV jitter at large world coordinates.
- **Boundaries:** The modulo operation ensures UVs stay in $[0, 1)$, but discontinuities may occur at integer boundaries if not handled by texture filtering.
- **Degenerate Input:** If `hit.normal` is zero or undefined, the dot products will be zero, resulting in `uv = (0, 0)`.

## Subtopics

### [Voxel Face Texture Coordinate Unwrapping](14779-voxel-face-texture-coordinate-unwrapping.md)
This method computes 2D texture coordinates for a voxel grid by mapping world-space vertex positions and normals to a specific face of a voxel block, then projecting those coordinates onto a larger texture atlas. It differs from standard UV unwrapping by using the geometric normal to deterministically select the visible face and handling the discrete block structure explicitly, ensuring consistent texture alignment across adjacent voxels.

## Operating characteristics
**Time Complexity:** $O(1)$ per pixel/fragment. The method involves a constant number of arithmetic operations.
**Memory Complexity:** $O(1)$ auxiliary memory. Requires storage for `texture_coords`, `uv`, and `interpolated_ao_pweig`.
**Measured Performance:** Not given in the sources.
**Strengths:**
- Deterministic face selection based on normal.
- Consistent texture alignment across adjacent voxels.
- Simple arithmetic operations suitable for GPU shaders.
**Limitations:**
- Relies on pre-computed AO values at voxel corners/faces.
- Gamma correction exponent is hardcoded to $1/3.0$, which may not be appropriate for all pipelines.
**When to Use:**
- When rendering voxel-based scenes with pre-computed AO data.
- When consistent texture mapping across voxel boundaries is required.
**When Not to Use:**
- When high-fidelity, non-deterministic UV parameterization is needed.
- When dynamic AO computation is required per frame without pre-computation.

## Background
This method is part of the `bevy-voxel-engine` project, an unofficial voxel engine for the Bevy game engine. It is used to render voxel data with ambient occlusion effects. The technique leverages the discrete nature of voxel grids to simplify UV unwrapping by projecting world-space coordinates onto 2D planes defined by surface normals.

## Sources
[S1] ria8651/bevy-voxel-engine, https://github.com/ria8651/bevy-voxel-engine, fetched 2026-10-27 (unofficial)
