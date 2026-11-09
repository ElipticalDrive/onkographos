# PCF Shadow Lookup with Slope Scaling and Edge Fade

Also called: PCF with Slope Scaling and Edge Fade

## Overview
Computes sun shadow occlusion at a world position using a comparison sampler and a dynamic PCF kernel, with slope-scaled depth bias and distance-based edge fading to prevent peter-panning and hard shadow edges [S1, unofficial]. It differs from standard shadow mapping by offsetting the sample position along the surface normal rather than the light direction to maintain contact stability, and by fading occlusion near the shadow map boundaries to avoid black artifacts at volume edges [S1, unofficial].

## How it works
**Inputs:**
- `world_pos`: World-space position of the point to shade.
- `normal`: Surface normal at `world_pos`, normalized.
- `light_dir`: Direction from the surface towards the light (sun), normalized.
- `scene.shadow_params`: A uniform vector containing:
  - `x`: Texel size in UV space.
  - `y`: Base normal offset magnitude.
  - `z`: Shadow strength multiplier (0 disables shadows).
  - `w`: Kernel radius in texels, dynamically adjusted.
- `shadow_map`: A `sampler2DShadow` comparison sampler [S1, unofficial].

**Outputs:**
- `occlusion`: A float in $[0, 1]$ representing the shadow factor. 0.0 is fully lit, 1.0 is fully shadowed.

**Data Structures:**
- `shadow_map`: A 2D shadow map texture. The comparison sampler returns 1.0 if the stored depth is less than or equal to the provided depth, 0.0 otherwise.

**Constants:**
- `SHADOW_PCF_MAX_RADIUS = 4`: Limits the kernel half-width to a maximum of 9x9 samples.
- `SHADOW_SLOPE_OFFSET = 2.0`: Constant float for slope scaling bias.
- `SHADOW_FADE_START = 0.85`: Fraction of map half-width where fade begins.

**Steps:**
1.  **Normal-Light Dot Product:** Calculate the alignment between the surface normal and the light direction.
    $$ n\_dot\_l = \text{clamp}(\text{dot}(\text{normal}, \text{light\_dir}), 0.0, 1.0) $$
2.  **Slope-Scaled Offset:** Compute the normal offset magnitude. The offset increases as the surface becomes more perpendicular to the light (smaller $n\_dot\_l$), providing greater bias against self-shadowing artifacts on grazing angles.
    $$ \text{offset} = \text{scene.shadow\_params.y} \times (1.0 + \text{SHADOW\_SLOPE\_OFFSET} \times (1.0 - n\_dot\_l)) $$
3.  **Transform to Light Clip Space:** Apply the offset along the normal to the world position, then transform to the light's clip space.
    $$ \text{light\_clip} = \text{scene.light\_view\_proj} \times \text{vec4}(\text{world\_pos} + \text{normal} \times \text{offset}, 1.0) $$
4.  **NDC Conversion:** Convert to Normalized Device Coordinates.
    $$ \text{ndc} = \text{light\_clip.xyz} / \text{light\_clip.w} $$
    If $\text{ndc.z} \le 0.0$ or $\text{ndc.z} \ge 1.0$, the point is outside the light's view frustum (behind the camera or beyond the far plane). Return 0.0 (fully lit) [S1, unofficial].
5.  **UV Calculation:** Convert NDC to texture coordinates.
    $$ \text{uv} = \text{ndc.xy} \times 0.5 + 0.5 $$
    If $\text{uv}$ is outside $[0, 1]$ in either dimension, the point is outside the shadow map bounds. Return 0.0 (fully lit) to avoid black walls at volume edges [S1, unofficial].
6.  **Kernel Setup:** Get texel size and clamp the kernel radius.
    $$ \text{texel} = \text{scene.shadow\_params.x} $$
    $$ \text{radius} = \text{clamp}(\text{int}(\text{scene.shadow\_params.w} + 0.5), 0, \text{SHADOW\_PCF\_MAX\_RADIUS}) $$
7.  **PCF Sampling:** Perform a square kernel PCF lookup. Accumulate occlusion by sampling the shadow map with the current depth.
    $$ \text{occlusion} = 0.0 $$
    For $y$ from $-\text{radius}$ to $\text{radius}$:
        For $x$ from $-\text{radius}$ to $\text{radius}$:
            $$ \text{tap} = \text{uv} + \text{vec2}(x, y) \times \text{texel} $$
            $$ \text{occlusion} += 1.0 - \text{texture}(\text{shadow\_map}, \text{vec3}(\text{tap}, \text{ndc.z})) $$
8.  **Averaging:** Average the accumulated occlusion over the kernel size.
    $$ \text{occlusion} /= \text{float}((2 \times \text{radius} + 1) \times (2 \times \text{radius} + 1)) $$
9.  **Edge Fade:** Compute distance from the center of the shadow map and apply a smooth fade to soften shadow edges near the map boundaries.
    $$ \text{edge} = \max(|\text{uv.x} - 0.5|, |\text{uv.y} - 0.5|) \times 2.0 $$
    $$ \text{fade} = 1.0 - \text{smoothstep}(\text{SHADOW\_FADE\_START}, 1.0, \text{edge}) $$
10. **Final Result:** Apply the fade and strength multiplier.
    $$ \text{return} \ \text{occlusion} \times \text{fade} \times \text{scene.shadow\_params.z} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pcf_shadow_lookup(world_pos, normal, light_dir, scene, shadow_map):
    # 1. Normal-Light Dot Product
    n_dot_l = clamp(dot(normal, light_dir), 0.0, 1.0)
    
    # 2. Slope-Scaled Offset
    offset = scene.shadow_params.y * (1.0 + SHADOW_SLOPE_OFFSET * (1.0 - n_dot_l))
    
    # 3. Transform to Light Clip Space
    light_clip = scene.light_view_proj * vec4(world_pos + normal * offset, 1.0)
    
    # 4. NDC Conversion
    ndc = light_clip.xyz / light_clip.w
    if ndc.z <= 0.0 or ndc.z >= 1.0:
        return 0.0
    
    # 5. UV Calculation
    uv = ndc.xy * 0.5 + 0.5
    if uv.x < 0.0 or uv.x > 1.0 or uv.y < 0.0 or uv.y > 1.0:
        return 0.0
    
    # 6. Kernel Setup
    texel = scene.shadow_params.x
    radius = clamp(int(scene.shadow_params.w + 0.5), 0, SHADOW_PCF_MAX_RADIUS)
    
    # 7. PCF Sampling
    occlusion = 0.0
    kernel_size = 2 * radius + 1
    for y in range(-radius, radius + 1):
        for x in range(-radius, radius + 1):
            tap = uv + vec2(x, y) * texel
            # Comparison sampler returns 1.0 if stored_depth <= provided_depth
            # We want occlusion, so we invert: 1.0 - shadow_sample
            shadow_sample = texture(shadow_map, vec3(tap, ndc.z))
            occlusion += 1.0 - shadow_sample
            
    # 8. Averaging
    occlusion /= float(kernel_size * kernel_size)
    
    # 9. Edge Fade
    edge = max(abs(uv.x - 0.5), abs(uv.y - 0.5)) * 2.0
    fade = 1.0 - smoothstep(SHADOW_FADE_START, 1.0, edge)
    
    # 10. Final Result
    return occlusion * fade * scene.shadow_params.z
```

**Edge Cases and Pitfalls:**
- **Peter-Panning:** Pure depth bias along the light direction can cause shadows to detach from surfaces (peter-panning). Offsetting along the surface normal keeps contact points fixed relative to the geometry [S1, unofficial].
- **Volume Edges:** If a point is outside the shadow map slab (UV out of bounds), reading as fully lit (0.0) prevents black walls at volume edges, which would occur if it returned fully shadowed [S1, unofficial].
- **Degenerate Normals:** If the normal is zero or near-zero, the offset calculation may become unstable. The implementation assumes a valid normalized normal is provided.

## Operating characteristics
- **Time Complexity:** $O(R^2)$ where $R$ is the kernel radius. With `SHADOW_PCF_MAX_RADIUS = 4`, the maximum number of texture samples is $9 \times 9 = 81$.
- **Memory Complexity:** $O(1)$ additional memory beyond the shadow map texture.
- **Strengths:**
  - Reduces aliasing and peter-panning through slope-scaled bias and PCF.
  - Softens shadow edges near the light's view frustum boundaries, preventing harsh cutoffs in volumetric scenes.
  - Simple to implement using standard hardware comparison samplers.
- **Limitations:**
  - Performance scales with kernel size; larger radii increase fill rate usage.
  - Does not handle soft shadows from area lights natively; it approximates softness via PCF and edge fade.
  - Bias parameters may need tuning for different scene scales.
- **When to Use:** Suitable for real-time sun shadows in volumetric rendering where performance is critical and hard shadow edges or peter-panning are undesirable.
- **When Not to Use:** Not suitable for accurate soft shadows from area lights or when precise contact shadows are required without bias artifacts.

## Background
This method is derived from the lighting implementation in VoxelPhase, a voxel-based rendering engine. It combines standard PCF shadow mapping techniques with specific adjustments for volumetric rendering, such as normal-based offsetting and edge fading to handle the unique artifacts of volume shading [S1, unofficial].

## Sources
[S1] migaelstrydom/VoxelPhase, https://github.com/migaelstrydom/VoxelPhase, fetched 2026-10-28 (unofficial)
