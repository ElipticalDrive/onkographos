# Deferred Rendering Pipeline
Also called: 8-Pass Deferred Rendering Pipeline
## Overview
The Deferred Rendering Pipeline is a multi-pass sequence that decouples geometry processing from shading by storing scene attributes (depth, position, normal, material) in separate framebuffer objects (FBOs) before computing lighting effects (shadows, SSAO) and final compositing [S1, unofficial]. It differs from the [Volume Rendering Pipeline](1318-volume-rendering-pipeline.md) by rasterizing explicit geometric primitives (such as an octree) into screen-space buffers rather than integrating volume density along rays, and from the [Fragment List-Based Rendering Pipeline](3488-fragment-list-based-rendering-pipeline.md) by using fixed, sequential passes over full-screen quads rather than dynamic fragment lists [S1, unofficial].
## How it works
**Inputs and Outputs:**
- **Input:** A scene representation (e.g., an octree or DAG4 structure), camera matrices (View, Projection, Inverse View, Inverse Projection), light positions, and configuration flags (materials, shadowing, SSAO, normal filtering) [S1, unofficial].
- **Output:** A final shaded image in the default framebuffer [S1, unofficial].

**Data Structures:**
The pipeline uses a sequence of Framebuffer Objects (FBOs) and associated textures. The specific implementation in [S1, unofficial] defines:
- `_glFBO[0]` to `_glFBO[6]`: Seven FBOs used sequentially.
- `_glTexDepth`: Bound to `_glFBO[0]`, stores depth information.
- `_glTexHitPos`: Stores world-space hit positions.
- `_glTexHitNorm`: Stores surface normals.
- `_glTexHitMaterial`: Stores material properties (albedo, roughness, etc.).
- `_glTexHitNormFiltered`: Stores filtered normals for smoothing.
- `_glTexShadows`: Stores shadow map or shadow accumulation buffer.
- `_glTexAO`: Stores ambient occlusion values.
- `_glTexAOBlurred`: Stores blurred ambient occlusion.
- `_glTexAONoise`: Stores noise texture for SSAO.
- `_glTexTBOMaterials`: Texture for material lookups.
These textures are bound to texture units `TEX_NUM_*` during state reset [S1, unofficial].

**Numbered Steps:**
1. **Pass 1 (Depth):** Bind `_glFBO[0]`, clear buffers. Set the renderer mode (e.g., `OctreeDDARenderer`) to `DEPTH`. Draw the scene geometry (e.g., octree leaves) to populate the depth buffer [S1, unofficial].
2. **Pass 2 (Position/Normal):** Bind `_glFBO[1]`, clear buffers. Use shader program `_programPosNorm`. Set uniforms: `viewMatInv`, `projMatInv`, `useDif`. Draw a full-screen quad to reconstruct world-space position and normal from depth [S1, unofficial].
3. **Pass 3 (Material):** If `_useMaterials` is true: Bind `_glFBO[2]`, clear buffers. Use `_programMaterial`. Set uniforms. Draw a full-screen quad to sample material properties from `_glTexTBOMaterials` or geometry attributes [S1, unofficial].
4. **Pass 4 (Normal Filtering):** If `_filterNormals` is true: Bind `_glFBO[3]`, clear buffers. Use `_programNormFilter`. Set uniforms `sigma` and `bsigma` (filtering radii). Draw a full-screen quad to smooth normals using `_glTexHitNorm` [S1, unofficial].
5. **Pass 5 (Shadows):** If `_doShadows` is true: Bind `_glFBO[4]`, clear buffers. Set `lightPos`. Set renderer mode to `SHADOW`. Draw the scene geometry to compute shadow contributions [S1, unofficial]. Else, clear `_glFBO[4]` to white (1.0), representing full illumination [S1, unofficial].
6. **Pass 6 (SSAO):** If `_doSSAO` is true: Bind `_glFBO[5]`, clear buffers. Use `_programSSAO`. Set uniforms: `projMat`, `viewMat`, `camPos`, `radius`. Draw a full-screen quad to compute screen-space ambient occlusion using `_glTexHitPos`, `_glTexHitNorm`, and `_glTexAONoise` [S1, unofficial].
7. **Pass 7 (AO Blur):** Bind `_glFBO[6]`, clear buffers. Use `_programBlur`. Draw a full-screen quad to blur the AO buffer (`_glTexAO`) for smoother results [S1, unofficial].
8. **Pass 8 (Final Shading):** Bind the default framebuffer, clear buffers. Use `_programShading`. Set uniforms: `isYUp`, `camPos`, `projMatInv`, `viewMatInv`, `lightPos`, `renderMode`, `useMaterials`. Draw a full-screen quad to composite all buffers (depth, position, normal, material, shadow, AO) into the final image [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def deferred_render_pipeline(scene, camera, lights, config):
    # 1. Depth Pass
    bind_fbo(_glFBO[0])
    clear()
    set_renderer_mode(OctreeDDARenderer, DEPTH)
    draw_scene(scene) # Renders to _glTexDepth

    # 2. Position/Normal Pass
    bind_fbo(_glFBO[1])
    clear()
    use_program(_programPosNorm)
    set_uniform("viewMatInv", camera.inv_view_matrix)
    set_uniform("projMatInv", camera.inv_proj_matrix)
    set_uniform("useDif", config.use_diffuse)
    draw_fullscreen_quad() # Reconstructs pos/norm from depth

    # 3. Material Pass
    if config.use_materials:
        bind_fbo(_glFBO[2])
        clear()
        use_program(_programMaterial)
        set_uniforms_for_materials()
        draw_fullscreen_quad() # Samples material data

    # 4. Normal Filtering Pass
    if config.filter_normals:
        bind_fbo(_glFBO[3])
        clear()
        use_program(_programNormFilter)
        set_uniform("sigma", config.normal_filter_sigma)
        set_uniform("bsigma", config.normal_filter_bsigma)
        draw_fullscreen_quad() # Smooths normals

    # 5. Shadow Pass
    if config.do_shadows:
        bind_fbo(_glFBO[4])
        clear()
        set_uniform("lightPos", lights.position)
        set_renderer_mode(OctreeDDARenderer, SHADOW)
        draw_scene(scene) # Computes shadows
    else:
        bind_fbo(_glFBO[4])
        clear_to_white() # Default fully lit

    # 6. SSAO Pass
    if config.do_ssao:
        bind_fbo(_glFBO[5])
        clear()
        use_program(_programSSAO)
        set_uniform("projMat", camera.proj_matrix)
        set_uniform("viewMat", camera.view_matrix)
        set_uniform("camPos", camera.position)
        set_uniform("radius", config.ssao_radius)
        draw_fullscreen_quad() # Computes AO

    # 7. AO Blur Pass
    bind_fbo(_glFBO[6])
    clear()
    use_program(_programBlur)
    draw_fullscreen_quad() # Blurs AO buffer

    # 8. Final Shading Pass
    bind_default_framebuffer()
    clear()
    use_program(_programShading)
    set_uniform("isYUp", config.is_y_up)
    set_uniform("camPos", camera.position)
    set_uniform("projMatInv", camera.inv_proj_matrix)
    set_uniform("viewMatInv", camera.inv_view_matrix)
    set_uniform("lightPos", lights.position)
    set_uniform("renderMode", config.render_mode)
    set_uniform("useMaterials", config.use_materials)
    draw_fullscreen_quad() # Composite and shade
```

**Parameters:**
- `sigma`, `bsigma`: Normal filtering radii. Not given in the sources.
- `radius`: SSAO sampling radius. Not given in the sources.
- `useMaterials`, `filterNormals`, `doShadows`, `doSSAO`: Boolean flags controlling pass execution.

**Edge Cases and Pitfalls:**
- When shadowing is disabled, the shadow buffer is cleared to white (1.0), which acts as a default fully lit state [S1, unofficial].
- The original octree-based AO path is disabled in favor of the screen-space quad pass (wrapped in `#if 0/#else`), so SSAO relies on screen-space reconstruction [S1, unofficial].
- Precision issues may arise in Pass 2 if depth buffer precision is insufficient for accurate world-space reconstruction [G].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of pixels for the full-screen quad passes (Passes 2-4, 6-8) and linear with respect to the number of visible primitives for the geometry passes (Passes 1, 5) [S1, unofficial].
- **Memory Complexity:** Proportional to screen resolution times the number of buffers (depth, position, normal, material, shadow, AO, blurred AO) [S1, unofficial].
- **Strengths:** Decouples shading from geometry complexity; allows efficient reuse of screen-space information for effects like SSAO and normal filtering [S1, unofficial].
- **Limitations:** Requires multiple passes over the screen; suffers from screen-space artifacts (e.g., SSAO bleeding, depth reconstruction errors at edges) [G].
- **When to use:** Real-time rendering where lighting complexity is high but geometry is static or moderately dynamic [S1, unofficial].
- **When not to use:** Scenes with extreme transparency or volumetric effects that require ray-marching integration rather than surface shading [S1, unofficial].

## Background
The deferred rendering pipeline described here originates from the `ssvdag` project by ajaspe, which implements an octree-based volume renderer using a DAG4 structure [S1, unofficial]. It adapts standard deferred shading techniques (depth, position, normal, material, lighting passes) to the specific needs of rendering hierarchical volume data structures, using screen-space reconstruction for lighting and occlusion rather than ray-casting for every pixel [S1, unofficial].

## Sources
[S1] ajaspe/ssvdag, https://github.com/ajaspe/ssvdag, fetched 2026-10-24 (unofficial)
