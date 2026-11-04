# Fullscreen Triangle Rendering

Also called: Fullscreen Texture Blit, Post-Process Quad, Screen Quad Texture Display, gl_VertexID fullscreen quad

## Overview
Fullscreen Triangle Rendering displays an off-screen render texture onto the entire viewport using a dedicated shader, enabling HUD overlays and screenshot captures [S1, unofficial]. It differs from sibling methods by generating a single oversized triangle via `gl_VertexID` to avoid diagonal seams and double-shading artifacts inherent in standard quad strip approaches [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A source texture (e.g., render target) and viewport dimensions (width, height).
- Output: The color buffer of the default framebuffer (screen).

**Data Structures:**
- **Vertex Layout:** If using explicit vertex buffers, the layout consists of 4 vertices. Each vertex contains 3 position floats and 2 texture coordinate floats, totaling 20 floats per quad [S1, unofficial].
- **Vertex Positions (Explicit):**
  - Vertex 0: `(-1, -1, 0, 0, 0)`
  - Vertex 1: `(-1, 1, 0, 0, 1)`
  - Vertex 2: `(1, -1, 0, 1, 0)`
  - Vertex 3: `(1, 1, 0, 1, 1)` [S1, unofficial]
- **Vertex Layout (gl_VertexID):** No explicit VBO is required. The vertex shader computes position and UV based on the built-in `gl_VertexID` [S2, unofficial].

**Steps:**
1. Bind the default framebuffer (ID 0) [S1, unofficial].
2. Clear the color buffer [S1, unofficial].
3. Bind the source render texture [S1, unofficial].
4. Bind the fullscreen triangle shader program [S1, unofficial].
5. Set uniforms: `screenWidth`, `screenHeight`, and any additional parameters (e.g., `crosshairType`) [S1, unofficial].
6. Draw the geometry using `GL_TRIANGLE_STRIP` with 4 vertices [S1, unofficial].

**Formulas:**
For the `gl_VertexID` approach [S2, unofficial]:
1. Compute UV coordinates:
   $$ uv = \text{vec2}((\text{gl\_VertexID} \ll 1) \& 2, \text{gl\_VertexID} \& 2) $$
2. Compute clip-space position:
   $$ \text{gl\_Position} = \text{vec4}(uv \times 2.0 - 1.0, 0.0, 1.0) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fullscreen_triangle_render(source_texture, width, height):
    # 1. Bind default framebuffer
    bind_framebuffer(0)
    
    # 2. Clear color buffer
    clear_color_buffer()
    
    # 3. Bind source texture
    bind_texture(source_texture, unit=0)
    
    # 4. Bind shader
    use_shader(fullscreen_triangle_shader)
    
    # 5. Set uniforms
    set_uniform("screenWidth", width)
    set_uniform("screenHeight", height)
    
    # 6. Draw using triangle strip
    # If using explicit VBO:
    # bind_vao(fullscreen_quad_vao)
    # draw_arrays(GL_TRIANGLE_STRIP, 0, 4)
    
    # If using gl_VertexID (no VBO needed):
    draw_arrays(GL_TRIANGLE_STRIP, 0, 4)
```

**Edge Cases and Pitfalls:**
- **Diagonal Seams:** Standard quad rendering can cause seams if the diagonal is shared incorrectly. The single-triangle approach avoids this [S2, unofficial].
- **Double-Shading:** Overlapping triangles in quad strips can cause double-shading. The triangle strip with 4 vertices ensures each pixel is shaded exactly once [S2, unofficial].
- **Aspect Ratio:** If the source texture does not match the viewport aspect ratio, distortion occurs. The shader should handle aspect correction if necessary [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ relative to scene complexity; depends only on screen resolution.
- **Memory Complexity:** $O(1)$ additional memory for vertex data (if using explicit VBO) or $O(0)$ if using `gl_VertexID`.
- **Strengths:** Simple implementation, no vertex buffer management required with `gl_VertexID`, avoids diagonal seams.
- **Limitations:** Limited to displaying a single texture; not suitable for complex geometry or multiple textures without additional passes.
- **When to Use:** Post-processing, HUD overlays, screenshot capture.
- **When Not to Use:** Complex 3D scene rendering, multi-texture blending in a single pass.

## Background
Fullscreen Triangle Rendering is a common technique in real-time graphics for displaying textures on the screen. It is often used in game engines and graphics applications for post-processing effects and UI rendering. The `gl_VertexID` approach is a modern optimization that reduces vertex buffer overhead [S2, unofficial].

## Sources
[S1] EmaMaker/voxel-engine, https://github.com/EmaMaker/voxel-engine, fetched 2026-10-26 (unofficial)
[S2] PIGJET/neon-raymarcher, https://github.com/PIGJET/neon-raymarcher, fetched 2026-10-27 (unofficial)
