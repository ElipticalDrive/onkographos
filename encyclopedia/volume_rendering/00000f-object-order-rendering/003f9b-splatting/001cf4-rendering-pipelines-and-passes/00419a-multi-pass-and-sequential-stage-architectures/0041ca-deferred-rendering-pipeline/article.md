# Deferred Rendering Pipeline
Also called: 8-Pass Deferred Rendering Pipeline
## Overview
A multi-pass deferred rendering sequence that decouples geometry traversal from lighting evaluation by storing scene attributes in separate framebuffer objects (FBOs) [S1, unofficial]. It computes depth, position/normal, materials, filtered normals, shadows, screen-space ambient occlusion (SSAO), blurred AO, and final shading in distinct stages [S1, unofficial]. It sets itself apart from the [Volume Rendering Pipeline](1318-volume-rendering-pipeline.md) by explicitly separating the octree traversal phase (depth/shadows) from the screen-space shading phases, and from the [Fragment List-Based Rendering Pipeline](3488-fragment-list-based-rendering-pipeline.md) by using fixed-function FBO attachments rather than dynamic fragment lists.
## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene geometry (represented as an octree or DAG4 structure), camera matrices (`viewMat`, `projMat`, `viewMatInv`, `projMatInv`), light position, and boolean flags for optional passes (`_useMaterials`, `_filterNormals`, `_doShadows`, `_doSSAO`).
*   **Outputs:** Final shaded image in the default framebuffer.

**Data Structures:**
*   **Framebuffers:** Eight FBOs (`_glFBO[0]` through `_glFBO[6]` and the default framebuffer) are used sequentially [S1, unofficial].
*   **Textures:** Bound to texture units `TEX_NUM_*` during `resetState()` [S1, unofficial]:
    *   `_glTexDepth`: Stores depth information.
    *   `_glTexHitPos`: Stores hit positions.
    *   `_glTexHitNorm`: Stores hit normals.
    *   `_glTexHitMaterial`: Stores material properties.
    *   `_glTexHitNormFiltered`: Stores filtered normals.
    *   `_glTexShadows`: Stores shadow map data.
    *   `_glTexAO`: Stores raw ambient occlusion values.
    *   `_glTexAOBlurred`: Stores blurred ambient occlusion values.
    *   `_glTexAONoise`: Stores noise for AO.
    *   `_glTexTBOMaterials`: Stores TBO material data.

**Numbered Steps:**
1.  **Pass 1 (Depth):** Bind `_glFBO[0]`, clear, set the `OctreeDDARenderer` mode to `DEPTH`, and draw the scene geometry [S1, unofficial].
2.  **Pass 2 (Position/Normal):** Bind `_glFBO[1]`, clear, use `_programPosNorm`, set uniforms `viewMatInv`, `projMatInv`, and `useDif`, and draw a full-screen quad [S1, unofficial].
3.  **Pass 3 (Materials):** If `_useMaterials` is true, bind `_glFBO[2]`, clear, use `_programMaterial`, set uniforms, and draw a full-screen quad [S1, unofficial].
4.  **Pass 4 (Filtered Normals):** If `_filterNormals` is true, bind `_glFBO[3]`, clear, use `_programNormFilter`, set uniforms `sigma` and `bsigma`, and draw a full-screen quad [S1, unofficial].
5.  **Pass 5 (Shadows):** If `_doShadows` is true, bind `_glFBO[4]`, clear, set `lightPos`, set `OctreeDDARenderer` mode to `SHADOW`, and draw the scene geometry [S1, unofficial]. If shadows are disabled, clear `_glFBO[4]` to white (1.0), acting as a default fully lit state [S1, unofficial].
6.  **Pass 6 (SSAO):** If `_doSSAO` is true, bind `_glFBO[5]`, clear, use `_programSSAO`, set uniforms `projMat`, `viewMat`, `camPos`, and `radius`, and draw a full-screen quad [S1, unofficial]. Note: The original octree-based AO path is disabled in favor of this screen-space quad pass [S1, unofficial].
7.  **Pass 7 (Blur AO):** Bind `_glFBO[6]`, clear, use `_programBlur`, and draw a full-screen quad [S1, unofficial].
8.  **Pass 8 (Final Shading):** Bind the default framebuffer, clear, use `_programShading`, set uniforms `isYUp`, `camPos`, `projMatInv`, `viewMatInv`, `lightPos`, `renderMode`, and `useMaterials`, and draw a full-screen quad [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def deferred_rendering_pipeline(scene, camera, lights, flags):
    # Initialize FBOs and textures as per resetState()
    fbo_depth = create_fbo()
    fbo_pos_norm = create_fbo()
    fbo_material = create_fbo()
    fbo_norm_filtered = create_fbo()
    fbo_shadows = create_fbo()
    fbo_ao = create_fbo()
    fbo_ao_blurred = create_fbo()
    
    # Pass 1: Depth
    bind_fbo(fbo_depth)
    clear_color_buffer()
    set_renderer_mode(OctreeDDARenderer, DEPTH)
    draw_scene(scene)
    
    # Pass 2: Position and Normal
    bind_fbo(fbo_pos_norm)
    clear_color_buffer()
    use_shader(_programPosNorm)
    set_uniform("viewMatInv", camera.view_mat_inv)
    set_uniform("projMatInv", camera.proj_mat_inv)
    set_uniform("useDif", True) # Example default
    draw_fullscreen_quad()
    
    # Pass 3: Materials
    if flags.use_materials:
        bind_fbo(fbo_material)
        clear_color_buffer()
        use_shader(_programMaterial)
        set_uniforms_for_materials()
        draw_fullscreen_quad()
        
    # Pass 4: Filtered Normals
    if flags.filter_normals:
        bind_fbo(fbo_norm_filtered)
        clear_color_buffer()
        use_shader(_programNormFilter)
        set_uniform("sigma", 1.0) # Typical value
        set_uniform("bsigma", 1.0) # Typical value
        draw_fullscreen_quad()
        
    # Pass 5: Shadows
    if flags.do_shadows:
        bind_fbo(fbo_shadows)
        clear_color_buffer()
        set_uniform("lightPos", lights[0].position)
        set_renderer_mode(OctreeDDARenderer, SHADOW)
        draw_scene(scene)
    else:
        # Clear to white for fully lit state
        bind_fbo(fbo_shadows)
        clear_color_buffer_with_value(1.0)
        
    # Pass 6: SSAO
    if flags.do_ssao:
        bind_fbo(fbo_ao)
        clear_color_buffer()
        use_shader(_programSSAO)
        set_uniform("projMat", camera.proj_mat)
        set_uniform("viewMat", camera.view_mat)
        set_uniform("camPos", camera.position)
        set_uniform("radius", 1.0) # Typical value
        draw_fullscreen_quad()
        
    # Pass 7: Blur AO
    bind_fbo(fbo_ao_blurred)
    clear_color_buffer()
    use_shader(_programBlur)
    draw_fullscreen_quad()
    
    # Pass 8: Final Shading
    bind_default_fbo()
    clear_color_buffer()
    use_shader(_programShading)
    set_uniform("isYUp", True)
    set_uniform("camPos", camera.position)
    set_uniform("projMatInv", camera.proj_mat_inv)
    set_uniform("viewMatInv", camera.view_mat_inv)
    set_uniform("lightPos", lights[0].position)
    set_uniform("renderMode", 0) # Example default
    set_uniform("useMaterials", flags.use_materials)
    draw_fullscreen_quad()
```

**Parameters:**
*   `sigma`, `bsigma`: Gaussian filter parameters for normal filtering. Typical values are not specified in the sources [S1, unofficial].
*   `radius`: SSAO sampling radius. Typical values are not specified in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
*   When shadows are disabled, the shadow buffer is cleared to white (1.0) to simulate full lighting [S1, unofficial].
*   The original octree-based AO path is disabled in favor of the screen-space quad pass, indicated by `#if 0/#else` in the source code [S1, unofficial].
## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** Linear with respect to the number of passes (8 passes) and the screen resolution. The geometry traversal passes (1 and 5) depend on the octree/DAG4 structure size. The screen-space passes (2-4, 6-8) depend on the screen resolution.
*   **Memory:** Proportional to the number of FBOs and textures (8 FBOs, 10 textures) [S1, unofficial].

**Measured Performance:**
*   Not given in the sources.

**Scaling:**
*   Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** Modular design allows independent optimization of stages (e.g., swapping shading models) [S1, unofficial]. Decouples geometry traversal from pixel evaluation [S1, unofficial].
*   **Limitations:** Requires multiple render passes, increasing overhead. Screen-space techniques (SSAO) may have artifacts at screen edges or for thin geometry.

**When to Use:**
*   When complex lighting and post-processing effects are required, and the scene geometry is static or changes infrequently.

**When Not to Use:**
*   When real-time performance is critical and the overhead of multiple passes is unacceptable.
*   When the scene contains highly dynamic geometry that changes every frame, as the deferred buffers need to be rebuilt every frame.
## Background
**Origin:**
*   The method is implemented in the `ssvdag` project by ajaspe [S1, unofficial].

**History and Development:**
*   The specific implementation details are found in the `ssvdag` repository [S1, unofficial].

**Where it is Used:**
*   Used in the `ssvdag` project for rendering octree/DAG4-based scenes [S1, unofficial].

## Sources
[S1] ajaspe/ssvdag, https://github.com/ajaspe/ssvdag, fetched 2026-10-24 (unofficial)
