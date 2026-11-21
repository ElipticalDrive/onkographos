# Volumetric Sun Shafts (God Rays)

Also called: Haze volumetrics, In-scattered light shafts

## Overview
A post-processing technique that renders volumetric light shafts (sun shafts) by marching a view ray through a sun shadow map, accumulating in-scattered light where the ray is lit and no light where it is shadowed, modulated by ground-hugging haze density [S1, unofficial]. It differs from screen-space alternatives by relying on a dedicated shadow map for occlusion rather than screen-space depth, allowing for more accurate volumetric occlusion from geometry outside the current view frustum, while trading off higher memory bandwidth for the shadow map [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Depth buffer, inverse projection matrix, inverse view matrix, sun shadow map (containing lit/shadowed status), sun color, sun direction, ground plane height, and a global intensity multiplier.
*   **Outputs:** A post-processing texture containing the accumulated sun shafts, composited over the scene.

**Data Structures:**
*   **VolumetricPass Target:** A render target (`WebGLRenderTarget`) with `HalfFloatType` precision, no depth buffer, sized at half the screen resolution (e.g., if screen is $W \times H$, target is $\frac{W}{2} \times \frac{H}{2}$) [S1, unofficial].
*   **Shadow Map:** A texture storing depth or binary lit/shadowed status for the sun light source.

**Parameters:**
*   `STEPS`: Number of march steps, set to 16 [S1, unofficial].
*   `uIntensity`: Global intensity multiplier, calculated as $0.07 \times \text{strength}$ [S1, unofficial].
*   `uGround`: Height of the ground plane for density calculation, passed as a uniform [S1, unofficial].
*   `MARCH_LENGTH`: The last 32 units of the ray are used for marching, or 0 if the ray is shorter than 32 units [S1, unofficial].

**Algorithm Steps:**
1.  **Resolution Reduction:** Render the effect at half resolution to quarter the per-pixel shadow-map sampling cost, relying on the soft nature of the haze effect to hide aliasing [S1, unofficial].
2.  **Jittering:** To hide banding artifacts in the volumetric march, jitter the start position of the ray march using interleaved gradient noise [S1, unofficial].
3.  **Ray Reconstruction:** For each pixel, reconstruct the world position from the depth buffer using the inverse projection and view matrices.
4.  **Ray Definition:** Compute the view ray direction $\mathbf{rd}$ from the camera position to the reconstructed world position and its length $L$.
5.  **March Setup:** Determine the march start distance as the last 32 units of the ray (or 0 if $L < 32$). Calculate the step length $\Delta s$ based on 16 steps over the march length:
    $$ \Delta s = \frac{\text{MARCH\_LENGTH}}{\text{STEPS}} $$
6.  **Integration Loop:** Loop `STEPS` (16) times:
    a.  Compute sample position $\mathbf{p}$ along the ray.
    b.  Transform $\mathbf{p}$ to shadow map space.
    c.  Determine if the sample is lit by checking shadow map bounds and depth. Let $S \in \{0, 1\}$ be the shadow factor (1 if lit, 0 if shadowed).
    d.  Calculate ground-hugging density $\rho$ using exponential falloff based on sample height $y$ relative to ground level $uGround$:
        $$ \rho = e^{-k(y - uGround)} $$
        (Note: The exact constant $k$ is not specified in the sources [S1, unofficial].)
    e.  Apply forward scattering phase function $P(\theta)$, which is brighter when looking toward the sun.
    f.  Scale the contribution by distance attenuation and density.
    g.  Accumulate the result: $Accum += S \cdot \rho \cdot P(\theta) \cdot \text{attenuation}$.
7.  **Output:** Output final color as sun color multiplied by the accumulated amount, phase, and intensity `uIntensity` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_volumetric_sun_shafts(depth_buffer, inv_proj, inv_view, shadow_map, sun_color, sun_dir, u_ground, strength):
    # Parameters
    STEPS = 16
    MARCH_LENGTH = 32.0
    u_intensity = 0.07 * strength
    
    # Output target is half resolution
    out_width = depth_buffer.width // 2
    out_height = depth_buffer.height // 2
    output = create_texture(out_width, out_height, type=HalfFloatType, depth=False)
    
    for y in range(out_height):
        for x in range(out_width):
            # 1. Reconstruct world position from depth
            # Note: Standard reconstruction from depth buffer using inv_proj and inv_view
            # This step assumes a helper function to unpack depth and apply matrices
            world_pos = reconstruct_world_pos(depth_buffer, x, y, inv_proj, inv_view)
            
            # 2. Compute ray
            camera_pos = get_camera_position()
            ray_dir = normalize(world_pos - camera_pos)
            ray_len = length(world_pos - camera_pos)
            
            # 3. Determine march length
            march_len = min(ray_len, MARCH_LENGTH)
            if march_len <= 0:
                output[x, y] = vec3(0.0)
                continue
                
            step_len = march_len / STEPS
            
            # 4. Jitter start position to hide banding
            # Interleaved gradient noise function
            jitter = interleaved_gradient_noise(x, y)
            start_offset = jitter * step_len
            
            accumulated_light = vec3(0.0)
            
            # 5. March loop
            for i in range(STEPS):
                # Sample position
                t = start_offset + i * step_len
                sample_pos = camera_pos + ray_dir * t
                
                # Transform to shadow map space
                shadow_uv = transform_to_shadow_space(sample_pos, shadow_map_matrix)
                
                # Check shadow
                # S is 1 if lit, 0 if shadowed
                S = sample_shadow(shadow_map, shadow_uv)
                
                # Ground-hugging density
                # Exponential falloff based on height relative to ground
                height_diff = sample_pos.y - u_ground
                # Assuming a decay constant k, not specified in sources [G]
                # Using a typical value for haze if k is missing, but strictly:
                # density = exp(-k * height_diff)
                # Since k is not given, we note it as a parameter [G]
                density = exp(-0.5 * max(0.0, height_diff)) 
                
                # Phase function (forward scattering)
                # Brighter when looking toward sun
                cos_theta = dot(ray_dir, sun_dir)
                phase = forward_scattering_phase(cos_theta)
                
                # Distance attenuation (simplified, exact formula not specified [G])
                attenuation = 1.0 / (t * t + 1.0) 
                
                accumulated_light += S * density * phase * attenuation * step_len
            
            # 6. Output
            output[x, y] = sun_color * accumulated_light * u_intensity
            
    return output
```

**Edge Cases and Pitfalls:**
*   **Banding:** Banding artifacts in the volumetric march are hidden by jittering the start position using interleaved gradient noise [S1, unofficial].
*   **Short Rays:** If the ray length is less than 32 units, the march length is 0, and no light is accumulated [S1, unofficial].
*   **Resolution:** Running at half resolution is critical for performance; running at full resolution would significantly increase cost without proportional visual benefit due to the soft nature of the effect [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(W \cdot H \cdot \text{STEPS})$, where $W$ and $H$ are the output resolution (half screen) and STEPS is 16. The cost is dominated by shadow map lookups.
*   **Memory Complexity:** $O(W \cdot H)$ for the output texture (half float, 3 channels) plus the shadow map size.
*   **Performance:** Runs at half resolution to reduce cost, relying on the soft nature of the haze effect [S1, unofficial].
*   **Strengths:** Accurate volumetric occlusion from shadow maps, allowing shafts to be cast by geometry outside the view frustum.
*   **Limitations:** Requires a dedicated shadow map and post-processing pass. Performance scales with resolution and step count.

## Background
This technique is part of the volumetric rendering pipeline in a custom WebAssembly-based voxel engine, specifically implemented in `src/volumetric.ts` [S1, unofficial]. It is designed to render atmospheric lighting effects in a browser environment.

## Sources
[S1] 12.8B destructible voxels rendered in the browser with Rust WASM custom engine, https://github.com/cktang88/2.5d-experiment, fetched 2026-10-26 (unofficial)
