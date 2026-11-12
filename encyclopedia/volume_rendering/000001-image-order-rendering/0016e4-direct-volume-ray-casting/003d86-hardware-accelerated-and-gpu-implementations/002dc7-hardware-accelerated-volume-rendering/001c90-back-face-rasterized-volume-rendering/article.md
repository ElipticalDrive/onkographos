# Back-face rasterized volume rendering

Also called: Back-Face Entry Point Method, Backface Rendering for Ray Casting, Backface Culling for Ray Casting Entry Points, Two-Pass Ray Setup, Ray Origin/Direction Texture Mapping

## Overview
Back-face rasterized volume rendering is a GPU-accelerated technique that determines ray entry and exit points by rasterizing the volume's bounding box geometry, avoiding explicit ray-box intersection calculations in the fragment shader [S2, unofficial][S3, unofficial][S6, unofficial]. It solves the problem of efficiently identifying which pixels require ray marching and establishing precise start/end coordinates for the volume traversal [S2, unofficial][S3, unofficial]. It differs from [3D Texture Mapping](9529-3d-texture-mapping.md) by using screen-space geometry rasterization to drive the ray setup rather than computing ray parameters directly from camera matrices in the fragment shader [S3, unofficial]. It differs from [Entry Point Adjustment](7406-entry-point-adjustment.md) by relying on the bounding box silhouette rather than block-level visibility culling [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D volume texture ($V$), transfer function texture ($TF$), camera projection/view matrices, bounding box geometry (unit cube or scaled cube).
*   **Outputs:** Final 2D image buffer with accumulated color and opacity.

**Data Structures:**
1.  **Bounding Box Geometry:** A cube defined by 8 vertices in object coordinates $[0,1]^3$ and 36 indices forming 12 triangles [S6, unofficial].
2.  **Back-Face Texture ($T_{back}$):** A 2D texture (resolution $W \times H$, format `GL_RGBA16F` or similar) storing the 3D entry/exit coordinates (or ray origins) derived from the back-face rasterization [S2, unofficial][S6, unofficial].
3.  **Frame Buffer Object (FBO):** Attached to $T_{back}$ for the first pass, and the default framebuffer for the second pass [S5, unofficial][S6, unofficial].

**Numbered Steps:**

1.  **Initialize Textures and FBO:** Create a 2D texture $T_{back}$ with width $W$ and height $H$ (screen resolution). Create an FBO and attach $T_{back}$ as the color attachment [S6, unofficial].
2.  **Pass 1: Back-Face Rasterization (Ray Origin/Exit Point):**
    *   Bind the FBO.
    *   Enable face culling and set `glCullFace` to `GL_FRONT` (to render only back faces) [S2, unofficial][S6, unofficial].
    *   Render the bounding box geometry using a vertex shader that outputs the vertex position (in object or world space) as the fragment color [S2, unofficial][S6, unofficial].
    *   The fragment shader writes this position to the color attachment ($T_{back}$) [S6, unofficial].
    *   Unbind the FBO [S2, unofficial].
3.  **Pass 2: Front-Face Rasterization and Ray Marching:**
    *   Bind the default framebuffer.
    *   Enable face culling and set `glCullFace` to `GL_BACK` (to render only front faces) [S2, unofficial][S6, unofficial].
    *   Bind $T_{back}$ as a uniform sampler (e.g., `uBackCoord`) [S2, unofficial].
    *   Render the bounding box geometry using the raycasting vertex/fragment shaders [S2, unofficial].
    *   **Vertex Shader:** Outputs the current vertex position (front face) as `frontColor` [S2, unofficial].
    *   **Fragment Shader:**
        *   Sample $T_{back}$ using the interpolated texture coordinates (or `gl_FragCoord`) to retrieve the back-face position (`backColor`) [S2, unofficial][S6, unofficial].
        *   Compute ray direction: $\vec{d} = \text{backColor}.rgb - \text{frontColor}.rgb$ [S2, unofficial].
        *   Compute ray length: $L = \|\vec{d}\|$ [S4].
        *   If $L \approx 0$ (ray misses or is tangent), discard the fragment [S6, unofficial].
        *   Normalize direction: $\vec{v} = \vec{d} / L$ [S2, unofficial].
        *   Calculate step size: $\Delta = L / N$, where $N$ is the number of steps (e.g., 256) [S2, unofficial][S4].
        *   Initialize accumulation variables: $C_{acc} = (0,0,0,0)$, $\alpha_{acc} = 0$, current position $p = \text{frontColor}.rgb$ [S2, unofficial].
        *   **Ray Marching Loop:** For $i = 0$ to $N-1$:
            1.  Sample volume value: $val = \text{texture}(V, p.xyz)$ [S2, unofficial].
            2.  If $val$ is outside $[uMinGrayVal, uMaxGrayVal]$, set opacity $\alpha = 0$ [S2, unofficial].
            3.  Else, lookup color $C$ and opacity $\alpha$ from Transfer Function $TF$ [S2, unofficial].
            4.  Apply compositing:
                $$ C_{acc} = C_{acc} + (1 - \alpha_{acc}) \alpha C $$
                $$ \alpha_{acc} = \alpha_{acc} + (1 - \alpha_{acc}) \alpha $$
            5.  Advance position: $p.xyz += \vec{v} \cdot \Delta$ [S2, unofficial].
            6.  **Early Ray Termination:** If $\alpha_{acc} \ge 1.0$ (or a threshold like 0.95), break loop [S2, unofficial][S3, unofficial].
            7.  **Boundary Check:** If $p$ goes outside $[0,1]$ bounds, break loop [S2, unofficial].
        *   Output $C_{acc}$ and $\alpha_{acc}$ to the framebuffer [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pass 1: Back-Face Rasterization
def pass1_backface_rasterization():
    bind_fbo(g_framebuffer, g_bfTexObj)
    set_viewport(0, 0, g_winWidth, g_winHeight)
    enable_culling(GL_FRONT) # Cull front faces, render back faces
    use_shader(g_bf_vert, g_bf_frag)
    draw_box() # Draws bounding box
    unbind_fbo()

# Pass 2: Front-Face Rasterization and Ray Marching
def pass2_frontface_raycasting():
    bind_default_fbo()
    enable_culling(GL_BACK) # Cull back faces, render front faces
    use_shader(g_rc_vert, g_rc_frag)
    set_uniform("uBackCoord", g_bfTexObj)
    set_uniform("uSteps", 256)
    set_uniform("uMinGrayVal", 0.001)
    set_uniform("uMaxGrayVal", 1.0)
    set_uniform("uTransferFunction", tf_texture)
    set_uniform("uVolume", volume_texture)
    draw_box() # Draws bounding box

# Fragment Shader Logic (Pass 2)
def fragment_shader():
    # Get front face position (ray start)
    front_pos = get_vertex_position() # Interpolated from vertex shader
    
    # Get back face position (ray end) from texture
    # Sample using fragment coordinates normalized to [0,1]
    uv = gl_FragCoord.xy / vec2(g_winWidth, g_winHeight)
    back_pos = texture(uBackCoord, uv).rgb
    
    # Compute ray vector
    ray_dir = back_pos - front_pos
    ray_len = length(ray_dir)
    
    # Discard if ray length is zero (misses volume)
    if ray_len < 1e-6:
        discard()
        
    # Normalize direction
    dir_norm = ray_dir / ray_len
    
    # Calculate step size
    step_size = ray_len / uSteps
    
    # Initialize accumulation
    acc_color = vec4(0.0)
    acc_alpha = 0.0
    current_pos = front_pos
    
    # Ray Marching Loop
    for i in range(int(uSteps)):
        # Sample volume
        val = texture(uVolume, current_pos).r
        
        # Check bounds
        if val < uMinGrayVal or val > uMaxGrayVal:
            alpha = 0.0
            color = vec3(0.0)
        else:
            # Lookup transfer function
            tf_sample = texture(uTransferFunction, vec2(val, 0.5))
            color = tf_sample.rgb
            alpha = tf_sample.a * uOpacityVal # Apply opacity scaling
            
        # Front-to-back compositing
        acc_color.rgb += (1.0 - acc_alpha) * alpha * color
        acc_alpha += (1.0 - acc_alpha) * alpha
        
        # Advance position
        current_pos += dir_norm * step_size
        
        # Early Ray Termination
        if acc_alpha >= 1.0:
            break
            
        # Boundary check (optional, depends on geometry precision)
        if any(current_pos < 0.0) or any(current_pos > 1.0):
            break
            
    # Output final color
    out_color = acc_color
```

**Parameters:**
*   `uSteps`: Number of sampling steps along the ray. Typical value: 256 [S2, unofficial].
*   `uOpacityVal`: Opacity scaling factor. Typical value: 4.0 [S2, unofficial].
*   `uMinGrayVal`: Minimum volume value threshold. Typical value: 0.001 [S2, unofficial].
*   `uMaxGrayVal`: Maximum volume value threshold. Typical value: 1.0 [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Zooming In:** Rendering front faces causes issues when the user zooms in, as front faces project behind the camera and are clipped, leaving pixels unrendered. Rendering only back faces in Pass 1 and front faces in Pass 2 (with back-face culling in Pass 2) handles zooming correctly [S3, unofficial].
*   **Ray Misses:** If the entry point equals the exit point (ray misses the box or is tangent), the ray length is zero. The shader must discard the fragment to avoid division by zero [S6, unofficial].
*   **Loop Termination:** Some implementations use a large constant (e.g., 4095) in the loop condition due to GLSL limitations, breaking explicitly when `i == uSteps` [S2, unofficial].
*   **Mismatched Steps:** Using `MAX_STEPS` in the loop condition while calculating `delta` using `u_Steps` can lead to mismatches between intended step size and actual steps if `MAX_STEPS != u_Steps` [S7, unofficial].

## Subtopics

### [Bounding Box Raymarching](2071-bounding-box-raymarching.md)
Explicitly utilizes a two-pass approach where the first pass captures back-face coordinates into a texture, and the second pass uses these coordinates to define ray endpoints for front-to-back compositing. Unlike [Volume Ray Casting with RayData Texture](3077-volume-ray-casting-with-raydata-texture.md), which stores ray data in a dedicated texture, this method computes ray directions from geometry attributes or camera matrices directly in the fragment shader.

### [Entry Point Buffer with Back-to-Front Projection](13565-entry-point-buffer-with-back-to-front-projection.md)
Uses block-level visibility culling and hidden surface removal to initialize rays precisely at the volume surface rather than the bounding box.

## Operating characteristics
*   **Time Complexity:** $O(W \cdot H \cdot N)$, where $W, H$ are screen dimensions and $N$ is the number of steps per ray. The two-pass setup adds a constant overhead of $O(W \cdot H)$ for rasterizing the bounding box.
*   **Memory Complexity:** $O(W \cdot H)$ for the back-face texture storage.
*   **Performance:** Efficient for sparse volumes as it skips pixels outside the bounding box silhouette [S3, unofficial]. Performance is limited by the fragment shader's ray marching loop and memory bandwidth for volume texture sampling.
*   **Strengths:** Avoids complex ray-box intersection math in the fragment shader [S5, unofficial]. Handles zooming correctly by rendering back faces for entry points [S3, unofficial]. Simple to implement on older GPUs (PS 3.0) [S1, unofficial].
*   **Limitations:** Requires two render passes and an additional texture [S7, unofficial]. Less precise than analytical ray-box intersection for non-axis-aligned bounding boxes or complex shapes. Not suitable for dynamic bounding boxes without re-rasterization.
*   **When to Use:** Real-time volume rendering on GPUs with limited fragment shader complexity or when the volume is contained within a simple axis-aligned bounding box.
*   **When Not to Use:** When high precision is required for ray entry/exit points, or when the volume has a complex shape that cannot be approximated by a simple bounding box.

## Background
Originating from early GPU volume rendering techniques, this method leverages the rasterization pipeline to determine ray boundaries. It was popularized in WebGL and OpenGL tutorials for direct volume rendering, where it provided a straightforward way to implement raycasting without complex shader math [S3, unofficial][S1, unofficial]. It is often cited in the context of direct volume rendering with pixel shaders up to version 3.0 [S1, unofficial].

## Sources
[S1] Any fast voxel rendering algorithms? - GameDev.net, https://gamedev.net/forums/topic/597410-any-fast-voxel-rendering-algorithms/4785388/, fetched 2026-10-26 (unofficial)
[S2] levl-dev/VolumeRayCasting_0, https://github.com/levl-dev/VolumeRayCasting_0, fetched 2026-10-26 (unofficial)
[S3] Volume Rendering with WebGL, https://www.willusher.io/webgl/2019/01/13/volume-rendering-with-webgl, fetched 2026-10-26 (unofficial)
[S4] Real-Time Isosurface Extraction Using the GPU Programmable Geometry Pipeline, https://doi.org/10.1145/1281500.1361219, fetched 2026-11-03
[S5] fargiolas/qvrc, https://github.com/fargiolas/qvrc, fetched 2026-10-26 (unofficial)
[S6] thePZH/VolumeRendering-OpenGL, https://github.com/thePZH/VolumeRendering-OpenGL, fetched 2026-10-26 (unofficial)
[S7] yngve-sk/Volume-Rendering-WebGL, https://github.com/yngve-sk/Volume-Rendering-WebGL, fetched 2026-10-27 (unofficial)
[S8] GPU-accelerated single-pass volumetric raycasting in Qt ..., https://martinopilia.com/posts/2018/09/17/volume-raycasting.html, fetched 2026-10-27 (unofficial)
