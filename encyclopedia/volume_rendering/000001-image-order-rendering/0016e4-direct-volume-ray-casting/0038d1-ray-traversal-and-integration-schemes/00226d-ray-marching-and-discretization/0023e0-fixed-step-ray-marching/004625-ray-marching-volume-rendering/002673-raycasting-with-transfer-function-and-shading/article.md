# Raycasting with Transfer Function and Shading

Also called: Front-to-back volume raycasting with gradient-based diffuse shading

## Overview
This method performs direct volume ray casting using a fragment shader to render a volume defined by 3D textures. It combines front-to-back compositing with a 1D transfer function for color/opacity mapping and gradient-based diffuse shading for surface appearance. It distinguishes itself from basic ray marching by explicitly computing surface normals from pre-stored gradients to apply Lambertian lighting, and by using jittered stepping to reduce banding artifacts. It builds on [Ray Marching Volume Rendering](4877-ray-marching-volume-rendering.md) by replacing procedural noise carving with explicit 3D texture lookups and adding explicit lighting calculations.

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `position` (vec4): Clip-space position of the fragment.
    *   `cubeFront` (sampler2D): Front face of the volume bounding box in clip space.
    *   `cubeBack` (sampler2D): Back face of the volume bounding box in clip space.
    *   `volume` (sampler3D): The scalar field data.
    *   `gradients` (sampler3D): Pre-computed gradient vectors, encoded as 2D vectors.
    *   `transferFunction` (sampler1D): Maps scalar values to RGBA.
    *   `bakedNoise` (sampler2D): 2D noise texture for jittering.
    *   `ciModelMatrixInverseTranspose` (mat3): Model matrix inverse transpose for normal transformation.
    *   `light` (struct): Contains `direction` (vec3), `ambient` (vec3), and `diffuse` (vec3).
    *   `threshold` (ivec2): Range of iso-values to render.
    *   `stepSize` (vec3): Step size per dimension.
    *   `iterations` (int): Maximum ray steps.
    *   `diffuseShading` (bool): Enable/disable lighting.
    *   `stepScale` (float): Scaling factor for opacity correction.
*   **Outputs:**
    *   `oColor` (vec4): Final color.
    *   `oNormal` (vec3): Surface normal at the hit point.
    *   `gl_FragDepth`: Depth value.

**Data Structures:**
*   **Encoded Normal:** Stored in `gradients` as a `vec2`. The z-component is reconstructed using the spheremap transform [S1, unofficial].
*   **Transfer Function:** 1D texture where the x-coordinate corresponds to the normalized scalar value (0.0 to 1.0).

**Steps:**
1.  **Ray Setup:**
    *   Convert fragment position to texture coordinates: $texC_x = 0.5 \frac{position_x}{position_w} + 0.5$, $texC_y = 0.5 \frac{position_y}{position_w} - 0.5$.
    *   Sample front and back intersection points from `cubeFront` and `cubeBack` using `texC`.
    *   Compute ray direction: $\vec{dir} = \text{normalize}(\vec{back} - \vec{front})$.
    *   Initialize ray position $\vec{pos} = \vec{front}$ with $w=0$.
    *   Initialize accumulators: $\vec{dst} = \vec{0}$ (destination color/alpha), $\vec{src} = \vec{0}$ (source color/alpha), $\vec{value} = \vec{0}$ (current sample).
    *   Compute step vector: $\vec{step} = \vec{dir} \times \vec{stepSize}$.
    *   Jitter start position: $\vec{pos}_{xyz} += \vec{step} \times \text{bakedNoise}(gl\_FragCoord.xy / 256).x$.

2.  **Ray Marching Loop:**
    *   Iterate $i$ from 0 to $iterations - 1$.
    *   Set $\vec{pos}_w = 0$.
    *   Sample volume: $v_a = \text{texture}(volume, \vec{pos}_{xyz}).x$.
    *   Compute integer iso-value: $iso = \lfloor 255.0 \times v_a \rfloor$.
    *   **Threshold Check:** If $iso \ge threshold.x$ and $iso \le threshold.y$:
        *   Sample transfer function: $\vec{src} = \text{texture}(transferFunction, v_a)$.
        *   **Opacity Correction:** Adjust opacity for step size: $src_a = 1 - (1 - src_a)^{\frac{stepScale}{0.5}}$.
        *   **Shading (if enabled):**
            *   Sample encoded gradient: $\vec{enc} = \text{texture}(gradients, \vec{pos}_{xyz}).xy$.
            *   Decode normal:
                *   $n_z = \vec{enc} \cdot \vec{enc} \times 2.0 - 1.0$.
                *   $n_{xy} = \text{normalize}(\vec{enc}_{xy}) \times \sqrt{1.0 - n_z^2}$.
                *   $\vec{n}_{local} = (n_{xy}.x, n_{xy}.y, n_z)$.
            *   Transform normal: $\vec{n}_{world} = \text{normalize}(ciModelMatrixInverseTranspose \times \vec{n}_{local})$.
            *   Compute light direction: $\vec{l} = \text{normalize}(-\vec{light}.direction)$.
            *   Compute Lambertian factor: $L = \max(\vec{n}_{world} \cdot \vec{l}, 0.0)$.
            *   Apply shading: $\vec{src}_{rgb} = \vec{light}.diffuse \times L \times \vec{src}_{rgb} + \vec{light}.ambient \times \vec{src}_{rgb}$.
        *   **Front-to-Back Blending:**
            *   $\vec{src}_{rgb} \times src_a$.
            *   $\vec{dst} = (1.0 - dst_a) \times \vec{src} + \vec{dst}$.
        *   **Early Termination:** If $dst_a \ge 0.95$, break loop.
    *   **Advance Position:** $\vec{pos}_{xyz} += \vec{step}$.
    *   **Boundary Check:** If $\vec{pos}_x > 1.0$ or $\vec{pos}_y > 1.0$ or $\vec{pos}_z > 1.0$, break loop.

3.  **Output:**
    *   Set `oColor` = $(\vec{dst}_{rgb}, 1.0)$.
    *   Set `oNormal` = $\vec{value}_{xyz}$ (last sampled gradient/normal).
    *   Set `gl_FragDepth` = $\vec{pos}_z$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def raycast_with_tf_and_shading(position, cube_front, cube_back, volume, gradients, transfer_function, baked_noise, ci_model_matrix_inverse_transpose, light, threshold, step_size, iterations, diffuse_shading, step_scale):
    # Ray Setup
    tex_c_x = 0.5 * (position.x / position.w) + 0.5
    tex_c_y = 0.5 * (position.y / position.w) - 0.5
    
    front = texture_2d(cube_front, vec2(tex_c_x, tex_c_y))
    back = texture_2d(cube_back, vec2(tex_c_x, tex_c_y))
    
    dir = normalize(back.xyz - front.xyz)
    pos = vec4(front.xyz, 0.0)
    dst = vec4(0.0)
    src = vec4(0.0)
    value = vec4(0.0)
    
    step = dir * step_size
    
    # Jitter start position
    noise_val = texture_2d(baked_noise, gl_frag_coord.xy / 256.0).x
    pos.xyz += step * noise_val
    
    for i in range(iterations):
        pos.w = 0.0
        
        # Sample volume
        value_a = texture_3d(volume, pos.xyz).x
        iso_value = int(255.0 * value_a)
        
        # Threshold check
        if threshold.x <= iso_value <= threshold.y:
            # Sample transfer function
            src = texture_1d(transfer_function, value_a)
            
            # Opacity correction
            src.a = 1.0 - pow(1.0 - src.a, step_scale / 0.5)
            
            # Shading
            if diffuse_shading:
                enc = texture_3d(gradients, pos.xyz).xy
                
                # Decode normal (Spheremap transform)
                n_z = dot(enc, enc) * 2.0 - 1.0
                n_xy = normalize(enc.xy) * sqrt(1.0 - n_z * n_z)
                n_local = vec3(n_xy.x, n_xy.y, n_z)
                
                # Transform normal
                normal = normalize(ci_model_matrix_inverse_transpose * n_local)
                
                # Light calculation
                light_dir = normalize(-light.direction)
                lambert = max(dot(normal, light_dir), 0.0)
                
                # Apply shading
                src.rgb = light.diffuse * lambert * src.rgb + light.ambient * src.rgb
            
            # Front-to-back blending
            src.rgb *= src.a
            dst = (1.0 - dst.a) * src + dst
            
            # Early termination
            if dst.a >= 0.95:
                break
        
        # Advance position
        pos.xyz += step
        
        # Boundary check
        if pos.x > 1.0 or pos.y > 1.0 or pos.z > 1.0:
            break
            
    o_color = vec4(dst.rgb, 1.0)
    o_normal = value.xyz
    gl_frag_depth = pos.z
    
    return o_color, o_normal, gl_frag_depth
```

## Subtopics

### [Gradient-Based Volume Rendering with Isovalue Lookup](10998-gradient-based-volume-rendering-with-isovalue-lookup.md)
This method performs direct volume ray casting by storing pre-computed integer isovalue and gradient magnitude data in a 3D texture, allowing the fragment shader to sample surface properties directly without numerical differentiation. It distinguishes itself from standard raycasting by weighting the transfer function opacity with the gradient magnitude, which enhances edges and surfaces, and by using a lookup-based approach for lighting and visibility selection. It builds on [Raycasting with Transfer Function and Shading](3773-raycasting-with-transfer-function-and-shading.md) by replacing on-the-fly gradient computation with pre-sampled gradient magnitudes and adding isovalue-based selection logic.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of iterations (ray steps) per pixel. $O(N \times M)$ where $N$ is pixel count and $M$ is iterations.
*   **Memory Complexity:** Depends on the size of the 3D volume texture, gradient texture, and transfer function. The fragment shader state is constant.
*   **Performance:** The method is GPU-friendly as it runs entirely in the fragment shader. However, it requires multiple texture lookups per step (volume, gradients, transfer function, noise).
*   **Strengths:** Produces high-quality images with smooth shading and accurate lighting. The jittering reduces banding. The thresholding allows for selective rendering of specific density ranges.
*   **Limitations:** Computationally expensive due to multiple texture samples per step. The fixed step size may lead to aliasing or missed details if not tuned correctly. The spheremap normal encoding has singularities at the poles, though this is mitigated by the normalization step.
*   **When to use:** When high-quality, shaded volume rendering is required and the volume data is available as 3D textures. Suitable for medical imaging, scientific visualization, and artistic volume rendering.
*   **When not to use:** For real-time applications with very large volumes or limited GPU memory, as the multiple texture lookups can be a bottleneck. For volumes where lighting is not important, simpler methods may be faster.

## Background
This method is implemented in the `VolumeRendering` project by Jose Villegas [S1, unofficial]. It utilizes techniques common in modern GPU-based volume rendering, such as front-to-back compositing, transfer functions, and gradient-based shading. The normal decoding technique uses a spheremap transform, attributed to Cry Engine 3 and Martin Mittring [S1, unofficial]. The implementation uses Cinder and ImGui for the application framework and UI.

## Sources
[S1] jose-villegas/VolumeRendering, https://github.com/jose-villegas/VolumeRendering, fetched 2026-10-27 (unofficial)
