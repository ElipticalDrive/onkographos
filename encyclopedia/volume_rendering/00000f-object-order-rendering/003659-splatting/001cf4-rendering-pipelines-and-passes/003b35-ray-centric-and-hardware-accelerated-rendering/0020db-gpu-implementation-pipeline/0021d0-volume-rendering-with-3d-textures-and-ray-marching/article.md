# Volume Rendering with 3D Textures and Ray Marching

Also called: 3D Texture Ray Marching, GPU Ray Casting for DICOM Volumes, GPU Volume Raycasting Implementation, GpuHypertexture Rendering, RaycastVolume Rendering Pipeline, WebGL Volume Raymarching with Lighting and Surface Occlusion, WebGPU Volume Raymarching

## Overview
This method performs object-order volume rendering by ray marching through a 3D texture stored in GPU memory, applying a transfer function to map scalar values to color and opacity, and compositing the result front-to-back [S2, unofficial]. It solves the problem of visualizing volumetric data (such as medical DICOM series or synthetic density fields) directly on the GPU without generating intermediate polygonal meshes [S4]. It differs from isosurface extraction (e.g., [Marching Cubes](53-marching-cubes.md)) by preserving internal volume information [S1, unofficial], and from splatting ([Splatting](10-splatting.md)) by avoiding sorting and blending overheads for dense volumes, though it is computationally heavier per pixel than texture-based splatting [G].

## How it works

### Inputs and Outputs
**Inputs:**
*   Volumetric scalar field data (e.g., 8-bit or 16-bit unsigned integers) [S3, unofficial].
*   Camera parameters: position, view matrix, projection matrix [S2, unofficial].
*   Transfer function: a 1D texture mapping scalar values to RGBA [S2, unofficial].
*   Optional: Surface depth map for occlusion, lighting parameters (direction, color), shadow maps [S2, S5, unofficial].

**Outputs:**
*   A 2D color buffer (RGBA) representing the rendered volume [S3, unofficial].
*   Optional: G-buffer attachments (normal, depth) for deferred rendering [S3, unofficial].

### Data Structures and Layouts

**3D Volume Texture:**
*   Format: `GL_RED` internal format with `GL_UNSIGNED_BYTE` (8-bit) or `GL_UNSIGNED_SHORT` (16-bit) data type [S3, unofficial].
*   Dimensions: $D_x \times D_y \times D_z$.
*   Compression: Supports `r8unorm` (1 byte/block), `bc4-r-unorm` (8 bytes/block, 4x4), or `astc-12x12-unorm` (16 bytes/block, 12x12) for WebGPU [S4].
*   Memory Layout for Compressed Upload:
    *   `blocksWide = ceil(width / blockLength)` [S4].
    *   `blocksHigh = ceil(height / blockLength)` [S4].
    *   `bytesPerRow = blocksWide * bytesPerBlock` [S4].
    *   Upload via `queue.writeTexture` with `{ bytesPerRow, rowsPerImage: blocksHigh }` [S4].

**Transfer Function Texture:**
*   Format: `GL_RGBA8` or `GL_RGBA16F`.
*   Dimensions: $256 \times 1$ (mapping 0–255 input values to RGBA) [S2, unofficial].

**Gradient Texture (Optional):**
*   Format: `GL_RG16F` (16-bit float per component).
*   Dimensions: $D_x \times D_y \times D_z$.
*   Filters: `GL_LINEAR`, Wrap: `GL_CLAMP_TO_BORDER` [S3, unofficial].

**Uniforms (Vertex Shader):**
*   `modelMatrix` (mat4), `modelViewMatrix` (mat4), `projectionMatrix` (mat4), `cameraPosition` (vec3) [S2, unofficial].
*   `inverseModelViewProjectionMatrix` (mat4) for WebGPU ray generation [S4].

**Uniforms (Fragment Shader):**
*   `volumeTex` (sampler3D), `transferTex` (sampler2D) [S2, unofficial].
*   `boxSize` (vec3): Physical dimensions of the volume bounding box [S2, unofficial].
*   `alphaScale`, `dtScale` (float): Scaling factors for opacity and step size [S2, unofficial].
*   `useLighting`, `useSurface`, `useWatermark` (bool) [S2, unofficial].
*   `light0/1/2` (vec3), `lightColor0/1/2` (vec3) [S2, unofficial].
*   `surfaceDepthTex` (sampler2D), `surfaceColorTex` (sampler2D) [S2, unofficial].

### Steps

1.  **Volume Loading and Preprocessing:**
    *   Load raw scalar data from file (e.g., DICOM stack or binary) [S3, unofficial].
    *   Create 3D texture. If 16-bit, use `GL_UNSIGNED_SHORT`; if 8-bit, use `GL_UNSIGNED_BYTE` [S3, unofficial].
    *   Compute histogram of volume data to determine normalization range (max value) [S3, unofficial].
    *   Compute gradients using a compute shader (if lighting is enabled):
        *   Dispatch `ceil(D_x/8) \times ceil(D_y/8) \times ceil(D_z/8)` blocks [S3, unofficial].
        *   Read from volume texture, write to `GL_RG16F` gradient texture [S3, unofficial].
        *   Apply smoothing pass by dispatching `smoothGradientsCompute` with the same block dimensions [S3, unofficial].

2.  **Ray Generation (Vertex Shader):**
    *   **Method A (WebGL/GLSL):** Compute unnormalized ray direction:
        $$ \text{rayDirUnnorm} = \text{position} - \text{cameraPosition} $$
        Normalize: $\text{rayDir} = \text{normalize}(\text{rayDirUnnorm})$.
        Apply X-axis mirroring if `useVolumeMirrorX` is true [S2, unofficial].
    *   **Method B (WebGPU/WGSL):** Generate a full-screen triangle. Calculate near and far points in object space by multiplying screen coordinates by `inverseModelViewProjectionMatrix` [S4].

3.  **Ray-Volume Intersection (Slab Method):**
    *   Define volume bounding box centered at origin with size `boxSize` (physical dimensions in mm, not voxel count) [S1, unofficial].
    *   Intersect ray with box to get interval $[t_0, t_1]$ [S2, unofficial].
    *   If `useSurface` is true:
        *   Sample `surfaceDepthTex` at fragment coordinate [S2, unofficial].
        *   Convert depth to camera distance.
        *   Clamp $t_1 = \min(t_1, \text{surfaceDistance})$ [S2, unofficial].
        *   Clamp $t_0 = \min(t_0, \text{surfaceDistance} - 0.0001)$ to prevent black artifacts from surface occlusion [S2, unofficial].
    *   If $t_0 \ge t_1$, discard fragment. Clamp $t_0 = \max(0.0, t_0)$ [S2, unofficial].

4.  **Step Size Calculation:**
    *   Compute base step size components:
        $$ dt_0 = \frac{1.0}{\text{volumeTexSize} \cdot |\text{rayDir}|} $$
        $$ dt = \min(dt_0.x, \min(dt_0.y, dt_0.z)) \cdot dtScale $$
    *   If $dt < 0.00001$, output black and return (prevents context loss) [S2, unofficial].
    *   Alternatively, calculate iterations: $N = \lceil \text{maxSize} \cdot (1.0 / \text{stepScale}) \cdot 2.0 \rceil$ [S3, unofficial].

5.  **Ray Initialization:**
    *   Calculate start position in real space:
        $$ p = \text{cameraPosition} + t_0 \cdot \text{rayDir} $$
    *   Apply dithering to reduce banding:
        $$ \text{random} = \text{fract}(\sin(\text{gl\_FragCoord.x} \cdot 12.9898 + \text{gl\_FragCoord.y} \cdot 78.233) \cdot 43758.5453) \cdot 5.0 $$
        $$ p = p + \text{random} \cdot dt \cdot \text{rayDir} $$
    *   Transform $p$ and step vector $dP = \text{rayDir} \cdot dt$ to texture space (normalized 0–1):
        $$ p_{\text{sized}} = \frac{p}{\text{boxSize}} + \text{vec3}(0.5) $$
        $$ dP_{\text{sized}} = \frac{\text{rayDir} \cdot dt}{\text{boxSize}} $$
    *   Compute alpha normalization factor to compensate for viewing angle:
        $$ l = \|\text{rayDir} \cdot \text{boxSize}\| $$
        $$ l_{\min} = \min(\text{boxSize}.x, \min(\text{boxSize}.y, \text{boxSize}.z)) $$
        $$ \text{alphaNormalization} = \frac{l_{\min}}{l} \cdot \text{alphaScale} $$ [S2, unofficial].

6.  **Ray Marching Loop (Fragment Shader):**
    *   Initialize accumulated color $C = (0,0,0,0)$ and transmittance $T = 1.0$.
    *   Loop $i$ from $t_0$ to $t_1$ with step $dt$ (or fixed number of steps, e.g., 64 [S4] or 512 [S1, unofficial]):
        1.  Sample volume: $v = \text{texture}(\text{volumeTex}, p_{\text{sized}}).r$.
        2.  Get color and opacity from transfer function:
            $$ c_{\text{vol}} = \text{texture}(\text{transferTex}, \text{vec2}(v, 0.5)) $$
            $$ c_{\text{vol}}.a \cdot= \text{alphaNormalization} $$
        3.  Compute lighting (if enabled and $c_{\text{vol}}.a > 0$):
            *   Sample gradient texture for normal.
            *   Compute diffuse/specular shading.
            *   Apply precomputed transmittance/shadow if available [S5, unofficial].
        4.  Front-to-back compositing:
            $$ C = C + c_{\text{vol}} \cdot T $$
            $$ T = T \cdot (1.0 - c_{\text{vol}}.a) $$
        5.  Advance position: $p_{\text{sized}} += dP_{\text{sized}}$ [S2, unofficial].
        6.  If $T < \epsilon$ (e.g., 0.001), break loop [G].

7.  **Post-Processing:**
    *   Apply gamma correction: $C_{\text{final}} = C^{\frac{1}{\text{finalGamma}}}$ (typical gamma 6.0 for microscopy data) [S2, unofficial].
    *   If `useWatermark` and fragment is at image center, write watermark value to blue channel [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vertex_main(position: list, camera_position: list, inv_mvp: list) -> tuple:
    # position is clip-space coordinate of full-screen triangle
    # Transform position to object space
    obj_pos = multiply_matrix_vector(inv_mvp, [position[0] * 2.0 - 1.0, position[1] * 2.0 - 1.0, 0.0, 1.0])
    # Ray direction is from camera to object point
    ray_dir_unnorm = [obj_pos[i] - camera_position[i] for i in range(3)]
    return obj_pos, ray_dir_unnorm

def fragment_main(ray_dir_unnorm: list, camera_position: list, box_size: list, volume_tex_size: list, 
                  dt_scale: float, alpha_scale: float, final_gamma: float, use_surface: bool, 
                  surface_depth_tex: dict, frag_coord: list, volume_tex: dict, transfer_tex: dict, 
                  use_lighting: bool) -> list:
    ray_dir = normalize(ray_dir_unnorm)
    t0, t1 = slab_intersect([0.0, 0.0, 0.0], ray_dir, box_size)
    
    if t0 >= t1:
        return [0.0, 0.0, 0.0, 1.0]

    if use_surface:
        depth = texture_sample(surface_depth_tex, frag_coord)
        surface_dist = camera_distance_from_depth(depth)
        t1 = min(t1, surface_dist)
        t0 = min(t0, surface_dist - 0.0001)

    if t0 < 0.0: t0 = 0.0
    if t0 >= t1:
        return [0.0, 0.0, 0.0, 1.0]

    # Step size
    dt0 = [1.0 / (volume_tex_size[i] * abs(ray_dir[i])) for i in range(3)]
    dt = min(dt0) * dt_scale
    
    if dt < 0.00001:
        return [0.0, 0.0, 0.0, 1.0]

    # Dithering
    random_val = fract(sin(frag_coord[0] * 12.9898 + frag_coord[1] * 78.233) * 43758.5453) * 5.0
    p = [camera_position[i] + t0 * ray_dir[i] + random_val * dt * ray_dir[i] for i in range(3)]
    
    # Transform to texture space
    p_sized = [p[i] / box_size[i] + 0.5 for i in range(3)]
    dP_sized = [ray_dir[i] * dt / box_size[i] for i in range(3)]
    
    # Alpha normalization
    l = length([ray_dir[i] * box_size[i] for i in range(3)])
    l_min = min(box_size)
    alpha_norm = (l_min / l) * alpha_scale

    acc_color = [0.0, 0.0, 0.0, 0.0]
    transmittance = 1.0
    
    # Fixed steps or loop until t1
    t = t0
    while t < t1:
        v = texture_sample(volume_tex, p_sized)
        v_color = texture_sample(transfer_tex, [v, 0.5])
        v_color[3] *= alpha_norm
        
        if v_color[3] > 0.0 and use_lighting:
            # Compute lighting (diffuse, specular, shadows)
            # ... lighting code ...
            v_color[0] *= lighting_factor
            v_color[1] *= lighting_factor
            v_color[2] *= lighting_factor
            
        acc_color[0] += v_color[0] * transmittance
        acc_color[1] += v_color[1] * transmittance
        acc_color[2] += v_color[2] * transmittance
        transmittance *= (1.0 - v_color[3])
        
        if transmittance < 0.001:
            break
            
        p_sized = [p_sized[i] + dP_sized[i] for i in range(3)]
        t += dt

    # Gamma correction
    acc_color[0] = pow(acc_color[0], 1.0 / final_gamma)
    acc_color[1] = pow(acc_color[1], 1.0 / final_gamma)
    acc_color[2] = pow(acc_color[2], 1.0 / final_gamma)
    
    return [acc_color[0], acc_color[1], acc_color[2], 1.0]
```

### Parameters
*   `dtScale`: Step size multiplier. Typical values depend on volume resolution and desired detail [S2, unofficial].
*   `alphaScale`: Opacity scaling factor [S2, unofficial].
*   `finalGamma`: Gamma correction exponent. Typical value: 6.0 for fluorescence microscopy [S2, unofficial].
*   `NumSteps`: Fixed number of iterations (e.g., 64 in WebGPU sample [S4], up to 512 in WebGL sample [S1, unofficial]).
*   `threshold`: Minimum opacity threshold for raycasting [S3, unofficial].
*   `m_absorption`: Absorption coefficient (default 0.1) [S5, unofficial].
*   `m_densityMult`: Density multiplier (default 1.0) [S5, unofficial].

### Edge Cases and Pitfalls
*   **Voxel Aspect Ratio:** CT slice spacing (mm) often differs from pixel spacing (<1mm). Rendering a volume as a cube flattens the anatomy. The bounding box `boxSize` must reflect physical dimensions in millimeters, not voxel counts [S1, unofficial].
*   **Step Size Underflow:** If `dt < 0.00001`, the context may be lost. Output black and return [S2, unofficial].
*   **Surface Occlusion Artifacts:** If a surface is between the camera and the volume, `t0` must be pushed back by a small tolerance (0.0001) to prevent black artifacts [S2, unofficial].
*   **Compressed Texture Upload:** Incorrect `bytesPerRow` and `rowsPerImage` calculations for compressed formats (BC4, ASTC) lead to texture corruption [S4].
*   **Performance:** High sample counts ("ridiculous number of samples") make this method expensive. Suitable for small volumes or infrequent updates [S5, unofficial].

## Operating characteristics

*   **Time Complexity:** $O(N \cdot P)$, where $N$ is the number of pixels and $P$ is the number of ray marching steps per pixel. Performance is linear with resolution and step count.
*   **Memory Complexity:** $O(V)$ for the 3D volume texture, plus $O(V)$ for gradients if computed. $O(P)$ for temporary buffers if deferred rendering is used.
*   **Measured Performance:**
    *   WebGL2 implementation with 512 steps per pixel can be slow on mobile devices [S1, unofficial].
    *   WebGPU implementation with 64 steps and MSAA (4x) is feasible for real-time on modern desktop GPUs [S4].
    *   GpuHypertexture method is noted as "expensive" and suitable only for experimentation or small 3D textures [S5, unofficial].
*   **Strengths:**
    *   High visual quality with accurate lighting and shadows.
    *   Supports complex transfer functions and surface occlusion.
    *   No sorting required (unlike splatting).
*   **Limitations:**
    *   Computationally intensive for large volumes or high step counts.
    *   Banding artifacts if dithering is not used.
    *   Requires careful handling of aspect ratios for medical data.
*   **When to Use:** Real-time visualization of moderate-sized volumes (e.g., 128³ to 256³) where lighting and transparency effects are critical.
*   **When Not to Use:** Very large volumes (e.g., 1024³) without level-of-detail techniques, or when frame rates are extremely critical and lighting is not needed.

## Background

*   **Origin:** Volume raycasting has been a standard technique since the 1980s [S1, unofficial]. The specific implementations cited here adapt the technique to modern web APIs (WebGL2, WebGPU) and GPU architectures.
*   **Development:**
    *   [S1, unofficial] adapts the technique for DICOM volumes in a web application, emphasizing physical accuracy (mm vs voxels).
    *   [S2, unofficial] implements a transfer function based on FluoRender for microscopy data, using Three.js and WebGL2.
    *   [S3, unofficial] provides a C++/OpenGL implementation with deferred rendering and gradient computation.
    *   [S4] demonstrates WebGPU-specific features like compressed texture upload and MSAA.
    *   [S5, unofficial] presents a "GpuHypertexture" approach for synthetic density fields with precomputed lighting.
*   **Usage:** Medical imaging (DICOM), scientific visualization (microscopy), and synthetic effects (clouds, fire) [S1, S2, S5, unofficial].

## Sources

[S1] feat(body-exposure): render a DICOM series in 3D, not a description of one, https://github.com/rizkyazhar486/Panacea/pull/1585, fetched 2026-10-26 (unofficial)
[S2] JaneliaSciComp/web-vol-viewer, https://github.com/JaneliaSciComp/web-vol-viewer, fetched 2026-10-26 (unofficial)
[S3] jose-villegas/VolumeRendering, https://github.com/jose-villegas/VolumeRendering, fetched 2026-10-27 (unofficial)
[S4] Volume Rendering and 3D Textures | webgpu/webgpu-samples | DeepWiki, https://deepwiki.com/webgpu/webgpu-samples/6.1-volume-rendering-and-3d-textures, fetched 2026-10-27
[S5] theomission/hypertexture, https://github.com/theomission/hypertexture, fetched 2026-10-27 (unofficial)
