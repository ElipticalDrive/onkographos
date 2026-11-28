# CVCT Cascade Light Sampling & Blending

Also called: Cascade-based volume sampling, Cascaded 3D Texture Sampling, Multi-resolution 3D texture interpolation

## Overview
CVCT Cascade Light Sampling & Blending samples a 3D texture containing volume and light data across multiple resolution cascades, interpolating between adjacent cascades based on distance to maintain consistent sampling density [S1, unofficial]. It solves the problem of fixed-resolution textures wasting memory or producing aliasing artifacts in real-time voxel cone tracing by providing smooth lighting transitions across varying distances from the light source [S1, unofficial]. It sets itself apart from its siblings by explicitly interpolating between adjacent cascade levels based on distance, whereas [Two-Bounce Indirect Illumination](4558-two-bounce-indirect-illumination.md) focuses on bounce count and [Multi-Cone Shading](5176-multi-cone-shading.md) focuses on cone geometry aggregation [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   `world_pos`: World-space position of the fragment [S1, unofficial].
*   `norm`: World-space surface normal vector [S1, unofficial].
*   `_CVCT_WorldToLightLocalMatrix`: 4x4 matrix transforming world coordinates to light/local space [S1, unofficial].
*   `_CVCT_LightTex3d`: 3D texture sampler holding the volume/light data [S1, unofficial].
*   `_CVCT_GridResolution`: A `float4` parameter where `.z` offsets the cascade calculation and `.y` scales the Y-axis for texture layout [S1, unofficial].

**Outputs:**
*   `light`: The final blended light value, modulated by the diffuse term [S1, unofficial].

### Data Structures
*   **`_CVCT_LightTex3d`**: A 3D texture. The shader reads only the red channel (`.r`) and assumes normalized coordinates after transformation [S1, unofficial].
*   **`_CVCT_GridResolution`**: A `float4` vector. `.z` provides a constant offset for cascade index calculation. `.y` scales the Y-axis coordinate during texture lookup. Other components are not specified in the sources [S1, unofficial].

### Steps
1.  **Transform to Light Space**: Transform `world_pos` and `norm` using `_CVCT_WorldToLightLocalMatrix` [S1, unofficial].
2.  **Backface Culling**: If `norm.z >= 0`, return 0. This culls fragments where the surface normal faces away from the light direction, assuming the light is along the negative Z axis in local space [S1, unofficial].
3.  **Compute Distance**: Calculate `central_distance` as the maximum absolute coordinate of the transformed position `pos` (i.e., `max(|pos.x|, |pos.y|, |pos.z|)`) [S1, unofficial].
4.  **Calculate Cascade Index**: Compute the floating-point cascade index `cascade_f` using the formula:
    $$
    cascade\_f = \log_2(\max(0.5, central\_distance)) + \_CVCT\_GridResolution.z
    $$
    The `max(0.5, ...)` guard prevents `log2(0)` or negative inputs [S1, unofficial].
5.  **Determine Integer Cascades**: Let `cascade` be the integer part of `cascade_f` and `fr` be the fractional blending factor between the current and next cascade [S1, unofficial].
6.  **Sample Current Cascade**:
    *   Scale `pos` to the current cascade bounds: `pos = pos * pow(0.5, cascade)` [S1, unofficial].
    *   Normalize `pos` to the [0, 1] range: `pos = pos * 0.5 + 0.5` [S1, unofficial].
    *   Adjust Y-coordinate for texture layout: `pos.y += cascade` [S1, unofficial].
    *   Rescale Y-coordinate: `pos.y *= _CVCT_GridResolution.y` [S1, unofficial].
    *   Sample the texture: If `pos.y < 1`, `light = tex3D(_CVCT_LightTex3d, pos).r`; otherwise, `light = 1`. This check acts as a fallback when Y exceeds texture bounds [S1, unofficial].
7.  **Sample Next Cascade**:
    *   Map coordinates to the next larger cascade: `pos = pos * 0.5` [S1, unofficial].
    *   Normalize coordinates to [0, 1]: `pos = pos * 0.5 + 0.5` [S1, unofficial].
    *   Adjust Y-coordinate: `pos.y += cascade + 1` [S1, unofficial].
    *   Rescale Y-coordinate: `pos.y *= _CVCT_GridResolution.y` [S1, unofficial].
    *   Sample the texture: If `pos.y < 1`, `light2 = tex3D(_CVCT_LightTex3d, pos).r`; otherwise, `light2 = 1` [S1, unofficial].
8.  **Blend**: Interpolate the results: `light = lerp(light, light2, fr)` [S1, unofficial].
9.  **Diffuse Modulation**: Return the final light value multiplied by the diffuse term: `light * (-normalize(norm).z)` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def cvct_cascade_light_sampling(world_pos, norm, CVCT_WorldToLightLocalMatrix, CVCT_LightTex3d, CVCT_GridResolution):
    """
    Samples CVCT light data with cascade blending.
    
    Args:
        world_pos: tuple(float, float, float), world space position
        norm: tuple(float, float, float), world space normal
        CVCT_WorldToLightLocalMatrix: list of lists (4x4), transformation matrix
        CVCT_LightTex3d: object with .sample(x, y, z) method returning a tuple
        CVCT_GridResolution: tuple(float, float, float, float), grid resolution parameters
        
    Returns:
        float, final light value
    """
    # 1. Transform to light space
    mat = CVCT_WorldToLightLocalMatrix
    pos = (
        mat[0][0]*world_pos[0] + mat[0][1]*world_pos[1] + mat[0][2]*world_pos[2] + mat[0][3],
        mat[1][0]*world_pos[0] + mat[1][1]*world_pos[1] + mat[1][2]*world_pos[2] + mat[1][3],
        mat[2][0]*world_pos[0] + mat[2][1]*world_pos[1] + mat[2][2]*world_pos[2] + mat[2][3]
    )
    norm = (
        mat[0][0]*norm[0] + mat[0][1]*norm[1] + mat[0][2]*norm[2],
        mat[1][0]*norm[0] + mat[1][1]*norm[1] + mat[1][2]*norm[2],
        mat[2][0]*norm[0] + mat[2][1]*norm[1] + mat[2][2]*norm[2]
    )
    
    # 2. Backface culling
    if norm[2] >= 0:
        return 0.0
    
    # 3. Compute central distance
    central_distance = max(abs(pos[0]), abs(pos[1]), abs(pos[2]))
    
    # 4. Calculate cascade index
    cascade_f = math.log2(max(0.5, central_distance)) + CVCT_GridResolution[2]
    
    # 5. Determine integer cascade and blend factor
    cascade = math.floor(cascade_f)
    fr = cascade_f - cascade
    
    # 6. Sample current cascade
    pos_curr = (pos[0] * (0.5 ** cascade), pos[1] * (0.5 ** cascade), pos[2] * (0.5 ** cascade))
    pos_curr = (pos_curr[0] * 0.5 + 0.5, pos_curr[1] * 0.5 + 0.5, pos_curr[2] * 0.5 + 0.5)
    pos_curr_y = pos_curr[1] + cascade
    pos_curr_y *= CVCT_GridResolution[1]
    light = CVCT_LightTex3d.sample(pos_curr[0], pos_curr[1], pos_curr[2])[0] if pos_curr_y < 1.0 else 1.0
    
    # 7. Sample next cascade
    pos_next = (pos_curr[0] * 0.5, pos_curr[1] * 0.5, pos_curr[2] * 0.5)
    pos_next = (pos_next[0] * 0.5 + 0.5, pos_next[1] * 0.5 + 0.5, pos_next[2] * 0.5 + 0.5)
    pos_next_y = pos_next[1] + cascade + 1
    pos_next_y *= CVCT_GridResolution[1]
    light2 = CVCT_LightTex3d.sample(pos_next[0], pos_next[1], pos_next[2])[0] if pos_next_y < 1.0 else 1.0
    
    # 8. Blend
    light = light + fr * (light2 - light)
    
    # 9. Diffuse modulation
    norm_len = math.sqrt(norm[0]**2 + norm[1]**2 + norm[2]**2)
    return light * (-norm[2] / norm_len)
```

### Parameters
*   `_CVCT_WorldToLightLocalMatrix`: 4x4 matrix [S1, unofficial].
*   `_CVCT_GridResolution`: `float4` [S1, unofficial].
*   `_CVCT_LightTex3d`: 3D texture sampler [S1, unofficial].
*   `cascade`: Integer or float index of the current voxel cascade level [S1, unofficial].
*   `fr`: Float blending factor between the current and next cascade [S1, unofficial].
*   `norm`: Surface normal vector [S1, unofficial].

### Edge Cases and Pitfalls
*   **Backface Culling**: If `norm.z >= 0`, the function returns 0. This assumes the light direction is along the negative Z axis in the local light space [S1, unofficial].
*   **Logarithm Domain**: The `max(0.5, central_distance)` ensures the argument to `log2` is never zero or negative, preventing undefined behavior [S1, unofficial].
*   **Texture Bounds**: The check `pos.y < 1` returns a value of 1 (full light) if the Y-coordinate exceeds the texture bounds. This acts as a fallback to prevent sampling out-of-bounds memory [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: O(1) per sample. The method performs exactly 2 texture fetches plus a constant number of arithmetic operations [S1, unofficial].
*   **Memory Complexity**: O(1) additional memory per sample, relying on the pre-computed 3D texture [S1, unofficial].
*   **Strengths**: Provides smooth lighting transitions across different voxel grid resolutions, maintaining consistent sampling density relative to distance from the light source [S1, unofficial].
*   **Limitations**: Requires a pre-computed multi-resolution 3D texture. Performance is constant but depends on texture fetch latency [S1, unofficial].
*   **When to Use**: Suitable for real-time voxel cone tracing where distance-based lighting attenuation and smooth transitions between resolution levels are required [S1, unofficial].
*   **When Not to Use**: Not suitable for scenes requiring high-frequency lighting details that exceed the resolution of the finest cascade, or where the light direction is not aligned with the assumed local space axis [S1, unofficial].

## Background
The method is part of the Cascaded Voxel Cone Tracing (CVCT) implementation, specifically designed to handle light sampling in participating media or for indirect illumination in voxel-based rendering [S1, unofficial]. It builds on the principles of [Customized Cone Tracing](3447-customized-cone-tracing.md) by refining the light evaluation step [S1, unofficial].

## Sources
[S1] arigo/CascatedVoxelConeTracing, https://github.com/arigo/CascatedVoxelConeTracing, fetched 2026-10-26 (unofficial)
