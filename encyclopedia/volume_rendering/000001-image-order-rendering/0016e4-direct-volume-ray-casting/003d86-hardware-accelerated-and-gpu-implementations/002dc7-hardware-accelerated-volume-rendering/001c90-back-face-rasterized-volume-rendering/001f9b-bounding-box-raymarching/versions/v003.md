# Bounding Box Raymarching

Also called: Raycasting via Bounding Box Rendering, Single-Pass GPU Raycasting, Volume Ray Marching with WebGL, WebGL2 Bounding Box Raymarching

## Overview
Bounding Box Raymarching is a GPU-accelerated volume rendering technique that renders a 3D voxel grid by casting rays through the volume's bounding box geometry [S1, S2, S3, unofficial]. It solves the problem of efficiently determining ray entry and exit points for volume traversal without performing expensive ray-cube intersection calculations in the final rendering pass [S2, S3, unofficial]. It differs from [Back-face rasterized volume rendering](1296-back-face-rasterized-volume-rendering.md) by explicitly utilizing a two-pass approach where the first pass captures back-face coordinates into a texture, and the second pass uses these coordinates to define ray endpoints for front-to-back compositing [S1, S2, S3, unofficial]. Unlike [Volume Ray Casting with RayData Texture](3077-volume-ray-casting-with-raydata-texture.md), which stores ray data in a dedicated texture, this method often computes ray directions from geometry attributes or camera matrices directly in the fragment shader [S5, unofficial].

## How it works
The method typically employs a two-pass rendering pipeline using a unit cube (or bounding box) geometry. The first pass renders the back faces of the cube to a framebuffer texture to store the ray exit coordinates (or entry coordinates in object space). The second pass renders the front faces to the screen, using the stored back coordinates and the current front coordinates to define the ray, then samples the volume texture along this ray.

### Inputs and Outputs
*   **Inputs**:
    *   Volume data: Stored in a 3D texture (`sampler3D`) or a 2D texture atlas of slices (`sampler2D` array) [S2, S3, S5, unofficial].
    *   Transfer function: Stored in a 1D texture (`sampler1D`) mapping scalar density to RGBA color/opacity [S2, S5, unofficial].
    *   Camera matrices: Model-View-Projection (MVP) and their inverses for coordinate transformation [S4, unofficial].
    *   Bounding Box Geometry: A cube defined by 8 vertices and 12 triangles [S2, unofficial].
*   **Outputs**:
    *   Final image: Rendered to the screen or a target framebuffer with front-to-back compositing [S1, S5, unofficial].

### Data Structures
1.  **Cube Geometry**:
    *   **Vertices**: 8 vertices defining a unit cube in object space (0 to 1).
        *   Layout: `(0,0,0), (0,0,1), (0,1,0), (0,1,1), (1,0,0), (1,0,1), (1,1,0), (1,1,1)` [S2, unofficial].
        *   Indices: 36 indices forming 12 triangles (6 faces), drawn counter-clockwise [S2, unofficial].
    *   **Texture Coordinates**: Vary from 0 to 1, representing the entry points of rays into the volume in texture space [S5, unofficial].
2.  **Frame Buffer (Pass 1)**:
    *   Color Attachment: 2D texture (e.g., `RGBA16F` or `RGBA`) storing world/object space coordinates of the back face [S1, S2, unofficial].
    *   Depth Attachment: Depth buffer (e.g., `DEPTH_COMPONENT16`) to ensure correct back-face rasterization [S1, S2, unofficial].
3.  **Textures**:
    *   **Volume Texture**: 3D texture (`GL_INTENSITY` or `RGBA`) with linear filtering and repeat wrapping [S2, S5, unofficial]. If using a 2D atlas, it requires slice mapping logic [S3, unofficial].
    *   **Transfer Function**: 1D texture (`GL_RGBA8`) with nearest filtering and repeat wrapping [S2, unofficial].
    *   **Back-Coord Texture**: 2D texture sampled in Pass 2 to retrieve ray endpoints [S3, unofficial].

### Numbered Steps

**Pass 1: Capture Back-Face Coordinates**
1.  Bind the framebuffer object (FBO) with the back-coordinate texture as color attachment.
2.  Enable front-face culling (`gl.CULL_FACE` with `gl.FRONT`) to render only the back faces of the cube [S1, S3, unofficial].
3.  Render the cube geometry.
4.  In the fragment shader, output the world-space (or object-space) coordinates of the fragment as the color [S1, S3, unofficial].
    *   *Note*: Some implementations use a single-pass approach where the ray direction is computed from the camera position and texture coordinates, bypassing the FBO [S5, unofficial]. However, the standard two-pass method uses this step to decouple ray intersection from sampling [S1, S2, unofficial].

**Pass 2: Ray Marching and Compositing**
1.  Bind the default framebuffer (screen).
2.  Disable face culling or enable back-face culling to render the front faces of the cube [S1, S3, unofficial].
3.  In the fragment shader:
    a.  **Retrieve Ray Endpoints**:
        *   Get the front position (`frontColor`) from the vertex shader varying (current fragment position in world/object space) [S3, unofficial].
        *   Get the back position (`backColor`) by sampling the texture from Pass 1 using the current screen coordinates (`texC = ((pos.xy/pos.w) + 1.0) / 2.0`) [S3, unofficial].
    b.  **Compute Ray Direction**:
        *   Calculate direction: $dir = backColor.rgb - frontColor.rgb$ [S3, unofficial].
        *   Normalize if necessary, though the step size calculation often absorbs the length [S3, unofficial].
    c.  **Initialize Accumulators**:
        *   Set accumulated color/opacity `accum = (0, 0, 0, 0)` [S3, S5, unofficial].
        *   Set current position `vpos = frontColor` [S3, unofficial].
    d.  **Marching Loop**:
        *   Iterate for a fixed number of steps (`uSteps`) or until transmittance is negligible [S3, S4, unofficial].
        *   **Step Size**: $Step = dir / uSteps$ [S3, unofficial]. Alternatively, a fixed step size (e.g., 0.001 or 0.01) may be used [S2, S5, unofficial].
        *   **Sample Volume**:
            *   If using 3D texture: `value = texture3D(volumeTex, vpos.xyz)` [S5, unofficial].
            *   If using 2D Atlas: Map 3D position to 2D slice coordinates using $s1Original = floor(volpos.z \times uNumberOfSlices)$, then interpolate between slices [S3, unofficial].
            *   Extract scalar value (e.g., from alpha channel `value.a`) [S5, unofficial].
        *   **Transfer Function**:
            *   Normalize scalar: $tf\_pos.x = (scalar - uMinGrayVal) / (uMaxGrayVal - uMinGrayVal)$ [S3, unofficial].
            *   Sample transfer function: `src = texture1D(tfTex, tf_pos.x)` [S5, unofficial].
        *   **Compositing**:
            *   Apply opacity factor: $sample.a = src.a \times uOpacityVal$ [S3, unofficial].
            *   Apply color factor: $sample.rgb = src.rgb \times uColorVal$ [S3, unofficial].
            *   Update accumulation (Standard Absorption):
                $$accum.rgb += (1.0 - accum.a) \times sample.rgb$$
                $$accum.a += sample.a \times (1.0 - accum.a)$$
                *Note: Simplified forms like `accum += sample` are used in some implementations if `sample.a` is pre-multiplied or small [S3, unofficial].*
            *   Break if $accum.a \ge 1.0$ [S3, unofficial].
        *   **Advance Position**:
            *   $vpos.xyz += Step$ [S3, unofficial].
            *   Check bounds: If using explicit bounds, break if $vpos$ exceeds cube limits ($x,y,z > 1.0$ or $< 0.0$) [S3, unofficial].
            *   Alternatively, check if inside volume extent using `dot(sign(pos - min), sign(max - pos)) < 3.0` [S5, unofficial].
4.  Output `accum` as the final fragment color [S3, S5, unofficial].

### Formulas
*   **Ray Direction**: $dir = backColor.rgb - frontColor.rgb$ [S3, unofficial].
*   **Step Vector**: $Step = dir / uSteps$ [S3, unofficial].
*   **Volume Value (2D Atlas)**:
    $$s1Original = floor(volpos.z \times uNumberOfSlices)$$
    $$tex1Index = floor(s1Original / (uSlicesOverX \times uSlicesOverY))$$
    $$s1 = mod(s1Original, uSlicesOverX \times uSlicesOverY)$$
    $$dx1 = fract(s1/uSlicesOverX)$$
    $$dy1 = floor(s1/uSlicesOverY)/uSlicesOverY$$
    $$texpos1.x = dx1 + (volpos.x/uSlicesOverX)$$
    $$texpos1.y = dy1 + (volpos.y/uSlicesOverY)$$
    $$value1 = texture2D(uSliceMaps[tex1Index], texpos1).x$$ [S3, unofficial].
*   **Transfer Function Normalization**: $tf\_pos.x = (gray\_val - uMinGrayVal) / (uMaxGrayVal - uMinGrayVal)$ [S3, unofficial].
*   **Inside Check**: $inside = dot(sign(position - volExtentMin), sign(volExtentMax - position))$ [S5, unofficial].

### Parameters
*   `steps`: Number of voxel samples per ray. Default: 256 [S1, unofficial].
*   `alphaCorrection` / `uOpacityVal`: Opacity factor applied to each sample. Default: 0.08 [S1, S3, unofficial].
*   `stepSize`: Fixed step size for marching. Typical: 0.001 to 0.01 [S2, S5, unofficial].
*   `uMinGrayVal`, `uMaxGrayVal`: Range for transfer function normalization [S3, unofficial].
*   `camera`: Camera position in texture space (normalized 0-1) [S5, unofficial].

### Edge Cases and Pitfalls
*   **Texture Loading**: If the volume texture is not loaded when rendering begins, the pass may return early [S1, unofficial].
*   **Depth Texture Support**: Requires `WEBGL_depth_texture` extension for the back-coordinate pass in WebGL [S1, unofficial].
*   **Warping Artifacts**: Occur if volume texture wrapping is not set to `GL_CLAMP` with a one-pixel empty margin, especially during rotation [S5, unofficial].
*   **Camera Space**: Camera position must be provided in texture space (normalized), not world space, to avoid incorrect ray directions [S5, unofficial].
*   **Loop Limits**: In some GLSL implementations, direct comparison `i > uSteps` is avoided due to precision issues; instead, a large constant (e.g., 4095) is used with an equality check `i == uSteps` [S3, unofficial].
*   **Out-of-Bounds Values**: Gray values outside `[uMinGrayVal, uMaxGrayVal]` should be handled (e.g., set to transparent) [S3, unofficial].

## Subtopics

### [Volume Ray Casting with RayData Texture](3077-volume-ray-casting-with-raydata-texture.md)
Uses a pre-pass to store ray entry/exit bounds in a dedicated texture, allowing the final pass to sample these values directly instead of computing them from geometry. Choose this when you need to decouple ray intersection logic from the volume sampling shader or support complex ray origins not easily derived from a simple cube.

## Operating characteristics
*   **Time Complexity**: $O(N \cdot S)$, where $N$ is the number of pixels and $S$ is the number of steps per ray. The two-pass approach adds a constant overhead for the back-face render.
*   **Memory Complexity**: $O(W \cdot H)$ for the back-coordinate texture (where $W, H$ are screen dimensions) plus the volume and transfer function textures.
*   **Performance**: Highly parallel on GPU due to per-pixel ray execution. Performance is limited by memory bandwidth (volume texture sampling) and fragment shader arithmetic [S5, unofficial].
*   **Strengths**: Simple geometry (unit cube), easy to implement in WebGL/OpenGL, handles zooming correctly by re-rasterizing the bounding box [S1, S2, unofficial].
*   **Limitations**: Fixed step size may lead to aliasing or slow convergence; requires full volume texture in memory; single-pass variants may suffer from warping artifacts if wrapping is not handled [S5, unofficial].
*   **When to Use**: Real-time volume visualization in web browsers (WebGL) or embedded systems where simplicity and compatibility are prioritized over advanced optimization techniques like hierarchical ray marching [S1, S3, unofficial].

## Background
Bounding Box Raymarching is a foundational technique in GPU-based volume rendering, popularized in the early days of WebGL and OpenGL shader programming. It leverages the rasterization pipeline to generate rays, avoiding the need for explicit ray-triangle intersection code in the fragment shader. Key implementations include the WebGL volume rendering demo by Chloe Arluck [S1, unofficial], the GLSL raycasting tutorial by ToolchainX [S2, unofficial], and community discussions on Three.js and Khronos forums [S3, S5, unofficial]. It serves as a baseline for more advanced methods like hierarchical ray marching or raycasting with raydata textures.

## Sources
[S1] chloearluck/volume_rendering, https://github.com/chloearluck/volume_rendering, fetched 2026-10-24 (unofficial)
[S2] Volume_Rendering_Using_GLSL, https://github.com/toolchainX/Volume_Rendering_Using_GLSL, fetched 2026-10-26 (unofficial)
[S3] Volume rendering in WebGL, https://stackoverflow.com/questions/26588568/volume-rendering-in-webgl, fetched 2026-10-27 (unofficial)
[S4] OKaluza/sharevol, https://github.com/OKaluza/sharevol, fetched 2026-10-27 (unofficial)
[S5] volume renderer - gpu raycast - OpenGL: Advanced Coding, https://community.khronos.org/t/volume-renderer-gpu-raycast/39262, fetched 2026-10-27 (unofficial)
