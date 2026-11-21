# PCF Shadow Lookup with Slope Scaling and Edge Fade

Also called: PCF with Slope Scaling and Edge Fade

## Overview
This method computes sun shadow occlusion at a world position by performing Percentage-Closer Filtering (PCF) on a shadow map, augmented with slope-scaled depth bias and distance-based edge fading [S1, unofficial]. It solves the twin problems of shadow acne/peter-panning (via normal-offset bias) and harsh, aliased shadow boundaries at the edges of the shadow map (via smooth fading) [S1, unofficial]. It differs from standard PCF Shadow Mapping by dynamically adjusting the depth bias based on surface orientation relative to the light and by softening the shadow intensity at the periphery of the shadow map to prevent hard cutoff artifacts [S1, unofficial].

## How it works
**Inputs:**
- `world_pos`: The 3D world-space position of the fragment being shaded.
- `normal`: The surface normal at `world_pos`, normalized.
- `scene.light_view_proj`: The 4x4 view-projection matrix transforming world space to light clip space.
- `scene.shadow_params`: A uniform vector containing:
  - `x`: Texel size in UV space.
  - `y`: Base normal offset magnitude.
  - `z`: Shadow strength multiplier (0 disables shadows).
  - `w`: Kernel radius in texels (dynamically adjusted).
- `shadow_map`: A `sampler2DShadow` comparison sampler bound at `set = 0, binding = 2` [S1, unofficial].

**Outputs:**
- `occlusion`: A float in `[0.0, 1.0]` representing the final shadow intensity (0.0 = fully lit, 1.0 = fully shadowed).

**Data Structures:**
- `shadow_map`: A 2D depth texture. The comparison sampler returns `1.0` if the stored depth is greater than the provided depth, and `0.0` otherwise.

**Parameters:**
- `SHADOW_PCF_MAX_RADIUS`: Integer constant, typically `4` [S1, unofficial]. Limits the kernel half-width to a 9x9 grid.
- `SHADOW_SLOPE_OFFSET`: Float constant, typically `2.0` [S1, unofficial]. Multiplier for the slope-scaled bias component.
- `SHADOW_FADE_START`: Float constant, typically `0.85` [S1, unofficial]. Fraction of the map half-width where edge fading begins.

**Algorithm Steps:**

1.  **Light Direction Dot Product:**
    Calculate the dot product between the surface normal and the light direction. The light direction is assumed to be the direction *from* the surface *towards* the light (consistent with `DirectionalLight.direction` in the source context).
    $$ n\_dot\_l = \text{clamp}(\text{dot}(\text{normal}, \text{light\_dir}), 0.0, 1.0) $$

2.  **Slope-Scaled Normal Offset:**
    Compute a depth bias offset along the surface normal to prevent peter-panning. The offset magnitude increases as the surface becomes more perpendicular to the light (smaller $n\_dot\_l$).
    $$ \text{offset} = \text{scene.shadow\_params.y} \times (1.0 + \text{SHADOW\_SLOPE\_OFFSET} \times (1.0 - n\_dot\_l)) $$
    *Note: Offsetting along the normal rather than the light direction keeps contact points fixed and avoids self-shadowing artifacts on sloped surfaces [S1, unofficial].*

3.  **Transform to Light Clip Space:**
    Apply the offset to the world position and transform to light clip space.
    $$ \text{light\_clip} = \text{scene.light\_view\_proj} \times \text{vec4}(\text{world\_pos} + \text{normal} \times \text{offset}, 1.0) $$

4.  **Convert to NDC:**
    Perform perspective division to obtain Normalized Device Coordinates.
    $$ \text{ndc} = \text{light\_clip.xyz} / \text{light\_clip.w} $$
    **Edge Case:** If `ndc.z <= 0.0` or `ndc.z >= 1.0`, the point is outside the light's view frustum (behind the camera or beyond the far plane). Return `0.0` (fully lit) [S1, unofficial].

5.  **Compute UV Coordinates:**
    Map NDC to texture UV space.
    $$ \text{uv} = \text{ndc.xy} \times 0.5 + 0.5 $$
    **Edge Case:** If `uv.x < 0.0`, `uv.x > 1.0`, `uv.y < 0.0`, or `uv.y > 1.0`, the point is outside the shadow map bounds. Return `0.0` (fully lit) to avoid black walls at volume edges [S1, unofficial].

6.  **Initialize PCF Kernel:**
    Get the texel size and clamp the kernel radius.
    $$ \text{texel} = \text{scene.shadow\_params.x} $$
    $$ \text{radius} = \text{clamp}(\text{int}(\text{scene.shadow\_params.w} + 0.5), 0, \text{SHADOW\_PCF\_MAX\_RADIUS}) $$

7.  **Accumulate Occlusion:**
    Sample the shadow map using a square grid centered on `uv`. The comparison sampler returns `1.0` if the fragment is in shadow (depth > stored depth) and `0.0` if lit.
    $$ \text{occlusion} = 0.0 $$
    For `y` from `-radius` to `radius`:
        For `x` from `-radius` to `radius`:
            $$ \text{tap} = \text{uv} + \text{vec2}(x, y) \times \text{texel} $$
            $$ \text{sample\_val} = \text{texture}(\text{shadow\_map}, \text{vec3}(\text{tap}, \text{ndc.z})) $$
            $$ \text{occlusion} += 1.0 - \text{sample\_val} $$

8.  **Average Occlusion:**
    Normalize the accumulated value by the number of samples.
    $$ \text{occlusion} = \text{occlusion} / \text{float}((2 \times \text{radius} + 1) \times (2 \times \text{radius} + 1)) $$

9.  **Edge Fade:**
    Compute the distance of the UV coordinate from the center of the shadow map to apply a smooth fade at the edges.
    $$ \text{edge} = \max(|\text{uv.x} - 0.5|, |\text{uv.y} - 0.5|) \times 2.0 $$
    $$ \text{fade} = 1.0 - \text{smoothstep}(\text{SHADOW\_FADE\_START}, 1.0, \text{edge}) $$

10. **Final Result:**
    Combine occlusion, fade, and strength.
    $$ \text{result} = \text{occlusion} \times \text{fade} \times \text{scene.shadow\_params.z} $$
    Return `result`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pcf_shadow_lookup_with_slope_scaling_and_edge_fade(
    world_pos: vec3,
    normal: vec3,
    light_dir: vec3,
    light_view_proj: mat4,
    shadow_params: vec4,
    shadow_map: sampler2DShadow,
    SHADOW_PCF_MAX_RADIUS: int = 4,
    SHADOW_SLOPE_OFFSET: float = 2.0,
    SHADOW_FADE_START: float = 0.85
) -> float:
    # 1. Light Direction Dot Product
    n_dot_l = clamp(dot(normal, light_dir), 0.0, 1.0)

    # 2. Slope-Scaled Normal Offset
    offset = shadow_params.y * (1.0 + SHADOW_SLOPE_OFFSET * (1.0 - n_dot_l))

    # 3. Transform to Light Clip Space
    light_clip = light_view_proj * vec4(world_pos + normal * offset, 1.0)

    # 4. Convert to NDC
    ndc = light_clip.xyz / light_clip.w

    # Edge case: Outside light frustum
    if ndc.z <= 0.0 or ndc.z >= 1.0:
        return 0.0

    # 5. Compute UV Coordinates
    uv = ndc.xy * 0.5 + 0.5

    # Edge case: Outside shadow map bounds
    if uv.x < 0.0 or uv.x > 1.0 or uv.y < 0.0 or uv.y > 1.0:
        return 0.0

    # 6. Initialize PCF Kernel
    texel = shadow_params.x
    radius = clamp(int(shadow_params.w + 0.5), 0, SHADOW_PCF_MAX_RADIUS)

    # 7. Accumulate Occlusion
    occlusion = 0.0
    kernel_size = 2 * radius + 1
    for y in range(-radius, radius + 1):
        for x in range(-radius, radius + 1):
            tap = uv + vec2(x, y) * texel
            # Comparison sampler: 1.0 if stored_depth > provided_depth, else 0.0
            sample_val = texture(shadow_map, vec3(tap, ndc.z))
            occlusion += 1.0 - sample_val

    # 8. Average Occlusion
    occlusion = occlusion / float(kernel_size * kernel_size)

    # 9. Edge Fade
    edge = max(abs(uv.x - 0.5), abs(uv.y - 0.5)) * 2.0
    fade = 1.0 - smoothstep(SHADOW_FADE_START, 1.0, edge)

    # 10. Final Result
    return occlusion * fade * shadow_params.z
```

## Subtopics

### [PCF Shadow Mapping](11337-pcf-shadow-mapping.md)
Applies a fixed bias and a uniform 3x3 kernel without dynamic slope scaling or edge fading, trading geometric precision for simplicity.

## Operating characteristics
- **Time Complexity:** $O(R^2)$ per fragment, where $R$ is the PCF radius. With $R=4$, this is 81 texture samples per fragment.
- **Memory Complexity:** $O(1)$ additional memory per fragment, plus the shadow map texture size.
- **Strengths:**
  - Reduces peter-panning and shadow acne more effectively than fixed bias by scaling with surface slope [S1, unofficial].
  - Prevents harsh, aliased shadow edges at the boundaries of the shadow map, which is critical for volume rendering where objects may extend beyond the light's view [S1, unofficial].
  - Simple to implement and integrates well with standard shadow map pipelines.
- **Limitations:**
  - Performance cost increases quadratically with kernel radius.
  - Relies on a single shadow map, which may not handle complex occlusion from multiple light sources or highly dynamic scenes as well as more advanced techniques like Variance Shadow Mapping.
  - The edge fade is a heuristic and may not perfectly match the physical falloff of light, but it is effective for visual continuity.
- **When to Use:**
  - When rendering static or semi-static scenes with a dominant directional light (sun).
  - When visual quality at shadow map boundaries is important, especially for volumes or terrain that may extend beyond the shadow map frustum.
  - When hardware supports efficient comparison sampling.
- **When Not to Use:**
  - For scenes with many dynamic shadow-casting objects where performance is critical and a fixed, small kernel is insufficient.
  - When physically accurate soft shadows are required, as this method produces hard shadows with filtered edges.

## Background
This method is derived from practical implementations in voxel-based rendering engines, specifically the `VoxelPhase` project [S1, unofficial]. It addresses common artifacts in shadow mapping, such as peter-panning (caused by insufficient depth bias) and hard shadow edges (caused by insufficient filtering or map boundaries). The slope-scaled bias technique is a well-known optimization in shadow mapping to reduce the need for large fixed biases, while the edge fade is a specific adaptation for volume rendering to prevent black artifacts at the edges of the shadow map.

## Sources
[S1] migaelstrydom/VoxelPhase, https://github.com/migaelstrydom/VoxelPhase, fetched 2026-10-28 (unofficial)
