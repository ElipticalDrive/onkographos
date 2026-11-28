# Volume Fragment Shader (Direct Ray Marching)

Also called: Fragment Shader Implementation, Ray-box volume rendering, Volume Rendering Shader

## Overview
This method implements direct volume ray casting entirely within a GPU fragment shader, computing per-pixel ray parameters from camera matrices and executing the emission-absorption compositing loop directly in the fragment shader [S2, unofficial]. It is distinguished from sibling methods by its reliance on hardware-accelerated texture sampling for trilinear interpolation and its specific handling of mask-based context visualization (ghosting) and intensity normalization within the shader logic itself [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D volume texture (`volumeTex`), 3D mask texture (`maskTex`), 1D lookup table (`lut`), 1D color transfer function (`CTF`), 1D gradient magnitude transfer function (`GMTF`), camera matrices (implicit via NDC), and UI parameters (`beta`, `surfaceVoxelsOnly`, `colorGhosting`, `roiMin`, `roiMax`, `isoValue`, `quality`, `dithering`, `rangeMin`, `rangeMax`, `gamma`) [S1, S2, unofficial].
*   **Outputs:** Final fragment color (`gl_FragColor`) [S1, unofficial].

**Data Structures:**
*   `volumeTex`: `sampler3D`, stores scalar volume values normalized to $[0, 1]$ [S2, unofficial].
*   `maskTex`: `sampler3D`, stores classification/mask data [S1, unofficial].
*   `lut`: `sampler1D`, maps volume intensity to color/opacity [S1, unofficial].
*   `CTF`: `sampler1D`, maps mask data to color [S1, unofficial].
*   `GMTF`: `sampler1D`, maps gradient magnitude to shading weight [S1, unofficial].
*   `transferTex`: `sampler2D`, used in some implementations to store 1-D LUTs as a single-row 2-D texture [S2, unofficial].

**Algorithm Steps:**

1.  **Coordinate Generation:** Compute Normalized Device Coordinates (NDC) from fragment coordinates to ensure robustness across platforms (e.g., macOS) [S2, unofficial].
2.  **Ray-AABB Intersection:**
    *   Define the bounding box (AABB) centered at the origin with half-extents `boxHalf` [S2, unofficial].
    *   Compute ray direction `rayDir` and origin `rayOrigin` from camera parameters [S2, unofficial].
    *   Calculate inverse direction: `invDir = 1.0 / rayDir`, handling zero components to avoid division-by-zero [G].
    *   Compute intersection intervals for each slab:
        $$ t_0 = (-\text{boxHalf} - \text{rayOrigin}) \times \text{invDir} $$
        $$ t_1 = (\text{boxHalf} - \text{rayOrigin}) \times \text{invDir} $$
    *   Determine $t_{\text{min}}$ and $t_{\text{max}}$:
        $$ t_{\text{min}} = \max(\min(t_0.x, t_1.x), \min(t_0.y, t_1.y), \min(t_0.z, t_1.z)) $$
        $$ t_{\text{max}} = \min(\max(t_0.x, t_1.x), \max(t_0.y, t_1.y), \max(t_0.z, t_1.z)) $$
    *   The ray intersects the box if $t_{\text{max}} > \max(t_{\text{min}}, 0.0)$ [S2, unofficial].
3.  **Volume & Mask Sampling:**
    *   Sample the volume texture at the fragment's position (`glTexCoord[0].stp`) to get $p$ [S1, unofficial].
    *   Extract gradient vector `grad` from $p.rgb$ [S1, unofficial].
    *   Sample the Lookup Table (LUT) using $p.a$ to get volume color `col_vol` [S1, unofficial].
    *   Sample the mask texture at the current fragment's texture coordinates (`glTexCoord[1].stp`) to get value $r$ [S1, unofficial].
    *   Sample the Color Transfer Function (CTF) using the alpha channel of $r$ ($r.a$) to obtain the mask color `col_mask` [S1, unofficial].
    *   Sample the Gradient Magnitude Transfer Function (GMTF) using $r.a$ to get `grad_weight` [S1, unofficial].
4.  **Intensity Normalization:** Before sampling, raw volume values are normalized:
    $$ v_{\text{norm}} = \frac{v_{\text{raw}} - \text{rangeMin}}{\text{rangeMax} - \text{rangeMin}} $$
    Clamped to $[0, 1]$. If $\text{rangeMax} - \text{rangeMin} < 10^{-6}$, return $0.0$ to avoid division by zero [S2, unofficial]. Gamma correction is applied: $v_{\text{final}} = v_{\text{norm}}^{\text{gamma}}$ [S2, unofficial].
5.  **Ray Marching & Compositing:**
    *   Initialize the output color and opacity accumulator `dest` to `vec4(0)` [S3, unofficial].
    *   Initialize the ray position at the far plane: `position = eye + ray_dir * tfar` [S3, unofficial].
    *   Execute the volume rendering loop, advancing the position along the ray direction: `position = position - ray_dir * step_size` [S3, unofficial].
    *   Implement opacity correction within the rendering loop [S3, unofficial].
    *   Update accumulated color $C'$ and opacity $\alpha'$ using front-to-back compositing:
        $$ C'_{i} = C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i C_i $$
        $$ \alpha'_{i} = \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i $$
    *   Implement Early Ray Termination to stop sampling when accumulated opacity exceeds a threshold (typically $0.95$ or $1.0$) [Parent notes, G].
6.  **Mask/Context Blending Logic:**
    *   If `col_mask.a` is $0.0$, discard the fragment [S1, unofficial].
    *   If `col_mask.a` is $1.0$: $\text{color} = \text{col\_vol} \times \beta + (1 - \beta) \times \text{col\_mask}$ [S1, unofficial].
    *   Else (partial mask): If `surfaceVoxelsOnly` is true AND the gradient magnitude is small (checked via `LargeGradient(grad, grad_weight)`): $\text{color}.a = 0.0$ [S1, unofficial].
    *   Else if `colorGhosting` is true: $\text{color} = \text{col\_mask}$ [S1, unofficial].
    *   Else: $\text{color} = \text{col\_vol} \times \beta + (1 - \beta) \times \text{col\_mask}$ [S1, unofficial].
7.  **Final Shading:** Calculate normals using central differences to enable shading [S3, unofficial]. Call the `shade` function with `color`, `grad`, and `grad_weight` to produce `gl_FragColor` [S1, unofficial].

**Dithering:** To prevent banding artifacts, a pseudo-random hash using `noiseSeed` adds a random offset to the ray start position [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_fragment(frag_coord, eye, ray_dir, volume_tex, mask_tex, lut, ctf, gmtf, beta, surface_voxels_only, color_ghosting, quality, dithering, range_min, range_max, gamma):
    # 1. Ray-AABB Intersection (Slab Method)
    box_min = vec3(-1.0)
    box_max = vec3(1.0)
    # Handle zero direction components to prevent inf/-inf propagation [G]
    inv_dir = vec3(1.0 / ray_dir.x if ray_dir.x != 0.0 else 1e30,
                   1.0 / ray_dir.y if ray_dir.y != 0.0 else 1e30,
                   1.0 / ray_dir.z if ray_dir.z != 0.0 else 1e30)
    t0 = (box_min - eye) * inv_dir
    t1 = (box_max - eye) * inv_dir
    t_near = max(min(t0.x, t1.x), min(t0.y, t1.y), min(t0.z, t1.z))
    t_far = min(max(t0.x, t1.x), max(t0.y, t1.y), max(t0.z, t1.z))
    if t_far < t_near or t_far < 0.0:
        return vec4(0.0)
    t_near = max(t_near, 0.0)
    t_near = max(t_near, -1.0) # Clamp entry distance [S3, unofficial]

    # 2. Dithering & Step Size
    dither_offset = dithering * pseudo_random_hash(frag_coord.xy)
    step_size = (t_far - t_near) * quality # Controls step count [S2, unofficial]

    # 3. Initialization
    dest = vec4(0.0) # Accumulated color and opacity [S3, unofficial]
    position = eye + ray_dir * t_far # Start at far plane [S3, unofficial]

    # 4. Ray Marching Loop
    while t_near < t_far:
        # World-to-texture mapping [S2, unofficial]
        tex_coord = (position / (2.0 * box_max)) + vec3(0.5)

        # Sample textures
        p = texture_lookup(volume_tex, tex_coord)
        r = texture_lookup(mask_tex, tex_coord)
        col_mask = texture_lookup(ctf, r.a)

        if col_mask.a == 0.0:
            return vec4(0.0) # Discard fragment [S1, unofficial]

        # Intensity normalization [S2, unofficial]
        raw_val = p.a
        denom = range_max - range_min
        val = 0.0 if denom < 1e-6 else clamp((raw_val - range_min) / denom, 0.0, 1.0)
        val = pow(val, gamma)

        # Sample transfer functions
        col_vol = texture_lookup(lut, val)
        grad = p.rgb
        grad_weight = texture_lookup(gmtf, r.a)

        # Opacity correction and compositing [Parent notes, S3, unofficial]
        alpha = col_vol.a
        corrected_alpha = alpha * (1.0 / (1.0 - alpha * step_size)) # [G] Standard opacity correction
        corrected_alpha = min(corrected_alpha, 1.0)

        dest.rgb += (1.0 - dest.a) * corrected_alpha * col_vol.rgb
        dest.a += (1.0 - dest.a) * corrected_alpha

        # Early Ray Termination [Parent notes, G]
        if dest.a > 0.95:
            break

        # Advance position
        position = position - ray_dir * step_size
        t_near += step_size

    # 5. Mask/Context Blending Logic [S1, unofficial]
    final_color = dest
    if col_mask.a == 1.0:
        final_color = col_vol * beta + (1.0 - beta) * col_mask
    else:
        if surface_voxels_only and not large_gradient(grad, grad_weight):
            final_color.a = 0.0
        elif color_ghosting:
            final_color = col_mask
        else:
            final_color = col_vol * beta + (1.0 - beta) * col_mask

    # 6. Shading
    final_color = shade(final_color, grad, grad_weight)
    return final_color

def large_gradient(grad, grad_weight):
    return grad_weight > 0.5 # [G] Threshold check inferred from context

def shade(color, grad, grad_weight):
    return color * grad_weight # [G] Simplified shading model based on gradient weight
```

**Parameters:**
*   `beta`: Float, modulates the blend between volume color and mask color. Default 1.0 [S1, S2, unofficial].
*   `surfaceVoxelsOnly`: Boolean, hides interior voxels if gradient is small [S1, unofficial].
*   `colorGhosting`: Boolean, displays mask color for context nodes [S1, unofficial].
*   `roiMin`, `roiMax`: `vec3`, texture coordinate bounds for Region of Interest clipping, default $(0,0,0)$ to $(1,1,1)$ [S2, unofficial].
*   `isoValue`: Float, threshold for isosurface mode [S2, unofficial].
*   `quality`: Float, controls step count (0.1 coarse, 1.0 fine) [S2, unofficial].
*   `dithering`: Float, controls random ray-start offset strength [S2, unofficial].

## Subtopics

### Volume Raycasting Fragment Shader (Integer Variant)
A variant that handles colormap and inversion logic differently, specifically separating inversion application for colormap vs scalar modes [S4, unofficial]. It uses integer-based fragment coordinates to determine inversion state [S4, unofficial].
*   **Definition:** Uses `highp usampler3D` for volume and `highp sampler2D` for colormap. Inversion is determined by comparing `comp` uniform with `gl_FragCoord` [S4, unofficial].
*   **Logic:**
    1.  Initialize `invert = false`. If `comp.x == int(gl_FragCoord.x)` or `comp.y == int(gl_FragCoord.y)`, set `invert = true` [S4, unofficial].
    2.  Sample volume at `samp`. Convert red channel to float and divide by 65536.0. If `samp` is outside $[0.0, 1.0]$, set value to 0.0 [S4, unofficial].
    3.  Default color is white RGB with alpha `val` [S4, unofficial].
    4.  If `use_colormap` is true: Sample colormap at `vec2(val, 0.5)`. Set color to RGB from colormap and alpha from `val` [S4, unofficial].
    5.  If `invert` is true AND `use_colormap` is false: Set color to white RGB and alpha `1.0 - val` [S4, unofficial].
    6.  If `invert` is true AND `use_colormap` is true: Set color to white RGB and alpha 1.0 [S4, unofficial].
*   **When to choose:** When integer-based coordinate inversion is required for specific visualization modes or when using 16-bit integer volume data mapped to colormap [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples per ray, which is controlled by the `quality` parameter [S2, unofficial].
*   **Memory Complexity:** Low, as it operates per-fragment without storing large intermediate buffers, relying on GPU texture memory [S2, unofficial].
*   **Performance:** Fast for small volumes or low quality settings; performance degrades with high `quality` (fine step count) and large volumes [S2, unofficial].
*   **Strengths:** Simple implementation, leverages hardware texture sampling, supports real-time interaction via UI parameters [S1, S2, unofficial].
*   **Limitations:** Limited to direct ray casting; does not support advanced lighting or global illumination without additional shader complexity [S1, unofficial]. Banding artifacts may occur without dithering [S2, unofficial].
*   **When to use:** Real-time visualization of scalar volumes with simple transfer functions [S1, S2, unofficial].
*   **When not to use:** Complex lighting requirements or very large volumes requiring hierarchical traversal [S1, unofficial].

## Background
This method originates from implementations in open-source visualization tools and educational assignments, adapting direct volume ray casting to fragment shaders [S1, S2, S3, unofficial]. It builds on the principles of hardware-accelerated volume rendering [Hardware-Accelerated Volume Rendering](5616-hardware-accelerated-volume-rendering.md) [G].

## Sources
[S1] Structural focus+context rendering of multiclassified volume data, http://hdl.handle.net/2117/86811, fetched 2026-10-26 (unofficial)
[S2] xwghua/clearvolume-plus, https://github.com/xwghua/clearvolume-plus, fetched 2026-10-26 (unofficial)
[S3] CS247-KAUST/Bonus_Assignment, https://github.com/CS247-KAUST/Bonus_Assignment, fetched 2026-10-26 (unofficial)
[S4] johanna-b/VisWeb, https://github.com/johanna-b/VisWeb, fetched 2026-10-27 (unofficial)
[S5] Pointifix/Vis2_Schott2010, https://github.com/Pointifix/Vis2_Schott2010, fetched 2026-10-26 (unofficial)
