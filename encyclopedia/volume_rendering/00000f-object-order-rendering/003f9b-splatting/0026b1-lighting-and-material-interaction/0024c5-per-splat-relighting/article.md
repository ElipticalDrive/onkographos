# Per-Splat Relighting


## Overview
Per-Splat Relighting computes a dynamic lighting multiplier for individual Gaussian splats by evaluating direct illumination, ambient occlusion, and confidence weights against a light rig. It differs from [Photon Splatting](2832-photon-splatting.md) by calculating lighting analytically per-splat in the fragment shader rather than accumulating photon densities, allowing for real-time updates when light positions or intensities change without re-radiating photons.

## How it works
**Inputs and Outputs:**
*   **Inputs:** `splat_lighting` buffer (packed lighting records), `light_rig` buffer (light definitions), `world_position` of the splat center, `unlit_level` (base color), `ambient` color, `gain` scalar.
*   **Output:** Relit color value.

**Data Structures:**
The `splat_lighting` buffer stores per-splat metadata packed into a `vec4` (or similar 4-component vector). The layout is defined as:
*   `rg`: Octahedral encoded normal.
*   `b`: Ambient Occlusion (AO) value.
*   `a`: Confidence value (weighting factor for direct light contribution).

The `light_rig` buffer is an array of floats, where each light occupies 4 consecutive elements (indices `i*4` to `i*4+3`):
*   `i*4+0`: `vec3` representing light position (for point/spot) or direction (for directional).
*   `i*4+1`: `vec3` tint (RGB) and `float` alpha (type indicator).
*   `i*4+2`: `vec3` setup parameters (`r`: inverse range, `g`: decay, `b`: spot cone cosine threshold, `a`: spot exponent).
*   `i*4+3`: `vec3` spot direction vector (used if type is spot).

**Numbered Steps:**
1.  **Unpack Lighting Record:** Retrieve the lighting record for the current splat index from `splat_lighting`. Extract the octahedral normal (`rg`), AO (`b`), and confidence (`a`).
2.  **Normalize Normal:** Convert the octahedral normal to a 3D vector. Calculate its length. If the length is greater than zero, normalize the vector. If the length is zero or near-zero, default the normal to `(0, 0, 1)` [G].
3.  **Initialize Direct Light:** Set `direct` illumination to zero.
4.  **Loop Over Lights:** For each light `i` in the `light_rig`:
    *   Extract `tint` from `light_rig[i*4+1]`.
    *   Extract `setup` parameters from `light_rig[i*4+2]`.
    *   Determine light `type` by rounding `tint.a` to the nearest integer.
    *   Extract initial `to_light` vector from `light_rig[i*4+0].xyz`.
    *   Set `attenuation` to `1.0`.
    *   **Distance Calculation:** If `type` is not 0 (not directional):
        *   Calculate `relative` vector: `light_rig[i*4+0].xyz - world_position`.
        *   Calculate `dist` as the length of `relative`.
        *   If `dist` is less than `1e-6`, skip this light (continue loop).
        *   Normalize `to_light` to `relative / dist`.
        *   Calculate `attenuation` using the `omni_attenuation` formula with `dist`, `setup.r` (inverse range), and `setup.g` (decay).
    *   **Spot Light Handling:** If `type` equals 2 (spot light):
        *   Calculate `cone_cosine` as `max(dot(-to_light, light_rig[i*4+3].xyz), setup.b)`.
        *   Calculate `rim` as `(1.0 - cone_cosine) / max(1e-4, 1.0 - setup.b)`.
        *   Update `attenuation` by multiplying it by `1.0 - pow(clamp(rim, 0, 1), setup.a)`.
    *   **Accumulate Direct Light:** Add to `direct`: `tint.rgb * attenuation * max(dot(world_normal, to_light), 0)`.
5.  **Compute Irradiance:** Calculate `irradiance` as `ambient * ao + direct * confidence`.
6.  **Return Result:** Return `unlit_level + gain * irradiance`.

**Formulas:**
*   **Omni Attenuation:**
    $$ \text{omni\_attenuation}(dist, inv\_range, decay) = \max(1.0 - (dist \cdot inv\_range)^4, 0)^2 \cdot dist^{-decay} $$
    Where:
    *   $dist$: Distance from splat to light.
    *   $inv\_range$: Inverse range of the light (setup.r).
    *   $decay$: Decay exponent (setup.g).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def per_splat_relight(splat_index, world_position, unlit_level, ambient, gain, splat_lighting, light_rig):
    # 1. Unpack lighting record
    record = splat_lighting[splat_index]
    oct_normal = record.rg
    ao = record.b
    confidence = record.a

    # 2. Normalize normal
    # Assuming oct_normal is decoded to a 3D vector elsewhere or here.
    # For implementation completeness, we assume a function decode_oct_normal exists.
    normal = decode_oct_normal(oct_normal)
    normal_length = length(normal)
    if normal_length > 0:
        world_normal = normal / normal_length
    else:
        world_normal = vec3(0, 0, 1)

    # 3. Initialize direct light
    direct = vec3(0, 0, 0)

    # 4. Loop over lights
    num_lights = len(light_rig) // 4
    for i in range(num_lights):
        base_idx = i * 4
        tint = light_rig[base_idx + 1]
        setup = light_rig[base_idx + 2]
        light_data = light_rig[base_idx + 0]
        
        # Type is in tint.a
        light_type = int(round(tint.a))
        
        to_light = light_data.xyz
        attenuation = 1.0
        
        if light_type != 0:
            relative = light_data.xyz - world_position
            dist = length(relative)
            
            if dist < 1e-6:
                continue
                
            to_light = relative / dist
            inv_range = setup.r
            decay = setup.g
            
            # omni_attenuation formula
            term = max(1.0 - (dist * inv_range)**4, 0.0)
            attenuation = (term ** 2) * (dist ** -decay)
            
        if light_type == 2:
            spot_dir = light_rig[base_idx + 3].xyz
            cone_cosine = max(dot(-to_light, spot_dir), setup.b)
            rim = (1.0 - cone_cosine) / max(1e-4, 1.0 - setup.b)
            attenuation *= 1.0 - pow(clamp(rim, 0.0, 1.0), setup.a)
            
        # Accumulate direct
        diff = max(dot(world_normal, to_light), 0.0)
        direct += tint.rgb * attenuation * diff

    # 5. Compute irradiance
    irradiance = ambient * ao + direct * confidence

    # 6. Return result
    return unlit_level + gain * irradiance
```

**Parameters:**
*   `depth_bias`: Used in the parent compositing step to reject splats behind the scene depth. Not part of the relighting formula itself but affects visibility.
*   `alpha_cutoff`: Minimum alpha for splat processing.

**Edge Cases and Pitfalls:**
*   **Zero Normal:** If the decoded normal has zero length, it defaults to `(0,0,1)`. This can cause incorrect shading for flat surfaces or degenerate splats.
*   **Division by Zero:** The spot light rim calculation uses `max(1e-4, 1.0 - setup.b)` to prevent division by zero when the cone angle is very narrow (setup.b close to 1.0).
*   **Near-Zero Distance:** Lights at the exact splat position are skipped to avoid NaNs or infinite attenuation.

## Subtopics

### Photon Splatting
Accumulates photon densities for indirect lighting. Choose when global illumination is required and lights are static. [Photon Splatting](2832-photon-splatting.md)

### Per-Splat Relighting
Calculates direct lighting analytically. Choose for dynamic lights and real-time performance.

## Operating characteristics
*   **Time Complexity:** $O(L)$ per splat, where $L$ is the number of lights. This is executed per fragment during splat rendering.
*   **Memory Complexity:** $O(1)$ additional memory per splat for the lighting record (packed in `splat_lighting`).
*   **Strengths:** Real-time capable; supports dynamic light movement and intensity changes without precomputation.
*   **Limitations:** Only handles direct lighting (and simple AO); no indirect bounces; performance scales linearly with light count.
*   **When to use:** Real-time applications with dynamic lighting.
*   **When not to use:** Offline rendering requiring global illumination or very high light counts where performance is critical.

## Background
Per-Splat Relighting is a technique used in real-time Gaussian Splatting renderers, such as the one implemented in the `godot-gaussian-splatting` repository. It adapts traditional deferred or forward lighting concepts to the splat-based rendering pipeline by storing lighting metadata (normal, AO, confidence) within the splat data structure.

## Sources
[S1] ReconWorldLab/godot-gaussian-splatting, https://github.com/ReconWorldLab/godot-gaussian-splatting, fetched 2026-10-27 (unofficial)
