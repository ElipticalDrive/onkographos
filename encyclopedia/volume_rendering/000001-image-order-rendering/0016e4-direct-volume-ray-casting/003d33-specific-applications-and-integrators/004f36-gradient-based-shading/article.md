# Gradient-based shading

Also called: DVR.shader, Front-to-Back DVR with Gradient-Based Shading, Phong Shading model, VolumeDVR

## Overview

Gradient-based shading is a direct volume rendering (DVR) technique that approximates surface lighting by estimating normals from the scalar field gradient and applying a Phong illumination model [S1]. It solves the problem of flat, unstructured appearance in ray-cast volumes by providing realistic surface orientation and specular highlights with low computational cost [S1]. Unlike volumetric shading methods that account for media occlusion or scattering, this method only considers adjacent voxels, making it unsuitable for capturing translucent scattering effects or global shadows [S1].

## How it works

**Inputs and Outputs**
*   **Input:** A 3D scalar volume texture (typically `RFloat`/`float32`), a 2D transfer function texture (`RGBAFloat`), camera parameters (position, direction), and light parameters (direction, intensity, ambient color).
*   **Output:** A 2D image with accumulated RGB color and alpha value per pixel.

**Data Structures**
*   **Volume Texture (`_VolumeTex`):** 3D texture, format `RFloat` (single channel `float32`). Memory order is x-fast, then y, then z [S2, unofficial].
*   **Transfer Function Texture (`_TFTex`):** 2D texture, format `RGBAFloat`. Dimensions are $256 \times 1$. The U coordinate maps to normalized intensity/label, and the V coordinate is fixed at $0.5$ [S2, unofficial].
*   **Metadata:**
    *   `_IsLabelMap`: Integer flag (1 for discrete labels, 0 for continuous intensities) [S2, unofficial].
    *   `_P1`, `_P99`: Percentiles for normalizing continuous values (used when `_IsLabelMap` is 0) [S2, unofficial].
    *   `_StepSize`: Distance advanced along the ray per sample. Range: $0.0005$ to $0.02$. Default: $0.003$ [S2, unofficial].
    *   `_Brightness`: Global multiplier for accumulated RGB. Range: $0$ to $5$. Default: $1.5$ [S2, unofficial].
    *   `_AlphaScale`: Multiplier for sampled alpha. Range: $0$ to $2$. Default: $0.5$ [S2, unofficial].
    *   `_LightDir`: World-space light direction. Default: $(0, 0, 1, 0)$ [S2, unofficial].
    *   `_LightIntensity`: Diffuse light intensity. Range: $0$ to $5$. Default: $1.0$ [S2, unofficial].
    *   `_Ambient`: Ambient lighting constant. Range: $0$ to $1$. Default: $0.2$ [S2, unofficial].
    *   `MAX_STEPS`: Maximum samples per ray. Set to $2048$ [S2, unofficial].

**Algorithm Steps**

1.  **Ray Initialization:**
    *   Reconstruct camera position and fragment position in object space (bounding box $[-0.5, +0.5]^3$) [S2, unofficial].
    *   Compute ray direction as the normalized difference between fragment and camera positions [S2, unofficial].
    *   Intersect the ray with the object-space bounding box $[-0.5, 0.5]^3$. If no intersection, return transparent black [S2, unofficial].
    *   Let $t_{enter}$ and $t_{exit}$ be the intersection distances. Initialize $t = \max(t_{enter}, 0.0)$ [S2, unofficial].
    *   Initialize accumulators: $accRGB = (0, 0, 0)$ and $accA = 0.0$ [S2, unofficial].
    *   Transform the world-space light direction to object space: $L_{obj}$ [S2, unofficial].

2.  **Ray Marching Loop:**
    *   Iterate step from $0$ to $MAX\_STEPS$.
    *   **Termination Check:** Break if $t > t_{exit}$ or if $accA \ge 0.9999$ [S2, unofficial].
    *   **Sample Position:** Compute current object-space position:
        $$ p_{obj} = camPos_{obj} + rayDir_{obj} \times t $$
    *   **Texture Coordinates:** Convert $p_{obj}$ to $[0, 1]^3$ UVW coordinates:
        $$ uvw = p_{obj} + 0.5 $$
        Break if any coordinate is outside $[0, 1]$ [S2, unofficial].
    *   **Voxel Sampling:** Sample the volume texture at $uvw$. Extract the red channel as the voxel value $val$ [S2, unofficial].
    *   **Transfer Function Sampling:**
        *   If `_IsLabelMap` is 1:
            *   Round $val$ to the nearest integer.
            *   Clamp to $[0, 255]$.
            *   Normalize to $[0, 1]$ by dividing by $255$:
                $$ u = \frac{\text{clamp}(\text{round}(val), 0, 255)}{255} $$
            *   Sample `_TFTex` at $(u, 0.5)$ to get $tf.rgb$ and $tf.a$ [S2, unofficial].
        *   If `_IsLabelMap` is 0:
            *   Normalize using percentiles:
                $$ u = \text{saturate}\left(\frac{val - P1}{\max(P99 - P1, 10^{-6})}\right) $$
            *   Sample `_TFTex` at $(u, 0.5)$ to get $tf.rgb$ and $tf.a$ [S2, unofficial].
    *   **Alpha Scaling:**
        $$ aSample = tf.a \times \_AlphaScale $$
    *   **Shading (if $aSample > 0.001$):**
        *   **Normal Estimation:** Estimate normal $N$ at $p_{obj}$ using central differences with $\epsilon = 0.002$ in object space:
            $$ grad = [V(x+\epsilon, y, z) - V(x-\epsilon, y, z), \quad V(x, y+\epsilon, z) - V(x, y-\epsilon, z), \quad V(x, y, z+\epsilon) - V(x, y, z-\epsilon)] $$
            Clamp $\|grad\|^2$ to $10^{-6}$ before square root to prevent division by zero:
            $$ N = \frac{grad}{\sqrt{\max(\|grad\|^2, 10^{-6})}} $$
        *   **Lighting Calculation:**
            *   Compute view direction $V_{obj}$ (normalized vector from $p_{obj}$ to camera).
            *   Compute light reflection vector $R$ based on $N$ and $L_{obj}$.
            *   Compute Lambertian diffuse term:
                $$ lambert = \text{saturate}(\text{dot}(\text{normalize}(N), \text{normalize}(L_{obj}))) $$
            *   Compute lighting intensity:
                $$ lighting = \_Ambient + \_LightIntensity \times lambert $$
            *   Compute lit color:
                $$ litColor = tf.rgb \times lighting $$
            *   *Note:* The full Phong model includes specular terms $k_s i_{m,s} (R \cdot V)^\alpha$ and attenuation $A$ for point lights, but the implementation notes specify using `_LightIntensity` for diffuse and `_Ambient` for ambient, effectively simplifying the model or assuming specific coefficient defaults [S1][S2, unofficial].
    *   **Front-to-Back Accumulation (if $aSample > 0.001$):**
        *   Compute remaining transparency:
            $$ remain = 1.0 - accA $$
        *   Update color accumulator:
            $$ accRGB += remain \times litColor \times aSample $$
        *   Update alpha accumulator:
            $$ accA += remain \times aSample $$
    *   **Step Advance:** Increment $t$ by `_StepSize` [S2, unofficial].

3.  **Final Output:**
    *   Multiply $accRGB$ by `_Brightness`.
    *   Return $float4(accRGB, accA)$ [S2, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_gradient_shading(volume_tex, tf_tex, cam_pos_obj, cam_dir_obj, light_dir_obj, 
                            is_label_map, p1, p99, step_size, alpha_scale, brightness, 
                            light_intensity, ambient, max_steps):
    """
    Renders a volume using gradient-based shading.
    
    Args:
        volume_tex: 3D texture (RFloat)
        tf_tex: 2D texture (RGBAFloat, 256x1)
        cam_pos_obj: vec3, camera position in object space
        cam_dir_obj: vec3, ray direction in object space
        light_dir_obj: vec3, light direction in object space
        is_label_map: bool, True if discrete labels, False if continuous
        p1: float, lower percentile for normalization
        p99: float, upper percentile for normalization
        step_size: float, distance per sample
        alpha_scale: float, alpha multiplier
        brightness: float, global brightness multiplier
        light_intensity: float, diffuse light intensity
        ambient: float, ambient light constant
        max_steps: int, maximum samples per ray
        
    Returns:
        vec4: Accumulated color and alpha
    """
    # 1. Ray Initialization
    # Intersect ray with bounding box [-0.5, 0.5]^3
    t_enter, t_exit = intersect_box(cam_pos_obj, cam_dir_obj, vec3(-0.5), vec3(0.5))
    if t_exit < 0:
        return vec4(0.0, 0.0, 0.0, 0.0)
    
    t = max(t_enter, 0.0)
    acc_rgb = vec3(0.0)
    acc_a = 0.0
    
    # 2. Ray Marching Loop
    for i in range(max_steps):
        if t > t_exit or acc_a >= 0.9999:
            break
            
        # Sample Position
        p_obj = cam_pos_obj + cam_dir_obj * t
        
        # Texture Coordinates
        uvw = p_obj + vec3(0.5)
        if any(coord < 0.0 or coord > 1.0 for coord in uvw):
            break
            
        # Voxel Sampling
        val = volume_tex.sample(uvw).r
        
        # Transfer Function Sampling
        if is_label_map:
            u = clamp(round(val), 0, 255) / 255.0
        else:
            denom = max(p99 - p1, 1e-6)
            u = clamp((val - p1) / denom, 0.0, 1.0)
            
        tf_color, tf_alpha = tf_tex.sample(vec2(u, 0.5)).rgb, tf_tex.sample(vec2(u, 0.5)).a
        
        # Alpha Scaling
        a_sample = tf_alpha * alpha_scale
        
        # Shading
        if a_sample > 0.001:
            # Normal Estimation
            epsilon = 0.002
            grad_x = volume_tex.sample(uvw + vec3(epsilon, 0, 0)).r - volume_tex.sample(uvw + vec3(-epsilon, 0, 0)).r
            grad_y = volume_tex.sample(uvw + vec3(0, epsilon, 0)).r - volume_tex.sample(uvw + vec3(0, -epsilon, 0)).r
            grad_z = volume_tex.sample(uvw + vec3(0, 0, epsilon)).r - volume_tex.sample(uvw + vec3(0, 0, -epsilon)).r
            grad = vec3(grad_x, grad_y, grad_z)
            
            grad_norm_sq = dot(grad, grad)
            if grad_norm_sq < 1e-6:
                grad_norm_sq = 1e-6
            N = grad / sqrt(grad_norm_sq)
            
            # Lighting
            V_obj = normalize(cam_pos_obj - p_obj)
            # Assuming standard Phong reflection vector calculation
            R_obj = reflect(-light_dir_obj, N)
            
            lambert = max(dot(normalize(N), normalize(light_dir_obj)), 0.0)
            lighting = ambient + light_intensity * lambert
            
            lit_color = tf_color * lighting
            
            # Front-to-Back Accumulation
            remain = 1.0 - acc_a
            acc_rgb += remain * lit_color * a_sample
            acc_a += remain * a_sample
            
        # Step Advance
        t += step_size
        
    # 3. Final Output
    acc_rgb *= brightness
    return vec4(acc_rgb, acc_a)
```

## Subtopics

### [Hybrid Scattering](12458-hybrid-scattering.md)
Hybrid Scattering is a stochastic shading technique for distance-field ray marching that switches between surface-based Bidirectional Reflectance Distribution Function (BRDF) and volumetric phase function scattering based on the local gradient magnitude. It solves the problem of rendering ambiguity in ill-defined regions by providing specular highlights where gradients are strong and volumetric scattering where gradients are weak, reducing sample variance compared to pure volumetric methods. Unlike conventional Phong shading, the underlying BRDF model is reciprocal and energy conserving.

## Operating characteristics

*   **Time Complexity:** Linear with respect to the number of samples per ray ($O(N)$) and the number of pixels. Normal estimation requires 6 additional texture lookups per sample, increasing memory bandwidth usage compared to unshaded DVR.
*   **Memory Complexity:** Depends on the volume texture size and transfer function size. The method itself uses constant memory per ray.
*   **Measured Performance:** Generally fast enough for real-time rendering (e.g., WebGL 2.0) due to the simplicity of the Phong model compared to volumetric scattering or path tracing [S1].
*   **Strengths:** Low time complexity, realistic surface lighting effects, easy adaptation to any lighting scenario, allows clinicians to quickly capture surface orientation and reflection [S1].
*   **Limitations:** Cannot capture scattering effects of translucent materials or media occlusions because it only considers adjacent voxels [S1].
*   **When to use:** When real-time performance is critical and surface detail/orientation is more important than volumetric translucency or shadowing.
*   **When not to use:** When accurate representation of translucent media, global shadows, or in-scattering is required.

## Background

Gradient-based shading was introduced as a widely used method for capturing surface reflection in volume rendering, particularly in medical imaging contexts [S1]. It replaces the in-scattering term in the volume rendering integral with a Phong shading model [S1]. The method is based on the normalized surface gradient of scalar values, offering a balance between computational efficiency and visual realism [S1]. It has been implemented in various systems, including WebGL-based viewers and Unity-based medical visualization tools (e.g., VRDF SDK) [S1][S2, unofficial].

## Sources

[S1] Interactive, in-browser cinematic volume rendering of medical images, https://doi.org/10.1080/21681163.2022.2145239, fetched 2026-10-26
[S2] guillaume-schneider/vrdf-sdk, https://github.com/guillaume-schneider/vrdf-sdk, fetched 2026-10-26 (unofficial)
