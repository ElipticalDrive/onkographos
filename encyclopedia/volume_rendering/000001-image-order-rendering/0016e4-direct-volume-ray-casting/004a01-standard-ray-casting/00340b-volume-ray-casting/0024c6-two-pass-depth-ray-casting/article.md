# Two-Pass Depth Ray Casting

Also called: Depth Peeling Ray Casting, Entry/Exit Depth Ray Casting, Krüger Ray-Casting, Ray-Casting de Krüger, Volumetric Ray-Casting

## Overview
Two-Pass Depth Ray Casting is a GPU-based direct volume rendering technique that determines ray entry and exit points by rendering the volume's bounding box geometry twice, then accumulates color along those rays in subsequent passes [S1, S2, S3, unofficial]. It differs from standard [Volume Ray Casting](3161-volume-ray-casting.md) by explicitly decoupling the geometric intersection calculation from the volumetric sampling, using depth-buffered rendering of the bounding box to provide precise start and end coordinates for each pixel's ray [S2, unofficial]. This approach enables early fragment rejection via depth testing when opacity thresholds are met, optimizing performance for opaque regions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D scalar volume texture ($t_{Volume}$), transfer function texture ($t_{TFFunc}$), viewport dimensions, volume resolution ($t_{DIM}$), value span ($v_{Min}, v_{Max}$), and number of steps ($steps$).
*   **Output:** Final composited color image in the render framebuffer.

**Data Structures:**
1.  **Bounding Box Geometry:** A cube defined by 8 vertices (each with 3 floats for position, 3 floats for color) and 36 indices (forming 12 triangles) [S2, unofficial].
2.  **Render Targets:**
    *   $ProjInFB$: Framebuffer storing entry coordinates. Resolution matches viewport. Texture format: 32-bit float RGB, NearestFilter, ClampToEdge [S2, S3, unofficial].
    *   $ProjOutFB$: Framebuffer storing exit coordinates. Same format as $ProjInFB$ [S2, S3, unofficial].
    *   $RenderFB$: Final output framebuffer [S2, unofficial].
3.  **Volume Texture:** 3D texture ($t_{Volume}$) with dimensions matching the dataset resolution, LinearFilter, ClampToEdge [S2, S3, unofficial].

**Steps:**

1.  **Pass 1 (Entry Point):** Render the bounding box with **back-face culling** enabled and depth test set to `DEPTH_LESS`. Clear depth buffer to 1.0. The fragment shader outputs the world-space coordinates of the back-face intersection (which is the entry point for the ray coming from the camera) to $ProjInFB$ [S2, S3, unofficial].
2.  **Pass 2 (Exit Point):** Render the bounding box with **front-face culling** enabled and depth test set to `DEPTH_GREATER`. Clear depth buffer to 0.0. The fragment shader outputs the world-space coordinates of the front-face intersection (exit point) to $ProjOutFB$ [S2, S3, unofficial].
3.  **Pass 3+ (Ray Casting & Accumulation):** Render a fullscreen quad (or the bounding box again) to $RenderFB$. Bind $ProjInFB$ and $ProjOutFB$ as textures. For each pixel:
    *   Retrieve entry position ($P_{entry}$) from $ProjInFB$ and exit position ($P_{exit}$) from $ProjOutFB$ [S3, unofficial].
    *   Calculate ray vector: $\vec{r} = P_{exit} - P_{entry}$ [S3, unofficial].
    *   Calculate ray length: $L = \|\vec{r}\|$ [S3, unofficial].
    *   Calculate step size: $\Delta = 1.0 / steps$ [S3, unofficial].
    *   Calculate step direction: $\vec{d} = \text{normalize}(\vec{r}) \cdot \Delta$ [S3, unofficial].
    *   Initialize current position $P_{curr} = P_{entry}$, accumulated color $C_{acc} = (0,0,0,0)$, accumulated length $L_{acc} = 0$ [S3, unofficial].
    *   Loop for $i = 0$ to $steps-1$:
        *   Sample volume: $I = t_{Volume}(P_{curr})$ [S3, unofficial].
        *   Apply transfer function and adjustments: $I' = \text{contrast} \cdot I + \text{brightness}$ [S3, unofficial].
        *   Threshold: If $I' \leq \text{colorTreshLow}$ or $I' \geq \text{colorTreshHigh}$, set $I' = 0$ [S3, unofficial].
        *   Compute color contribution $C_{step}$:
            *   If $I' \geq 0.5$: $C_{step} = (I' \cdot 2.56 \cdot \text{alphaCorrection} \cdot \Delta, 0, 0, I' \cdot 2.56 \cdot \text{alphaCorrection} \cdot \Delta)$ [S3, unofficial].
            *   Else: $C_{step} = (I' \cdot 2.56 \cdot \text{alphaCorrection} \cdot \Delta, I' \cdot 2.56 \cdot \text{alphaCorrection} \cdot \Delta, I' \cdot 2.56 \cdot \text{alphaCorrection} \cdot \Delta, I' \cdot 2.56 \cdot \text{alphaCorrection} \cdot \Delta)$ [S3, unofficial].
        *   Accumulate: $C_{acc} += C_{step}$ [S3, unofficial].
        *   Update: $L_{acc} += \|\vec{d}\|$, $P_{curr} += \vec{d}$ [S3, unofficial].
        *   Early Termination: If $L_{acc} \geq L$ or $C_{acc}.a \geq 1.0$, break loop [S3, unofficial].
    *   Write $C_{acc}$ to $RenderFB$ [S3, unofficial].
    *   **Early Depth Test Optimization:** In subsequent passes (if iterative), write a depth value forcing rejection for fragments that have reached maximum opacity, utilizing GPU early depth test to skip fragment shader execution for fully opaque pixels [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pass 1: Entry Point
def render_entry_point():
    set_depth_test(LESS)
    set_face_culling(BACK)
    clear_depth(1.0)
    bind_framebuffer(ProjInFB)
    draw_cube()
    # Fragment Shader: gl_FragColor = vec4(worldSpaceCoords, 1.0)

# Pass 2: Exit Point
def render_exit_point():
    set_depth_test(GREATER)
    set_face_culling(FRONT)
    clear_depth(0.0)
    bind_framebuffer(ProjOutFB)
    draw_cube()
    # Fragment Shader: gl_FragColor = vec4(worldSpaceCoords, 1.0)

# Pass 3+: Ray Casting
def render_volume():
    bind_framebuffer(RenderFB)
    bind_texture(ProjInFB, unit=0) # Entry coords
    bind_texture(ProjOutFB, unit=1) # Exit coords
    bind_texture(t_Volume, unit=2)
    
    draw_fullscreen_quad()
    
    # Fragment Shader Logic:
    # 1. texc = vec2(((gl_FragCoord.x / viewport.w) * 2.0 - 1.0), ((gl_FragCoord.y / viewport.h) * 2.0 - 1.0))
    # 2. P_entry = texture(ProjInFB, texc).xyz
    # 3. P_exit = texture(ProjOutFB, texc).xyz
    # 4. if P_entry == vec3(0.0): return vec4(0.0) # Invalid intersection
    # 5. ray = P_exit - P_entry
    # 6. ray_len = length(ray)
    # 7. delta = 1.0 / steps
    # 8. delta_dir = normalize(ray) * delta
    # 9. curr_pos = P_entry
    # 10. acc_color = vec4(0.0)
    # 11. for i in range(steps):
    #       intensity = texture(t_Volume, curr_pos).r
    #       intensity = contrast * intensity + brightness
    #       if intensity <= colorTreshLow or intensity >= colorTreshHigh:
    #           intensity = 0.0
    #       if intensity >= 0.5:
    #           color_step = vec4(intensity * 2.56 * alphaCorrection * delta, 0.0, 0.0, intensity * 2.56 * alphaCorrection * delta)
    #       else:
    #           color_step = vec4(intensity * 2.56 * alphaCorrection * delta, intensity * 2.56 * alphaCorrection * delta, intensity * 2.56 * alphaCorrection * delta, intensity * 2.56 * alphaCorrection * delta)
    #       acc_color += color_step
    #       curr_pos += delta_dir
    #       if length(curr_pos - P_entry) >= ray_len or acc_color.a >= 1.0:
    #           break
    # 12. gl_FragColor = acc_color
```

**Parameters:**
*   `steps`: Number of samples along the ray. Default: 256 [S3, unofficial].
*   `contrast`: Multiplier for intensity. Default: 1.0 [S3, unofficial].
*   `brightness`: Offset for intensity. Default: 0.0 [S3, unofficial].
*   `alphaCorrection`: Scalar for alpha scaling. Default: 2.0 [S3, unofficial].
*   `colorTreshLow`: Lower intensity threshold. Default: 0.0 [S3, unofficial].
*   `colorTreshHigh`: Upper intensity threshold. Default: 1.0 [S3, unofficial].
*   `t_DIM`: Volume resolution (voxels) [S2, unofficial].
*   `v_Min, v_Max`: Volume value span [S2, unofficial].

**Edge Cases:**
*   If the sampled back position is $(0,0,0)$, it indicates an invalid intersection (e.g., at the cube edge where no back-face was rendered). The shader returns black $(0,0,0,0)$ [S3, unofficial].
*   Zero-sized viewport causes invalid framebuffer operations. Must check `viewport.width > 0` and `viewport.height > 0` before resizing [S2, unofficial].

## Subtopics

### [Volume Raycasting with Front/Back Face Depth Textures](13618-volume-raycasting-with-front-back-face-depth-textures.md)
Encodes full 3D coordinates of entry/exit faces in 2D textures, avoiding per-ray intersection tests. Chose when bounding box geometry is static and coordinate mapping is preferred over depth buffer tricks.

### [Single-pass Raycasting](6121-single-pass-raycasting.md)
Combines intersection and casting in one pass. Chose for simpler pipelines where early depth test optimization is less critical or hardware supports it natively.

### [Backward Mapping (Ray Tracing)](3009-backward-mapping-ray-tracing.md)
Traces rays from surface to volume. Chose for surface-volume hybrid rendering.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$ per pixel, where $N$ is the number of pixels and $S$ is the number of steps. The two initial passes are $O(N)$ for bounding box rasterization. Subsequent passes are $O(N \cdot S)$ but benefit from early depth test rejection [S1, S3, unofficial].
*   **Memory Complexity:** $O(V)$ for the 3D volume texture, plus $O(N)$ for the two coordinate textures (entry/exit) [S2, S3, unofficial].
*   **Performance:** Performance is significantly impacted by data volume size due to the rasterization of support primitives (bounding box) and the fixed number of steps [S1, unofficial]. Early depth testing improves performance for opaque regions [S1, unofficial].
*   **Strengths:** Precise entry/exit points; efficient early termination for opaque pixels; simple implementation on GPU [S1, S2, unofficial].
*   **Limitations:** Does not scale well with large data volumes; fixed step count may lead to aliasing or excessive computation; limited to regular voxel grids [S1, unofficial].
*   **When to Use:** Interactive visualization of moderate-sized regular volumes where precise ray endpoints are needed and early termination is beneficial [S1, S2, unofficial].
*   **When Not to Use:** Very large datasets where memory bandwidth is a bottleneck; irregular volumes requiring complex intersection tests [S1, unofficial].

## Background
The method was first proposed by Krüger [KW03] for Shader Model 2.0 GPUs [S1, unofficial]. It was later extended by Scharsach [Sch05] to leverage Shader Model 3.0 features, including dynamic branching and block-based volume loading [S1, unofficial]. It is a foundational technique for GPU-based direct volume rendering, particularly in medical imaging and scientific visualization [S1, S3, unofficial].

## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S2] hands133/opengl-directVolumeRenderer, https://github.com/hands133/opengl-directVolumeRenderer, fetched 2026-10-26 (unofficial)
[S3] arisetiawan4601/dicom-volume-renderer, https://github.com/arisetiawan4601/dicom-volume-renderer, fetched 2026-10-07 (unofficial)
