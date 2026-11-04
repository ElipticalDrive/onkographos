# Fog Blending with Clouds

Also called: Cloud Rendering with Fog, Exponential Fog Cloud Blending, Weighted Depth Fog Blending

## Overview
This method renders a cloud layer by mixing a fixed cloud color with the background (sky) color using distance-based exponential fog, outputting a semi-transparent result [S1, unofficial]. It sets itself apart from siblings like [Height-based Fog with Directional In-scatter](4400-height-based-fog-with-directional-in-scatter.md) by avoiding per-sample fog calculations; instead, it computes a single weighted average depth for the ray passing through the cloud and applies the fog blend only at that specific depth [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `cloud_color`: Fixed `vec3` color for the cloud layer (typically white, e.g., `vec3(1.0)`) [S1, unofficial].
    *   `bg_color`: Uniform `vec3` background (sky) color [S1, unofficial].
    *   `gl_FragCoord`: Built-in fragment shader variable containing window-relative coordinates, where `z` is the clip-space depth and `w` is the clip-space w-component [S1, unofficial].
*   **Outputs:**
    *   `fragColor`: Output `vec4` containing the blended RGB color and a fixed alpha of 0.8 [S1, unofficial].

**Data Structures:**
No complex data structures are required. The method relies on standard fragment shader built-ins and uniform variables.

**Steps:**
1.  **Compute Fog Distance:** Calculate the linear fog distance from the clip-space depth.
    $$ d = \frac{z_{clip}}{w_{clip}} $$
    Where $z_{clip}$ is `gl_FragCoord.z` and $w_{clip}$ is `gl_FragCoord.w` [S1, unofficial].
2.  **Calculate Weighted Average Depth (Optional/Optimization):** To optimize, calculate the weighted average depth of the ray passing through the cloud, using Transmittance as the weight. This allows sampling and blending fog only at this specific depth, avoiding per-sample fog calculations [S2, unofficial]. *Note: The basic implementation below uses the direct fragment depth.*
3.  **Compute Fog Factor:** Calculate the exponential fog factor using the squared distance.
    $$ f = 1.0 - e^{-k \cdot d^2} $$
    Where $k = 0.000001$ [S1, unofficial].
4.  **Blend Colors:** Mix the cloud color and background color using the fog factor.
    $$ C_{out} = \text{mix}(C_{cloud}, C_{bg}, f) $$
5.  **Output:** Set the fragment color to the blended RGB with a fixed alpha of 0.8 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs: cloud_color (vec3), bg_color (vec3)
# Built-in: gl_FragCoord (vec4)

def fog_blend_clouds(cloud_color, bg_color):
    # Step 1: Compute fog distance from clip-space depth
    # gl_FragCoord.z is clip-space Z, gl_FragCoord.w is clip-space W
    fog_dist = gl_FragCoord.z / gl_FragCoord.w
    
    # Step 2: Compute fog factor using exponential decay
    # k = 0.000001
    fog_factor = 1.0 - exp(-0.000001 * fog_dist * fog_dist)
    
    # Step 3: Mix cloud color with background color
    blended_rgb = mix(cloud_color, bg_color, fog_factor)
    
    # Step 4: Output with fixed alpha
    return vec4(blended_rgb, 0.8)
```

**Parameters:**
*   `k` (density constant): $0.000001$ [S1, unofficial].
*   `alpha`: $0.8$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Using `gl_FragCoord.z` directly assumes a standard projection matrix. If non-standard projections are used, the depth calculation may be inaccurate [G].
*   **Performance:** The basic implementation calculates fog per fragment. For large clouds, the weighted average depth optimization [S2, unofficial] is recommended to reduce overdraw and calculation cost.
*   **Artifacts:** The weighted average depth method may introduce artifacts in some cases if the transmittance distribution is highly non-uniform [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per fragment for the basic implementation. The weighted average depth approach requires additional passes or accumulation buffers but reduces the number of fog calculations.
*   **Memory Complexity:** $O(1)$ additional memory for uniforms.
*   **Strengths:** Simple to implement; computationally inexpensive for basic cloud layers; provides a smooth exponential fade to the sky [S1, unofficial].
*   **Limitations:** Lacks volumetric detail (no in-scatter, no height variation); fixed alpha may not blend correctly with all background scenes; weighted average method can cause artifacts [S2, unofficial].
*   **When to use:** Ideal for simple, distant cloud layers or skyboxes where volumetric accuracy is not required.
*   **When not to use:** For detailed, close-up volumetric clouds requiring realistic lighting and scattering [S2, unofficial].

## Background
This method is derived from the exponential fog technique based on the Beer-Lambert law, as implemented in various real-time rendering contexts. The specific optimization of using a weighted average depth for fog blending in cloud rendering is attributed to Unreal Engine 4's approach, as described in industry articles [S2, unofficial]. The basic implementation is found in unofficial Minecraft shader repositories [S1, unofficial].

## Sources
[S1] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
[S2] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
