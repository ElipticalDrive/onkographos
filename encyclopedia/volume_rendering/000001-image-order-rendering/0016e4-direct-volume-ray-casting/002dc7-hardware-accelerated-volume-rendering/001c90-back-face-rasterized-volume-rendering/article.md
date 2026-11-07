# Back-face rasterized volume rendering

Also called: Back-Face Entry Point Method, Backface Rendering for Ray Casting, Backface Culling for Ray Casting Entry Points

## Overview
Back-face rasterized volume rendering is a hardware-accelerated technique that determines ray entry and exit points by rasterizing the volume's bounding box geometry rather than computing intersections analytically in the fragment shader [S1, S4, unofficial][S2, unofficial][S3, unofficial]. It solves the problem of efficiently restricting fragment shader execution to pixels intersecting the volume while correctly handling camera zooming, which can cause front-face clipping if standard rasterization is used [S2, unofficial]. It differs from [Bounding Box Raymarching](2071-bounding-box-raymarching.md) by explicitly utilizing a two-pass approach where the first pass captures back-face coordinates into a texture, and the second pass uses these coordinates to define ray endpoints for front-to-back compositing [S1, S2, S3, unofficial]. Unlike [Volume Ray Casting with RayData Texture](3077-volume-ray-casting-with-raydata-texture.md), which stores ray data in a dedicated texture, this method often computes ray directions from geometry attributes or camera matrices directly in the fragment shader [S5, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D volume texture ($V$), transfer function texture ($TF$), camera projection and view matrices ($P, V_{iew}$), camera position ($\mathbf{o}$), viewport resolution ($W, H$), and bounding box geometry (unit cube vertices).
*   **Outputs:** Final color buffer image.

**Data Structures:**
1.  **Bounding Box Geometry:** A unit cube defined by 8 vertices in $[0,1]^3$ and 36 indices forming 12 triangles [S4, unofficial].
    *   Vertices: $\{(0,0,0), (0,0,1), (0,1,0), (0,1,1), (1,0,0), (1,0,1), (1,1,0), (1,1,1)\}$ [S4, unofficial].
2.  **Back-Face Texture ($T_{back}$):** A 2D texture of size $W \times H$ with internal format `GL_RGBA16F` (or equivalent high-precision float) [S4, unofficial]. Stores the 3D entry/exit coordinates (object space) of the back-face intersection as RGB color values [S1, S4, unofficial].
3.  **Frame Buffer Object (FBO):** Attached to $T_{back}$ for the first pass [S3, S4, unofficial].

**Numbered Steps:**

1.  **Pass 1: Capture Back-Face Entry Points**
    1.  Bind the FBO attached to $T_{back}$.
    2.  Set viewport to $(0, 0, W, H)$.
    3.  Enable face culling with `GL_FRONT` (cull front faces, render back faces) [S1, S4, unofficial].
    4.  Render the bounding box geometry using a vertex shader that transforms vertices to clip space and a fragment shader that outputs the vertex position (in object/local coordinates) as the fragment color [S1, S4, unofficial].
    5.  Unbind the FBO. The texture $T_{back}$ now contains the 3D coordinates where rays enter the volume (or exit, depending on culling convention; typically the "back" face relative to the camera is the entry point for front-to-back rendering if front faces are culled) [S1, S4, unofficial].

2.  **Pass 2: Ray Casting and Compositing**
    1.  Bind the default framebuffer.
    2.  Set uniforms: $T_{back}$, volume texture $V$, transfer function $TF$, camera matrices, and step parameters.
    3.  Enable face culling with `GL_BACK` (cull back faces, render front faces) [S1, S4, unofficial].
    4.  Render the bounding box geometry.
        *   **Vertex Shader:** Outputs the current vertex position (front face) as `EntryPoint` [S4, unofficial].
        *   **Fragment Shader:**
            1.  Sample $T_{back}$ using normalized fragment coordinates ($gl\_FragCoord.xy / [W, H]$) to retrieve the `ExitPoint` (back face position) [S1, S4, unofficial].
            2.  If `EntryPoint` equals `ExitPoint` (ray misses or is tangent), discard the fragment [S4, unofficial].
            3.  Compute ray direction: $\mathbf{d} = \text{ExitPoint} - \text{EntryPoint}$ [S1, S4, unofficial].
            4.  Compute ray length: $L = \|\mathbf{d}\|$ [S4, unofficial].
            5.  Compute step vector: $\mathbf{s} = \frac{\mathbf{d}}{N}$, where $N$ is the number of steps (or fixed step size $\Delta$) [S1, S4, unofficial].
            6.  Initialize accumulated color $C_{acc} = (0,0,0,0)$ and position $\mathbf{p} = \text{EntryPoint}$ [S1, S4, unofficial].
            7.  **Ray Marching Loop:** For $i = 0$ to $N-1$:
                *   Sample volume value: $v = V(\mathbf{p})$ [S1, S4, unofficial].
                *   Apply transfer function: $(c, \alpha) = TF(v)$ [S1, S4, unofficial].
                *   If $\alpha > 0$:
                    *   Update color: $C_{acc}.rgb \leftarrow C_{acc}.rgb + (1 - C_{acc}.a) \cdot \alpha \cdot c$ [S1, S4, unofficial].
                    *   Update alpha: $C_{acc}.a \leftarrow C_{acc}.a + (1 - C_{acc}.a) \cdot \alpha$ [S1, S4, unofficial].
                *   Advance position: $\mathbf{p} \leftarrow \mathbf{p} + \mathbf{s}$ [S1, S4, unofficial].
                *   **Termination Conditions:**
                    *   If $C_{acc}.a \ge 1.0$ (or threshold 0.95), break (Early Ray Termination) [S1, S2, S3, unofficial].
                    *   If $\mathbf{p}$ exceeds $[0,1]$ bounds in any dimension, break [S1, unofficial].
                    *   If loop counter reaches $N$, break [S1, S4, unofficial].
            8.  Composite with background (if needed) and output $C_{acc}$ [S4, unofficial].

**Formulas:**
*   Ray Parametric Equation: $\mathbf{p}(t) = \mathbf{o} + t \mathbf{v}$ [S5, unofficial].
*   Focal Length from Field of View $\alpha$: $f = \frac{1}{\tan(\frac{\alpha}{2})}$ [S5, unofficial].
*   Front-to-Back Compositing:
    $$
    C'_{i} = C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i C_i
    $$
    $$
    \alpha'_{i} = \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i
    $$
    Where $C_i, \alpha_i$ are the color and opacity of the current sample, and $C', \alpha'$ are the accumulated values [Parent].

**Parameters:**
*   `uSteps` ($N$): Number of sampling steps. Typical value: 256 [S1, unofficial].
*   `uOpacityVal`: Opacity scaling factor. Typical value: 4.0 [S1, unofficial].
*   `uMinGrayVal`, `uMaxGrayVal`: Volume value thresholds. Typical: 0.001, 1.0 [S1, unofficial].
*   `DELTA`: Constant step size. Value: 0.005 [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Zooming:** Rendering front faces causes clipping when the camera zooms inside the volume. Rendering only back faces (Pass 1) and using them as entry points avoids this [S2, unofficial].
*   **Missed Rays:** If the ray does not intersect the box (entry == exit), the fragment must be discarded to prevent division by zero or infinite loops [S4, unofficial].
*   **Precision:** Using `GL_RGBA16F` for the back-face texture helps maintain precision for coordinate storage [S4, unofficial].
*   **GLSL Loop Limits:** Some implementations use a large constant loop bound (e.g., 1600 or 4095) with an explicit break condition due to compiler limitations [S1, S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pass 1: Capture Back-Face Coordinates
def pass1_capture_backface():
    bind_fbo(target_texture=g_bfTexObj, width=win_width, height=win_height, format=RGBA16F)
    set_viewport(0, 0, win_width, win_height)
    enable_cull_face(GL_FRONT) # Cull front, render back
    
    # Vertex Shader: Output vertex position as color
    # Fragment Shader: Write vertex position (object coords) to gl_FragColor
    
    draw_bounding_box()
    
    unbind_fbo()

# Pass 2: Ray Casting
def pass2_ray_casting():
    bind_default_framebuffer()
    set_uniforms(back_face_tex=g_bfTexObj, volume_tex=V, transfer_func=TF)
    enable_cull_face(GL_BACK) # Cull back, render front
    
    # Vertex Shader: Output vertex position as EntryPoint
    # Fragment Shader:
    #   1. Sample g_bfTexObj at gl_FragCoord.xy / resolution to get ExitPoint
    #   2. dir = ExitPoint - EntryPoint
    #   3. if length(dir) == 0: discard
    #   4. step = dir / uSteps
    #   5. pos = EntryPoint
    #   6. acc_color = (0,0,0,0), acc_alpha = 0
    #   7. for i in 0..uSteps:
    #       val = sample_volume(pos)
    #       (c, a) = sample_transfer_function(val)
    #       if a > 0:
    #           acc_color.rgb += (1 - acc_alpha) * a * c
    #           acc_alpha += (1 - acc_alpha) * a
    #       pos += step
    #       if acc_alpha >= 1.0: break
    #       if pos outside [0,1]: break
    #   8. output acc_color
    
    draw_bounding_box()
```

## Subtopics

### [Bounding Box Raymarching](2071-bounding-box-raymarching.md)
Explicitly utilizes a two-pass approach where the first pass captures back-face coordinates into a texture, and the second pass uses these coordinates to define ray endpoints for front-to-back compositing. Choose when you need to avoid explicit ray-box intersection math in the fragment shader for exit determination.

### [Volume Ray Casting with RayData Texture](3077-volume-ray-casting-with-raydata-texture.md)
Stores ray data in a dedicated texture. Choose when you need to decouple ray generation from the bounding box rasterization or support more complex ray data structures.

## Operating characteristics
*   **Time Complexity:** $O(W \cdot H \cdot N)$, where $W, H$ are viewport dimensions and $N$ is the number of steps per ray. The two-pass approach adds a constant overhead for the first pass.
*   **Memory Complexity:** $O(W \cdot H)$ for the back-face texture storage [S4, unofficial].
*   **Performance:** Efficiently restricts fragment processing to pixels touching the volume, avoiding unnecessary work for empty screen regions [S2, unofficial]. Early Ray Termination (ERT) significantly reduces average step count for opaque regions [S1, S2, S3, unofficial].
*   **Strengths:** Correctly handles zooming into the volume (unlike front-face only rasterization) [S2, unofficial]; avoids complex ray-box intersection code in the fragment shader for determining bounds [S3, S4, unofficial].
*   **Limitations:** Requires two rendering passes and an extra texture bind/sampling [S4, unofficial]; precision of entry/exit points depends on the back-face texture format [S4, unofficial].
*   **When to Use:** Standard hardware-accelerated volume rendering where camera zooming is expected and explicit ray-box intersection is undesirable or too costly in the fragment shader.
*   **When Not to Use:** Single-pass optimization is critical and ray-box intersection is cheap; or when the volume has no simple convex bounding box.

## Background
The method originates from GPU-based volume rendering implementations that leverage rasterization hardware to determine ray bounds. It is commonly found in WebGL and OpenGL tutorials and prototypes for interactive volume visualization [S2, S3, S4, unofficial]. The technique addresses the limitation of standard front-face rasterization, which fails when the camera enters the volume bounds [S2, unofficial].

## Sources
[S1] levl-dev/VolumeRayCasting_0, https://github.com/levl-dev/VolumeRayCasting_0, fetched 2026-10-26 (unofficial)
[S2] Volume Rendering with WebGL, https://www.willusher.io/webgl/2019/01/13/volume-rendering-with-webgl, fetched 2026-10-26 (unofficial)
[S3] fargiolas/qvrc, https://github.com/fargiolas/qvrc, fetched 2026-10-26 (unofficial)
[S4] thePZH/VolumeRendering-OpenGL, https://github.com/thePZH/VolumeRendering-OpenGL, fetched 2026-10-26 (unofficial)
[S5] GPU-accelerated single-pass volumetric raycasting in Qt ..., https://martinopilia.com/posts/2018/09/17/volume-raycasting.html, fetched 2026-10-27 (unofficial)
