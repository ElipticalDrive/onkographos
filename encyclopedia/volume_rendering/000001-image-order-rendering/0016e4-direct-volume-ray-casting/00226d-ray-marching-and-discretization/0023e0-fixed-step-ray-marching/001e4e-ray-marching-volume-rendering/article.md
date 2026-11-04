# Ray Marching Volume Rendering

Also called: Volumetric Raymarching, Volumetric Raycasting

## Overview
Ray Marching Volume Rendering is a real-time GPU technique that visualizes 3D scalar fields (such as smoke, clouds, or density volumes) by casting rays from the camera through the volume and accumulating color and opacity along the ray path [S1, S2, S4, unofficial]. It solves the problem of rendering heterogeneous, translucent media without requiring explicit mesh extraction [S1, unofficial]. Unlike sibling methods that may use adaptive stepping or discrete integral approximations, this method typically employs a fixed step size with front-to-back compositing, often integrating gradient-based shading and transfer functions for material appearance [S4, S7, unofficial]. It is distinguished by its implementation in frameworks like Taichi for high-performance real-time visualization, supporting features such as refraction, shadows, and spherical harmonics encoding [S1, S6, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A 3D scalar field (density) `volume_texture` (or Taichi field `smoke_density_taichi`), an optional 3D color field `smoke_color_taichi`, an optional 3D index of refraction (IOR) field, a 1D transfer function texture `trfunc_texture`, camera parameters (position, direction, field of view), and light sources [S1, S4, S6, unofficial].
*   **Outputs:** A 2D image buffer containing the accumulated color and opacity for each pixel [S1, S4, unofficial].

**Data Structures:**
*   **Volume Texture:** A 3D texture where density values are stored in the red channel (accessed via `.r`) [S4, unofficial]. In Taichi implementations, this is a `ti.field` of shape `(N, N, N)` with dtype `f32` or `f16` [S1, S6, unofficial].
*   **Transfer Function:** A 1D texture mapping density values to RGBA color [S4, unofficial].
*   **Light Density:** A 3D field of shape `(N, N, N, 3)` storing pre-computed light contributions per voxel, initialized to ones or zeros depending on the shadowing model [S1, unofficial].
*   **Ray Buffers:** For GPU implementations, buffers for ray origins `rays_o` (3x f16), ray directions `rays_d` (3x f16), and hit distances `hits_t` (2x f16) [S6, unofficial].

**Parameters:**
*   `NUM_RAY_STEPS` or `SAMPLE_COUNT`: Maximum number of steps per ray (e.g., 128, 1024) [S2, S4, S6, unofficial].
*   `step_size`: Fixed distance between samples. Can be derived from `pixel_size * ray_tracing_step_size_factor` or `MAX_DISTANCE / NUM_STEPS` [S1, S4, S8, unofficial].
*   `ray_tracing_stop_threshold`: Termination threshold for transmittance (e.g., 0.01) [S1, S9, unofficial].
*   `alphaScale`: Scaling factor for alpha normalization [S3, unofficial].
*   `finalGamma`: Gamma correction exponent (typically 2.2 or 1/2.2) [S3, unofficial].
*   `densityCoef`: Global density multiplier (e.g., 50.0) [S2, unofficial].

**Numbered Steps:**

1.  **Ray Generation:** For each pixel, compute the ray origin `ro` (camera position) and direction `rd` from camera parameters [S2, unofficial].
    $$ \mathbf{d} = \mathbf{d}_{cam} + u \cdot \frac{\text{fov} \cdot (i - \text{width}/2)}{\text{width}} + v \cdot \frac{\text{fov} \cdot (j/\text{width} - 0.5)}{\text{width}} $$
    Normalize $\mathbf{d}$ [S1, unofficial].

2.  **Ray-Box Intersection:** Determine the entry ($t_{min}$) and exit ($t_{max}$) distances of the ray with the volume's bounding box (typically $[-0.5, 0.5]^3$) using the slab method [S2, unofficial].
    $$ t_{bot} = \frac{B_{min} - r_o}{r_d}, \quad t_{top} = \frac{B_{max} - r_o}{r_d} $$
    $$ t_{min} = \max(t_{bot}), \quad t_{max} = \min(t_{top}) $$

3.  **Initialization:** Initialize accumulated color $C_{acc} = (0,0,0)$, accumulated opacity $\alpha_{acc} = 0$, and transmittance $T = 1.0$ [S1, S4, unofficial].

4.  **Marching Loop:** Iterate $i$ from 0 to `NUM_RAY_STEPS` or until $t > t_{max}$:
    a. **Position Update:** Compute sample position $\mathbf{p} = \mathbf{r}_o + t \cdot \mathbf{r}_d$ [S7, unofficial].
    b. **Sampling:** Sample density $D$ from the volume texture [S4, unofficial].
    c. **Alpha Calculation:** Compute the opacity contribution for this step.
        $$ \alpha_{sample} = D \cdot \text{step\_size} \cdot \text{density\_coef} $$
        Or using Beer's Law approximation:
        $$ \alpha_{sample} = 1 - e^{-\sigma_t \cdot \Delta d} $$
        where $\sigma_t$ is the extinction coefficient derived from density [S4, S7, unofficial].
    d. **Termination Check:** If $\alpha_{acc} \ge 1.0$ or $T < 0.01$, break the loop (Early Ray Termination) [S1, S9, unofficial].
    e. **Lighting (Optional):** If lighting is enabled, compute the light contribution at $\mathbf{p}$ [S1, S8, unofficial].
        $$ L = \text{sample\_light}(\mathbf{p}) $$
    f. **Color Sampling:** Map density to color using the transfer function.
        $$ \mathbf{c}_{sample} = \text{texture}(\text{trfunc\_texture}, \text{vec2}(D, 0.5)).rgb $$
        Apply lighting: $\mathbf{c}_{sample} \leftarrow \mathbf{c}_{sample} \cdot L$ [S4, S8, unofficial].
    g. **Compositing (Front-to-Back):**
        $$ \mathbf{c}_{acc} \leftarrow \mathbf{c}_{acc} + (1 - \alpha_{acc}) \cdot \alpha_{sample} \cdot \mathbf{c}_{sample} $$
        $$ \alpha_{acc} \leftarrow \alpha_{acc} + (1 - \alpha_{acc}) \cdot \alpha_{sample} $$
        $$ T \leftarrow T \cdot (1 - \alpha_{sample}) $$
    h. **Advance:** $t \leftarrow t + \text{step\_size}$ [S4, S7, unofficial].

5.  **Background and Final Color:** Add the background color modulated by the final transmittance.
    $$ \mathbf{c}_{final} = \mathbf{c}_{acc} + \mathbf{c}_{bg} \cdot T $$
    Apply gamma correction: $\mathbf{c}_{final} = \mathbf{c}_{final}^{1/\gamma}$ [S3, unofficial].

6.  **Refraction (Optional):** If an IOR field is provided, modify the ray direction at each step based on the gradient of the IOR field.
    $$ \nabla \text{IOR} = \frac{\text{IOR}(\mathbf{p} + \Delta) - \text{IOR}(\mathbf{p} - \Delta)}{2\Delta} $$
    Use Snell's law to bend the ray. If total internal reflection occurs ($\sin \theta_{new} > 1$), terminate the ray [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_marching_volume_rendering(camera_pos, camera_dir, volume_texture, transfer_func, 
                                  num_steps, step_size, bg_color, use_lighting=False, 
                                  light_density_field=None, ior_field=None, density_coef=50.0):
    # 1. Ray Generation
    ray_origin = camera_pos
    ray_direction = normalize(camera_dir)
    
    # 2. Ray-Box Intersection (slab method for box [-0.5, 0.5]^3)
    box_min = vec3(-0.5)
    box_max = vec3(0.5)
    # Handle zero direction components to avoid division by zero
    inv_dir = vec3(1.0 / ray_direction.x if ray_direction.x != 0 else 1e9,
                   1.0 / ray_direction.y if ray_direction.y != 0 else 1e9,
                   1.0 / ray_direction.z if ray_direction.z != 0 else 1e9)
    
    t0 = (box_min - ray_origin) * inv_dir
    t1 = (box_max - ray_origin) * inv_dir
    t_min = max(min(t0.x, t1.x), min(t0.y, t1.y), min(t0.z, t1.z))
    t_max = min(max(t0.x, t1.x), max(t0.y, t1.y), max(t0.z, t1.z))
    
    if t_min > t_max:
        return bg_color # Ray misses volume

    # 3. Initialization
    acc_color = vec3(0.0)
    acc_alpha = 0.0
    transmittance = 1.0
    
    t = t_min
    
    # 4. Marching Loop
    for i in range(num_steps):
        if t > t_max:
            break
            
        # Sample position
        pos = ray_origin + ray_direction * t
        
        # Sample Density (trilinear interpolation)
        density = sample_3d_texture(volume_texture, pos)
        
        # Calculate Alpha for this step
        alpha_sample = 1.0 - exp(-density * step_size * density_coef)
        
        # Early Ray Termination
        if acc_alpha >= 1.0 or transmittance < 0.01:
            break
            
        # Lighting
        light_contribution = vec3(1.0)
        if use_lighting and light_density_field is not None:
            light_contribution = sample_3d_texture(light_density_field, pos)
            
        # Transfer Function
        tf_color = sample_1d_texture(transfer_func, vec2(density, 0.5))
        color_sample = tf_color.rgb * light_contribution
        
        # Compositing (Front-to-Back)
        acc_color += (1.0 - acc_alpha) * alpha_sample * color_sample
        acc_alpha += (1.0 - acc_alpha) * alpha_sample
        transmittance *= (1.0 - alpha_sample)
        
        # Advance
        t += step_size
        
    # 5. Final Color
    final_color = acc_color + bg_color * transmittance
    
    # Gamma Correction
    gamma = 2.2
    final_color = pow(final_color, vec3(1.0 / gamma))
    
    return final_color

def sample_3d_texture(tex, p):
    # Assumes tex is a 3D array of shape (W, H, D)
    # Normalize p to [0, 1] assuming box is [-0.5, 0.5]
    p_norm = p + 0.5
    ix = floor(p_norm.x * (W - 1))
    iy = floor(p_norm.y * (H - 1))
    iz = floor(p_norm.z * (D - 1))
    fx = p_norm.x * (W - 1) - ix
    fy = p_norm.y * (H - 1) - iy
    fz = p_norm.z * (D - 1) - iz
    # Clamp indices
    ix = clamp(ix, 0, W-1); iy = clamp(iy, 0, H-1); iz = clamp(iz, 0, D-1)
    # Fetch 8 corners
    c000 = tex[ix, iy, iz]
    c100 = tex[ix+1, iy, iz]
    c010 = tex[ix, iy+1, iz]
    c110 = tex[ix+1, iy+1, iz]
    c001 = tex[ix, iy, iz+1]
    c101 = tex[ix+1, iy, iz+1]
    c011 = tex[ix, iy+1, iz+1]
    c111 = tex[ix+1, iy+1, iz+1]
    # Lerp
    c00 = lerp(c000, c100, fx)
    c10 = lerp(c010, c110, fx)
    c01 = lerp(c001, c101, fx)
    c11 = lerp(c011, c111, fx)
    c0 = lerp(c00, c10, fy)
    c1 = lerp(c01, c11, fy)
    return lerp(c0, c1, fz)

def sample_1d_texture(tex, uv):
    # 1D texture lookup
    idx = floor(uv.x * (len(tex) - 1))
    idx = clamp(idx, 0, len(tex)-1)
    return tex[idx]
```

## Subtopics

### [Ray Marching Volume Rendering with Geometry Occlusion](2066-ray-marching-volume-rendering-with-geometry-occlusion.md)
A front-to-back ray marching technique that renders a 3D volume texture by stepping along rays from the camera, compositing color and opacity, and using pre-rendered opaque geometry to occlude the volume and adjust sampling rates. It solves the problem of self-occlusion in volume rendering without requiring full volumetric integration or multiple rendering passes. It is suitable for real-time applications where geometry occlusion is handled by the parent method.

### [Volumetric Ray Marching for Fire](2578-volumetric-ray-marching-for-fire.md)
Volumetric Ray Marching for Fire is a GPU-friendly technique that samples a 3D volume along a ray to render fire simulation results stored in 3D textures. It distinguishes itself from smoke rendering by accumulating values based on a reaction coordinate (fuel) rather than blending density values, using an artist-defined color gradient to map these coordinates to flame colors.

### [Raycasting with Transfer Function and Shading](3773-raycasting-with-transfer-function-and-shading.md)
This method performs direct volume rendering by marching a ray through a 3D scalar field (volume texture) and accumulating color and opacity using front-to-back compositing. It distinguishes itself from basic raycasting by integrating a 1D transfer function for material mapping and gradient-based diffuse shading for surface appearance, while using jittered stepping to reduce aliasing artifacts. It serves as the core rendering loop for real-time GPU volume visualization, building on the discretization and ray setup provided by [Ray Marching Volume Rendering](1735-ray-marching-volume-rendering.md).

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of pixels and $S$ is the number of steps per ray [S1, S8, unofficial].
*   **Memory Complexity:** $O(V)$ for the volume texture, plus $O(V)$ for optional light density fields or hash tables [S1, S6, unofficial].
*   **Performance:** Up to 66 fps on an NVIDIA 3090 GPU at 800x800 resolution [S6, unofficial].
*   **Strengths:** Real-time performance on modern GPUs; supports complex lighting, shadows, and refraction; no need for mesh extraction [S1, S6, unofficial].
*   **Limitations:** Fixed step size can cause aliasing (wood-grain artifacts) at low sample counts, mitigated by stochastic jittering [S2, unofficial]. Requires tuning of step size and transfer functions [S10].
*   **When to Use:** Real-time visualization of smoke, clouds, fluids, and medical data [S1, S2, unofficial].
*   **When Not to Use:** Not given in the sources.

## Background
Ray Marching Volume Rendering is a fundamental technique in computer graphics for visualizing volumetric data [S4, unofficial]. It is based on the volume rendering integral, which approximates the light transport through a participating medium [S4, S8, unofficial]. The method has been widely implemented in various frameworks, including OpenGL shaders [S3, S5, S9, unofficial], CUDA engines [S7, unofficial], and high-level frameworks like Taichi for real-time applications [S1, S6, unofficial]. The Taichi NGP Renderer specifically adapts the Instant-NGP approach for volume rendering, using hash encoding and fully fused MLPs for density and color prediction, enabling real-time rendering of complex scenes [S6, unofficial]. The technique is commonly used in scientific visualization, visual effects, and real-time fluid simulation [S1, S2, unofficial].

## Sources

[S1] ShengzhiWu/taichi-volume-renderer, https://github.com/ShengzhiWu/taichi-volume-renderer, fetched 2026-10-26 (unofficial)
[S2] tzhazuma/Fluid_simulation, https://github.com/tzhazuma/Fluid_simulation, fetched 2026-10-26 (unofficial)
[S3] JaneliaSciComp/web-vol-viewer, https://github.com/JaneliaSciComp/web-vol-viewer, fetched 2026-10-26 (unofficial)
[S4] 1vx-437312114/webgl_volume_renderer, https://github.com/1vx-437312114/webgl_volume_renderer, fetched 2026-10-27 (unofficial)
[S5] Volumetric Fog Using 3D Textures in OpenGL - opengl, https://salivity.github.io/opengl/article/volumetric-fog-using-3-d-textures-in-open-gl, fetched 2026-10-27 (unofficial)
[S6] Linyou/taichi-ngp-renderer, https://github.com/Linyou/taichi-ngp-renderer, fetched 2026-10-27 (unofficial)
[S7] Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, https://github.com/Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, fetched 2026-10-27 (unofficial)
[S8] Volumetric Cloud Rendering - Jack Tollenaar, https://www.jacktollenaar.top/articles/clouds.html, fetched 2026-10-29 (unofficial)
[S9] Volumetric Clouds with 3D Perlin-Worley Noise in OpenGL, https://salivity.github.io/opengl/article/volumetric-clouds-with-3-d-perlin-worley-noise-in-open-gl, fetched 2026-10-29 (unofficial)
