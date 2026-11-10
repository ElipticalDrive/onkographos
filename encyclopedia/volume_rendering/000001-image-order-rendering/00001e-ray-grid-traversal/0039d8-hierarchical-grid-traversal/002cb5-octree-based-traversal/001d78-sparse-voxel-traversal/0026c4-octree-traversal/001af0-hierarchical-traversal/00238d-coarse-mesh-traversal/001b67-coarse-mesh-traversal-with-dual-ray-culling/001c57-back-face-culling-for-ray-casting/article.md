# Back-Face Culling for Ray Casting

Also called: Backface Raycasting, Raycasting with depth buffers, Raycasting with depth pre-pass, Front/Backface Buffer Raycasting, Two-pass depth buffer raycasting, Back-face depth culling raycasting, Back-face culling for entry/exit determination, Two-Pass Raycasting with Back-Face Depth Buffering, Two-Pass Raycasting with Backface Entry/Exit Points

## Overview
Back-face culling for ray casting is an optimization technique that determines the entry and exit points of rays into a volume by rendering the back faces of the volume's bounding box to a texture, thereby avoiding the need to compute ray-cube intersections analytically in the raycasting shader [S1, unofficial][S2, unofficial][S3, unofficial][S4, unofficial]. It builds on [Coarse Mesh Traversal with Dual-Ray Culling](987-coarse-mesh-traversal-with-dual-ray-culling.md) by applying the principle of using front/back face distinction to a simple bounding box primitive, offloading geometry processing to the GPU rasterizer [S1, unofficial][S2, unofficial]. Unlike single-pass methods that compute intersections in the fragment shader, this approach uses a two-pass rendering pipeline: the first pass captures the back-face depth/position, and the second pass uses this data to initialize the ray march, skipping empty space outside the volume [S2, unofficial][S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Volume density data (3D texture) [S3, unofficial], transfer function (2D texture) [S3, unofficial], camera view and projection matrices [S4, unofficial], and bounding box geometry (cube) [S1, unofficial].
*   **Outputs:** Final composited image (color buffer) [S3, unofficial].

**Data Structures:**
1.  **Bounding Box Geometry:** A cube defined by 8 vertices and 12 triangles [S1, unofficial]. Each vertex contains:
    *   `VerPos` (vec3): Position in object space.
    *   `VerClr` (vec3): Color, used to pass the 3D position of the vertex to the fragment shader [S2, unofficial].
2.  **Back-Face Texture (`g_bfTexObj` / `texBackFaces`):** A 2D texture (RGBA) storing the 3D entry point coordinates for each pixel [S2, unofficial][S4, unofficial].
    *   Dimensions: Width × Height (screen resolution) [S4, unofficial].
    *   Initialization: Zeros [S4, unofficial].
    *   Filters: `LINEAR` for MIN/MAG, `CLAMP_TO_EDGE` for S/T [S4, unofficial].
3.  **Volume Texture (`g_volTexObj` / `uVolData`):** A 3D texture (`sampler3D`) storing density values [S2, unofficial][S3, unofficial][S4, unofficial]. Requires the `OES_texture_float` extension in WebGL for floating-point data [S4, unofficial].
4.  **Transfer Function Texture (`trfunc_texture`):** A 2D texture (`sampler2D`) mapping density to color [S3, unofficial].
5.  **Frame Buffer:** A framebuffer object containing a `DEPTH_COMPONENT16` renderbuffer [S4, unofficial] and a 2D color texture attachment [S4, unofficial].

**Numbered Steps:**

1.  **Pass 1: Back-Face Depth Capture**
    1.  Bind the auxiliary framebuffer [S4, unofficial].
    2.  Enable depth testing [S4, unofficial].
    3.  Set depth function to `GEQUAL` [S4, unofficial].
    4.  Enable front-face culling (cull front faces, render back faces) [S1, unofficial][S3, unofficial]. The method assumes counter-clockwise (CCW) winding order where normals point outward; back faces yield a negative dot product with the view direction [S1, unofficial].
    5.  Use the "back" shader program [S4, unofficial].
    6.  Draw the bounding box cube.
        *   **Vertex Shader:** Transforms `VerPos` to clip space via `gl_Position = projection_matrix * view_matrix * vec4(VerPos, 1.0)` [S4, unofficial]. Passes `VerClr` as `EntryPoint` to the fragment shader [S2, unofficial].
        *   **Fragment Shader:** Outputs the interpolated vertex color (`VerClr`), which encodes the 3D entry point position, as `FragColor` [S2, unofficial].
    7.  Store the resulting color buffer into `g_bfTexObj` [S2, unofficial].

2.  **Pass 2: Raycasting**
    1.  Bind the default framebuffer [S4, unofficial].
    2.  Enable depth testing [S4, unofficial].
    3.  Set depth function to `LEQUAL` [S4, unofficial].
    4.  Enable back-face culling (cull back faces, render front faces) [S3, unofficial].
    5.  Bind `g_bfTexObj` to texture unit 0 (`uBackCoord`) [S4, unofficial].
    6.  Bind the volume texture to texture unit 1 (`uVolData`) [S4, unofficial].
    7.  Use the "raycast" shader program [S4, unofficial].
    8.  Draw the bounding box cube again.
        *   **Vertex Shader:** Transforms `VerPos` to clip space. Passes `VerClr` as `EntryPoint` (front face position) [S2, unofficial].
        *   **Fragment Shader:**
            1.  Sample `uBackCoord` using screen coordinates (`gl_FragCoord.st / ScreenSize`) to retrieve the 3D exit point (`exitPoint`) [S2, unofficial].
            2.  If `EntryPoint == exitPoint` (no back-face hit, e.g., background), discard the fragment [S2, unofficial].
            3.  Calculate ray direction: `dir = exitPoint - EntryPoint` [S3, unofficial].
            4.  Calculate ray length: `len = length(dir)` [S3, unofficial].
            5.  Calculate step vector: `deltaDir = normalize(dir) * StepSize` [S3, unofficial].
            6.  Initialize accumulators: `col_acc = vec4(0,0,0,0)`, `alpha_acc = 0.0`, `length_acc = 0.0`, `ray_pos = EntryPoint` [S3, unofficial].
            7.  Loop for `NUM_RAY_STEPS` iterations:
                1.  Sample density: `density = texture(uVolData, ray_pos).x` [S3, unofficial].
                2.  Map to color: `color_sample = texture(trfunc_texture, vec2(density, 0.5))` [S3, unofficial].
                3.  Calculate sample alpha: `alpha_sample = density * StepSize` [S3, unofficial].
                4.  Front-to-back blending: `col_acc += (1.0 - alpha_acc) * color_sample * alpha_sample * 40.0` [S3, unofficial].
                5.  Update alpha: `alpha_acc += alpha_sample` [S3, unofficial].
                6.  Update length: `length_acc += length(deltaDir)` [S3, unofficial].
                7.  Advance ray: `ray_pos += deltaDir` [S3, unofficial].
                8.  Termination Conditions:
                    *   If `length_acc >= len`, composite background (white, alpha 0) and break [S2, unofficial].
                    *   If `alpha_acc > 1.0`, clamp to 1.0 and break [S2, unofficial].
            8.  Output `col_acc` as fragment color [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def back_face_culling_raycasting(screen_width, screen_height, volume_texture, transfer_func, step_size, num_steps):
    # Pass 1: Back-Face Depth Capture
    # 1. Bind auxiliary framebuffer and enable depth testing.
    # 2. Set depth function to GEQUAL.
    # 3. Enable front-face culling (cull front faces, render back faces).
    # 4. Draw bounding box cube with back shader.
    #    Vertex Shader: gl_Position = projection_matrix * view_matrix * vec4(VerPos, 1.0)
    #                   EntryPoint = VerClr
    #    Fragment Shader: FragColor = vec4(EntryPoint, 1.0)
    # 5. Read framebuffer color buffer into back_face_texture.

    # Pass 2: Raycasting
    # 1. Bind default framebuffer and enable depth testing.
    # 2. Set depth function to LEQUAL.
    # 3. Enable back-face culling (cull back faces, render front faces).
    # 4. Bind back_face_texture to texture unit 0, volume_texture to unit 1.
    # 5. Draw bounding box cube with raycast shader.
    #    Vertex Shader: gl_Position = projection_matrix * view_matrix * vec4(VerPos, 1.0)
    #                   EntryPoint = VerClr
    #    Fragment Shader:
    #       uv = gl_FragCoord.xy / vec2(screen_width, screen_height)
    #       exit_point = texture(back_face_texture, uv).xyz
    #       if EntryPoint == exit_point: discard
    #       dir = exit_point - EntryPoint
    #       ray_len = length(dir)
    #       delta_dir = normalize(dir) * step_size
    #       col_acc = vec4(0.0)
    #       alpha_acc = 0.0
    #       length_acc = 0.0
    #       ray_pos = EntryPoint
    #       for i in range(num_steps):
    #           density = texture(volume_texture, ray_pos).x
    #           color_sample = texture(transfer_func, vec2(density, 0.5)).rgb
    #           alpha_sample = density * step_size
    #           col_acc += (1.0 - alpha_acc) * color_sample * alpha_sample * 40.0
    #           alpha_acc += alpha_sample
    #           length_acc += length(delta_dir)
    #           ray_pos += delta_dir
    #           if length_acc >= ray_len or alpha_acc > 1.0: break
    #       FragColor = col_acc
```

**Parameters:**
*   `StepSize`: Controls the step length in ray marching [S2, unofficial][S3, unofficial].
*   `NUM_RAY_STEPS`: Maximum iterations per ray. Source [S3, unofficial] uses 512. Source [S2, unofficial] uses 1600.
*   `ScreenSize`: Resolution of the screen (width, height) [S2, unofficial].
*   `uBackCoord`: Sampler uniform for the back-face texture [S4, unofficial].
*   `uVolData`: Sampler uniform for the volume data texture [S4, unofficial].

**Edge Cases and Pitfalls:**
*   **Coordinate System Mismatch:** `ExitPointCoord` in the vertex shader is in clip-space [S2, unofficial]. Sampling the texture using `gl_FragCoord` (window coordinates) works because the rasterizer maps clip-space to window-space, and the texture was written during rasterization [S2, unofficial].
*   **AMD Debugging Issue:** The `VerClr` input variable must be used in the shader; otherwise, debugging may fail on AMD hardware [S2, unofficial].
*   **Floating-Point Textures:** WebGL implementations require the `OES_texture_float` extension to support floating-point textures for volume data [S4, unofficial].
*   **Winding Order:** The method assumes counter-clockwise (CCW) winding order for front faces [S1, unofficial].

## Subtopics

### [Volume Raycasting with MRT Endpoints](1584-volume-raycasting-with-mrt-endpoints.md)
Volume Raycasting with MRT Endpoints is a two-pass GPU technique that determines ray entry and exit points by rendering the volume's bounding box to multiple render targets (MRT), explicitly separating front-face and back-face coordinates into distinct textures to avoid analytical ray-cube intersection calculations. It builds on [Back-Face Culling for Ray Casting](1236-back-face-culling-for-ray-casting.md) by using MRT to store both entry and exit points simultaneously in the first pass, rather than relying on depth buffering and a single pass for back-faces. Unlike single-pass methods that compute intersections in the fragment shader, this approach offloads geometry processing to the rasterizer, simplifying the raycasting shader at the cost of an additional rendering pass and texture lookups.

### Raycasting Phase
In the raycasting phase, the two auxiliary buffers are bound to distinct texture units and a viewport-filling quadrilateral is projected [S5]. For each pixel, the main fragment shader fetches back-facing texture coordinates; if invalid, the fragment is discarded, otherwise front-facing coordinates are retrieved to compute the step vector along the ray according to a predefined step length $d$ and the total number of iterations [S5]. Each slab of thickness $2d$ is defined by three successive samples $(s_f, s_m, s_b)$, with the next slab indexed by $(s_f, s_m, s_b)$ [S5].

## Operating characteristics
*   Time and memory complexity: Not given in the sources [G].
*   Measured performance: Not given in the sources [G].
*   Scaling: Not given in the sources [G].
*   Strengths and limitations: Not given in the sources [G].
*   When to use and when not: Not given in the sources [G].

## Background
Back-face culling for ray casting originated as a technique to optimize volume rendering by leveraging the GPU's rasterization pipeline to determine ray entry and exit points [S1, unofficial][S2, unofficial][S3, unofficial][S4, unofficial]. It was popularized in early WebGL and OpenGL tutorials for volume rendering, such as those by Khizar Siddiqui [S1, unofficial], toolchainX [S2, unofficial], and others [S3, unofficial][S4, unofficial]. The method is a practical application of depth buffering and face culling to avoid analytical geometry calculations in shaders [S4, unofficial]. It is commonly used in educational contexts and simple real-time volume renderers where simplicity and GPU compatibility are prioritized over advanced traversal algorithms [S4, unofficial].

## Sources
[S1] khizarsiddiqui/Volume-Rendering, https://github.com/khizarsiddiqui/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S2] Volume_Rendering_Using_GLSL, https://github.com/toolchainX/Volume_Rendering_Using_GLSL, fetched 2026-10-26 (unofficial)
[S3] 1vx-437312114/webgl_volume_renderer, https://github.com/1vx-437312114/webgl_volume_renderer, fetched 2026-10-27 (unofficial)
[S4] kapadia/volumetric.js, https://github.com/kapadia/volumetric.js, fetched 2026-10-27 (unofficial)
[S5] Second Order Pre-Integrated Volume Rendering, https://people.freedesktop.org/~marcheu/pacificvis08.pdf, fetched 2026-10-29
