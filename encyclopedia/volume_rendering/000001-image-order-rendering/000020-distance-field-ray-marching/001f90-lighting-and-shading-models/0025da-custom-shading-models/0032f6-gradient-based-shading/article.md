# Gradient-Based Shading

Also called: DVR.shader, Front-to-Back DVR with Gradient-Based Shading, VolumeDVR

## Overview

Gradient-Based Shading is a discrete volume rendering (DVR) technique that renders 3D textures using front-to-back accumulation, shading voxels via a pseudo-normal estimated from the volume gradient and a directional light model [S1, unofficial]. It differs from sibling methods like [GGX Microfacet BRDF](3715-ggx-microfacet-brdf.md) or [Blinn-Phong with Metallic Reflections](4252-blinn-phong-with-metallic-reflections.md) by relying on a simple Lambertian diffuse model driven by finite-difference normals rather than complex microfacet distributions or metallic reflections. It is distinct from [Fast Shading for Voxel Engines](5606-fast-shading-for-voxel-engines.md) by its explicit use of a transfer function LUT and continuous/discrete normalization modes within a standard raymarching loop.

## How it works

### Inputs and Outputs
*   **Input:** A 3D volume texture (`_VolumeTex`) containing single-channel `float32` data, and a 2D transfer function texture (`_TFTex`) mapping intensity/label to RGBA.
*   **Output:** A `float4` color value (RGB + accumulated alpha) for each fragment.

### Data Structures
*   **Volume Texture (`_VolumeTex`):** 3D texture, format `RFloat` (single channel `float32`). Dimensions correspond to the volume data.
*   **Transfer Function Texture (`_TFTex`):** 2D texture, format `RGBAFloat`. Dimensions $256 \times 1$. The U coordinate maps to normalized intensity/label, and the V coordinate is fixed at $0.5$.

### Parameters
*   `_StepSize`: Distance advanced along the ray per sample. Range: $[0.0005, 0.02]$. Default: $0.003$.
*   `_Brightness`: Global multiplier for accumulated RGB. Range: $[0, 5]$. Default: $1.5$.
*   `_AlphaScale`: Multiplier for sampled alpha. Range: $[0, 2]$. Default: $0.5$.
*   `_LightDir`: World-space light direction. Default: $(0, 0, 1, 0)$.
*   `_LightIntensity`: Diffuse light intensity. Range: $[0, 5]$. Default: $1.0$.
*   `_Ambient`: Ambient lighting constant. Range: $[0, 1]$. Default: $0.2$.
*   `_IsLabelMap`: Integer flag ($1$ or $0$). $1$ for discrete labels, $0$ for continuous intensities.
*   `_P1`: Lower percentile for continuous normalization.
*   `_P99`: Upper percentile for continuous normalization.
*   `MAX_STEPS`: Maximum samples per ray. Default: $2048$.

### Algorithm Steps

1.  **Ray Setup:**
    *   Reconstruct camera position (`camPosObj`) and fragment position in object space (cube $[-0.5, +0.5]^3$).
    *   Compute ray direction: $\mathbf{d}_{obj} = \text{normalize}(\text{fragPosObj} - \text{camPosObj})$.
    *   Transform world-space light direction to object space: $\mathbf{L}_{obj}$.
    *   Intersect ray with object-space bounding box $[-0.5, 0.5]^3$. If no intersection, return transparent black. Let $t_{enter}$ and $t_{exit}$ be the intersection distances.

2.  **Initialization:**
    *   $t = \max(t_{enter}, 0.0)$.
    *   $\text{accRGB} = (0, 0, 0)$.
    *   $\text{accA} = 0.0$.

3.  **Raymarching Loop:**
    *   Iterate $i$ from $0$ to `MAX_STEPS`.
    *   **Termination Check:** If $t > t_{exit}$ or $\text{accA} \ge 0.9999$, break loop.
    *   **Position:** $\mathbf{p}_{obj} = \text{camPosObj} + \mathbf{d}_{obj} \times t$.
    *   **Coordinates:** $\text{uvw} = \mathbf{p}_{obj} + 0.5$. If any component of $\text{uvw}$ is outside $[0, 1]$, break loop.
    *   **Sampling:** Sample volume texture at $\text{uvw}$ to get `voxelVal` (red channel).
    *   **Transfer Function:**
        *   If `_IsLabelMap == 1`:
            *   $u = \frac{\text{clamp}(\text{round}(\text{voxelVal}), 0, 255)}{255}$.
        *   If `_IsLabelMap == 0`:
            *   $u = \text{saturate}\left(\frac{\text{voxelVal} - \_P1}{\max(\_P99 - \_P1, 10^{-6})}\right)$.
        *   Sample `_TFTex` at $(u, 0.5)$ to get $\text{tf.rgb}$ and $\text{tf.a}$.
    *   **Alpha Scaling:** $a_{sample} = \text{tf.a} \times \_AlphaScale$.
    *   **Shading:**
        *   If $a_{sample} > 0.001$:
            *   Estimate normal $\mathbf{N}$ at $\mathbf{p}_{obj}$ using finite differences (see Formula).
            *   $\mathbf{n} = \text{normalize}(\mathbf{N})$.
            *   $\mathbf{l} = \text{normalize}(\mathbf{L}_{obj})$.
            *   $\text{lambert} = \text{saturate}(\text{dot}(\mathbf{n}, \mathbf{l}))$.
            *   $\text{lighting} = \_Ambient + \_LightIntensity \times \text{lambert}$.
            *   $\text{litColor} = \text{tf.rgb} \times \text{lighting}$.
    *   **Accumulation:**
        *   If $a_{sample} > 0.001$:
            *   $\text{remain} = 1.0 - \text{accA}$.
            *   $\text{accRGB} += \text{remain} \times \text{litColor} \times a_{sample}$.
            *   $\text{accA} += \text{remain} \times a_{sample}$.
    *   **Advance:** $t += \_StepSize$.

4.  **Final Output:**
    *   $\text{outColor} = \text{accRGB} \times \_Brightness$.
    *   Return $\text{float4}(\text{outColor}, \text{accA})$.

### Formulas

**Normal Estimation:**
The normal $\mathbf{N}$ is estimated using central finite differences with step size $\epsilon = 0.002$ in object space:
$$ \mathbf{N} = \frac{[V(x+\epsilon, y, z) - V(x-\epsilon, y, z), \quad V(x, y+\epsilon, z) - V(x, y-\epsilon, z), \quad V(x, y, z+\epsilon) - V(x, y, z-\epsilon)]}{\sqrt{\max(\|\mathbf{grad}\|^2, 10^{-6})}} $$
where the denominator is clamped to $10^{-6}$ before the square root to prevent division by zero [S1, unofficial].

**Continuous Normalization:**
$$ u = \text{saturate}\left(\frac{val - P1}{\max(P99 - P1, 10^{-6})}\right) $$
If $P99 = P1$, the denominator becomes $10^{-6}$ to avoid division by zero [S1, unofficial].

**Discrete Label Mapping:**
$$ u = \frac{\text{clamp}(\text{round}(val), 0, 255)}{255} $$

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gradient_based_shading(cam_pos_obj, frag_pos_obj, light_dir_world, volume_tex, tf_tex, params):
    # 1. Ray Setup
    ray_dir_obj = normalize(frag_pos_obj - cam_pos_obj)
    light_dir_obj = transform_world_to_object(light_dir_world)
    
    t_enter, t_exit = intersect_ray_box(ray_dir_obj, cam_pos_obj, box_min=(-0.5,-0.5,-0.5), box_max=(0.5,0.5,0.5))
    if t_enter is None:
        return (0.0, 0.0, 0.0, 0.0)
    
    t = max(t_enter, 0.0)
    acc_rgb = (0.0, 0.0, 0.0)
    acc_a = 0.0
    
    # 2. Raymarching Loop
    for i in range(params.MAX_STEPS):
        if t > t_exit or acc_a >= 0.9999:
            break
            
        p_obj = cam_pos_obj + ray_dir_obj * t
        uvw = p_obj + 0.5
        
        if not (0.0 <= uvw.x <= 1.0 and 0.0 <= uvw.y <= 1.0 and 0.0 <= uvw.z <= 1.0):
            break
            
        voxel_val = volume_tex.sample(uvw).r
        
        # Transfer Function
        if params.IsLabelMap:
            u = clamp(round(voxel_val), 0, 255) / 255.0
        else:
            denom = max(params.P99 - params.P1, 1e-6)
            u = saturate((voxel_val - params.P1) / denom)
            
        tf_sample = tf_tex.sample((u, 0.5))
        tf_rgb = tf_sample.rgb
        tf_a = tf_sample.a
        
        a_sample = tf_a * params.AlphaScale
        
        # Shading
        if a_sample > 0.001:
            # Estimate Normal
            eps = 0.002
            grad_x = volume_tex.sample((uvw.x + eps, uvw.y, uvw.z)).r - volume_tex.sample((uvw.x - eps, uvw.y, uvw.z)).r
            grad_y = volume_tex.sample((uvw.x, uvw.y + eps, uvw.z)).r - volume_tex.sample((uvw.x, uvw.y - eps, uvw.z)).r
            grad_z = volume_tex.sample((uvw.x, uvw.y, uvw.z + eps)).r - volume_tex.sample((uvw.x, uvw.y, uvw.z - eps)).r
            
            grad_sq = grad_x**2 + grad_y**2 + grad_z**2
            norm_len = sqrt(max(grad_sq, 1e-6))
            
            n = normalize((grad_x / norm_len, grad_y / norm_len, grad_z / norm_len))
            l = normalize(light_dir_obj)
            
            lambert = saturate(dot(n, l))
            lighting = params.Ambient + params.LightIntensity * lambert
            lit_color = tf_rgb * lighting
            
            # Accumulation
            remain = 1.0 - acc_a
            acc_rgb += remain * lit_color * a_sample
            acc_a += remain * a_sample
            
        t += params.StepSize
        
    out_rgb = acc_rgb * params.Brightness
    return (out_rgb.r, out_rgb.g, out_rgb.b, acc_a)
```

### Edge Cases and Pitfalls
*   **Division by Zero in Normalization:** If `P99` equals `P1`, the denominator in continuous normalization is zero. This is handled by using `max(_P99 - _P1, 1e-6)` [S1, unofficial].
*   **Zero-Length Gradient:** If the gradient vector is zero, normal estimation divides by zero. This is handled by clamping the squared length to $10^{-6}$ before the square root [S1, unofficial].
*   **Premature Termination:** The loop breaks if accumulated alpha reaches $0.9999$ to allow some background to show through, preventing total opacity too early [S1, unofficial].

## Operating characteristics

*   **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of steps along the ray (up to `MAX_STEPS`).
*   **Memory Complexity:** $O(V)$ for the volume texture and $O(1)$ for the transfer function (fixed size).
*   **Strengths:** Simple to implement; provides basic surface-like shading for volumetric data; supports both continuous and discrete label maps.
*   **Limitations:** Lambertian shading may appear flat for complex internal structures; finite difference normals can be noisy if the volume is not smooth; performance depends heavily on `MAX_STEPS` and `StepSize`.
*   **When to use:** Real-time visualization of medical or scientific volumes where basic surface appearance is sufficient and hardware resources are limited.
*   **When not to use:** When high-fidelity specular highlights or complex material properties are required (use [GGX Microfacet BRDF](3715-ggx-microfacet-brdf.md) instead).

## Background

Gradient-Based Shading is implemented in the `VolumeDVR.shader` component of the VRDF SDK, a tool for real-time volume rendering of 3D medical or scientific data [S1, unofficial]. It was developed by Guillaume Schneider as part of the `vrdf-sdk` repository, designed to work with Unity's URP (Universal Render Pipeline) [S1, unofficial]. The method encapsulates both continuous anatomy (MRI/CT) and discrete segmentations (AI masks, tumor maps) within a single binary container format (`.vrdf`) [S1, unofficial].

## Sources

[S1] guillaume-schneider/vrdf-sdk, https://github.com/guillaume-schneider/vrdf-sdk, fetched 2026-10-26 (unofficial)
