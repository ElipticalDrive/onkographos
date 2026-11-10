# Gradient-Based Shading

Also called: DVR.shader, Front-to-Back DVR with Gradient-Based Shading, VolumeDVR

## Overview
Gradient-Based Shading is a discrete volume rendering (DVR) technique that renders 3D textures using front-to-back accumulation, shading sampled voxels via a pseudo-normal estimated from the volume gradient and a directional light model [S1, unofficial]. It solves the problem of flat, unlit volume appearance by introducing surface-like shading cues derived from the scalar field's spatial derivatives, allowing standard lighting models to be applied to volumetric data [S1, unofficial]. Unlike pure volumetric scattering methods, it treats the volume as a collection of semi-transparent surfaces, offering a balance between performance and visual fidelity for medical and scientific data [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume texture (`_VolumeTex`) containing scalar values (float32), a 2D transfer function texture (`_TFTex`), camera parameters (position, direction), and light parameters (direction, intensity).
*   **Output:** A 2D image with accumulated RGB color and alpha.

**Data Structures:**
*   **Volume Texture (`_VolumeTex`):** 3D texture, format `RFloat` (single channel `float32`). Memory order is x-fast, then y, then z [S1, unofficial].
*   **Transfer Function Texture (`_TFTex`):** 2D texture, format `RGBAFloat`. Dimensions are 256x1. The U coordinate maps to normalized intensity/label, and the V coordinate is fixed at 0.5 [S1, unofficial].

**Parameters:**
*   `_StepSize`: Distance advanced along the ray per sample. Range: 0.0005 to 0.02. Default: 0.003 [S1, unofficial].
*   `_Brightness`: Global multiplier applied to accumulated RGB. Range: 0 to 5. Default: 1.5 [S1, unofficial].
*   `_AlphaScale`: Multiplier for sampled alpha. Range: 0 to 2. Default: 0.5 [S1, unofficial].
*   `_LightDir`: Light direction in world space. Default: (0,0,1,0) [S1, unofficial].
*   `_LightIntensity`: Diffuse light intensity. Range: 0 to 5. Default: 1.0 [S1, unofficial].
*   `_Ambient`: Ambient lighting constant. Range: 0 to 1. Default: 0.2 [S1, unofficial].
*   `_IsLabelMap`: Integer flag (1 or 0). 1 for discrete labels, 0 for continuous intensities [S1, unofficial].
*   `_P1`, `_P99`: Percentiles for continuous normalization. Used when `_IsLabelMap` is 0 [S1, unofficial].
*   `MAX_STEPS`: Maximum samples per ray. Default: 2048 [S1, unofficial].

**Algorithm Steps:**

1.  **Vertex Shader:** Transform vertex position to clip space and compute world position [S1, unofficial].
2.  **Ray Setup (Fragment Shader):**
    *   Reconstruct camera position (`camPosObj`) and fragment position in object space (cube `[-0.5, +0.5]^3`) [S1, unofficial].
    *   Compute ray direction: `rayDirObj = normalize(fragmentPosObj - camPosObj)` [S1, unofficial].
    *   Transform light direction to object space: `Lobj = normalize(TransformWorldToObject(_LightDir))` [S1, unofficial].
3.  **Ray-Box Intersection:**
    *   Intersect `rayDirObj` with the object-space bounding box `[-0.5, 0.5]^3` [S1, unofficial].
    *   Calculate entry (`tEnter`) and exit (`tExit`) distances.
    *   If no intersection, return transparent black `float4(0,0,0,0)` [S1, unofficial].
4.  **Initialization:**
    *   Set `t = max(tEnter, 0.0)` [S1, unofficial].
    *   Initialize accumulators: `accRGB = (0,0,0)`, `accA = 0.0` [S1, unofficial].
5.  **Ray Marching Loop:**
    *   Iterate `step` from 0 to `MAX_STEPS`.
    *   **Termination Check:** Break if `t > tExit` or `accA >= 0.9999` [S1, unofficial].
    *   **Sample Position:** Compute object-space position: `pObj = camPosObj + rayDirObj * t` [S1, unofficial].
    *   **Texture Coordinates:** Convert `pObj` to `[0,1]^3` UVW: `uvw = pObj + 0.5` [S1, unofficial].
    *   **Boundary Check:** Break if any coordinate in `uvw` is outside `[0,1]` [S1, unofficial].
    *   **Voxel Sampling:** Sample volume texture: `voxelVal = texture(_VolumeTex, uvw).r` [S1, unofficial].
    *   **Transfer Function Sampling:**
        *   If `_IsLabelMap == 1`:
            *   Round `val` to nearest integer, clamp to `[0, 255]`.
            *   Normalize: `u = clamp(round(val), 0, 255) / 255.0` [S1, unofficial].
            *   Sample TF: `tf = texture(_TFTex, float2(u, 0.5))` [S1, unofficial].
        *   If `_IsLabelMap == 0`:
            *   Normalize continuous value: `u = saturate((voxelVal - _P1) / max(_P99 - _P1, 1e-6))` [S1, unofficial].
            *   Sample TF: `tf = texture(_TFTex, float2(u, 0.5))` [S1, unofficial].
    *   **Alpha Scaling:** `aSample = tf.a * _AlphaScale` [S1, unofficial].
    *   **Shading (if `aSample > 0.001`):**
        *   Estimate normal `N` at `pObj` using finite differences (see Formula).
        *   Compute Lambertian diffuse: `lambert = saturate(dot(normalize(N), normalize(Lobj)))` [S1, unofficial].
        *   Compute lighting: `lighting = _Ambient + _LightIntensity * lambert` [S1, unofficial].
        *   Compute lit color: `litColor = tf.rgb * lighting` [S1, unofficial].
    *   **Front-to-Back Accumulation (if `aSample > 0.001`):**
        *   `remain = 1.0 - accA` [S1, unofficial].
        *   `accRGB += remain * litColor * aSample` [S1, unofficial].
        *   `accA += remain * aSample` [S1, unofficial].
    *   **Step Advance:** `t += _StepSize` [S1, unofficial].
6.  **Final Output:**
    *   `accRGB *= _Brightness` [S1, unofficial].
    *   Return `float4(accRGB, accA)` [S1, unofficial].

**Formulas:**

*   **Normal Estimation:**
    The normal $\mathbf{N}$ is estimated using central finite differences in object space:
    $$ \mathbf{N} = \frac{[V(x+\epsilon, y, z) - V(x-\epsilon, y, z), \quad V(x, y+\epsilon, z) - V(x, y-\epsilon, z), \quad V(x, y, z+\epsilon) - V(x, y, z-\epsilon)]}{\sqrt{\max(\|\mathbf{grad}\|^2, 10^{-6})}} $$
    where $\epsilon = 0.002$ in object space, and the denominator is clamped to $10^{-6}$ before the square root to prevent division by zero [S1, unofficial].

*   **Continuous Normalization:**
    $$ u = \text{saturate}\left(\frac{val - P1}{\max(P99 - P1, 10^{-6})}\right) $$
    where $P1$ and $P99$ are the 1st and 99th percentiles of the volume data [S1, unofficial].

*   **Discrete Label Mapping:**
    $$ u = \frac{\text{clamp}(\text{round}(val), 0, 255)}{255} $$
    used when `_IsLabelMap` is 1 [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gradient_based_shading(cam_pos_obj, ray_dir_obj, light_dir_world, volume_tex, tf_tex, params):
    """
    cam_pos_obj: vec3, camera position in object space
    ray_dir_obj: vec3, normalized ray direction in object space
    light_dir_world: vec3, light direction in world space
    volume_tex: 3D texture (RFloat)
    tf_tex: 2D texture (RGBAFloat, 256x1)
    params: dict containing _StepSize, _Brightness, _AlphaScale, _LightIntensity, 
            _Ambient, _IsLabelMap, _P1, _P99, MAX_STEPS
    """
    # Transform light to object space
    light_dir_obj = normalize(transform_world_to_object(light_dir_world))
    
    # Ray-box intersection for bounding box [-0.5, 0.5]^3
    t_enter, t_exit = intersect_box(ray_dir_obj, min=-0.5, max=0.5)
    
    if t_enter > t_exit:
        return (0.0, 0.0, 0.0, 0.0)
    
    t = max(t_enter, 0.0)
    acc_rgb = (0.0, 0.0, 0.0)
    acc_a = 0.0
    
    for step in range(params['MAX_STEPS']):
        if t > t_exit or acc_a >= 0.9999:
            break
            
        # Sample position
        p_obj = cam_pos_obj + ray_dir_obj * t
        uvw = p_obj + 0.5
        
        # Boundary check
        if not (0.0 <= uvw.x <= 1.0 and 0.0 <= uvw.y <= 1.0 and 0.0 <= uvw.z <= 1.0):
            t += params['_StepSize']
            continue
            
        # Voxel sampling
        voxel_val = volume_tex.sample(uvw).r
        
        # Transfer function sampling
        if params['_IsLabelMap'] == 1:
            u = clamp(round(voxel_val), 0, 255) / 255.0
        else:
            denom = max(params['_P99'] - params['_P1'], 1e-6)
            u = saturate((voxel_val - params['_P1']) / denom)
            
        tf_color = tf_tex.sample((u, 0.5))
        a_sample = tf_color.a * params['_AlphaScale']
        
        if a_sample > 0.001:
            # Normal estimation
            eps = 0.002
            grad_x = volume_tex.sample((uvw.x + eps, uvw.y, uvw.z)).r - volume_tex.sample((uvw.x - eps, uvw.y, uvw.z)).r
            grad_y = volume_tex.sample((uvw.x, uvw.y + eps, uvw.z)).r - volume_tex.sample((uvw.x, uvw.y - eps, uvw.z)).r
            grad_z = volume_tex.sample((uvw.x, uvw.y, uvw.z + eps)).r - volume_tex.sample((uvw.x, uvw.y, uvw.z - eps)).r
            
            grad_sq = grad_x**2 + grad_y**2 + grad_z**2
            norm_len = sqrt(max(grad_sq, 1e-6))
            
            normal = (grad_x / norm_len, grad_y / norm_len, grad_z / norm_len)
            
            # Lighting
            lambert = saturate(dot(normalize(normal), normalize(light_dir_obj)))
            lighting = params['_Ambient'] + params['_LightIntensity'] * lambert
            lit_color = tf_color.rgb * lighting
            
            # Accumulation
            remain = 1.0 - acc_a
            acc_rgb = (
                acc_rgb[0] + remain * lit_color[0] * a_sample,
                acc_rgb[1] + remain * lit_color[1] * a_sample,
                acc_rgb[2] + remain * lit_color[2] * a_sample
            )
            acc_a += remain * a_sample
            
        t += params['_StepSize']
        
    acc_rgb = (acc_rgb[0] * params['_Brightness'], acc_rgb[1] * params['_Brightness'], acc_rgb[2] * params['_Brightness'])
    return (acc_rgb[0], acc_rgb[1], acc_rgb[2], acc_a)
```

## Subtopics

### [Hybrid Scattering](12458-hybrid-scattering.md)
Hybrid Scattering is a stochastic shading technique for distance-field ray marching that switches between surface-based Bidirectional Reflectance Distribution Function (BRDF) and volumetric phase function scattering based on the local gradient magnitude. It solves the problem of rendering ambiguity in ill-defined regions by providing specular highlights where gradients are strong and volumetric scattering where gradients are weak, reducing sample variance compared to pure volumetric methods. Unlike conventional Phong shading, the underlying BRDF model is reciprocal and energy conserving.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of pixels and $S$ is the number of steps per ray (up to `MAX_STEPS`). The complexity is linear with respect to the number of samples taken along each ray.
*   **Memory Complexity:** $O(V)$ for the volume texture and $O(T)$ for the transfer function, where $V$ is the number of voxels and $T$ is the size of the LUT.
*   **Measured Performance:** Performance depends heavily on `_StepSize` and `MAX_STEPS`. Smaller step sizes increase accuracy but reduce performance. The use of early termination (`accA >= 0.9999`) helps maintain performance in opaque regions.
*   **Strengths:** Simple to implement, computationally efficient compared to full volumetric path tracing, provides intuitive surface-like shading for volumes.
*   **Limitations:** The pseudo-normal estimation can be noisy in low-gradient regions, leading to shading artifacts. It does not account for subsurface scattering or complex light transport within the volume.
*   **When to use:** Suitable for real-time visualization of medical images (CT/MRI) and scientific data where surface appearance is desired.
*   **When not to use:** Not suitable for highly translucent or scattering media where volumetric effects are critical.

## Background
Gradient-Based Shading is a method used in real-time volume rendering, particularly within frameworks like the VRDF SDK for Unity. It leverages the gradient of the scalar field to approximate surface normals, allowing standard lighting models to be applied to volumetric data. This approach is common in medical imaging visualization to enhance the perception of depth and structure.

## Sources
[S1] guillaume-schneider/vrdf-sdk, https://github.com/guillaume-schneider/vrdf-sdk, fetched 2026-10-26 (unofficial)
