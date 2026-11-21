# Screen Quad Rendering

Also called: FBO composition pass, Screen Render Pass, Two-pass rendering pipeline

## Overview
Screen Quad Rendering is the final compositing stage in a two-pass splatting pipeline, where pre-rendered volumetric data stored in off-screen Framebuffer Objects (FBOs) is composited into the default framebuffer [S1, S2, unofficial]. It solves the problem of transferring intermediate rendering results (such as color and auxiliary buffers) from GPU memory to the display surface [G]. It differs from its siblings by explicitly decoupling the volume splatting geometry pass from the final screen composition [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:**
  - `colorTexture`: 2D texture containing the accumulated splat color, bound to unit 0. Format: `GL_RGBA`, type `GL_UNSIGNED_BYTE` [S1, unofficial].
  - `positionTexture`: 2D texture containing auxiliary data (e.g., world-space positions), bound to unit 3. Format: `GL_RGBA32F`, type `GL_FLOAT` [S1, unofficial].
  - `viewportSize`: Vector2 uniform specifying the target resolution [S1, unofficial].
  - `upscale`: Scalar uniform dividing the first-pass texture resolution. Default value 1 [S1, unofficial].
  - `crossSection`: Scalar uniform for debug overlay positioning, clamped to $[0, 1]$ [S1, unofficial].
- **Outputs:**
  - Default framebuffer (screen) with the composited image [S1, unofficial].

**Data Structures:**
The intermediate rendering state is stored in an FBO with two attachments [S1, unofficial]:
1.  `GL_COLOR_ATTACHMENT0`: `colorTexture` (RGBA, Unsigned Byte).
2.  `GL_COLOR_ATTACHMENT3`: `positionTexture` (RGBA32F, Float).
Draw buffers are configured as `drawBuffers[] = {GL_COLOR_ATTACHMENT0, GL_NONE, GL_NONE, GL_COLOR_ATTACHMENT3}` [S1, unofficial].

**Steps:**
1.  **Pass 1 (Splatting):** Render the volume primitives into the FBO described above. The texture resolution is scaled by `upscale` (default 1) [G].
2.  **Pass 2 (Screen Render Pass):**
    a.  Bind the default framebuffer (ID 0) [S1, unofficial].
    b.  Bind `colorTexture` to texture unit 0 and `positionTexture` to texture unit 3 [S1, unofficial].
    c.  Set uniforms: `viewportSize`, `upscale`, and `crossSection` [S1, unofficial].
    d.  Clear the color attachment with `{r:0.0, g:0.0, b:0.0, a:1.0}` and set `loadOp` to 'clear', `storeOp` to 'store' [S2, unofficial].
    e.  Draw a full-screen triangle using the screen pass shader [S1, unofficial].
    f.  End the render pass [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def screen_quad_render_pass(color_texture, position_texture, viewport_size, upscale, cross_section):
    # Clamp cross-section to [0, 1]
    if cross_section < 0:
        cross_section = 0
    if cross_section > 1:
        cross_section = 1

    # Bind default framebuffer (ID 0)
    bind_framebuffer(0)

    # Bind textures to units 0 and 3
    bind_texture(color_texture, unit=0)
    bind_texture(position_texture, unit=3)

    # Set uniforms
    set_uniform("viewportSize", viewport_size)
    set_uniform("upscale", upscale)
    set_uniform("crossSection", cross_section)

    # Begin render pass with clear value {r:0.0, g:0.0, b:0.0, a:1.0}
    begin_render_pass(
        color_attachment=color_texture,
        clear_value=(0.0, 0.0, 0.0, 1.0),
        load_op="clear",
        store_op="store"
    )

    # Draw full-screen triangle
    draw_fullscreen_triangle()

    # End render pass
    end_render_pass()

# Standard graphics pipeline operations
def bind_framebuffer(id): pass
def bind_texture(tex, unit): pass
def set_uniform(name, val): pass
def begin_render_pass(color_attachment, clear_value, load_op, store_op): pass
def draw_fullscreen_triangle(): pass
def end_render_pass(): pass
```

**Edge Cases and Pitfalls:**
- **Wayland Compositing:** On Wayland, forcing output alpha to 1.0 in the fragment shader prevents compositing artifacts [S1, unofficial].
- **Upscale Factor:** If `upscale` > 1, the intermediate textures are smaller than the viewport, requiring filtering during the final draw [S1, unofficial].

## Subtopics

### [Screen-space blit / Resample copy](2563-screen-space-blit-resample-copy.md)
Performs a screen-space copy with injected debug logic and alpha forcing.

### [Fullscreen Triangle Rendering](4230-fullscreen-triangle-rendering.md)
Uses a single oversized triangle to avoid diagonal seams and double-shading artifacts.

## Operating characteristics
- **Time Complexity:** Depends only on screen resolution, not volume size [G].
- **Memory Complexity:** Proportional to the intermediate FBO attachments [S1, unofficial].
- **Strengths:** Decouples volume rendering from display output, enabling post-processing and debug overlays [S1, unofficial].
- **Limitations:** Requires two passes; intermediate FBOs consume VRAM [S1, unofficial].
- **When to use:** When post-processing, debug overlays, or separate auxiliary buffer handling is required [S1, unofficial].

## Background
This method originates from standard OpenGL and WebGPU rendering pipelines [G]. It is implemented in real-time volume visualization applications [G]. It leverages hardware-accelerated rasterization for the final composition step [G].

## Sources
[S1] gabrielmuller/zazen, https://github.com/gabrielmuller/zazen, fetched 2026-10-26 (unofficial)
[S2] dongeronimo/webgpu_volume_renderer_demo, https://github.com/dongeronimo/webgpu_volume_renderer_demo, fetched 2026-10-29 (unofficial)
