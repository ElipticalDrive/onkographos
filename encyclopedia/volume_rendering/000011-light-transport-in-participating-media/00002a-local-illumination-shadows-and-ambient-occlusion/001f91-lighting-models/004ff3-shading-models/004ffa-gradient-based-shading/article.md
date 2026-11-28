# Gradient-based shading

Also called: DVR.shader, Front-to-Back DVR with Gradient-Based Shading, Phong Shading model, VolumeDVR

## Overview

Gradient-based shading is a local illumination technique for direct volume rendering (DVR) that approximates surface lighting by estimating normals from the scalar field gradient and applying a Phong-like reflection model [S1][S2, unofficial]. It solves the problem of flat, "cloud-like" appearance in standard volume rendering by providing crisp depth cues and surface orientation information [S1, S3]. Unlike volumetric shading which accounts for scattering and global occlusion, gradient-based shading is computationally efficient but limited to local surface properties, making it unsuitable for translucent media where subsurface scattering is significant [S1].

## How it works

**Inputs and Outputs**
- **Input**: A 3D scalar volume texture (voxel values), a transfer function (TF) mapping values to color/alpha, light direction, and camera parameters [S2, unofficial].
- **Output**: A 2D image with accumulated color and alpha [S2, unofficial].

**Data Structures**
- **Volume Texture (`_VolumeTex`)**: 3D texture, format `RFloat` (single channel `float32`). Memory order is x-fast, then y, then z [S2, unofficial].
- **Transfer Function (`_TFTex`)**: 2D texture, format `RGBAFloat`. Dimensions $256 \times 1$. The U coordinate maps to normalized intensity/label, V is fixed at $0.5$ [S2, unofficial].
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

1.  **Ray Setup**: Reconstruct camera position and fragment position in object space (cube $[-0.5, +0.5]^3$) [S2, unofficial]. Compute ray direction $\vec{V}_{obj}$ as normalized difference [S2, unofficial]. Intersect ray with object-space bounding box $[-0.5, 0.5]^3$. If no intersection, return transparent black [S2, unofficial]. Transform light direction $\vec{L}_{world}$ to object space $\vec{L}_{obj}$ [S2, unofficial].
2.  **Initialization**: Set $t = \max(t_{enter}, 0.0)$ [S2, unofficial]. Initialize accumulators: $\text{accRGB} = (0,0,0)$, $\text{accA} = 0.0$ [S2, unofficial].
3.  **Raymarching Loop** (iterate $i$ from $0$ to `MAX_STEPS`):
    - **Termination**: Break if $t > t_{exit}$ or $\text{accA} \ge 0.9999$ [S2, unofficial].
    - **Position**: Compute object-space position $\vec{p}_{obj} = \vec{p}_{cam, obj} + \vec{V}_{obj} \cdot t$ [S2, unofficial].
    - **Coordinates**: Convert $\vec{p}_{obj}$ to UVW coordinates in $[0,1]^3$: $\text{uvw} = \vec{p}_{obj} + 0.5$. Break if any coordinate is outside $[0,1]$ [S2, unofficial].
    - **Sampling**: Sample volume texture at `uvw`. Extract red channel as voxel value $val$ [S2, unofficial].
    - **Normalization**: If `_IsLabelMap` == $1$: $u = \frac{\text{clamp}(\text{round}(val), 0, 255)}{255}$ [S2, unofficial]. If `_IsLabelMap` == $0$: $u = \text{saturate}\left(\frac{val - P1}{\max(P99 - P1, 10^{-6})}\right)$ [S2, unofficial]. Sample TF at $(u, 0.5)$ to get $\text{tf.rgb}$ and $\text{tf.a}$ [S2, unofficial]. Compute sample alpha: $a_{sample} = \text{tf.a} \cdot \text{\_AlphaScale}$ [S2, unofficial].
    - **Shading** (if $a_{sample} > 0.001$): Estimate normal $\vec{N}$ using central differences with $\epsilon = 0.002$ in object space: $\nabla V = [V(x+\epsilon, y, z) - V(x-\epsilon, y, z), V(x, y+\epsilon, z) - V(x, y-\epsilon, z), V(x, y, z+\epsilon) - V(x, y, z-\epsilon)]$ [S2, unofficial]. Compute $\vec{N} = \frac{\nabla V}{\max(\|\nabla V\|, 10^{-6})}$ [S2, unofficial]. Compute diffuse term: $\text{lambert} = \text{saturate}(\vec{N} \cdot \vec{L}_{obj})$ [S2, unofficial]. Compute lighting: $\text{lighting} = \text{\_Ambient} + \text{\_LightIntensity} \cdot \text{lambert}$ [S2, unofficial]. Compute lit color: $\text{litColor} = \text{tf.rgb} \cdot \text{lighting}$ [S2, unofficial]. The full Phong model includes specular terms $k_s i_{m,s} (R \cdot V)^\alpha$ and attenuation $A$ for point lights [S1].
    - **Accumulation** (if $a_{sample} > 0.001$): $\text{remain} = 1.0 - \text{accA}$, $\text{accRGB} += \text{remain} \cdot \text{litColor} \cdot a_{sample}$, $\text{accA} += \text{remain} \cdot a_{sample}$ [S2, unofficial].
    - **Advance**: $t += \text{\_StepSize}$ [S2, unofficial].
4.  **Final Output**: $\text{accRGB} *= \text{\_Brightness}$ [S2, unofficial]. Return $\text{float4}(\text{accRGB}, \text{accA})$ [S2, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gradient_based_shading(ray_origin_obj, ray_dir_obj, light_dir_obj, volume_tex, tf_tex, params):
    # Ray-Box Intersection (slab method)
    t_min = -float('inf')
    t_max = float('inf')
    for i in range(3):
        inv_d = 1.0 / ray_dir_obj[i]
        t0 = (-0.5 - ray_origin_obj[i]) * inv_d
        t1 = (0.5 - ray_origin_obj[i]) * inv_d
        t_min = max(t_min, min(t0, t1))
        t_max = min(t_max, max(t0, t1))
    if t_max < t_min or t_max < 0.0:
        return (0.0, 0.0, 0.0, 0.0)
    t = max(t_min, 0.0)

    acc_rgb = (0.0, 0.0, 0.0)
    acc_a = 0.0

    for _ in range(params.MAX_STEPS):
        if t > t_max or acc_a >= 0.9999:
            break

        p_obj = (ray_origin_obj[0] + ray_dir_obj[0] * t,
                 ray_origin_obj[1] + ray_dir_obj[1] * t,
                 ray_origin_obj[2] + ray_dir_obj[2] * t)
        uvw = (p_obj[0] + 0.5, p_obj[1] + 0.5, p_obj[2] + 0.5)

        if not (0.0 <= uvw[0] <= 1.0 and 0.0 <= uvw[1] <= 1.0 and 0.0 <= uvw[2] <= 1.0):
            t += params.StepSize
            continue

        val = volume_tex.sample(uvw) # Extract red channel

        if params.IsLabelMap:
            u = max(0.0, min(1.0, round(val) / 255.0))
        else:
            denom = max(params.P99 - params.P1, 1e-6)
            u = max(0.0, min(1.0, (val - params.P1) / denom))

        tf_color, tf_alpha = tf_tex.sample(u, 0.5)
        a_sample = tf_alpha * params.AlphaScale

        if a_sample > 0.001:
            eps = 0.002
            # Gradient in object space
            gx = volume_tex.sample((uvw[0]+eps, uvw[1], uvw[2]))[0] - volume_tex.sample((uvw[0]-eps, uvw[1], uvw[2]))[0]
            gy = volume_tex.sample((uvw[0], uvw[1]+eps, uvw[2]))[0] - volume_tex.sample((uvw[0], uvw[1]-eps, uvw[2]))[0]
            gz = volume_tex.sample((uvw[0], uvw[1], uvw[2]+eps))[0] - volume_tex.sample((uvw[0], uvw[1], uvw[2]-eps))[0]

            grad_norm = max((gx**2 + gy**2 + gz**2)**0.5, 1e-6)
            N = (gx/grad_norm, gy/grad_norm, gz/grad_norm)

            lambert = max(0.0, min(1.0, N[0]*light_dir_obj[0] + N[1]*light_dir_obj[1] + N[2]*light_dir_obj[2]))
            lighting = params.Ambient + params.LightIntensity * lambert
            lit_color = (tf_color[0]*lighting, tf_color[1]*lighting, tf_color[2]*lighting)

            remain = 1.0 - acc_a
            acc_rgb = (acc_rgb[0] + remain * lit_color[0] * a_sample,
                       acc_rgb[1] + remain * lit_color[1] * a_sample,
                       acc_rgb[2] + remain * lit_color[2] * a_sample)
            acc_a += remain * a_sample

        t += params.StepSize

    acc_rgb = (acc_rgb[0]*params.Brightness, acc_rgb[1]*params.Brightness, acc_rgb[2]*params.Brightness)
    return (acc_rgb[0], acc_rgb[1], acc_rgb[2], acc_a)
```

## Subtopics

### [Hybrid Gradient Volume Raycasting with Lasso Masking](2059-hybrid-gradient-volume-raycasting-with-lasso-masking.md)
This method performs direct volume raycasting using pre-computed Hounsfield Unit (HU) gradients for lighting direction, applying Phong shading only when a significant opacity change is detected along the gradient vector. It combines this hybrid lighting model with lasso-based voxel masking to selectively render regions of interest and cutting cube bounds to clip the volume. It differs from standard Phong shading by decoupling the lighting direction from the local sample gradient, relying instead on global HU gradients, and by gating the lighting contribution based on opacity thresholds to prevent shading artifacts on smooth structures like vessels.

### [Hybrid Scattering](12458-hybrid-scattering.md)
Hybrid Scattering is a stochastic shading technique for distance-field ray marching that switches between surface-based Bidirectional Reflectance Distribution Function (BRDF) and volumetric phase function scattering based on the local gradient magnitude. It solves the problem of rendering ambiguity in ill-defined regions by providing specular highlights where gradients are strong and volumetric scattering where gradients are weak, reducing sample variance compared to conventional ray casters. Unlike conventional Phong shading, the underlying BRDF model is reciprocal and energy conserving.

### [Hybrid Gradient Volume Raycasting with Lasso Masking](2059-hybrid-gradient-volume-raycasting-with-lasso-masking.md)
A technique to obtain crisp and solid-looking images from volume-rendered turbulent flow data by applying additional gradient shading, solving the problem of 'cloud like or fuzzy' images characteristic of direct volume rendering of space filling 3-D data. It applies solid-texture-mapping to map one velocity component onto another, then applies gradient shading with ambient, diffuse, specular, and glossiness contributions. Unlike geometrical rendering techniques, this preserves the full integrity of the original dataset, making it more adaptable to analysis.

### [Gradient-based Surface Model](14087-gradient-based-surface-model.md)
The Gradient-based Surface Model is a local illumination technique for discrete volume rendering (DVR) that simulates surface reflection and depth cues by estimating a pseudo-normal from the local volume gradient. It is distinct from global shading methods because it reflects local geometric details rather than global occlusion, making it particularly effective for visualizing structured data like CT scans, though it can introduce an overly metallic appearance in noisy data such as ultrasound.

## Operating characteristics

- **Time Complexity**: Linear with respect to the number of samples per ray and the number of rays (pixels) [S1].
- **Memory Complexity**: Dominated by the volume texture and transfer function [S2, unofficial].
- **Strengths**: Relatively low time complexity compared to scattering, realistic surface lighting effect, and easy adaptation to any lighting scenario [S1].
- **Limitations**: Only considers adjacent voxels and cannot capture the scattering effects of translucent materials or media occlusions [S1].
- **When to Use**: When real-time performance is critical and the volume has well-defined surfaces or high-contrast boundaries [S2, unofficial].
- **When Not to Use**: For translucent media where subsurface scattering is important [S1].

## Background

Gradient-based shading was introduced as a widely used method to capture surface reflection in volume rendering by using the normalized surface gradient of scalar values [S1]. It is based on the Phong Shading model [S1]. The technique has been applied to various domains, including medical imaging (CT, MRI) and scientific visualization (turbulent flow) [S1, S3]. The method addresses the "cloud-like" appearance of standard DVR by adding surface-like lighting cues [S3].

## Sources

[S1] Interactive, in-browser cinematic volume rendering of medical images, https://doi.org/10.1080/21681163.2022.2145239, fetched 2026-10-26
[S2] guillaume-schneider/vrdf-sdk, https://github.com/guillaume-schneider/vrdf-sdk, fetched 2026-10-26 (unofficial)
[S3] Direct volume rendering of curvilinear volumes, https://doi.org/10.1145/99308.99318, fetched 2026-10-28
