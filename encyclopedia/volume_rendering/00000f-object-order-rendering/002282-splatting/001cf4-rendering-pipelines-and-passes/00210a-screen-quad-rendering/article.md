# Screen Quad Rendering

Also called: FBO composition pass, Screen Render Pass, Two-pass rendering pipeline

## Overview
Screen Quad Rendering is the final compositing stage in a two-pass volume rendering pipeline, responsible for transferring computed volume data (such as color and position) from off-screen render targets to the default framebuffer for display [S1, S2, unofficial]. It solves the problem of presenting the rendered result to the user by drawing a geometry that covers the entire screen, allowing fragment shaders to sample the intermediate textures and apply final tone mapping or debugging overlays [S1, S2, unofficial]. It differs from standard blits by often injecting specific pixel-level logic (like debug crosshairs or tone mapping) and explicitly managing draw buffers to handle multiple attachments [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Intermediate textures generated in the first pass (e.g., `colorTexture` for RGBA color, `positionTexture` for 3D position or depth). Uniforms including `viewportSize` (resolution), `upscale` factor, and optional debug parameters like `xSection` [S1, S2, unofficial].
*   **Outputs:** The default framebuffer (screen) containing the final composited image [S1, S2, unofficial].

**Data Structures and Layout:**
The first pass typically writes to a Framebuffer Object (FBO) with specific attachments [S1, unofficial]:
*   `GL_COLOR_ATTACHMENT0`: Bound to `colorTexture`. Format: `GL_RGBA`, Type: `GL_UNSIGNED_BYTE` [S1, unofficial].
*   `GL_COLOR_ATTACHMENT3`: Bound to `positionTexture`. Format: `GL_RGBA32F`, Type: `GL_FLOAT` [S1, unofficial].
*   **Draw Buffers:** The FBO is configured to write to `GL_COLOR_ATTACHMENT0` and `GL_COLOR_ATTACHMENT3` simultaneously, with other attachments disabled: `drawBuffers[] = {GL_COLOR_ATTACHMENT0, GL_NONE, GL_NONE, GL_COLOR_ATTACHMENT3}` [S1, unofficial].

**Steps:**
1.  **Initialize Render Pass:** Begin a render pass (e.g., 'Screen Render Pass') targeting the current context's texture [S2, unofficial].
2.  **Clear Attachment:** Clear the color attachment with `{r:0.0, g:0.0, b:0.0, a:1.0}` using `loadOp: 'clear'` and `storeOp: 'store'` [S2, unofficial].
3.  **Bind Resources:** Bind the intermediate textures to specific texture units (e.g., `colorTexture` at unit 0, `positionTexture` at unit 3) [S1, unofficial]. Set uniform values: `viewportSize` (width/height), `upscale` (default 1), and `xSection` (clamped to [0, 1]) [S1, S2, unofficial].
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
    # Target: current context texture
    # Clear: {r:0.0, g:0.0, b:0.0, a:1.0}
    # LoadOp: clear, StoreOp: store
    context.begin_render_pass(
        attachments=[{
            'texture': context.get_current_texture(),
            'clear_value': (0.0, 0.0, 0.0, 1.0),
            'load_op': 'clear',
            'store_op': 'store'
        }]
    )

    # 2. Bind Textures to Units
    # colorTexture at unit 0
    context.bind_texture(unit=0, texture=color_texture)
    # positionTexture at unit 3
    context.bind_texture(unit=3, texture=position_texture)

    # 3. Set Uniforms
    context.set_uniform('viewportSize', (width, height))
    context.set_uniform('upscale', upscale)
    context.set_uniform('xSection', x_section)

    # 4. Draw Fullscreen Geometry
    # Using the pre-configured screen pass pipeline
    context.draw_fullscreen_quad(pipeline='gQuadRendererPipeline')

    # 5. End Render Pass
    context.end_render_pass()
```

**Edge Cases and Pitfalls:**
*   **Upscale Factor:** The `upscale` parameter divides the texture resolution (`width / upscale`, `height / upscale`) for the first pass, affecting memory usage and performance [S1, unofficial].
*   **Clamping:** Debug parameters like `xSection` must be clamped to valid ranges to prevent out-of-bounds sampling [S1, unofficial].
*   **Alpha Handling:** Some variants force alpha to 1.0 to prevent compositing artifacts on certain windowing systems [S2563].

## Subtopics

### [Screen-space blit / Resample copy](2563-screen-space-blit-resample-copy.md)
This method performs a screen-space copy (blit) of a source texture (typically a compute-shader output) into a target render target, while simultaneously overlaying a debug crosshair. It solves the problem of transferring data between render targets without CPU intervention and providing a visual aid for screen-space alignment. It differs from a standard blit by injecting specific pixel-level logic (the crosshair) and forcing alpha to 1.0 to prevent compositing artifacts on certain windowing systems.

### Fullscreen Triangle Rendering
Rendering a fullscreen quad using a single oversized triangle generated from `gl_VertexID`, avoiding diagonal seams and double-shading [S3, unofficial].
*   **UV Computation:** `uv = vec2((gl_VertexID << 1) & 2, gl_VertexID & 2)` [S3, unofficial].
*   **Clip Position:** `gl_Position = vec4(uv * 2.0 - 1.0, 0.0, 1.0)` [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** O(1) relative to scene complexity; O(N) relative to screen resolution (N = width * height), as every pixel is processed once [S1, S2, unofficial].
*   **Memory Complexity:** O(1) additional memory beyond the intermediate textures [S1, unofficial].
*   **Performance:** Highly efficient on modern GPUs due to fixed-function rasterization of a single primitive [S3, unofficial].
*   **Strengths:** Simple implementation, low overhead, flexible for post-processing effects [S1, S2, unofficial].
*   **Limitations:** Requires two-pass architecture; not suitable for single-pass ray marching without significant modification [S1, S2, unofficial].
*   **When to Use:** Final compositing stage in offline or real-time volume rendering where intermediate data needs to be presented [S1, S2, unofficial].
*   **When Not to Use:** When single-pass rendering is required or when the output does not need to be displayed on the screen [S1, unofficial].

## Background
Originating from standard OpenGL and WebGPU rendering practices, this method is a fundamental component of two-pass rendering pipelines [S1, S2, unofficial]. It is widely used in volume rendering demos and applications to separate the heavy computation of volume integration from the final presentation [S1, S2, unofficial]. The use of fullscreen triangles to avoid quad diagonal seams is a common optimization in shader programming [S3, unofficial].

## Sources
[S1] gabrielmuller/zazen, https://github.com/gabrielmuller/zazen, fetched 2026-10-26 (unofficial)
[S2] dongeronimo/webgpu_volume_renderer_demo, https://github.com/dongeronimo/webgpu_volume_renderer_demo, fetched 2026-10-29 (unofficial)
[S3] PIGJET/neon-raymarcher, https://github.com/PIGJET/neon-raymarcher, fetched 2026-10-27 (unofficial)
