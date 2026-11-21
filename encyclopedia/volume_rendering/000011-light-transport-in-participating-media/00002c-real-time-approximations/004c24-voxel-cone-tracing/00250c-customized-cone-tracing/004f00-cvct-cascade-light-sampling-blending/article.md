# CVCT Cascade Light Sampling & Blending

Also called: Cascade-based volume sampling, Cascaded 3D Texture Sampling, Multi-resolution 3D texture interpolation

## Overview
CVCT Cascade Light Sampling & Blending is a technique for evaluating volumetric light transport within the Customized Cone Tracing framework by sampling a multi-resolution 3D texture of light data. It solves the problem of maintaining consistent sampling density and smooth lighting transitions across varying distances from the light source, which is critical for real-time voxel cone tracing where fixed-resolution textures would either waste memory or produce aliasing artifacts. It sets itself apart from its siblings by explicitly interpolating between adjacent cascade levels based on distance, whereas [Two-Bounce Indirect Illumination](4558-two-bounce-indirect-illumination.md) focuses on bounce count and [Multi-Cone Shading](5176-multi-cone-shading.md) focuses on cone geometry aggregation.

## How it works
The method samples a 3D texture containing volume/light data across multiple resolution cascades. It transforms world-space coordinates into light space, determines the appropriate cascade level based on distance, and blends samples from two adjacent cascades to ensure smooth transitions.

### Inputs and Outputs
**Inputs:**
*   `world_pos`: World-space position of the fragment.
*   `norm`: World-space surface normal vector.
*   `_CVCT_WorldToLightLocalMatrix`: 4x4 matrix transforming world coordinates to light/local space [S1, unofficial].
*   `_CVCT_LightTex3d`: 3D texture sampler holding the volume/light data [S1, unofficial].
*   `_CVCT_GridResolution`: A `float4` parameter where `.z` offsets the cascade calculation and `.y` scales the Y-axis for texture layout [S1, unofficial].

**Outputs:**
*   `light`: The final blended light value, modulated by the diffuse term.

### Data Structures
*   **`_CVCT_LightTex3d`**: A 3D texture. The shader reads only the red channel (`.r`) and assumes normalized coordinates after transformation [S1, unofficial].
*   **`_CVCT_GridResolution`**: A `float4` vector.
    *   `.z`: Constant offset for cascade index calculation.
    *   `.y`: Scaling factor for the Y-axis coordinate during texture lookup.
    *   Other components are not specified in the sources [S1, unofficial].

### Steps
1.  **Transform to Light Space**: Transform `world_pos` and `norm` using `_CVCT_WorldToLightLocalMatrix`. Let the transformed position be `pos` and the transformed normal be `norm`.
2.  **Backface Culling**: If `norm.z >= 0`, return 0. This culls fragments where the surface normal faces away from the light direction, assuming the light is along the negative Z axis in local space [S1, unofficial].
3.  **Compute Distance**: Calculate `central_distance` as the maximum absolute coordinate of the transformed position `pos` (i.e., `max(|pos.x|, |pos.y|, |pos.z|)`) [S1, unofficial].
4.  **Calculate Cascade Index**: Compute the floating-point cascade index `cascade_f` using the formula:
    $$
    cascade\_f = \log_2(\max(0.5, central\_distance)) + \_CVCT\_GridResolution.z
    $$
    The `max(0.5, ...)` guard prevents `log2(0)` or negative inputs [S1, unofficial].
5.  **Determine Integer Cascades**: Let `cascade` be the integer part of `cascade_f` (or the current cascade level being processed) and `fr` be the fractional blending factor between the current and next cascade [S1, unofficial].
6.  **Sample Current Cascade**:
    *   Scale `pos` to the current cascade bounds: `pos = pos * pow(0.5, cascade)`.
    *   Normalize `pos` to the [0, 1] range: `pos = pos * 0.5 + 0.5`.
    *   Adjust Y-coordinate for texture layout: `pos.y += cascade`.
    *   Rescale Y-coordinate: `pos.y *= _CVCT_GridResolution.y`.
    *   Sample the texture: If `pos.y < 1`, `light = tex3D(_CVCT_LightTex3d, pos).r`; otherwise, `light = 1`. This check acts as a fallback when Y exceeds texture bounds [S1, unofficial].
7.  **Sample Next Cascade**:
    *   Map coordinates to the next larger cascade: `pos1 = pos * 0.5` (Note: `pos` here refers to the position scaled/normalized for the *current* cascade before the Y-adjustment, or the original scaled position depending on implementation order. Based on the code snippet, `pos1` is derived from the position before the final Y-adjustment of the current cascade, or specifically, the code snippet shows `pos = pos1 * 0.5` implying `pos1` was the previous state. Let's follow the code snippet logic strictly:
        *   Let `pos_curr` be the position normalized to [0,1] for the current cascade (before Y-add).
        *   `pos_next_base = pos_curr * 0.5`.
        *   `pos_next = pos_next_base * 0.5 + 0.5`.
        *   `pos_next.y += cascade + 1`.
        *   `pos_next.y *= _CVCT_GridResolution.y`.
        *   If `pos_next.y < 1`, `light2 = tex3D(_CVCT_LightTex3d, pos_next).r`; otherwise, `light2 = 1`.
8.  **Blend**: Interpolate the results: `light = lerp(light, light2, fr)` [S1, unofficial].
9.  **Diffuse Modulation**: Return the final light value multiplied by the diffuse term: `light * (-normalize(norm).z)` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cvct_cascade_light_sampling(world_pos, norm, CVCT_WorldToLightLocalMatrix, CVCT_LightTex3d, CVCT_GridResolution):
    """
    Samples CVCT light data with cascade blending.
    
    Args:
        world_pos: float3, world space position
        norm: float3, world space normal
        CVCT_WorldToLightLocalMatrix: float4x4, transformation matrix
        CVCT_LightTex3d: 3D texture sampler
        CVCT_GridResolution: float4, grid resolution parameters
        
    Returns:
        float, final light value
    """
    # 1. Transform to light space
    pos = mul(CVCT_WorldToLightLocalMatrix, float4(world_pos, 1.0)).xyz
    norm = mul(CVCT_WorldToLightLocalMatrix, float4(norm, 0.0)).xyz
    
    # 2. Backface culling
    if norm.z >= 0:
        return 0.0
    
    # 3. Compute central distance
    central_distance = max(abs(pos.x), max(abs(pos.y), abs(pos.z)))
    
    # 4. Calculate cascade index
    # Guard against log2(0) or negative inputs
    cascade_f = log2(max(0.5, central_distance)) + CVCT_GridResolution.z
    
    # Determine integer cascade and blend factor
    cascade = floor(cascade_f)
    fr = frac(cascade_f)
    
    # 5. Sample Current Cascade
    # Scale position to current cascade bounds
    pos_curr = pos * pow(0.5, cascade)
    
    # Normalize to [0, 1]
    pos_curr = pos_curr * 0.5 + 0.5
    
    # Store base position for next cascade calculation
    pos_next_base = pos_curr * 0.5
    
    # Adjust Y for current cascade
    pos_curr_y = pos_curr.y + cascade
    pos_curr_y *= CVCT_GridResolution.y
    
    # Sample current cascade
    if pos_curr_y < 1:
        light = CVCT_LightTex3d.Sample(pos_curr.x, pos_curr.y, pos_curr.z).r
    else:
        light = 1.0
        
    # 6. Sample Next Cascade
    # Map to next larger cascade
    pos_next = pos_next_base * 0.5 + 0.5
    
    # Adjust Y for next cascade
    pos_next_y = pos_next.y + (cascade + 1)
    pos_next_y *= CVCT_GridResolution.y
    
    # Sample next cascade
    if pos_next_y < 1:
        light2 = CVCT_LightTex3d.Sample(pos_next.x, pos_next.y, pos_next.z).r
    else:
        light2 = 1.0
        
    # 7. Blend
    light = lerp(light, light2, fr)
    
    # 8. Diffuse modulation
    return light * (-normalize(norm).z)
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
*   **Strengths**: Provides smooth lighting transitions across different voxel grid resolutions, maintaining consistent sampling density relative to distance from the light source.
*   **Limitations**: Requires a pre-computed multi-resolution 3D texture. Performance is constant but depends on texture fetch latency.
*   **When to Use**: Suitable for real-time voxel cone tracing where distance-based lighting attenuation and smooth transitions between resolution levels are required.
*   **When Not to Use**: Not suitable for scenes requiring high-frequency lighting details that exceed the resolution of the finest cascade, or where the light direction is not aligned with the assumed local space axis.

## Background
The method is part of the Cascaded Voxel Cone Tracing (CVCT) implementation, specifically designed to handle light sampling in participating media or for indirect illumination in voxel-based rendering. It builds on the principles of [Customized Cone Tracing](3447-customized-cone-tracing.md) by refining the light evaluation step.

## Sources
[S1] arigo/CascatedVoxelConeTracing, https://github.com/arigo/CascatedVoxelConeTracing, fetched 2026-10-26 (unofficial)
