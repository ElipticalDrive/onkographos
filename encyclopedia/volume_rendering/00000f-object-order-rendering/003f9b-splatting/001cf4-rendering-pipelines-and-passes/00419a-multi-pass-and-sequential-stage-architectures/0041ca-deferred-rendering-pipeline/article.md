# Deferred Rendering Pipeline
Also called: 8-Pass Deferred Rendering Pipeline
## Overview
A multi-pass deferred rendering sequence that decouples geometric depth/position acquisition from lighting and post-processing effects by storing intermediate scene data in separate framebuffer objects (FBOs) [S1, unofficial]. It differs from forward rendering by deferring shading calculations until after all geometry is processed, allowing complex effects like Screen-Space Ambient Occlusion (SSAO) and shadow mapping to be applied uniformly to the entire scene without per-object overhead [S1, unofficial].
## How it works
### Inputs and Outputs
**Inputs:**
- Volume data (implicit or explicit primitives, e.g., OctreeDDA) [S1, unofficial].
- Camera matrices: View ($V$), Projection ($P$), Inverse View ($V^{-1}$), Inverse Projection ($P^{-1}$) [S1, unofficial].
- Light position ($L$) [S1, unofficial].
- Configuration flags: `_useMaterials`, `_filterNormals`, `_doShadows`, `_doSSAO` [S1, unofficial].

**Outputs:**
- Final shaded image rendered to the default framebuffer [S1, unofficial].

### Data Structures
The pipeline uses a chain of Framebuffer Objects (FBOs) and associated textures. The specific layout is defined by the source implementation [S1, unofficial]:

1.  **FBO[0] (`_glFBO[0]`)**: Depth buffer (`_glTexDepth`). Used in Pass 1.
2.  **FBO[1] (`_glFBO[1]`)**: Position (`_glTexHitPos`) and Normal (`_glTexHitNorm`) buffers. Used in Pass 2.
3.  **FBO[2] (`_glFBO[2]`)**: Material buffer (`_glTexHitMaterial`). Used in Pass 3 (conditional).
4.  **FBO[3] (`_glFBO[3]`)**: Filtered Normal buffer (`_glTexHitNormFiltered`). Used in Pass 4 (conditional).
5.  **FBO[4] (`_glFBO[4]`)**: Shadow map (`_glTexShadows`). Used in Pass 5.
6.  **FBO[5] (`_glFBO[5]`)**: Ambient Occlusion buffer (`_glTexAO`). Used in Pass 6.
7.  **FBO[6] (`_glFBO[6]`)**: Blurred AO buffer (`_glTexAOBlurred`). Used in Pass 7.
8.  **Default Framebuffer**: Final output. Used in Pass 8.

Textures are bound to texture units `TEX_NUM_*` during state reset [S1, unofficial]. Additional textures include `_glTexAONoise` and `_glTexTBOMaterials` [S1, unofficial].

### Numbered Steps
1.  **Pass 1 (Depth):** Bind FBO[0]. Clear buffers. Set renderer mode to `DEPTH`. Draw the volume geometry (e.g., OctreeDDA primitives) to populate the depth buffer [S1, unofficial].
2.  **Pass 2 (Position/Normal):** Bind FBO[1]. Clear buffers. Use shader `_programPosNorm`. Set uniforms: `viewMatInv`, `projMatInv`, `useDif`. Draw a full-screen quad to reconstruct world-space position and normal from depth [S1, unofficial].
3.  **Pass 3 (Material):** If `_useMaterials` is true: Bind FBO[2]. Clear buffers. Use shader `_programMaterial`. Set uniforms. Draw a full-screen quad to encode material properties [S1, unofficial].
4.  **Pass 4 (Normal Filtering):** If `_filterNormals` is true: Bind FBO[3]. Clear buffers. Use shader `_programNormFilter`. Set uniforms: `sigma`, `bsigma`. Draw a full-screen quad to filter normals [S1, unofficial].
5.  **Pass 5 (Shadows):** If `_doShadows` is true: Bind FBO[4]. Clear buffers. Set `lightPos`. Set renderer mode to `SHADOW`. Draw the volume geometry to generate shadow maps [S1, unofficial]. Else: Clear FBO[4] to white (1.0), representing a fully lit state [S1, unofficial].
6.  **Pass 6 (SSAO):** If `_doSSAO` is true: Bind FBO[5]. Clear buffers. Use shader `_programSSAO`. Set uniforms: `projMat`, `viewMat`, `camPos`, `radius`. Draw a full-screen quad to compute screen-space ambient occlusion [S1, unofficial]. Note: The original octree-based AO path is disabled (`#if 0`) in favor of this screen-space pass [S1, unofficial].
7.  **Pass 7 (Blur):** Bind FBO[6]. Clear buffers. Use shader `_programBlur`. Draw a full-screen quad to blur the AO buffer (`_glTexAO`) into `_glTexAOBlurred` [S1, unofficial].
8.  **Pass 8 (Final Shading):** Bind default framebuffer. Clear buffers. Use shader `_programShading`. Set uniforms: `isYUp`, `camPos`, `projMatInv`, `viewMatInv`, `lightPos`, `renderMode`, `useMaterials`. Draw a full-screen quad to composite all buffers (depth, position, normal, material, shadow, AO) into the final image [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def deferred_render_pipeline(volume_renderer, camera, light, config):
    # Setup FBOs and textures as defined in Data Structures
    fbo_depth = create_fbo([GL_DEPTH_COMPONENT])
    fbo_pos_norm = create_fbo([GL_RGBA, GL_RGBA]) # Position, Normal
    fbo_material = create_fbo([GL_RGBA])
    fbo_norm_filter = create_fbo([GL_RGBA])
    fbo_shadow = create_fbo([GL_DEPTH_COMPONENT])
    fbo_ao = create_fbo([GL_RGBA])
    fbo_ao_blur = create_fbo([GL_RGBA])

    # Pass 1: Depth
    bind_fbo(fbo_depth)
    clear_buffer(GL_DEPTH_BUFFER_BIT)
    volume_renderer.set_mode("DEPTH")
    volume_renderer.draw()

    # Pass 2: Position and Normal
    bind_fbo(fbo_pos_norm)
    clear_buffer(GL_COLOR_BUFFER_BIT)
    use_shader("programPosNorm")
    set_uniform("viewMatInv", camera.inverse_view)
    set_uniform("projMatInv", camera.inverse_projection)
    set_uniform("useDif", True)
    draw_fullscreen_quad()

    # Pass 3: Material (Conditional)
    if config.use_materials:
        bind_fbo(fbo_material)
        clear_buffer(GL_COLOR_BUFFER_BIT)
        use_shader("programMaterial")
        set_uniforms_for_material()
        draw_fullscreen_quad()

    # Pass 4: Normal Filtering (Conditional)
    if config.filter_normals:
        bind_fbo(fbo_norm_filter)
        clear_buffer(GL_COLOR_BUFFER_BIT)
        use_shader("programNormFilter")
        set_uniform("sigma", 1.0) # Example value
        set_uniform("bsigma", 1.0) # Example value
        draw_fullscreen_quad()

    # Pass 5: Shadows
    if config.do_shadows:
        bind_fbo(fbo_shadow)
        clear_buffer(GL_DEPTH_BUFFER_BIT)
        set_uniform("lightPos", light.position)
        volume_renderer.set_mode("SHADOW")
        volume_renderer.draw()
    else:
        # Default fully lit state
        bind_fbo(fbo_shadow)
        clear_color(1.0, 1.0, 1.0, 1.0)

    # Pass 6: SSAO
    if config.do_ssao:
        bind_fbo(fbo_ao)
        clear_buffer(GL_COLOR_BUFFER_BIT)
        use_shader("programSSAO")
        set_uniform("projMat", camera.projection)
        set_uniform("viewMat", camera.view)
        set_uniform("camPos", camera.position)
        set_uniform("radius", 0.5) # Example value
        draw_fullscreen_quad()

    # Pass 7: Blur AO
    bind_fbo(fbo_ao_blur)
    clear_buffer(GL_COLOR_BUFFER_BIT)
    use_shader("programBlur")
    draw_fullscreen_quad()

    # Pass 8: Final Shading
    bind_default_fbo()
    clear_buffer(GL_COLOR_BUFFER_BIT)
    use_shader("programShading")
    set_uniform("isYUp", True)
    set_uniform("camPos", camera.position)
    set_uniform("projMatInv", camera.inverse_projection)
    set_uniform("viewMatInv", camera.inverse_view)
    set_uniform("lightPos", light.position)
    set_uniform("renderMode", "FINAL")
    set_uniform("useMaterials", config.use_materials)
    draw_fullscreen_quad()
```

### Parameters and Edge Cases
-   **Sigma/Bsigma:** Used in normal filtering. Values not explicitly defined in sources [S1, unofficial].
-   **SSAO Radius:** Affects occlusion calculation range. Value not explicitly defined in sources [S1, unofficial].
-   **Shadow Clear Value:** When shadows are disabled, FBO[4] is cleared to white (1.0). This acts as a default fully lit state, preventing black artifacts if shadow sampling is not properly guarded [S1, unofficial].
-   **SSAO Implementation:** The source code indicates the original octree-based AO path is disabled (`#if 0`) in favor of the screen-space quad pass. Implementers should be aware that the octree-specific AO logic is not active in this version [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** Linear with respect to the number of passes (8 passes) multiplied by the screen resolution for quad passes, plus the cost of volume rendering for depth and shadow passes.
-   **Memory Complexity:** Proportional to the screen resolution times the number of FBOs (8 textures).
-   **Strengths:** Decouples lighting from geometry, allowing complex effects like SSAO and shadows to be applied efficiently.
-   **Limitations:** Requires multiple render passes, increasing latency. Screen-space effects (SSAO) suffer from edge artifacts and limited view range.
-   **When to use:** Real-time rendering with complex lighting and post-processing effects.
-   **When not to use:** Simple scenes where forward rendering is more efficient, or when hardware does not support multiple FBOs efficiently.

## Background
The deferred rendering pipeline is a standard technique in real-time graphics. The specific 8-pass implementation described here is derived from the `ssvdag` project, which uses an OctreeDDA renderer for volume visualization [S1, unofficial]. The approach adapts traditional deferred shading principles to volumetric data, leveraging the GPU's rasterization capabilities for screen-space effects.

## Sources
[S1] ajaspe/ssvdag, https://github.com/ajaspe/ssvdag, fetched 2026-10-24 (unofficial)
