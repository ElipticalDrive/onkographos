# Shadow Mapping for Volumes

Also called: Image-based shadow algorithms

## Overview
Shadow Mapping for Volumes generates a depth texture (shadow map) from the light's perspective to determine if sample points within a participating medium are occluded by scene geometry [S1, S2, unofficial]. It solves the problem of volumetric light leaking through opaque objects by providing a binary shadow decision for each raymarching step. Unlike [Half-Angle Slicing for Incremental Lighting](3446-half-angle-slicing-for-incremental-lighting.md), which computes lighting incrementally per slice, this method decouples shadow calculation from the volume integration loop, allowing standard opaque shadow maps to be reused for volume shading [S1, S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** Scene geometry, light position/direction/cone angle, raymarching sample positions in world space, light view/projection matrices, shadow map texture.
*   **Outputs:** A binary shadow factor (0.0 for shadow, 1.0 for lit) per sample point.

**Data Structures**
*   **Shadow FBO (`shadowFBO`):** A render target configured with `depth: true` and a `DepthTexture` of type `THREE.FloatType` (or equivalent high-precision depth buffer) at resolution `shadowMapSize` x `shadowMapSize` [S1, unofficial].
*   **Shadow Map Texture:** Stores the depth values of the first hit points from the light's perspective [S2, unofficial].

**Parameters**
*   `shadowMapSize`: Resolution of the shadow map. Default is 256x256. Larger sizes increase computational cost in the raymarching loop [S1, unofficial].
*   `shadowBias`: A float uniform added to the depth comparison to prevent shadow acne or peter-panning [S1, unofficial].
*   `shadowThreshold`: Marks the intensity limit for shadowing [S2, unofficial].

**Steps**
1.  **Setup Light Camera:** Create a dedicated perspective camera (`lightCamera`) positioned at the light's location. Set its Field of View (FOV) to match the light's `coneAngle` or radius, depending on whether the light is conical or cylindrical [S1, unofficial].
2.  **Render Shadow Map:** Clear the `shadowFBO`. Render the scene geometry using `lightCamera`. This captures the depth of the first occluder hit for each pixel [S1, S2, unofficial].
3.  **Raymarching with Shadow Check:** During the volume raymarching loop, for each sample point `worldPosition`:
    a. Transform `worldPosition` to clip space:
       $$lightClipPos = lightProjectionMatrix \times lightViewMatrix \times vec4(worldPosition, 1.0)$$
    b. Convert to Normalized Device Coordinates (NDC):
       $$lightNDC = lightClipPos.xyz / lightClipPos.w$$
    c. Convert NDC to shadow map UV coordinates and depth:
       $$shadowCoord = lightNDC.xy \times 0.5 + 0.5$$
       $$lightDepth = lightNDC.z \times 0.5 + 0.5$$
    d. **Edge Case Handling:** If `shadowCoord.x` or `shadowCoord.y` is outside $[0, 1]$, or if `lightDepth > 1.0`, the point is outside the light's frustum. Return 1.0 (lit) to avoid sampling errors [S1, unofficial].
    e. Sample the shadow map:
       $$shadowMapDepth = texture2D(shadowMap, shadowCoord).x$$
    f. **Depth Comparison:** If `lightDepth > shadowMapDepth + shadowBias`, the point is occluded. Return 0.0. Otherwise, return 1.0 [S1, unofficial].
4.  **Accumulation Control:** If the shadow check returns 0.0, skip the lighting accumulation for that step (increment step size `t` and continue) but do not break the raymarching loop, as subsequent points may be lit [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Uniforms and Textures
# shadowMap: sampler2D
# lightViewMatrix: mat4
# lightProjectionMatrix: mat4
# shadowBias: float

def calculate_shadow(world_position: vec3) -> float:
    # Transform world position to light clip space
    light_clip_pos = light_projection_matrix * light_view_matrix * vec4(world_position, 1.0)
    
    # Convert to NDC
    light_ndc = light_clip_pos.xyz / light_clip_pos.w
    
    # Convert to shadow map coordinates
    shadow_coord = light_ndc.xy * 0.5 + 0.5
    light_depth = light_ndc.z * 0.5 + 0.5
    
    # Handle points outside the light's frustum
    if (shadow_coord.x < 0.0 or shadow_coord.x > 1.0 or
        shadow_coord.y < 0.0 or shadow_coord.y > 1.0 or
        light_depth > 1.0):
        return 1.0
        
    # Sample shadow map depth
    shadow_map_depth = texture2D(shadowMap, shadow_coord).x
    
    # Compare depths with bias
    if light_depth > shadow_map_depth + shadow_bias:
        return 0.0
        
    return 1.0

# Usage in Raymarching Loop
for i in range(num_steps):
    sample_pos = ray_origin + ray_dir * t
    
    # Break if ray exits scene bounds
    if t > scene_depth or t > camera_far:
        break
        
    shadow_factor = calculate_shadow(sample_pos)
    
    # Skip accumulation if in shadow, but continue marching
    if shadow_factor == 0.0:
        t += step_size
        continue
        
    # Accumulate lighting contribution here
    # ...
    
    t += step_size
```

## Subtopics

### Alpha Shadow Maps
A variant that supports semi-transparent occluders by incorporating multiple layers of depth and transparency, where each layer is a pair of depth and transparency values [S2, unofficial]. It saves control points of piecewise linear functions for each pixel. Construction involves saving the first key value at the first hit point and further key values based on an error function (e.g., 0.01 or 0.00005) [S2, unofficial]. This allows for smoother shadow transitions compared to the binary opaque-only standard shadow map [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ per raymarching step for the shadow lookup, where $N$ is the number of steps. The shadow map generation is $O(G)$, where $G$ is the number of geometry pixels.
*   **Memory Complexity:** $O(W \times H)$ for the shadow map texture, where $W, H$ are `shadowMapSize` dimensions [S1, unofficial].
*   **Performance:** Default 256x256 resolution is computationally cheap but may show aliasing. Higher resolutions increase raymarching loop intensity [S1, unofficial]. Tested on Intel Core2 6600 and nVidia GeForce 8800GTX [S2, unofficial].
*   **Strengths:** Simple to implement; reuses standard shadow map infrastructure; effective for sharp shadows from opaque objects [S1, S2, unofficial].
*   **Limitations:** Standard maps support opaque occluders only [S2, unofficial]. Artifacts like shadow acne or peter-panning require bias tuning [S1, unofficial]. Filtering is needed to reduce artifacts [S2, unofficial]. Does not handle self-shadowing of the volume itself unless the volume geometry is also rendered to the shadow map [G].

## Background
Originated in the context of GPU-based volume raycasting with advanced illumination techniques [S2, unofficial]. Timo Ropinski (University of Münster) and colleagues (Hadwiger, Ljung, Rezk Salama, etc.) discussed these methods in 2008 [S2, unofficial]. The specific implementation details for real-time volumetric lighting with post-processing and raymarching were popularized in unofficial guides such as Maxime Heckel's blog [S1, unofficial].

## Sources
[S1] On Shaping Light: Real-Time Volumetric Lighting with Post-Processing ..., https://blog.maximeheckel.com/posts/shaping-light-volumetric-lighting-with-post-processing-and-raymarching/, fetched 2026-10-28 (unofficial)
[S2] GPU-Based Volume Ray-Casting with Advanced Illumination, https://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/viscom/2008/advanced_illumination.pdf, fetched 2026-10-29 (unofficial)
