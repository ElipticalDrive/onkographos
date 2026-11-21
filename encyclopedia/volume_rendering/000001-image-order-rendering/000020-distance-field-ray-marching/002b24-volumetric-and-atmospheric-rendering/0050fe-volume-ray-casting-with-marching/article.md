# Volume Ray Casting with Marching

Also called: Front-to-Back Ray Casting, Marching along the point of view, Ray Casting with Adaptive Step Size and Early Ray Termination, Volume Rendering via Ray Casting

## Overview
Volume Ray Casting with Marching is an image-order rendering technique that casts rays from the camera through a volumetric scalar field, sampling the field at discrete intervals to accumulate color and opacity using front-to-back compositing [S2, unofficial]. It solves the problem of visualizing 3D scalar data (such as medical CT scans or atmospheric density) by projecting it onto a 2D screen while preserving depth cues and transparency [S2, unofficial]. Unlike [SDF Cloud Rendering](2064-sdf-cloud-rendering.md), which relies on signed distance fields for surface-like rendering, this method integrates density directly, and unlike [Atmospheric Ray Marching](4874-atmospheric-ray-marching.md), which often targets specific post-processing effects like god rays, this method provides a general-purpose integration of the radiative transfer equation for arbitrary volume textures [S1, unofficial].

## How it works
The method computes the final pixel color by integrating light contribution along a ray $r(t) = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is the camera origin and $\mathbf{d}$ is the normalized direction vector. The integration proceeds from the camera outward (front-to-back) through the volume bounds.

### Inputs and Outputs
*   **Inputs**:
    *   Camera parameters: Origin $\mathbf{o}$, direction $\mathbf{d}$, view matrix, projection matrix.
    *   Volume data: A 3D texture (scalar field) defined by dimensions $V_{dims} = (V_x, V_y, V_z)$ and bounds $[box_{min}, box_{max}]$.
    *   Transfer function: A 2D colormap texture mapping scalar values to RGBA color.
    *   Parameters: Step size scale factor $dt_{scale}$, early termination threshold $T_{thresh}$ (typically 0.95 opacity).
*   **Outputs**:
    *   Final pixel color $C_{final}$ and accumulated opacity $A_{final}$.

### Data Structures
*   **Volume Texture**: 3D texture storing scalar values. For 16-bit data, values are normalized by 65536.0; for 8-bit/float, values are used directly [S2, unofficial].
*   **Colormap Texture**: 2D texture where the x-coordinate represents the normalized scalar value and the y-coordinate is fixed (e.g., 0.5) to sample the transfer function [S2, unofficial].
*   **Ray State**:
    *   $t_{entry}$: Distance to volume entry.
    *   $t_{exit}$: Distance to volume exit.
    *   $p$: Current position in world space.
    *   $C$: Accumulated color (RGB).
    *   $A$: Accumulated opacity (A).

### Numbered Steps

1.  **Calculate Back Plate Position**: Determine the position of the back of the volume bounding box relative to the camera to establish the maximum ray extent [S1, unofficial].
2.  **Calculate Ray Direction**: Normalize the view ray direction vector $\mathbf{d}$ [S2, unofficial].
3.  **Calculate Front Plate Position**: Determine the position of the front of the volume bounding box [S1, unofficial].
4.  **Intersect Ray with Volume Bounds**: Compute the entry distance $t_0$ and exit distance $t_1$ where the ray intersects the bounding box defined by $box_{min}$ and $box_{max}$ [S2, unofficial].
    *   If $t_0 > t_1$, discard the ray (the ray does not intersect the volume) [S2, unofficial].
    *   Clamp the entry distance to non-negative values: $t_{entry} = \max(t_0, 0.0)$ [S2, unofficial].
5.  **Calculate Step Size**: Determine the step size $\Delta t$ based on the volume dimensions, ray direction, and a scaling factor $dt_{scale}$ [S2, unofficial]. The base step is derived from the voxel traversal rates:
    $$ \Delta t = dt_{scale} \times \min \left( \frac{1}{|d_x| \cdot V_x}, \frac{1}{|d_y| \cdot V_y}, \frac{1}{|d_z| \cdot V_z} \right) $$
    [S2, unofficial]. Note: In implementation, this is often computed as `1.0 / (vec3(volume_dims) * abs(ray_dir))` [S2, unofficial].
6.  **Generate Jitter Offset**: To reduce aliasing artifacts (specifically "wood grain" patterns), generate a random offset for the starting position within the first voxel [S2, unofficial]. The offset is calculated using a hash function (e.g., `wang_hash`) on the fragment coordinates:
    $$ \text{offset} = \text{wang\_hash}(\text{int}(gl\_FragCoord.x + 640.0 \cdot gl\_FragCoord.y)) $$
    [S2, unofficial].
7.  **Initialize Sample Position**: Set the initial sample position $p$ at the entry point plus the jittered offset:
    $$ p = \mathbf{o} + (t_{entry} + \text{offset} \cdot \Delta t) \cdot \mathbf{d} $$
    [S2, unofficial].
8.  **Marching Loop**: Iterate along the ray from $t_{entry}$ to $t_{exit}$ with step $\Delta t$. In each iteration:
    a. **Sample Volume Value**: Sample the scalar value $v$ from the volume texture at position $p$ [S2, unofficial].
        *   For 16-bit data: $v = \text{texture}(\text{volume}, p).r / 65536.0$ [S2, unofficial].
        *   For 8-bit/float data: $v = \text{texture}(\text{volume}, p).r$ [S2, unofficial].
    b. **Map to Color**: Sample the colormap texture using the normalized value $v$ to get the transfer function output [S2, unofficial]:
        $$ \text{trans\_sample} = \text{texture}(\text{colormap}, \text{vec2}(v, 0.5)) $$
        [S2, unofficial].
    c. **Combine Color and Opacity**: Construct the sample color $C_{sample}$ and opacity $A_{sample}$ [S2, unofficial]:
        $$ C_{sample} = \text{trans\_sample}.rgb $$
        $$ A_{sample} = v \cdot \text{trans\_sample}.a $$
        [S2, unofficial].
    d. **Apply Opacity Correction**: Correct the sample opacity for the step size to approximate the integral of absorption over the segment [S2, unofficial]:
        $$ A_{corrected} = 1 - (1 - A_{sample})^{dt_{scale}} $$
        [S2, unofficial].
    e. **Front-to-Back Compositing**: Accumulate the color and opacity into the running totals $C$ and $A$ [S2, unofficial]:
        $$ C \leftarrow C + (1.0 - A) \cdot A_{corrected} \cdot C_{sample} $$
        $$ A \leftarrow A + (1.0 - A) \cdot A_{corrected} $$
        [S2, unofficial].
    f. **Advance Position**: Move to the next sample point:
        $$ p \leftarrow p + \mathbf{d} \cdot \Delta t $$
        [S2, unofficial].
    g. **Early Ray Termination (ERT)**: If the accumulated opacity $A \ge 0.95$, break the loop [S2, unofficial].
9.  **Update Controller and View Matrix**: After the marching pass, update the controller and view matrix based on the previous state, applying rescaling if necessary [S1, unofficial].
10. **Draw Frame and Generate Color Maps**: Render the frame around the volume and generate any necessary color maps for the current frame [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_ray_casting(camera_origin, camera_direction, volume_texture, colormap_texture, volume_dims, dt_scale, t_thresh):
    """
    Renders a single pixel using volume ray casting with marching.
    
    Args:
        camera_origin: vec3, camera position.
        camera_direction: vec3, normalized ray direction.
        volume_texture: 3D texture object.
        colormap_texture: 2D texture object.
        volume_dims: vec3, dimensions of the volume (Vx, Vy, Vz).
        dt_scale: float, step size multiplier.
        t_thresh: float, opacity threshold for early termination.
        
    Returns:
        vec4, final color with accumulated opacity.
    """
    # 1. Intersect ray with bounding box [box_min, box_max]
    # Note: box_min and box_max are assumed to be available in the shader context
    t0, t1 = intersect_box(camera_origin, camera_direction, box_min, box_max)
    
    # If no intersection, return transparent black
    if t0 > t1:
        return vec4(0.0, 0.0, 0.0, 0.0)
    
    # Clamp entry to non-negative
    t_entry = max(t0, 0.0)
    t_exit = t1
    
    # 2. Calculate step size
    # dt_vec = 1.0 / (volume_dims * abs(camera_direction))
    # dt = dt_scale * min(dt_vec)
    dt_vec = vec3(1.0) / (volume_dims * abs(camera_direction))
    dt = dt_scale * min(dt_vec.x, dt_vec.y, dt_vec.z)
    
    # 3. Generate jitter offset
    # Assuming gl_FragCoord is available in the shader context
    frag_coord = get_frag_coord() # vec2
    offset = wang_hash(int(frag_coord.x + 640.0 * frag_coord.y))
    
    # 4. Initialize position
    p = camera_origin + (t_entry + offset * dt) * camera_direction
    
    # 5. Initialize accumulators
    acc_color = vec3(0.0)
    acc_alpha = 0.0
    
    # 6. Marching loop
    t = t_entry + offset * dt
    while t < t_exit:
        # Sample volume value
        val_raw = texture(volume_texture, p).r
        # Normalize if 16-bit, else use directly
        val = val_raw / 65536.0 if is_16bit else val_raw
        
        # Map to color via colormap
        trans_sample = texture(colormap_texture, vec2(val, 0.5))
        
        # Combine color and opacity
        sample_color = trans_sample.rgb
        sample_alpha = val * trans_sample.a
        
        # Apply opacity correction for step size
        corrected_alpha = 1.0 - pow(1.0 - sample_alpha, dt_scale)
        
        # Front-to-back compositing
        # C_new = C_old + (1 - A_old) * A_sample * C_sample
        # A_new = A_old + (1 - A_old) * A_sample
        acc_color += (1.0 - acc_alpha) * corrected_alpha * sample_color
        acc_alpha += (1.0 - acc_alpha) * corrected_alpha
        
        # Early Ray Termination
        if acc_alpha >= t_thresh:
            break
            
        # Advance
        t += dt
        p = camera_origin + t * camera_direction
        
    return vec4(acc_color, acc_alpha)
```

### Parameters
*   `dt_scale`: A uniform float parameter controlling the step size multiplier. It scales the base voxel traversal step. Typical value not specified in sources [S2, unofficial].
*   `t_thresh`: Early termination threshold. Default value is 0.95 opacity [S2, unofficial].

### Edge Cases and Pitfalls
*   **Aliasing**: Not jittering the starting position causes "wood grain" aliasing artifacts [S2, unofficial]. However, jittering can cause artifacts on flat surfaces ("with offset is bad for flat surfaces") [S2, unofficial].
*   **Double Launch**: Calculating the front and back plate positions separately may require launching the process twice. It is recommended to calculate the front plate and start marching in the same shader pass to avoid redundant computation [S1, unofficial].
*   **Precision**: Ensure correct normalization of volume texture data (e.g., dividing by 65536.0 for 16-bit data) to prevent incorrect density interpretation [S2, unofficial].

## Subtopics

### [Shadow Occlusion via Occlusion Camera](16371-shadow-occlusion-via-occlusion-camera.md)
This method uses a dedicated child camera (the "Occlusion Camera") to render a depth map from the light's perspective, allowing scene geometry to block volumetric light beams. Unlike standard URP shadows which depend on the light's Shadow Type and global URP shadow settings, this occlusion map operates independently, enabling volumetric shadows even when the light casts no regular geometric shadows.

### [Basic Volumetric Cone Lighting](17756-basic-volumetric-cone-lighting.md)
Basic Volumetric Cone Lighting is a post-processing technique that renders a volumetric light beam shaped like a cone originating from a light position and pointing in a specific direction, using screen-space raymarching. It solves the problem of rendering visible light shafts (god rays) in real-time by accumulating light density along rays cast from the camera into the scene. It is distinguished from sibling methods like [Shadow Occlusion via Occlusion Camera](16371-shadow-occlusion-via-occlusion-camera.md) by its reliance on a single depth buffer for culling rather than a secondary camera pass, and from [Volume Ray Casting with Marching](14353-volume-ray-casting-with-marching.md) by its specific application to spotlight-shaped volumetric effects rather than general volume density integration.

### Neural Asset Shader
A technique to convert a neural radiance field model into efficient GLSL or HLSL shader code, enabling real-time volumetric raycasting that blends with existing engine subsystems [S3, unofficial]. It transpiles the optimized model architecture (grid features + MLPs) into pixel shader code, executes volumetric raycasting in the pixel shader by aggregating sample points to compute the final color, and blends the rendered neural asset result with the background consisting of other mesh- or volume-based objects [S3, unofficial].

### Raymarched Anisotropic Scattering with Soft Shadows
Enhanced raymarching for volumetric media that supports configurable dual-lobe anisotropic scattering, soft screen-space penumbrae, and spatial step jitter [S4, unofficial]. It packs shadow steps and anisotropy values into a single uniform slot for compatibility, applies spatial step jitter during raymarching to reduce aliasing and banding artifacts, modulates density based on height for natural atmospheric effects, generates coherent wisps and axis-weighted dust patterns for visual detail, and calculates soft screen-space penumbrae for shadows cast by volumetric objects [S4, unofficial].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of samples along the ray, which depends on the volume depth and step size. The step size is inversely proportional to the ray direction components and volume dimensions [S2, unofficial].
*   **Memory Complexity**: Depends on the size of the 3D volume texture and the 2D colormap texture. No additional large buffers are required beyond standard render targets [S2, unofficial].
*   **Performance**: The method is computationally intensive due to the large number of texture samples required for smooth rendering. Early Ray Termination (ERT) significantly reduces the average number of samples per ray, especially for opaque regions [S2, unofficial].
*   **Strengths**: Provides high-quality, physically-based volume rendering with accurate transparency and depth cues. It is flexible and works with any scalar field data [S2, unofficial].
*   **Limitations**: Susceptible to aliasing artifacts ("wood grain") if not jittered, and jittering can introduce artifacts on flat surfaces [S2, unofficial]. Performance degrades with large volumes or high resolution.
*   **When to Use**: Suitable for general-purpose volume visualization where quality is prioritized over extreme performance, or when ERT can effectively cull opaque regions.
*   **When Not to Use**: Not suitable for real-time applications with very large volumes or strict performance constraints without optimization (e.g., hierarchical level-of-detail).

## Background
The technique is a fundamental method in volume rendering, rooted in the discretization of the Radiative Transfer Equation (RTE) along rays [S2, unofficial]. It is widely used in medical imaging, scientific visualization, and computer graphics for rendering 3D scalar data. The specific implementation details, such as jittering and early termination, are common optimizations to improve visual quality and performance [S2, unofficial]. The method is often implemented in fragment shaders for GPU acceleration [S2, unofficial].

## Sources
[S1] WebGL for Numerical PDEs (Session 11): Volume Ray Casting, https://www.youtube.com/watch?v=X095f9QzSBo, fetched 2026-10-27 (unofficial)
[S2] johanna-b/VisWeb, https://github.com/johanna-b/VisWeb, fetched 2026-10-27 (unofficial)
[S3] Neural Assets: Volumetric Object Capture and Rendering for Interactive Environments, https://core.ac.uk/download/552466290.pdf, fetched 2026-10-30 (unofficial)
[S4] Improve volumetric lighting and emissive glow proxies, https://github.com/EVEngine/EVEngine/pull/462, fetched 2026-10-26 (unofficial)
