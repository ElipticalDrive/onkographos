# Two-Pass Ray Marching Volume Rendering

Also called: INTMeshVolumeRendering, Multi-pass Raycasting with OpenGL FBOs, Multi-pass raycasting - OpenGL, Two-Pass Raycasting

## Overview
Two-Pass Ray Marching Volume Rendering is a GPU-based direct volume rendering technique that decouples the determination of ray entry and exit points from the volumetric integration process [S1, unofficial][S2, unofficial][S3, unofficial]. It solves the problem of efficiently computing ray-volume intersections without requiring complex ray-triangle intersection shaders for every sample, by rasterizing the volume's bounding box faces to offscreen textures in the first pass and using those depths to drive a raymarching loop in the second pass [S1, unofficial][S3, unofficial]. It differs from single-pass raycasting by reducing per-pass shader complexity at the cost of an additional render target and pass overhead [S3, unofficial], and from [Half-angle slicing](1255-half-angle-slicing.md) by maintaining continuous ray integration rather than discrete slice projection.

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D volume data (scalar field), camera parameters (position, orientation, projection), volume bounding box dimensions ($size_x, size_y, size_z$), number of marching steps ($N_{steps}$), and a transfer function mapping scalar values to RGBA [S2, unofficial].
*   **Outputs:** A 2D color buffer containing the final composited image [S1, unofficial][S3, unofficial].

**Data Structures:**
1.  **Bounding Box Geometry:** A cube defined by vertices spanning the volume extent. Must support double-sided rendering or separate front/back face culling [S2, unofficial][S3, unofficial].
2.  **Render Targets (FBOs):**
    *   `BackfaceTexture`: 2D texture, format `GL_RGBA16F_ARB` (or `FloatType`/`RGBAFormat` in WebGL), dimensions matching the viewport ($W \times H$) [S2, unofficial][S3, unofficial]. Stores the world-space position of the ray entry point (back face intersection).
    *   `FrontfaceTexture`: 2D texture, format `GL_RGBA16F_ARB` (or `FloatType`/`RGBAFormat` in WebGL), dimensions matching the viewport ($W \times H$) [S2, unofficial][S3, unofficial]. Stores the world-space position of the ray exit point (front face intersection).
    *   `VolumeTexture`: 3D texture (`RedFormat`/`AlphaFormat`, `FloatType`) containing the padded volume data [S1, unofficial][S2, unofficial].
    *   `TransferFunctionTexture`: 1D texture (e.g., $8 \times 256$ or $1 \times 255$), `RGBAFormat`, `ClampToEdgeWrapping` [S1, unofficial][S2, unofficial].
3.  **Uniforms/Parameters:**
    *   `size`: Bounding box dimensions $(x, y, z)$ [S2, unofficial].
    *   `steps`: Number of samples per ray (default 50) [S2, unofficial].
    *   `alphaCorrection`: Opacity correction factor (default 0.2) [S2, unofficial].
    *   `dataMin`, `dataMax`: Min/max scalar values for normalization [S2, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Create the bounding box geometry with double-sided rendering enabled or prepare for separate culling [S2, unofficial].
    *   Create two 2D render targets (`BackfaceTexture`, `FrontfaceTexture`) with high-precision floating-point formats (e.g., `GL_RGBA16F_ARB`) to store precise world coordinates [S3, unofficial].
    *   Load volume data into a 3D texture. If the data length does not fit a rectangular power-of-two or max-size texture, pad with zeros [S2, unofficial].
    *   Generate the transfer function texture from user-defined color/alpha stops [S1, unofficial][S2, unofficial].

2.  **Pass 1: Boundary Rasterization:**
    *   Bind the FBO containing `BackfaceTexture` as the color attachment [S3, unofficial].
    *   Enable face culling to exclude the front faces (`glCullFace(GL_FRONT)` in OpenGL, or render with `BackSide` material in WebGL) [S2, unofficial][S3, unofficial].
    *   Render the bounding box. The fragment shader outputs the world-space position of the back face intersection for each pixel [S3, unofficial].
    *   Bind the FBO containing `FrontfaceTexture` [S3, unofficial].
    *   Enable face culling to exclude the back faces (`glCullFace(GL_BACK)` in OpenGL, or render with `FrontSide` material in WebGL) [S2, unofficial][S3, unofficial].
    *   Render the bounding box. The fragment shader outputs the world-space position of the front face intersection [S3, unofficial].
    *   Unbind the FBO [S3, unofficial].

3.  **Pass 2: Ray Marching and Compositing:**
    *   Render a fullscreen quad (or the screen itself) using the raycasting shader [S1, unofficial][S3, unofficial].
    *   For each pixel, sample `FrontfaceTexture` to get $P_{start}$ and `BackfaceTexture` to get $P_{end}$ [S3, unofficial].
    *   Calculate the ray vector $\vec{R} = P_{end} - P_{start}$ and ray length $L = \|\vec{R}\|$ [S3, unofficial].
    *   Calculate step size $\Delta s = L / N_{steps}$ [S3, unofficial].
    *   Initialize accumulated color $C_{acc} = (0,0,0,0)$ and current position $P_{curr} = P_{end}$ (back-to-front accumulation) [S3, unofficial].
    *   Loop $i$ from 0 to $N_{steps}-1$:
        *   Sample volume at $P_{curr}$ to get scalar value $v$ [S3, unofficial].
        *   Normalize $v$ using `dataMin` and `dataMax` to index `TransferFunctionTexture`, obtaining source color $C_{src}$ and opacity $\alpha_{src}$ [S3, unofficial].
        *   Apply alpha correction: $\alpha_{src} = \alpha_{src} \times \alpha_{Correction}$ [S2, unofficial].
        *   Composite:
            $$ C_{acc}.rgb = C_{acc}.rgb \times (1 - \alpha_{src}) + C_{src}.rgb \times \alpha_{src} $$
            $$ C_{acc}.a = C_{acc}.a \times (1 - \alpha_{src}) + \alpha_{src} $$
        *   Move $P_{curr}$ towards start: $P_{curr} = P_{curr} - \Delta s \cdot \frac{\vec{R}}{\|\vec{R}\|}$ [S3, unofficial].
        *   Check Early Ray Termination (ERT): If $C_{acc}.a > 0.99$ or distance traveled $\ge L$, break loop [S3, unofficial].
    *   Output $C_{acc}$ to the screen buffer [S3, unofficial].

**Pseudocode (Fragment Shader Logic):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pass 1 Fragment Shader (Back Face)
def pass1_back_face(position_world):
    return position_world

# Pass 1 Fragment Shader (Front Face)
def pass1_front_face(position_world):
    return position_world

# Pass 2 Fragment Shader
def pass2_ray_marching(uv):
    # 1. Retrieve boundaries
    start_pos = texture2D(frontfaceTex, uv).rgb
    end_pos = texture2D(backfaceTex, uv).rgb
    
    # 2. Calculate Ray
    ray_vec = end_pos - start_pos
    ray_length = length(ray_vec)
    if ray_length < 1e-6:
        return vec4(0.0, 0.0, 0.0, 0.0)
    
    step_size = ray_length / steps
    step_dir = normalize(ray_vec)
    
    # 3. Initialize Accumulators (Back-to-Front)
    acc_color = vec4(0.0, 0.0, 0.0, 0.0)
    current_pos = end_pos
    
    # 4. Marching Loop
    for i in range(steps):
        # Sample Volume
        # Note: Normalization of voxel value depends on specific implementation
        # Assuming texture3D returns scalar in .r or .a
        voxel_val = texture3D(volumeTex, current_pos).r
        
        # Map to Transfer Function
        # tf_input is normalized scalar, typically [0,1]
        tf_input = clamp((voxel_val - dataMin) / (dataMax - dataMin), 0.0, 1.0)
        tf_sample = texture1D(transferFuncTex, tf_input)
        
        src_color = tf_sample.rgb
        src_alpha = tf_sample.a * alphaCorrection
        
        # Composite
        # Over operator: C_out = C_src * alpha_src + C_dst * (1 - alpha_src)
        acc_color.rgb = src_color * src_alpha + acc_color.rgb * (1.0 - src_alpha)
        acc_color.a = src_alpha + acc_color.a * (1.0 - src_alpha)
        
        # Early Ray Termination
        if acc_color.a > 0.99:
            break
            
        # Move backwards along ray
        current_pos = current_pos - step_dir * step_size
        
        # Boundary Check (optional, loop limit usually suffices)
        if length(end_pos - current_pos) > ray_length:
            break
            
    return acc_color
```

**Edge Cases and Pitfalls:**
*   **Precision:** Use `GL_RGBA16F_ARB` or higher for boundary textures to avoid z-fighting or inaccurate ray origins, especially for large bounding boxes [S3, unofficial].
*   **Culling:** Incorrect culling order (e.g., culling back instead of front for the first pass) results in swapped start/end points and visual artifacts [S3, unofficial].
*   **Data Padding:** Volume data must be padded to fit the rectangular texture dimensions; failure to pad causes out-of-bounds sampling or garbage data at texture edges [S2, unofficial].
*   **Missing Parameters:** Omitting renderer, camera, or size parameters causes silent failure or runtime errors in some implementations (e.g., Three.js wrappers) [S2, unofficial].
*   **AutoClear:** If rendering additional overlays (like cutting planes) after the raycaster, `autoClear` must be disabled during the raycast pass to preserve the framebuffer if multi-pass accumulation is used, or managed carefully if not [S1, unofficial].

## Subtopics

### [Two-Pass GPU Volume Ray Casting for Light Field Displays](6523-two-pass-gpu-volume-ray-casting-for-light-field-displays.md)
Specialized for multi-projector light field displays, decoupling 3D raycasting from 2D optical distortion correction. Chosen when rendering for naked-eye 3D displays with multiple projectors.

## Operating characteristics
*   **Time Complexity:** $O(W \cdot H \cdot N_{steps})$, where $W, H$ are viewport dimensions and $N_{steps}$ is the number of samples per ray. The two-pass structure adds a constant overhead of two full-screen renders and texture copies.
*   **Memory Complexity:** $O(W \cdot H)$ for the two boundary textures (plus $O(V)$ for the volume data). The boundary textures require high-precision formats (16-bit float per channel), consuming significant VRAM for high resolutions [S3, unofficial].
*   **Performance:** The method is efficient on GPUs with strong fill-rate capabilities. The separation of intersection logic (rasterization) from integration (raymarching) allows the rasterizer to handle culling and depth testing efficiently [S3, unofficial]. However, the overhead of rendering the bounding box twice and the memory bandwidth required for the FBOs can be a bottleneck at very high resolutions [S3, unofficial].
*   **Strengths:** Simple shader logic for raymarching; leverages hardware rasterization for intersection; supports early ray termination effectively; easy to implement in WebGL/OpenGL [S1, unofficial][S2, unofficial].
*   **Limitations:** Fixed step size (unless adaptive stepping is added in the shader); precision issues with large bounding boxes if low-precision textures are used; not suitable for complex, non-convex volumes without additional geometry handling [S3, unofficial].

## Background
The technique originates from the work of Krüger and Westermann (2003), who demonstrated the use of OpenGL Framebuffer Objects (FBOs) to store ray start and end positions by rasterizing the volume's bounding box faces [S3, unofficial]. It has since been adapted into various WebGL libraries, such as the `INTMeshVolumeRendering` class in Three.js-based projects, to provide interactive volume visualization in browsers [S1, unofficial][S2, unofficial]. It is commonly used in medical visualization and scientific data exploration where interactive frame rates are required [S1, unofficial].

## Sources
[S1] SubairProject1/Volume-Rendering-Visualization-Project, https://github.com/SubairProject1/Volume-Rendering-Visualization-Project, fetched 2026-10-26 (unofficial)
[S2] pyx61198/volume_rendering, https://github.com/pyx61198/volume_rendering, fetched 2026-10-27 (unofficial)
[S3] PDF Slide 1, https://www.nvidia.com/content/GTC/documents/1102_GTC09.pdf, fetched 2026-10-29 (unofficial)
