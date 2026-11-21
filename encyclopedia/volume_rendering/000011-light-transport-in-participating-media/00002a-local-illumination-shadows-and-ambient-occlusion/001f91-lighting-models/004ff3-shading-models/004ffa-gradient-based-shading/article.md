# Gradient-based shading

Also called: DVR.shader, Front-to-Back DVR with Gradient-Based Shading, Phong Shading model, VolumeDVR

## Overview

Gradient-based shading is a local illumination technique for direct volume rendering (DVR) that approximates surface lighting by estimating normals from the scalar field gradient and applying a Phong-like reflection model [S1][S2, unofficial]. It solves the problem of flat, "cloud-like" appearance in standard volume rendering by providing crisp depth cues and surface orientation information [S1, S3]. Unlike volumetric shading which accounts for scattering and global occlusion, gradient-based shading is computationally efficient but limited to local surface properties, making it unsuitable for translucent media where subsurface scattering is significant [S1][S4, unofficial].

## How it works

**Inputs and Outputs**
- **Input**: A 3D scalar volume texture (voxel values), a transfer function (TF) mapping values to color/alpha, light direction, and camera parameters.
- **Output**: A 2D image with accumulated color and alpha.

**Data Structures**
- **Volume Texture (`_VolumeTex`)**: 3D texture, format `RFloat` (single channel `float32`). Memory order is x-fast, then y, then z [S2, unofficial].
- **Transfer Function (`_TFTex`)**: 2D texture, format `RGBAFloat`. Dimensions $256 \times 1$. The U coordinate maps to normalized intensity/label, V is fixed at 0.5 [S2, unofficial].
- **Metadata**: Includes dimensions, voxel spacing, affine matrix, and normalization percentiles ($P1, P99$) [S2, unofficial].

**Parameters**
- `_StepSize`: Distance advanced per sample. Range: $0.0005$ to $0.02$. Default: $0.003$ [S2, unofficial].
- `_Brightness`: Global RGB multiplier. Range: $0$ to $5$. Default: $1.5$ [S2, unofficial].
- `_AlphaScale`: Alpha multiplier before accumulation. Range: $0$ to $2$. Default: $0.5$ [S2, unofficial].
- `_LightDir`: World-space light direction. Default: $(0,0,1,0)$ [S2, unofficial].
- `_LightIntensity`: Diffuse intensity. Range: $0$ to $5$. Default: $1.0$ [S2, unofficial].
- `_Ambient`: Ambient constant. Range: $0$ to $1$. Default: $0.2$ [S2, unofficial].
- `_IsLabelMap`: Integer flag ($1$ or $0$). $1$ for discrete labels, $0$ for continuous intensities [S2, unofficial].
- `_P1`, `_P99`: Percentiles for continuous normalization [S2, unofficial].
- `MAX_STEPS`: Maximum samples per ray. Default: $2048$ [S2, unofficial].

**Algorithm Steps**

1.  **Ray Setup**:
    - Reconstruct camera position and fragment position in object space (cube $[-0.5, +0.5]^3$) [S2, unofficial].
    - Compute ray direction $\vec{V}_{obj}$ as normalized difference.
    - Intersect ray with object-space bounding box $[-0.5, 0.5]^3$. If no intersection, return transparent black [S2, unofficial].
    - Transform light direction $\vec{L}_{world}$ to object space $\vec{L}_{obj}$ [S2, unofficial].

2.  **Initialization**:
    - Set $t = \max(t_{enter}, 0.0)$.
    - Initialize accumulators: $\text{accRGB} = (0,0,0)$, $\text{accA} = 0.0$ [S2, unofficial].

3.  **Raymarching Loop** (iterate $i$ from $0$ to `MAX_STEPS`):
    - **Termination**: Break if $t > t_{exit}$ or $\text{accA} \ge 0.9999$ [S2, unofficial].
    - **Position**: Compute object-space position $\vec{p}_{obj} = \vec{p}_{cam, obj} + \vec{V}_{obj} \cdot t$ [S2, unofficial].
    - **Coordinates**: Convert $\vec{p}_{obj}$ to UVW coordinates in $[0,1]^3$: $\text{uvw} = \vec{p}_{obj} + 0.5$. Break if any coordinate is outside $[0,1]$ [S2, unofficial].
    - **Sampling**:
        - Sample volume texture at `uvw`. Extract red channel as voxel value $val$ [S2, unofficial].
        - **Normalization**:
            - If `_IsLabelMap` == $1$: $u = \frac{\text{clamp}(\text{round}(val), 0, 255)}{255}$ [S2, unofficial].
            - If `_IsLabelMap` == $0$: $u = \text{saturate}\left(\frac{val - P1}{\max(P99 - P1, 10^{-6})}\right)$ [S2, unofficial].
        - Sample TF at $(u, 0.5)$ to get $\text{tf.rgb}$ and $\text{tf.a}$ [S2, unofficial].
        - Compute sample alpha: $a_{sample} = \text{tf.a} \cdot \text{\_AlphaScale}$ [S2, unofficial].
    - **Shading** (if $a_{sample} > 0.001$):
        - **Normal Estimation**: Compute gradient $\nabla V$ using central differences with $\epsilon = 0.002$ in object space:
          $$ \nabla V = [V(x+\epsilon, y, z) - V(x-\epsilon, y, z), V(x, y+\epsilon, z) - V(x, y-\epsilon, z), V(x, y, z+\epsilon) - V(x, y, z-\epsilon)] $$
          Estimate normal $\vec{N} = \frac{\nabla V}{\max(\|\nabla V\|, 10^{-6})}$ [S2, unofficial].
        - **Lighting**:
          - Compute diffuse term: $\text{lambert} = \text{saturate}(\vec{N} \cdot \vec{L}_{obj})$ [S2, unofficial].
          - Compute lighting: $\text{lighting} = \text{\_Ambient} + \text{\_LightIntensity} \cdot \text{lambert}$ [S2, unofficial].
          - Compute lit color: $\text{litColor} = \text{tf.rgb} \cdot \text{lighting}$ [S2, unofficial].
          - *Note*: The full Phong model includes specular terms $k_s i_{m,s} (R \cdot V)^\alpha$ and attenuation $A$ for point lights [S1].
    - **Accumulation** (if $a_{sample} > 0.001$):
      $$ \text{remain} = 1.0 - \text{accA} $$
      $$ \text{accRGB} += \text{remain} \cdot \text{litColor} \cdot a_{sample} $$
      $$ \text{accA} += \text{remain} \cdot a_{sample} $$
    - **Advance**: $t += \text{\_StepSize}$ [S2, unofficial].

4.  **Final Output**:
    - $\text{accRGB} *= \text{\_Brightness}$ [S2, unofficial].
    - Return $\text{float4}(\text{accRGB}, \text{accA})$ [S2, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gradient_based_shading(ray_origin_obj, ray_dir_obj, light_dir_obj, volume_tex, tf_tex, params):
    # 1. Ray-Box Intersection
    t_enter, t_exit = intersect_box(ray_origin_obj, ray_dir_obj, bounds=[-0.5, 0.5])
    if t_exit < 0 or t_enter > t_exit:
        return (0.0, 0.0, 0.0, 0.0)
    
    t = max(t_enter, 0.0)
    acc_rgb = (0.0, 0.0, 0.0)
    acc_a = 0.0
    
    # 2. Loop
    for i in range(params.MAX_STEPS):
        if t > t_exit or acc_a >= 0.9999:
            break
            
        # 3. Position & Coordinates
        p_obj = ray_origin_obj + ray_dir_obj * t
        uvw = p_obj + 0.5
        
        if not (0.0 <= uvw.x <= 1.0 and 0.0 <= uvw.y <= 1.0 and 0.0 <= uvw.z <= 1.0):
            t += params.StepSize
            continue
            
        # 4. Sampling
        val = volume_tex.sample(uvw) # Extract red channel
        
        # Normalization
        if params.IsLabelMap:
            u = clamp(round(val), 0, 255) / 255.0
        else:
            denom = max(params.P99 - params.P1, 1e-6)
            u = saturate((val - params.P1) / denom)
            
        tf_color, tf_alpha = tf_tex.sample(u, 0.5)
        a_sample = tf_alpha * params.AlphaScale
        
        # 5. Shading
        if a_sample > 0.001:
            # Normal Estimation
            eps = 0.002
            grad_x = volume_tex.sample(uvw + (eps, 0, 0)).x - volume_tex.sample(uvw + (-eps, 0, 0)).x
            grad_y = volume_tex.sample(uvw + (0, eps, 0)).x - volume_tex.sample(uvw + (0, -eps, 0)).x
            grad_z = volume_tex.sample(uvw + (0, 0, eps)).x - volume_tex.sample(uvw + (0, 0, -eps)).x
            
            grad_norm_sq = grad_x**2 + grad_y**2 + grad_z**2
            grad_norm = max(grad_norm_sq, 1e-6)**0.5
            
            N = normalize((grad_x, grad_y, grad_z))
            
            # Lighting (Phong-like)
            lambert = saturate(dot(N, light_dir_obj))
            lighting = params.Ambient + params.LightIntensity * lambert
            
            # Specular and Attenuation (Simplified in S2, Full in S1)
            # S2 uses only ambient + diffuse. S1 includes specular and attenuation.
            # Here we follow S2's explicit shader steps but acknowledge S1's model.
            lit_color = tf_color * lighting
            
            # 6. Accumulation
            remain = 1.0 - acc_a
            acc_rgb = acc_rgb + remain * lit_color * a_sample
            acc_a = acc_a + remain * a_sample
            
        t += params.StepSize
        
    acc_rgb = acc_rgb * params.Brightness
    return (acc_rgb[0], acc_rgb[1], acc_rgb[2], acc_a)
```

## Subtopics

### [Hybrid Gradient Volume Raycasting with Lasso Masking](2059-hybrid-gradient-volume-raycasting-with-lasso-masking.md)
This method performs direct volume raycasting using pre-computed Hounsfield Unit (HU) gradients for lighting direction, applying Phong shading only when a significant opacity change is detected along the gradient vector. It combines this hybrid lighting model with lasso-based voxel masking to selectively render regions of interest and cutting cube bounds to clip the volume. It differs from standard Phong shading by decoupling the lighting direction from the local sample gradient, relying instead on global HU gradients, and by gating the lighting contribution based on opacity thresholds to prevent shading artifacts on smooth structures like vessels.

### [Hybrid Scattering](12458-hybrid-scattering.md)
Hybrid Scattering is a stochastic shading technique for distance-field ray marching that switches between surface-based Bidirectional Reflectance Distribution Function (BRDF) and volumetric phase function scattering based on the local gradient magnitude. It solves the problem of rendering ambiguity in ill-defined regions by providing specular highlights where gradients are strong and volumetric scattering where gradients are weak, reducing sample variance compared to pure volumetric methods. Unlike conventional Phong shading, the underlying BRDF model is reciprocal and energy conserving.

### Gradient Shading for Turbulent Flow Visualization
A technique to obtain crisp and solid-looking images from volume-rendered turbulent flow data by applying additional gradient shading, solving the problem of 'cloud like or fuzzy' images characteristic of direct volume rendering of space filling 3-D data [S3]. It applies solid-texture-mapping to map one velocity component onto another, then applies gradient shading with ambient, diffuse, specular, and glossiness contributions [S3]. Unlike geometrical rendering techniques, this preserves the full integrity of the original dataset, making it more adaptable to analysis [S3].

### Gradient-based Surface Model
A shading technique that uses local gradients to simulate surface details, providing local shadowing and depth cues. It reflects local details and is often preferred for CT scans [S4, unofficial]. The local gradient can be computed using either the opacity normal or the density normal; opacity gradient is particularly useful in medical images where tissues are made transparent [S4, unofficial]. Unlike global shading, gradient shading reflects local details. It can be combined with volumetric shading or LAO, but not both simultaneously [S4, unofficial].

## Operating characteristics

- **Time Complexity**: Linear with respect to the number of samples per ray and the number of rays (pixels). Normal estimation requires 6 additional texture lookups per sample [S2, unofficial].
- **Memory Complexity**: Dominated by the volume texture and transfer function. The method itself uses minimal additional memory per ray [S2, unofficial].
- **Performance**: Generally faster than volumetric scattering or shadow ray techniques because it avoids secondary ray casts [S1]. Performance on a Stardent GS-1000 was demonstrated for 54 GB turbulent flow data [S3].
- **Strengths**: Low computational cost, realistic surface lighting effects, easy adaptation to lighting scenarios, provides clear depth cues [S1].
- **Limitations**: Cannot capture scattering effects of translucent materials or media occlusions [S1]. Can impose an overwhelmingly metallic feeling, making ultrasound data look unrealistic [S4, unofficial]. Limited to local details, ignoring global lighting interactions [S4, unofficial].
- **When to Use**: When real-time performance is critical and the volume has well-defined surfaces or high-contrast boundaries. Suitable for CT scans and structural visualization [S4, unofficial].
- **When Not to Use**: For translucent media where subsurface scattering is important, or for noisy data like ultrasound where the metallic artifact is undesirable [S1][S4, unofficial].

## Background

Gradient-based shading was introduced as a widely used method to capture surface reflection in volume rendering by using the normalized surface gradient of scalar values [S1]. It is based on the Phong Shading model [S1]. The technique has been applied to various domains, including medical imaging (CT, MRI) and scientific visualization (turbulent flow) [S1, S3][S4, unofficial]. In medical visualization, it is often implemented in frameworks like VTK.js and Unity-based renderers (e.g., `VolumeDVR.shader`) [S2, S4, unofficial]. The method addresses the "cloud-like" appearance of standard DVR by adding surface-like lighting cues [S3].

## Sources

[S1] Interactive, in-browser cinematic volume rendering of medical images, https://doi.org/10.1080/21681163.2022.2145239, fetched 2026-10-26
[S2] guillaume-schneider/vrdf-sdk, https://github.com/guillaume-schneider/vrdf-sdk, fetched 2026-10-26 (unofficial)
[S3] Direct volume rendering of curvilinear volumes, https://doi.org/10.1145/99308.99318, fetched 2026-10-28
[S4] Cinematic Volume Rendering in Your Web Browser: VolView and VTK.js, https://www.kitware.com/cinematic-volume-rendering/, fetched 2026-10-27 (unofficial)
