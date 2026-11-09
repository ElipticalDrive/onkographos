# Raycasting with Style Transfer and Contour Lines

Also called: FGC_T3 Raycasting, Style-based Volume Raycasting

## Overview
This method extends direct volume raycasting by integrating artistic style transfer and illustrative contour lines to produce non-photorealistic visualizations [S1, unofficial]. It differs from standard raycasting by mapping surface normals and material indices to a 2D array texture to apply artistic styles, while using curvature and view-angle approximations to render contour lines [S1, unofficial].

## How it works
The method operates on the fundamental loop structure of [Image-order rendering](16-image-order-rendering.md): for each pixel, a ray $r(t) = \mathbf{o} + t\mathbf{d}$ is defined, intersected with the volume domain, and sampled/composited along its path [G]. The traversal order is determined by the pixel's position in the image plane [G].

### Inputs and Outputs
**Inputs:**
*   **Volume Data:** A 3D texture (`VolumeTex`) storing scalar density values [S1, unofficial].
*   **Entry Points:** A 2D texture (`ExitPoints`) storing the 3D exit coordinates for each pixel, derived from a prior pass [S1, unofficial].
*   **Transfer Function:** A 1D texture (`transferFunctionTexture`) mapping density to opacity (green channel) and material index (red channel) [S1, unofficial].
*   **Index Function:** A 1D texture (`indexFunctionTexture`) mapping material indices to style indices [S1, unofficial].
*   **Style Texture:** A 2D array texture (`styleTransferTexture`) containing multiple artistic styles, indexed by the style index [S1, unofficial].
*   **Uniforms:**
    *   `NormalMatrix`: $4 \times 4$ matrix for transforming normals [S1, unofficial].
    *   `StepSize`: Floating-point step length along the ray (default 0.01) [S1, unofficial].
    *   `Threshold`: Density threshold for sampling (default 0.15) [S1, unofficial].
    *   `ScreenSize`: Resolution of the output image [S1, unofficial].
    *   `AVAILABLE_STYLE_COUNT`: Integer count of styles in the array texture (value 34) [S1, unofficial].

**Outputs:**
*   **FragColor:** The final composited color and opacity for the pixel [S1, unofficial].

### Data Structures
*   **EntryPoint:** `vec3` representing the camera origin or entry point into the volume for the current pixel [S1, unofficial].
*   **ExitPointCoord:** `vec4` used to sample the exit point from the `ExitPoints` texture [S1, unofficial].
*   **Accumulators:**
    *   `pos`: `vec3` current ray position [S1, unofficial].
    *   `dst`: `vec4` accumulated destination color and opacity [S1, unofficial].
    *   `normal`: `vec3` current surface normal [S1, unofficial].
    *   `baseColor`: `vec4` color sampled from the style texture [S1, unofficial].
    *   `src`: `vec4` source color with opacity for compositing [S1, unofficial].

### Steps
1.  **Retrieve Exit Point:** Sample the exit point from the `ExitPoints` texture using normalized fragment coordinates:
    $$ \mathbf{p}_{exit} = \text{texture}(\text{ExitPoints}, \frac{\mathbf{gl\_FragCoord.st}}{\text{ScreenSize}}).xyz $$
2.  **Discard Background:** If the entry point equals the exit point, discard the fragment [S1, unofficial].
3.  **Ray Setup:** Calculate ray direction $\mathbf{d} = \mathbf{p}_{exit} - \mathbf{p}_{entry}$ and length $L = \|\mathbf{d}\|$. Compute step vector $\mathbf{s} = \text{StepSize} \cdot \frac{\mathbf{d}}{L}$ [S1, unofficial].
4.  **Initialize:** Set ray start position $\mathbf{pos} = \mathbf{p}_{entry}$. If jittering is enabled, add noise: $\mathbf{pos} += \mathbf{s} \cdot \text{snoise}(\frac{\mathbf{gl\_FragCoord.xy}}{2.5})$ [S1, unofficial]. Initialize accumulators: $\mathbf{dst} = (0,0,0,0)$, $\mathbf{normal} = (1,0,0)$, $\mathbf{baseColor} = (0,0,0,0)$, $\mathbf{src} = (0,0,0,0)$ [S1, unofficial].
5.  **Ray Marching Loop:** While $\mathbf{dst}.a < 1.0$ and $L > 0$:
    a.  **Sample Density:** $v = \text{texture}(\text{VolumeTex}, \mathbf{pos}).x$ [S1, unofficial].
    b.  **Threshold Check:** If `USE_THRESHOLD` is defined and $v \le \text{Threshold}$, skip to step 5g [S1, unofficial].
    c.  **Transfer Function:** Sample opacity $\alpha = \text{texture}(\text{transferFunctionTexture}, v).g$ and material index $idx = \text{texture}(\text{transferFunctionTexture}, v).r$ [S1, unofficial].
    d.  **Style Index:** Determine style index $k = \lfloor \text{texture}(\text{indexFunctionTexture}, idx).x \cdot \text{AVAILABLE\_STYLE\_COUNT} \rfloor$ [S1, unofficial].
    e.  **Normal Estimation:** Compute gradient $\mathbf{g}$ using central differences with step size `StepSize` $L_s$:
        $$ \mathbf{g} = \begin{bmatrix} \text{texture}(\text{VolumeTex}, \mathbf{pos} + (L_s, 0, 0)).x - v \\ \text{texture}(\text{VolumeTex}, \mathbf{pos} + (0, L_s, 0)).x - v \\ \text{texture}(\text{VolumeTex}, \mathbf{pos} + (0, 0, L_s)).x - v \end{bmatrix} $$
        Transform normal: $\mathbf{n}_{new} = \text{NormalMatrix} \cdot \begin{bmatrix} \mathbf{g} \\ 0 \end{bmatrix}$.xyz [S1, unofficial].
    f.  **Style and Contour Application:**
        i.  **LitSphere Mapping:** Map normal to 2D coordinates: $\mathbf{uv} = (\mathbf{n}_{new}.x, -\mathbf{n}_{new}.y) \cdot 0.5 + 0.5$ [S1, unofficial].
        ii. **Base Style:** Sample style texture: $\mathbf{baseColor} = \text{texture}(\text{styleTransferTexture}, \mathbf{uv}, k)$ [S1, unofficial].
        iii. **Contour Check:** If `USE_CONTOUR` is defined:
            *   Compute curvature approximation: $\kappa = \arccos\left(\frac{\mathbf{n}_{new} \cdot \mathbf{normal}}{\|\mathbf{n}_{new}\| \|\mathbf{normal}\|}\right) \cdot L_s$ [S1, unofficial].
            *   Compute contour condition: $T_{kv} = \text{thickness} \cdot \kappa$, $C = \sqrt{T_{kv}(2 - T_{kv})}$ (where thickness=1.0) [S1, unofficial].
            *   Compute view dot product: $n_v = |\mathbf{n}_{new} \cdot \frac{-\mathbf{pos}}{\|\mathbf{pos}\|}|$ [S1, unofficial].
            *   If $n_v \le C$, apply matcap style:
                *   $\Delta = 1.0 - \min(1.0, \frac{C - n_v}{C})$ [S1, unofficial].
                *   $\mathbf{uv}_{matcap} = \text{matcap}(\mathbf{pos}, \mathbf{n}_{new}).xy$ [S1, unofficial].
                *   $\mathbf{baseColor} = \text{texture}(\text{styleTransferTexture}, \mathbf{uv}_{matcap}, k)$ [S1, unofficial].
    g.  **Composite:**
        i.  $\mathbf{src} = (\mathbf{baseColor}.rgb, \alpha)$ [S1, unofficial].
        ii. Pre-multiply alpha: $\mathbf{src}.rgb \cdot= \mathbf{src}.a$ [S1, unofficial].
        iii. Front-to-back compositing: $\mathbf{dst} = (1.0 - \mathbf{dst}.a)\mathbf{src} + \mathbf{dst}$ [S1, unofficial].
    h.  **Advance:** $\mathbf{pos} += \mathbf{s}$, $L -= \text{StepSize}$ [S1, unofficial].
6.  **Output:** Set `FragColor = dst` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def raycast_with_style_transfer_and_contours(
    entry_point: vec3,
    exit_points_texture: Texture2D,
    volume_texture: Texture3D,
    transfer_function_texture: Texture1D,
    index_function_texture: Texture1D,
    style_transfer_texture: Texture2DArray,
    normal_matrix: mat4,
    screen_size: vec2,
    step_size: float = 0.01,
    threshold: float = 0.15,
    available_style_count: int = 34,
    use_threshold: bool = True,
    use_contour: bool = True,
    use_noise_jitter: bool = False
) -> vec4:
    """
    Performs volume raycasting with style transfer and contour lines.
    """
    # 1. Retrieve Exit Point
    normalized_coord = vec2(gl_frag_coord_st / screen_size)
    exit_point = exit_points_texture.sample(normalized_coord).xyz
    
    # 2. Discard Background
    if entry_point == exit_point:
        return vec4(0.0) # Discard

    # 3. Ray Setup
    ray_direction = exit_point - entry_point
    ray_length = length(ray_direction)
    step_vector = step_size * ray_direction / ray_length
    
    # 4. Initialize
    pos = entry_point
    if use_noise_jitter:
        # snoise is a simplex noise function
        jitter = snoise(normalized_coord / 2.5)
        pos += step_vector * jitter
        
    dst = vec4(0.0)
    normal = vec3(1.0, 0.0, 0.0)
    base_color = vec4(0.0)
    src = vec4(0.0)
    
    # 5. Ray Marching Loop
    while dst.a < 1.0 and ray_length > 0.0:
        # a. Sample Density
        density = volume_texture.sample(pos).x
        
        # b. Threshold Check
        if use_threshold and density <= threshold:
            pos += step_vector
            ray_length -= step_size
            continue
            
        # c. Transfer Function
        tf_sample = transfer_function_texture.sample(density)
        opacity = tf_sample.g
        index = tf_sample.r
        
        # d. Style Index
        idx_sample = index_function_texture.sample(index)
        style_index = int(idx_sample.x * available_style_count)
        
        # e. Normal Estimation
        # Central differences with step_size
        L = step_size
        look_up = density
        grad_x = volume_texture.sample(pos + vec3(L, 0, 0)).x - look_up
        grad_y = volume_texture.sample(pos + vec3(0, L, 0)).x - look_up
        grad_z = volume_texture.sample(pos + vec3(0, 0, L)).x - look_up
        gradient = vec3(grad_x, grad_y, grad_z)
        
        previous_normal = normal
        normal = (normal_matrix * vec4(gradient, 0.0)).xyz
        
        # f. Style and Contour Application
        # LitSphere Mapping
        litsphere_uv = vec2(normal.x, -normal.y) * 0.5 + 0.5
        base_color = style_transfer_texture.sample(litsphere_uv, style_index)
        
        if use_contour:
            # Curvature approximation
            magnitudes = length(normal) * length(previous_normal)
            if magnitudes > 0.0:
                curvature = acos(dot(normal, previous_normal) / magnitudes) * step_size
            else:
                curvature = 0.0
                
            thickness = 1.0
            t_kv = thickness * curvature
            cond = sqrt(t_kv * (2.0 - t_kv))
            
            # View dot product
            view_dir = normalize(-pos)
            n_dot_v = abs(dot(normal, view_dir))
            
            if n_dot_v <= cond:
                lit_delta = 1.0 - min(1.0, (cond - n_dot_v) / cond)
                # matcap is a helper function mapping pos/normal to UV
                matcap_uv = matcap(pos, normal).xy
                base_color = style_transfer_texture.sample(matcap_uv, style_index)
                
        # g. Composite
        src = vec4(base_color.rgb, opacity)
        src.rgb *= src.a
        dst = (1.0 - dst.a) * src + dst
        
        # h. Advance
        pos += step_vector
        ray_length -= step_size
        
    return dst
```

### Parameters
*   **StepSize:** Distance between samples. Default: 0.01 [S1, unofficial].
*   **Threshold:** Density value below which sampling is skipped. Default: 0.15 [S1, unofficial].
*   **AVAILABLE_STYLE_COUNT:** Number of styles in the style transfer texture. Value: 34 [S1, unofficial].

### Edge Cases and Pitfalls
*   **Background Detection:** If the entry point equals the exit point, the fragment is discarded to avoid rendering background [S1, unofficial].
*   **Normal Magnitude:** The curvature calculation divides by the product of normal magnitudes. If magnitudes are zero, curvature is set to 0.0 [G].
*   **Precision:** The curvature approximation uses `acos`, which is sensitive to floating-point errors when normals are nearly parallel. The condition `magnitudes > 0.0` guards against division by zero [G].
*   **Boundary Conditions:** The gradient calculation assumes the volume texture is sampled with appropriate border handling or that the ray does not step outside the volume bounds during gradient estimation [G].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples per ray $N$. Each sample involves texture lookups and vector operations.
*   **Memory Complexity:** Depends on the size of the volume texture, transfer function, index function, and style transfer texture. The style transfer texture is a 2D array with 34 layers [S1, unofficial].
*   **Strengths:** Produces illustrative, artistic visualizations with contour lines that highlight surface features and material boundaries [S1, unofficial].
*   **Limitations:** Computationally more expensive than standard raycasting due to additional texture lookups (style, index, transfer) and normal estimation [S1, unofficial]. The quality of contours depends on the step size and curvature approximation [S1, unofficial].

## Background
The method is implemented in the `FGC_T3` project, an open-source repository for illustrative volume rendering [S1, unofficial]. It integrates style transfer techniques to map 3D volume data to 2D artistic styles based on surface normals and material indices [S1, unofficial].

## Sources
[S1] jose-villegas/IllustrativeVolumeRendering, https://github.com/jose-villegas/IllustrativeVolumeRendering, fetched 2026-10-26 (unofficial)
