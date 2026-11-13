# Raycasting with Style Transfer and Contour Lines

Also called: FGC_T3 Raycasting, Style-based Volume Raycasting

## Overview
Raycasting with Style Transfer and Contour Lines is a direct volume rendering technique that applies artistic styles to volumetric data based on surface normals and material indices, while optionally rendering contour lines to emphasize structural features [S1, unofficial]. It distinguishes itself from standard direct volume raycasting by replacing fixed color transfer functions with a Style Transfer Texture lookup driven by normal-to-UV mapping, and from basic illustrative rendering by integrating curvature-based contour detection directly into the raycasting loop [S1, 11409-raycasting-with-style-transfer-and-contour-lines.md, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**: A 3D volume texture (`VolumeTex`), an `EntryPoint` texture (storing front entry points per pixel), an `ExitPoints` texture (storing back exit points per pixel), a 1D `transferFunctionTexture` (mapping density to opacity and index), a 1D `indexFunctionTexture` (mapping transfer function index to style index), and a 2D array `styleTransferTexture` (containing $N$ style images, typically lit spheres or matcaps).
*   **Outputs**: A final color buffer (`FragColor`).

**Data Structures**
*   `VolumeTex`: 3D texture of density values.
*   `ExitPoints`: 2D texture where each texel stores a `vec3` exit point.
*   `EntryPoint`: 2D texture where each texel stores a `vec3` entry point.
*   `transferFunctionTexture`: 1D texture. Red channel stores material index, Green channel stores opacity.
*   `indexFunctionTexture`: 1D texture. Red channel maps material index to normalized style index $[0, 1)$.
*   `styleTransferTexture`: 2D array texture. Dimensions are $W \times H \times N$, where $N$ is the number of styles.

**Parameters**
*   `StepSize`: Distance between samples along the ray. Default: $0.01$ [S1, unofficial].
*   `Threshold`: Minimum density value to process. Default: $0.15$ [S1, unofficial].
*   `AVAILABLE_STYLE_COUNT`: Number of styles in the array texture. Value: $34$ [S1, unofficial].
*   `USE_NOISE_JITTER`: Boolean flag for jittering ray start to reduce banding [S1, unofficial].
*   `USE_CONTOUR`: Boolean flag to enable contour line rendering [S1, unofficial].
*   `USE_THRESHOLD`: Boolean flag to enable density thresholding [S1, unofficial].

**Algorithm Steps**

1.  **Ray Initialization**:
    *   Retrieve the fragment's exit point from the `ExitPoints` texture using normalized screen coordinates: $\mathbf{p}_{exit} = \text{texture}(\text{ExitPoints}, \frac{\mathbf{gl\_FragCoord.st}}{\text{ScreenSize}}).xyz$.
    *   Retrieve the entry point $\mathbf{p}_{entry}$ from the `EntryPoint` texture.
    *   If $\mathbf{p}_{entry} == \mathbf{p}_{exit}$, discard the fragment (background) [S1, unofficial].
    *   Compute ray direction $\mathbf{d} = \mathbf{p}_{exit} - \mathbf{p}_{entry}$ and length $L = \|\mathbf{d}\|$.
    *   Compute step vector $\mathbf{s} = \text{StepSize} \cdot \frac{\mathbf{d}}{L}$.
    *   Initialize ray position $\mathbf{pos} = \mathbf{p}_{entry}$.
    *   If `USE_NOISE_JITTER` is enabled, jitter the start position: $\mathbf{pos} \leftarrow \mathbf{pos} + \mathbf{s} \cdot \text{snoise}(\frac{\mathbf{gl\_FragCoord.xy}}{2.5})$, where `snoise` is a simplex noise function [S1, unofficial].

2.  **Accumulator Initialization**:
    *   Initialize destination color $\mathbf{c}_{dst} = (0, 0, 0, 0)$.
    *   Initialize current normal $\mathbf{n} = (1, 0, 0)$ [G] (arbitrary initial value, overwritten in first iteration).
    *   Initialize previous normal $\mathbf{n}_{prev} = \mathbf{n}$ [G].
    *   Initialize base style color $\mathbf{c}_{base} = (0, 0, 0, 0)$.

3.  **Raycasting Loop**:
    While $\mathbf{c}_{dst}.a < 1.0$ and $L > 0$:
    *   Sample density $D = \text{texture}(\text{VolumeTex}, \mathbf{pos}).x$.
    *   If `USE_THRESHOLD` is enabled and $D \le \text{Threshold}$, skip to next step.
    *   Sample opacity $\alpha$ and material index $I_{mat}$ from `transferFunctionTexture` at $D$:
        $$ \alpha = \text{texture}(\text{transferFunctionTexture}, D).g $$
        $$ I_{mat} = \text{texture}(\text{transferFunctionTexture}, D).r $$
    *   Determine style index $k$:
        $$ k = \lfloor \text{texture}(\text{indexFunctionTexture}, I_{mat}).r \cdot \text{AVAILABLE\_STYLE\_COUNT} \rfloor $$
    *   Compute gradient $\mathbf{g}$ at $\mathbf{pos}$ using central differences or forward differences with step `StepSize` $h$:
        $$ \mathbf{g} = \left( \text{texture}(\text{VolumeTex}, \mathbf{pos} + (h, 0, 0)).x - D, \quad \text{texture}(\text{VolumeTex}, \mathbf{pos} + (0, h, 0)).x - D, \quad \text{texture}(\text{VolumeTex}, \mathbf{pos} + (0, 0, h)).x - D \right) $$
        *Note: The source code uses forward differences relative to the current density $D$ [S1, unofficial].*
    *   Transform gradient to normal space using `NormalMatrix` $\mathbf{M}_N$:
        $$ \mathbf{n}_{new} = \text{normalize}((\mathbf{M}_N \cdot (\mathbf{g}, 0)).xyz) $$
    *   Store previous normal: $\mathbf{n}_{prev} = \mathbf{n}$.
    *   Update current normal: $\mathbf{n} = \mathbf{n}_{new}$.
    *   Map normal to UV coordinates for lit sphere style:
        $$ \mathbf{uv} = (\mathbf{n}.x, -\mathbf{n}.y) \cdot 0.5 + 0.5 $$
    *   Sample base color from style transfer texture:
        $$ \mathbf{c}_{base} = \text{texture}(\text{styleTransferTexture}, (\mathbf{uv}, k)) $$
    *   **Contour Line Calculation** (if `USE_CONTOUR` is enabled):
        *   Compute curvature approximation $\kappa$:
            $$ \kappa = \arccos\left(\frac{\mathbf{n} \cdot \mathbf{n}_{prev}}{\|\mathbf{n}\| \|\mathbf{n}_{prev}\|}\right) \cdot \text{StepSize} $$
        *   Compute contour condition threshold $T_{kv}$:
            $$ T_{kv} = \text{thickness} \cdot \kappa $$
            where `thickness` is a constant (default $1.0$) [S1, unofficial].
        *   Compute contour mask value $C$:
            $$ C = \sqrt{T_{kv} \cdot (2.0 - T_{kv})} $$
        *   Compute view dot product $V_{dot}$:
            $$ V_{dot} = |\mathbf{n} \cdot \frac{-\mathbf{pos}}{\|\mathbf{pos}\|}| $$
            *Note: The source code uses `normalize(-pos)` which assumes the camera is at the origin or `pos` is in view space [S1, unofficial].*
        *   If $V_{dot} \le C$:
            *   Compute lighting delta: $\Delta L = 1.0 - \min(1.0, \frac{C - V_{dot}}{C})$.
            *   Adjust normal length: $L_{adj} = \min(1.0, \frac{\|\mathbf{n}\|}{\Delta L})$.
            *   Sample matcap style instead of lit sphere:
                $$ \mathbf{uv}_{matcap} = \text{matcap}(\mathbf{pos}, \mathbf{n}).xy $$
                $$ \mathbf{c}_{base} = \text{texture}(\text{styleTransferTexture}, (\mathbf{uv}_{matcap}, k)) $$
    *   **Compositing**:
        *   Set source color with opacity: $\mathbf{c}_{src} = (\mathbf{c}_{base}.rgb \cdot \alpha, \alpha)$.
        *   Pre-multiply alpha: $\mathbf{c}_{src}.rgb \leftarrow \mathbf{c}_{src}.rgb \cdot \mathbf{c}_{src}.a$.
        *   Composite: $\mathbf{c}_{dst} = (1.0 - \mathbf{c}_{dst}.a) \cdot \mathbf{c}_{src} + \mathbf{c}_{dst}$.
    *   Advance ray: $\mathbf{pos} \leftarrow \mathbf{pos} + \mathbf{s}$, $L \leftarrow L - \text{StepSize}$.

4.  **Output**:
    *   Write $\mathbf{c}_{dst}$ to `FragColor` [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def raycast_with_style_transfer_and_contour_lines(
    volume_tex, exit_points_tex, entry_points_tex,
    transfer_func_tex, index_func_tex, style_tex,
    normal_matrix, screen_size, step_size=0.01,
    threshold=0.15, available_style_count=34,
    use_noise_jitter=False, use_contour=False,
    use_threshold=True, thickness=1.0
):
    """
    Renders a single pixel's ray using style transfer and optional contours.
    Assumes access to GLSL-style texture sampling functions.
    """
    # 1. Ray Initialization
    exit_point = exit_points_tex.sample(gl_frag_coord.st / screen_size)
    entry_point = entry_points_tex.sample(gl_frag_coord.st / screen_size)
    
    if entry_point == exit_point:
        return vec4(0.0) # Background

    ray_direction = exit_point - entry_point
    ray_length = length(ray_direction)
    step_vector = step_size * normalize(ray_direction)
    
    ray_pos = entry_point
    if use_noise_jitter:
        # snoise is a simplex noise function
        ray_pos += step_vector * snoise(gl_frag_coord.xy / 2.5)

    # 2. Accumulators
    dst_color = vec4(0.0)
    current_normal = vec3(1.0, 0.0, 0.0) # [G] Initial arbitrary normal
    prev_normal = current_normal
    
    # 3. Raycasting Loop
    while dst_color.a < 1.0 and ray_length > 0.0:
        density = volume_tex.sample(ray_pos).x
        
        if use_threshold and density <= threshold:
            ray_pos += step_vector
            ray_length -= step_size
            continue
            
        # Sample Transfer Function
        opacity = transfer_func_tex.sample(density).g
        mat_index = transfer_func_tex.sample(density).r
        
        # Determine Style Index
        style_norm = index_func_tex.sample(mat_index).r
        style_idx = int(style_norm * available_style_count)
        
        # Compute Gradient and Normal
        h = step_size
        grad = vec3(
            volume_tex.sample(ray_pos + vec3(h, 0, 0)).x - density,
            volume_tex.sample(ray_pos + vec3(0, h, 0)).x - density,
            volume_tex.sample(ray_pos + vec3(0, 0, h)).x - density
        )
        
        # Transform normal
        normal_vec = normal_matrix * vec4(grad, 0.0)
        prev_normal = current_normal
        current_normal = normalize(normal_vec.xyz)
        
        # Map Normal to UV (Lit Sphere)
        uv = vec2(current_normal.x, -current_normal.y) * 0.5 + 0.5
        
        # Sample Base Style
        base_color = style_tex.sample(vec3(uv, style_idx))
        
        # Contour Lines
        if use_contour:
            magnitudes = length(current_normal) * length(prev_normal)
            if magnitudes > 0.0:
                curvature = acos(dot(current_normal, prev_normal) / magnitudes) * step_size
            else:
                curvature = 0.0
                
            t_kv = thickness * curvature
            cond = sqrt(t_kv * (2.0 - t_kv))
            
            # View vector assumes camera at origin in view space
            view_dir = normalize(-ray_pos)
            n_dot_v = abs(dot(current_normal, view_dir))
            
            if n_dot_v <= cond:
                lit_delta = 1.0 - min(1.0, (cond - n_dot_v) / cond)
                adjusted_len = min(1.0, length(current_normal) / lit_delta)
                
                # Matcap UV mapping (implementation of matcap() depends on specific lighting model)
                # Assuming matcap() returns UVs for a matcap texture based on pos and normal
                uv_matcap = matcap_uv(ray_pos, current_normal) 
                base_color = style_tex.sample(vec3(uv_matcap, style_idx))
                
        # Compositing
        src_color = vec4(base_color.rgb * opacity, opacity)
        src_color.rgb *= src_color.a # Pre-multiply
        dst_color = (1.0 - dst_color.a) * src_color + dst_color
        
        ray_pos += step_vector
        ray_length -= step_size
        
    return dst_color
```

## Operating characteristics
*   **Time Complexity**: $O(R \cdot S)$, where $R$ is the number of samples per ray (determined by ray length and `StepSize`) and $S$ is the cost of texture lookups and normal transformations per sample.
*   **Memory Complexity**: $O(V + T)$, where $V$ is the volume texture size and $T$ is the size of the style transfer array texture ($W \times H \times N$).
*   **Performance**: Performance is heavily dependent on the number of styles $N$ and the resolution of the style transfer textures. The curvature calculation adds a small constant overhead per sample.
*   **Strengths**: Provides high-quality illustrative rendering with minimal user effort (no manual brush strokes). Contour lines effectively highlight internal structures and boundaries.
*   **Limitations**: The forward-difference gradient estimation can be noisy. The contour detection relies on curvature approximation which may produce artifacts in low-density regions. The method assumes a specific normal-to-UV mapping (lit sphere/matcap) which may not suit all artistic styles.
*   **When to use**: When artistic, non-photorealistic visualization of volumetric data is required, especially for medical or scientific data where structural emphasis (contours) is valuable.
*   **When not to use**: When photorealism is required, or when the volume data is extremely sparse and contour lines would be too dense or noisy.

## Background
This method originates from the work on Illustrative Volume Rendering with Style Transfer, specifically the implementation known as FGC_T3 (Fuzzy Gradient Classification with Style Transfer) [S1, unofficial]. It extends the principles of style transfer from surface rendering to volume rendering by leveraging the volume gradient as a proxy for surface normals. The technique was developed to allow artists and scientists to apply pre-defined artistic styles (like lit spheres or matcaps) to volumetric datasets dynamically.

## Sources
[S1] jose-villegas/IllustrativeVolumeRendering, https://github.com/jose-villegas/IllustrativeVolumeRendering, fetched 2026-10-26 (unofficial)
