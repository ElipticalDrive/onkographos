# Screen Quad Rendering

Also called: FBO composition pass, Screen Render Pass, Two-pass rendering pipeline

## Overview
Screen Quad Rendering is the final presentation stage in a volume rendering pipeline, responsible for transferring computed volume data from off-screen render targets to the default framebuffer or current context texture for display [S1, S2, unofficial]. It differs from standard blits by explicitly managing multiple draw buffers and allowing fragment shaders to sample intermediate textures for final compositing or debugging overlays [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Intermediate textures (`colorTexture`, `positionTexture`) and uniforms (`viewportSize`, `upscale`, `xSection`) [S1, unofficial].
*   **Outputs:** The default framebuffer (0) or current context texture containing the final composited image [S1, S2, unofficial].

**Data Structures and Layout:**
The first pass typically writes to a Framebuffer Object (FBO) with specific attachments [S1, unofficial]:
*   `GL_COLOR_ATTACHMENT0`: Bound to `colorTexture`. Format: `GL_RGBA`, Type: `GL_UNSIGNED_BYTE` [S1, unofficial].
*   `GL_COLOR_ATTACHMENT3`: Bound to `positionTexture`. Format: `GL_RGBA32F`, Type: `GL_FLOAT` [S1, unofficial].
*   **Draw Buffers:** The FBO is configured to write to `GL_COLOR_ATTACHMENT0` and `GL_COLOR_ATTACHMENT3` simultaneously, with other attachments disabled: `drawBuffers[] = {GL_COLOR_ATTACHMENT0, GL_NONE, GL_NONE, GL_COLOR_ATTACHMENT3}` [S1, unofficial].

**Steps:**
1.  **Initialize Render Pass:** Begin a render pass (e.g., 'Screen Render Pass') targeting the current context texture, with clear value `{r:0.0, g:0.0, b:0.0, a:1.0}`, `loadOp: 'clear'`, and `storeOp: 'store'` [S2, unofficial].
2.  **Bind Resources:** Bind `colorTexture` to texture unit 0 and `positionTexture` to texture unit 3 [S1, unofficial].
3.  **Set Uniforms:** Set `viewportSize`, `upscale` (default 1), and `xSection` (clamped to [0, 1]) [S1, unofficial].
4.  **Render Geometry:** Draw a fullscreen primitive (quad or triangle) using the configured pipeline [S1, S2, unofficial].
5.  **End Pass:** Finalize the render pass [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def screen_quad_render_pass(context, color_texture, position_texture, width, height, upscale=1, x_section=0.5):
    # Clamp x_section to [0, 1]
    if x_section < 0:
        x_section = 0
    if x_section > 1:
        x_section = 1

    # 1. Begin Render Pass
    context.begin_render_pass(
        attachments=[{
            'texture': context.get_current_texture(),
            'clear_value': (0.0, 0.0, 0.0, 1.0),
            'load_op': 'clear',
            'store_op': 'store'
        }]
    )

    # 2. Bind Textures to Units
    context.bind_texture(unit=0, texture=color_texture)
    context.bind_texture(unit=3, texture=position_texture)

    # 3. Set Uniforms
    context.set_uniform('viewportSize', (width, height))
    context.set_uniform('upscale', upscale)
    context.set_uniform('xSection', x_section)

    # 4. Draw Fullscreen Geometry
    context.render(pipeline='gQuadRendererPipeline')

    # 5. End Render Pass
    context.end_render_pass()
```

**Edge Cases and Pitfalls:**
*   **Upscale Factor:** The `upscale` parameter divides the first pass texture resolution (`width / upscale`, `height / upscale`), affecting memory usage and performance [S1, unofficial].
*   **Clamping:** Debug parameters like `xSection` must be clamped to valid ranges to prevent out-of-bounds sampling [S1, unofficial].

## Subtopics

### [Screen-space blit / Resample copy](2563-screen-space-blit-resample-copy.md)
This method performs a screen-space copy of a source texture into a target render target while overlaying a debug crosshair. It solves the problem of transferring data between render targets without CPU intervention and provides a visual aid for screen-space alignment. It differs from a standard blit by injecting specific pixel-level logic (the crosshair) and forcing alpha to 1.0 to prevent compositing artifacts on certain windowing systems [G].

### [Fullscreen Triangle Rendering](4230-fullscreen-triangle-rendering.md)
This method displays an off-screen render texture onto the entire viewport using a dedicated shader. It differs from sibling methods by generating a single oversized triangle via `gl_VertexID` to avoid diagonal seams and double-shading artifacts inherent in standard quad strip approaches [G].

## Operating characteristics
*   **Time Complexity:** O(N) relative to screen resolution (N = width × height), as every pixel is processed once [G].
*   **Performance:** Highly efficient on modern GPUs due to fixed-function rasterization of a single primitive [G].
*   **When to Use:** Final compositing stage in volume rendering where intermediate data needs to be presented [S1, S2, unofficial].
*   **When Not to Use:** When single-pass rendering is required or when the output does not need to be displayed on the screen [G].

## Background
Used in volume rendering pipelines to separate the heavy computation of volume integration from the final presentation [S1, S2, unofficial]. The use of fullscreen triangles to avoid quad diagonal seams is a common optimization in shader programming [G].

## Sources
[S1] gabrielmuller/zazen, https://github.com/gabrielmuller/zazen, fetched 2026-10-26 (unofficial)
[S2] dongeronimo/webgpu_volume_renderer_demo, https://github.com/dongeronimo/webgpu_volume_renderer_demo, fetched 2026-10-29 (unofficial)
