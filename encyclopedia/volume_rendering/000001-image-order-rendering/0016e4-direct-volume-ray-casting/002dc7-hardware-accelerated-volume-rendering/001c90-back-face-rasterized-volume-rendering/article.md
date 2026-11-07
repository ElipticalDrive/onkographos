# Back-face rasterized volume rendering

Also called: Back-Face Entry Point Method, Backface Rendering for Ray Casting, Backface Culling for Ray Casting Entry Points

## Overview
Back-face rasterized volume rendering is a GPU-accelerated technique that determines ray entry and exit points for volume traversal by rasterizing the volume's bounding box geometry, thereby avoiding expensive explicit ray-box intersection calculations in the final rendering pass [S1, S2, S3, S4, unofficial]. It solves the problem of efficiently identifying which pixels correspond to rays intersecting the volume and correctly computing the start and end coordinates for raymarching, while handling zooming scenarios where front faces might project behind the camera [S2, S3, unofficial]. It differs from [Bounding Box Raymarching](2071-bounding-box-raymarching.md) by explicitly utilizing a two-pass approach where the first pass captures back-face coordinates into a texture, and the second pass uses these coordinates to define ray endpoints for front-to-back compositing [S1, S2, S3, unofficial]. Unlike [Volume Ray Casting with RayData Texture](3077-volume-ray-casting-with-raydata-texture.md), which stores ray data in a dedicated texture, this method often computes ray directions from geometry attributes or camera matrices directly in the fragment shader [S5, unofficial].

## How it works
The method employs a two-pass rendering pipeline. The first pass renders the back faces of the volume's bounding box to a 2D texture to store the ray entry (or exit, depending on culling convention) points. The second pass renders the front faces, retrieves the corresponding back-face points from the texture, and performs the volume raycasting integration using the computed ray segment [S1, S2, S3, S4, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Volume data $V(\mathbf{x})$ stored in a 3D texture (`sampler3D`) [S1, S2, S4, unofficial].
    *   Transfer function mapping scalar values to color and opacity, typically a 1D or 2D texture (`sampler1D` or `sampler2D`) [S1, S2, S4, unofficial].
    *   Camera parameters: projection matrix, view matrix, and eye position [S2, S4, unofficial].
    *   Volume bounding box geometry, typically a unit cube $[0,1]^3$ scaled to volume dimensions [S2, S4, unofficial].
*   **Outputs:**
    *   Final 2D image (color buffer) with composited volume data [S1, S3, S4, unofficial].

### Data Structures
1.  **Bounding Box Geometry:**
    *   Vertices: 8 vertices defining a cube in object coordinates $[0,1]^3$: $(0,0,0), (0,0,1), (0,1,0), (0,1,1), (1,0,0), (1,0,1), (1,1,0), (1,1,1)$ [S4, S3, unofficial].
    *   Indices: 36 indices defining 12 triangles covering the cube surface [S4, S3, unofficial].
2.  **Back-Face Texture (`g_bfTexObj` / `u_ExitPoints`):**
    *   Type: 2D texture (`sampler2D`) [S1, S4, unofficial].
    *   Dimensions: Width and height equal to the viewport resolution ($g\_winWidth \times g\_winHeight$) [S4, S1, unofficial].
    *   Internal Format: `GL_RGBA16F` (16-bit floating point per channel) to preserve precision of 3D coordinates [S4, S1, unofficial].
    *   Pixel Format: `GL_RGBA`, Type: `GL_FLOAT` [S4, unofficial].
    *   Content: Stores the 3D position of the back-face intersection point as RGB color values, with alpha set to 1.0 [S1, S4, unofficial].
3.  **Frame Buffer Object (FBO):**
    *   Attachments: Color attachment bound to `g_bfTexObj` (`GL_COLOR_ATTACHMENT0`), Depth attachment bound to a renderbuffer (`GL_DEPTH_ATTACHMENT`) [S3, S4, unofficial].

### Numbered Steps

**Pass 1: Capture Back-Face Entry/Exit Points**
1.  Create and bind a Frame Buffer Object (FBO) attached to the 2D texture `g_bfTexObj` [S4, S3, unofficial].
2.  Set the viewport to the window dimensions $(0, 0, g\_winWidth, g\_winHeight)$ [S1, S4, unofficial].
3.  Enable face culling and set `glCullFace` to `GL_FRONT` to render only the back faces of the bounding box [S1, S4, unofficial].
4.  Execute the back-face vertex and fragment shaders:
    *   **Vertex Shader:** Transforms the bounding box vertices using the Model-View-Projection (MVP) matrix. Outputs the vertex position in object/local coordinates as `fragPos` [S4, S1, unofficial].
    *   **Fragment Shader:** Writes `fragPos` (the 3D coordinate) as the fragment color into the FBO texture `g_bfTexObj` [S4, S1, unofficial].
5.  Unbind the FBO and restore the default framebuffer [S1, S4, unofficial].

**Pass 2: Raycasting and Compositing**
1.  Bind the default framebuffer [S1, S4, unofficial].
2.  Enable face culling and set `glCullFace` to `GL_BACK` to render only the front faces of the bounding box [S1, S4, unofficial].
3.  Execute the raycasting vertex and fragment shaders:
    *   **Vertex Shader:** Outputs the current vertex position (front face) as `EntryPoint` (or `frontColor`) [S1, S4, unofficial].
    *   **Fragment Shader:**
        1.  Sample the back-face texture `g_bfTexObj` using the interpolated fragment coordinates (normalized by resolution) to retrieve the `ExitPoint` (or `backColor`) [S1, S4, unofficial].
        2.  Calculate the ray direction: $\mathbf{dir} = \text{ExitPoint} - \text{EntryPoint}$ [S1, S4, unofficial].
        3.  Calculate the ray length: $L = \|\mathbf{dir}\|$ [S4, unofficial].
        4.  If $L \approx 0$ (entry equals exit, e.g., tangent ray), discard the fragment to avoid division by zero [S4, unofficial].
        5.  Compute the step vector: $\mathbf{step} = \frac{\mathbf{dir}}{N}$, where $N$ is the number of sampling steps (or compute step size based on voxel dimensions) [S1, S2, S4, unofficial].
        6.  Initialize accumulation variables: $C_{acc} = (0,0,0,0)$, $\alpha_{acc} = 0$, current position $\mathbf{p} = \text{EntryPoint}$ [S1, S4, unofficial].
        7.  Loop for $i = 0$ to $N-1$:
            a.  Sample volume value $v = V(\mathbf{p})$ [S1, S4, unofficial].
            b.  Map $v$ to color $C_i$ and opacity $\alpha_i$ using the transfer function [S1, S4, unofficial].
            c.  Apply front-to-back compositing:
                $$
                C_{acc} = C_{acc} + (1 - \alpha_{acc}) \alpha_i C_i
                $$
                $$
                \alpha_{acc} = \alpha_{acc} + (1 - \alpha_{acc}) \alpha_i
                $$
            d.  Advance position: $\mathbf{p} = \mathbf{p} + \mathbf{step}$ [S1, S4, unofficial].
            e.  **Early Termination:** If $\alpha_{acc} \ge 1.0$ (or a threshold like 0.95), break the loop [S1, S2, S3, S4, unofficial].
            f.  **Boundary Check:** If $\mathbf{p}$ goes outside the volume bounds $[0,1]^3$, break the loop [S1, unofficial].
        8.  Output $C_{acc}$ as the final pixel color [S1, S4, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pass 1: Back-Face Capture
def pass1_capture_backface():
    bind_fbo(g_frameBuffer, texture=g_bfTexObj)
    set_viewport(0, 0, g_winWidth, g_winHeight)
    enable_cull_face()
    set_cull_face(GL_FRONT) # Render only back faces
    
    use_shader(backface_vert, backface_frag)
    set_uniform("MVP", projection * view * model)
    
    draw_box() # Draws the bounding box cube
    
    unbind_fbo()

# Back-Face Fragment Shader Logic
def backface_frag_shader(frag_pos):
    # frag_pos is the vertex position in object coordinates
    write_color_to_texture(g_bfTexObj, vec4(frag_pos, 1.0))

# Pass 2: Raycasting
def pass2_raycast():
    bind_default_framebuffer()
    enable_cull_face()
    set_cull_face(GL_BACK) # Render only front faces
    
    use_shader(raycast_vert, raycast_frag)
    set_uniform("u_ExitPoints", g_bfTexObj)
    set_uniform("u_VolumeTex", volume_texture)
    set_uniform("u_TransferFunc", transfer_function)
    set_uniform("u_Resolution", vec2(g_winWidth, g_winHeight))
    set_uniform("uSteps", 256) # Typical value
    
    draw_box() # Draws the bounding box cube

# Raycasting Fragment Shader Logic
def raycast_frag_shader(entry_point):
    # entry_point is the front face vertex position
    # Retrieve exit point from the texture sampled at fragment coordinates
    exit_point = texture(g_bfTexObj, gl_FragCoord.xy / u_Resolution).xyz
    
    if length(exit_point - entry_point) < epsilon:
        discard()
        
    ray_dir = exit_point - entry_point
    ray_len = length(ray_dir)
    step_vec = ray_dir / uSteps
    
    accum_color = vec4(0.0)
    accum_alpha = 0.0
    current_pos = entry_point
    
    for i in range(uSteps):
        # Sample volume
        vol_val = texture(u_VolumeTex, current_pos).r
        
        # Transfer function
        tf_sample = texture(u_TransferFunc, vol_val)
        c_i = tf_sample.rgb
        a_i = tf_sample.a
        
        # Compositing
        if a_i > 0.0:
            accum_color.rgb += (1.0 - accum_alpha) * a_i * c_i
            accum_alpha += (1.0 - accum_alpha) * a_i
            
        # Advance
        current_pos += step_vec
        
        # Early termination
        if accum_alpha >= 1.0:
            break
            
        # Boundary check (optional if box is tight)
        if not is_inside_volume(current_pos):
            break
            
    write_color(accum_color)
```

### Parameters
*   **uSteps:** Number of sampling steps along the ray. Typical value: 256 [S1, unofficial].
*   **uOpacityVal:** Opacity scaling factor. Typical value: 4.0 [S1, unofficial].
*   **uColorVal:** Color scaling factor. Typical value: 1.0 [S1, unofficial].
*   **uMinGrayVal / uMaxGrayVal:** Minimum and maximum volume value thresholds for sampling. Typical values: 0.001 and 1.0 [S1, unofficial].
*   **uAbsorptionModeIndex:** Switches between rendering modes: 0.0 (Simple accumulation), 1.0 (Standard alpha blending), 2.0 (Max projection) [S1, unofficial].
*   **Geometry Scale:** The bounding box is often a unit cube scaled by `volume_scale` to support non-uniform volumes [S2, unofficial].

### Edge Cases and Pitfalls
*   **Zooming:** Rendering only front faces causes issues when zooming in, as front faces project behind the camera and are clipped, leaving pixels unrendered. Rendering back faces avoids this [S2, unofficial].
*   **Tangent/Missing Rays:** If the entry point equals the exit point (ray misses or is tangent to the box), the ray length is zero. The shader must discard the fragment to avoid division by zero or infinite loops [S4, unofficial].
*   **Loop Termination:** Some implementations use a large constant loop bound (e.g., 4095) with an explicit break condition (`i == uSteps`) due to GLSL limitations [S1, unofficial].
*   **Precision:** Using `GL_RGBA16F` for the back-face texture helps maintain precision for 3D coordinates, but 32-bit floats may be needed for very large volumes or high precision requirements [S4, unofficial].
*   **Compositing Modes:** Some variants support different compositing modes (e.g., 'mida'), but these may not work correctly in all implementations [S3, unofficial].

## Subtopics

### [Bounding Box Raymarching](2071-bounding-box-raymarching.md)
A variant that explicitly utilizes a two-pass approach where the first pass captures back-face coordinates into a texture, and the second pass uses these coordinates to define ray endpoints for front-to-back compositing. It differs from other methods by avoiding explicit ray-box intersection math in the fragment shader for exit determination.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot R)$, where $N$ is the number of pixels in the viewport and $R$ is the number of sampling steps per ray. The two-pass approach adds a constant overhead for the back-face rendering pass.
*   **Memory Complexity:** $O(W \cdot H)$ for the back-face texture, where $W$ and $H$ are the viewport dimensions. The volume data itself is $O(D_x \cdot D_y \cdot D_z)$ [S4, S1, unofficial].
*   **Performance:** Efficient for large volumes as it restricts fragment processing to pixels intersecting the volume bounding box, avoiding unnecessary processing for pixels outside the volume [S2, unofficial]. Performance scales with viewport resolution and number of steps.
*   **Strengths:**
    *   Avoids complex ray-box intersection calculations in the fragment shader [S3, S4, unofficial].
    *   Handles zooming correctly by rendering back faces [S2, unofficial].
    *   Simple implementation using standard rasterization primitives [S1, S4, unofficial].
*   **Limitations:**
    *   Requires two render passes, increasing overhead compared to single-pass methods [S3, S4, unofficial].
    *   Limited to convex bounding boxes (typically axis-aligned) [S2, S4, unofficial].
    *   Precision of entry/exit points depends on the texture format used [S4, unofficial].
*   **When to use:** Suitable for real-time volume rendering on GPUs where simplicity and correctness with zooming are prioritized over single-pass efficiency. Ideal for interactive applications [S2, S4, unofficial].
*   **When not to use:** When single-pass performance is critical and ray-box intersection is cheap, or for non-convex volumes where bounding box approximation is insufficient [S3, S4, unofficial].

## Background
The method originated as a practical solution for GPU-based volume rendering, leveraging the rasterization pipeline to determine ray bounds. It is commonly implemented in educational and open-source volume rendering projects using OpenGL and WebGL [S1, S2, S3, S4, unofficial]. The technique is described in various online tutorials and code repositories focusing on WebGL and OpenGL volume rendering [S2, S5, unofficial].

## Sources
[S1] levl-dev/VolumeRayCasting_0, https://github.com/levl-dev/VolumeRayCasting_0, fetched 2026-10-26 (unofficial)
[S2] Volume Rendering with WebGL, https://www.willusher.io/webgl/2019/01/13/volume-rendering-with-webgl, fetched 2026-10-26 (unofficial)
[S3] fargiolas/qvrc, https://github.com/fargiolas/qvrc, fetched 2026-10-26 (unofficial)
[S4] thePZH/VolumeRendering-OpenGL, https://github.com/thePZH/VolumeRendering-OpenGL, fetched 2026-10-26 (unofficial)
[S5] GPU-accelerated single-pass volumetric raycasting in Qt ..., https://martinopilia.com/posts/2018/09/17/volume-raycasting.html, fetched 2026-10-27 (unofficial)
