# Deferred Rendering Pipeline
Also called: 8-Pass Deferred Rendering Pipeline
## Overview
The Deferred Rendering Pipeline is a multi-pass sequence that decouples geometry processing from shading by storing scene attributes (depth, position, normals, materials) in separate framebuffer objects (FBOs) before computing final lighting [S1, unofficial]. It differs from the [Volume Rendering Pipeline](1318-volume-rendering-pipeline.md) by rasterizing geometry into screen-space buffers rather than compositing volume primitives along rays, and from the [Fragment List-Based Rendering Pipeline](3488-fragment-list-based-rendering-pipeline.md) by using fixed, sequential passes over screen-space textures rather than dynamic fragment lists [S1, unofficial].
## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene geometry (via an OctreeDDARenderer), camera matrices (`viewMat`, `projMat`, inverses), light position, and boolean flags for optional features (`_useMaterials`, `_filterNormals`, `_doShadows`, `_doSSAO`) [S1, unofficial].
*   **Outputs:** Final shaded image in the default framebuffer [S1, unofficial].

**Data Structures:**
*   **Framebuffers:** Eight FBOs (`_glFBO[0]` to `_glFBO[6]` and the default framebuffer) are used sequentially [S1, unofficial].
*   **Textures:** Bound to texture units `TEX_NUM_*` during `resetState()` [S1, unofficial]:
    *   `_glTexDepth`: Stores depth information [S1, unofficial].
    *   `_glTexHitPos`: Stores hit positions [S1, unofficial].
    *   `_glTexHitNorm`: Stores hit normals [S1, unofficial].
    *   `_glTexHitMaterial`: Stores material properties [S1, unofficial].
    *   `_glTexHitNormFiltered`: Stores filtered normals [S1, unofficial].
    *   `_glTexShadows`: Stores shadow map data [S1, unofficial].
    *   `_glTexAO`: Stores ambient occlusion values [S1, unofficial].
    *   `_glTexAOBlurred`: Stores blurred ambient occlusion [S1, unofficial].
    *   `_glTexAONoise`: Stores noise for AO [S1, unofficial].
    *   `_glTexTBOMaterials`: Stores TBO material data [S1, unofficial].

**Steps:**
1.  **Depth Pass:** Bind `_glFBO[0]`, clear it, set `OctreeDDARenderer` mode to `DEPTH`, and draw the scene [S1, unofficial].
2.  **Position/Normal Pass:** Bind `_glFBO[1]`, clear it, use `_programPosNorm`, set uniforms `viewMatInv`, `projMatInv`, `useDif`, and draw a full-screen quad [S1, unofficial].
3.  **Material Pass (Optional):** If `_useMaterials` is true, bind `_glFBO[2]`, clear it, use `_programMaterial`, set uniforms, and draw a full-screen quad [S1, unofficial].
4.  **Normal Filtering Pass (Optional):** If `_filterNormals` is true, bind `_glFBO[3]`, clear it, use `_programNormFilter`, set uniforms `sigma` and `bsigma`, and draw a full-screen quad [S1, unofficial].
5.  **Shadow Pass (Optional):** If `_doShadows` is true, bind `_glFBO[4]`, clear it, set `lightPos`, set `OctreeDDARenderer` mode to `SHADOW`, and draw the scene [S1, unofficial]. If disabled, clear `_glFBO[4]` to white (1.0), representing a fully lit state [S1, unofficial].
6.  **SSAO Pass (Optional):** If `_doSSAO` is true, bind `_glFBO[5]`, clear it, use `_programSSAO`, set uniforms `projMat`, `viewMat`, `camPos`, `radius`, and draw a full-screen quad [S1, unofficial]. Note: The original octree-based AO path is disabled in favor of this screen-space quad pass [S1, unofficial].
7.  **AO Blur Pass:** Bind `_glFBO[6]`, clear it, use `_programBlur`, and draw a full-screen quad [S1, unofficial].
8.  **Final Shading Pass:** Bind the default framebuffer, clear it, use `_programShading`, set uniforms `isYUp`, `camPos`, `projMatInv`, `viewMatInv`, `lightPos`, `renderMode`, `useMaterials`, and draw a full-screen quad [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def deferred_rendering_pipeline(scene, camera, light, flags):
    # Pass 1: Depth
    bind_fbo(_glFBO[0])
    clear()
    set_renderer_mode(OctreeDDARenderer, DEPTH)
    draw_scene(scene)

    # Pass 2: Position and Normal
    bind_fbo(_glFBO[1])
    clear()
    use_program(_programPosNorm)
    set_uniform("viewMatInv", camera.view_mat_inv)
    set_uniform("projMatInv", camera.proj_mat_inv)
    set_uniform("useDif", True) # Assumed default based on source
    draw_quad()

    # Pass 3: Materials (Optional)
    if flags.use_materials:
        bind_fbo(_glFBO[2])
        clear()
        use_program(_programMaterial)
        set_material_uniforms()
        draw_quad()

    # Pass 4: Normal Filtering (Optional)
    if flags.filter_normals:
        bind_fbo(_glFBO[3])
        clear()
        use_program(_programNormFilter)
        set_uniform("sigma", 1.0) # Typical value, not explicitly given
        set_uniform("bsigma", 1.0) # Typical value, not explicitly given
        draw_quad()

    # Pass 5: Shadows (Optional)
    if flags.do_shadows:
        bind_fbo(_glFBO[4])
        clear()
        set_uniform("lightPos", light.position)
        set_renderer_mode(OctreeDDARenderer, SHADOW)
        draw_scene(scene)
    else:
        bind_fbo(_glFBO[4])
        clear_to_white() # Default fully lit state

    # Pass 6: SSAO (Optional)
    if flags.do_ssao:
        bind_fbo(_glFBO[5])
        clear()
        use_program(_programSSAO)
        set_uniform("projMat", camera.proj_mat)
        set_uniform("viewMat", camera.view_mat)
        set_uniform("camPos", camera.position)
        set_uniform("radius", 1.0) # Typical value, not explicitly given
        draw_quad()

    # Pass 7: AO Blur
    bind_fbo(_glFBO[6])
    clear()
    use_program(_programBlur)
    draw_quad()

    # Pass 8: Final Shading
    bind_default_fbo()
    clear()
    use_program(_programShading)
    set_uniform("isYUp", True) # Assumed default based on source
    set_uniform("camPos", camera.position)
    set_uniform("projMatInv", camera.proj_mat_inv)
    set_uniform("viewMatInv", camera.view_mat_inv)
    set_uniform("lightPos", light.position)
    set_uniform("renderMode", "FINAL") # Assumed default based on source
    set_uniform("useMaterials", flags.use_materials)
    draw_quad()
```

**Edge Cases and Pitfalls:**
*   When shadow pass is disabled, the shadow buffer must be cleared to white (1.0) to ensure objects are fully lit, rather than leaving undefined values [S1, unofficial].
*   The SSAO pass replaces an older octree-based AO implementation; using the old path may yield incorrect results if not properly disabled [S1, unofficial].
## Operating characteristics
**Time and Memory Complexity:**
*   Time complexity is linear with respect to the number of passes (8 fixed passes) and the screen resolution, as each pass typically involves full-screen quad rasterization or geometry rendering [S1, unofficial].
*   Memory complexity is proportional to the number of textures bound to the FBOs (depth, position, normal, material, filtered normal, shadows, AO, blurred AO, noise, TBO materials) [S1, unofficial].

**Measured Performance:**
*   Not given in the sources.

**Scaling:**
*   Not given in the sources.

**Strengths and Limitations:**
*   Strengths: Decouples geometry and shading, allowing for complex lighting effects (shadows, SSAO) without per-fragment lighting calculations during geometry pass [S1, unofficial].
*   Limitations: Requires multiple passes and large texture memory; performance scales with screen resolution and number of enabled features [S1, unofficial].

**When to use it and when not:**
*   Use when complex lighting and post-processing effects are required and geometry complexity is high [S1, unofficial].
*   Do not use when memory bandwidth is a bottleneck or when simple lighting is sufficient [S1, unofficial].
## Background
**Origin:**
*   The method is implemented in the `ssvdag` project by ajaspe [S1, unofficial].

**History and Development:**
*   The implementation uses an OctreeDDARenderer for geometry traversal and rendering [S1, unofficial].

**Where it is used:**
*   Used in real-time volume rendering applications requiring deferred shading techniques [S1, unofficial].

## Sources
[S1] ajaspe/ssvdag, https://github.com/ajaspe/ssvdag, fetched 2026-10-24 (unofficial)
