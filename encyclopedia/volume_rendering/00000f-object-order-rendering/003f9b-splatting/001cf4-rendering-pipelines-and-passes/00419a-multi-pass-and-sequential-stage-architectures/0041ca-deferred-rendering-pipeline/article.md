# Deferred Rendering Pipeline

Also called: 8-Pass Deferred Rendering Pipeline

## Overview
This method implements a multi-pass deferred rendering sequence for volume visualization, computing depth, position, normals, materials, shadows, and ambient occlusion in separate framebuffer objects before a final shading pass [S1, unofficial]. It distinguishes itself from object-order splatting by decoupling geometric traversal from screen-space shading, allowing complex lighting effects like SSAO and filtered normals to be applied via screen-space quads rather than per-voxel computation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An octree-based volume representation (specifically SVDAG structure), camera matrices (`viewMat`, `projMat`, `viewMatInv`, `projMatInv`), light position, and boolean flags for optional features (`_useMaterials`, `_filterNormals`, `_doShadows`, `_doSSAO`).
*   **Output:** A final color image rendered to the default framebuffer.

**Data Structures:**
The pipeline utilizes a sequence of Framebuffer Objects (FBOs) and attached textures. The specific layout involves:
*   `_glFBO[0]` through `_glFBO[6]`: Seven intermediate framebuffers.
*   `_glTexDepth`: Attached to FBO[0], stores depth information.
*   `_glTexHitPos`: Attached to FBO[1], stores hit positions.
*   `_glTexHitNorm`: Attached to FBO[2] (conditional), stores hit normals.
*   `_glTexHitMaterial`: Attached to FBO[3] (conditional), stores material properties.
*   `_glTexHitNormFiltered`: Attached to FBO[4] (conditional), stores filtered normals.
*   `_glTexShadows`: Attached to FBO[5], stores shadow map data.
*   `_glTexAO`: Attached to FBO[6], stores screen-space ambient occlusion values.
*   `_glTexAOBlurred`: Intermediate texture for blurred AO.
*   `_glTexAONoise`: Noise texture for AO variation.
*   `_glTexTBOMaterials`: Texture Buffer Object for materials.
*   These textures are bound to texture units `TEX_NUM_*` during the `resetState()` initialization phase [S1, unofficial].

**Numbered Steps:**
1.  **Pass 1 (Depth):** Bind `_glFBO[0]`. Clear the buffer. Set the `OctreeDDARenderer` mode to `DEPTH`. Draw the volume geometry to populate the depth buffer.
2.  **Pass 2 (Position/Normal):** Bind `_glFBO[1]`. Clear the buffer. Use the `_programPosNorm` shader. Set uniforms `viewMatInv`, `projMatInv`, and `useDif`. Draw a full-screen quad to reconstruct position and normal from depth.
3.  **Pass 3 (Materials):** If `_useMaterials` is true: Bind `_glFBO[2]`. Clear the buffer. Use `_programMaterial`. Set relevant material uniforms. Draw a full-screen quad.
4.  **Pass 4 (Normal Filtering):** If `_filterNormals` is true: Bind `_glFBO[3]`. Clear the buffer. Use `_programNormFilter`. Set uniforms `sigma` and `bsigma` for filtering parameters. Draw a full-screen quad.
5.  **Pass 5 (Shadows):** If `_doShadows` is true: Bind `_glFBO[4]`. Clear the buffer. Set `lightPos`. Set `OctreeDDARenderer` mode to `SHADOW`. Draw the volume. Else, clear `_glFBO[4]` to white (1.0), representing a fully lit state [S1, unofficial].
6.  **Pass 6 (SSAO):** If `_doSSAO` is true: Bind `_glFBO[5]`. Clear the buffer. Use `_programSSAO`. Set uniforms `projMat`, `viewMat`, `camPos`, and `radius`. Draw a full-screen quad. Note: The original octree-based AO path is disabled (`#if 0`) in favor of this screen-space pass [S1, unofficial].
7.  **Pass 7 (AO Blur):** Bind `_glFBO[6]`. Clear the buffer. Use `_programBlur`. Draw a full-screen quad to blur the AO texture.
8.  **Pass 8 (Final Shading):** Bind the default framebuffer. Clear the buffer. Use `_programShading`. Set uniforms `isYUp`, `camPos`, `projMatInv`, `viewMatInv`, `lightPos`, `renderMode`, and `useMaterials`. Draw a full-screen quad to composite all accumulated data into the final image.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def deferred_render_pipeline(volume_data, camera, light, flags):
    # Initialization
    reset_state() # Binds _glTex* to _glFBO* and texture units
    
    # Pass 1: Depth
    bind_fbo(_glFBO[0])
    clear()
    set_renderer_mode(OctreeDDARenderer, DEPTH)
    draw_volume(volume_data)
    
    # Pass 2: Position and Normal
    bind_fbo(_glFBO[1])
    clear()
    use_shader(_programPosNorm)
    set_uniform("viewMatInv", camera.viewMatInv)
    set_uniform("projMatInv", camera.projMatInv)
    set_uniform("useDif", True)
    draw_quad()
    
    # Pass 3: Materials (Conditional)
    if flags.useMaterials:
        bind_fbo(_glFBO[2])
        clear()
        use_shader(_programMaterial)
        set_uniforms_for_materials()
        draw_quad()
        
    # Pass 4: Normal Filtering (Conditional)
    if flags.filterNormals:
        bind_fbo(_glFBO[3])
        clear()
        use_shader(_programNormFilter)
        set_uniform("sigma", 1.0) # Typical value, source specific
        set_uniform("bsigma", 1.0) # Typical value, source specific
        draw_quad()
        
    # Pass 5: Shadows
    if flags.doShadows:
        bind_fbo(_glFBO[4])
        clear()
        set_uniform("lightPos", light.position)
        set_renderer_mode(OctreeDDARenderer, SHADOW)
        draw_volume(volume_data)
    else:
        bind_fbo(_glFBO[4])
        clear(color=vec4(1.0, 1.0, 1.0, 1.0)) # White = fully lit
        
    # Pass 6: SSAO
    if flags.doSSAO:
        bind_fbo(_glFBO[5])
        clear()
        use_shader(_programSSAO)
        set_uniform("projMat", camera.projMat)
        set_uniform("viewMat", camera.viewMat)
        set_uniform("camPos", camera.position)
        set_uniform("radius", 0.5) # Typical value, source specific
        draw_quad()
        
    # Pass 7: Blur AO
    bind_fbo(_glFBO[6])
    clear()
    use_shader(_programBlur)
    draw_quad()
    
    # Pass 8: Final Shading
    bind_default_fbo()
    clear()
    use_shader(_programShading)
    set_uniform("isYUp", True)
    set_uniform("camPos", camera.position)
    set_uniform("projMatInv", camera.projMatInv)
    set_uniform("viewMatInv", camera.viewMatInv)
    set_uniform("lightPos", light.position)
    set_uniform("renderMode", 0)
    set_uniform("useMaterials", flags.useMaterials)
    draw_quad()
```

**Edge Cases and Pitfalls:**
*   **Shadow Default:** When shadows are disabled, Pass 5 clears to white (1.0). This acts as a default "fully lit" state. If the shading model expects a different default (e.g., black or ambient-only), this may cause incorrect lighting [S1, unofficial].
*   **AO Path:** The screen-space AO pass (Pass 6) replaces the original octree-based AO. Ensure the shader `_programSSAO` is correctly configured for screen-space reconstruction, as the geometric AO path is disabled (`#if 0`) [S1, unofficial].
*   **Precision:** Depth reconstruction in Pass 2 relies on `viewMatInv` and `projMatInv`. Precision errors in these matrices or the depth buffer can lead to normal artifacts in subsequent passes.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of passes (8 passes) and the screen resolution (for quad-based passes). The volume traversal passes (1 and 5) depend on the octree size and view frustum culling.
*   **Memory Complexity:** High. Requires storage for 7 intermediate FBOs and multiple textures (depth, position, normal, material, shadow, AO, blurred AO, noise, TBO).
*   **Strengths:** Decouples geometry from lighting, allowing efficient screen-space effects (SSAO, blurring) and easy integration of multiple light sources without re-traversing the volume.
*   **Limitations:** High memory bandwidth usage due to multiple FBO reads/writes. Depth reconstruction artifacts can affect normal accuracy. Not suitable for real-time applications with strict memory budgets.
*   **When to use:** Offline or high-end real-time rendering where visual quality (SSAO, filtered normals) is prioritized over memory efficiency.
*   **When not to use:** Memory-constrained environments or when simple lighting is sufficient.

## Background
This pipeline is derived from the `ssvdag` implementation by ajaspe, which utilizes a Sparse Vectorized DAG (SVDAG) octree structure for volume rendering [S1, unofficial]. It adapts standard deferred rendering techniques (depth pre-pass, G-buffer, lighting pass) to the specific needs of octree-based volume visualization, such as handling depth-only traversal for shadow mapping and reconstructing normals from depth for screen-space effects.

## Sources
[S1] ajaspe/ssvdag, https://github.com/ajaspe/ssvdag, fetched 2026-10-24 (unofficial)
