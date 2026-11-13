# Illustrative Volume Rendering with Style Transfer

Also called: Style Transfer Function (STF) Volume Rendering

## Overview
This method applies artistic styles to volumetric data by combining standard direct volume ray casting with a Style Transfer Function (STF). It solves the problem of visualizing scalar fields with non-photorealistic aesthetics by mapping volume values to discrete style indices and surface normals to 2D style textures. It builds on [Style Transfer and Illustrative Rendering](13031-style-transfer-and-illustrative-rendering.md) by implementing a specific two-pass rendering pipeline that captures exit points for accurate ray length calculation.

## How it works
**Inputs and Outputs**
- **Input**: A 3D scalar volume texture (dimensions $W \times H \times D$), a 1D Transfer Function texture (size 256), a 1D Index Function texture, a 2D Style Transfer Texture array (containing lit sphere or matcap images), and camera parameters (projection, view, model matrices).
- **Output**: A 2D color image with illustrative styling.

**Data Structures**
1.  **Volume Texture**: 3D texture (`GL_TEXTURE_3D`) with dimensions $W \times H \times D$. Internal format `GL_INTENSITY`. Wrap modes `GL_CLAMP` for S, T, R. Min/Mag filter `GL_LINEAR`. Data source is normalized float array `dataScalars`.
2.  **Transfer Function Texture**: 1D texture (`GL_TEXTURE_1D`) of size 256. Internal format `GL_RGBA8`. Wrap mode `GL_REPEAT`. Min/Mag filter `GL_NEAREST`. Stores `glm::vec4` values where `transferFunc[i] = vec4((float)i / 255.f)` by default.
3.  **Index Function Texture**: 1D texture (`GL_TEXTURE_1D`) mapping scalar values to material indices.
4.  **Style Transfer Texture**: 2D array texture (`GL_TEXTURE_2D_ARRAY`) containing multiple style images (e.g., lit spheres).
5.  **Back Face Texture**: 2D texture (`GL_TEXTURE_2D`) with size equal to the window resolution ($W_{win} \times H_{win}$). Internal format `GL_RGBA32F`. Wrap modes `GL_REPEAT`. Min/Mag filter `GL_NEAREST`. Used to store exit points.
6.  **Frame Buffer Object (FBO)**: Contains the Back Face Texture as a color attachment and a depth renderbuffer (`GL_DEPTH_COMPONENT`). Depth test is enabled.
7.  **Vertex Buffer Object (VBO)**: Layout of 24 floats defining 8 vertices of a unit cube (0.0-1.0 range). Indices: 36 unsigned ints defining 12 triangles (2 per face, counter-clockwise). VAO binds attribute 0 (vertex location) and attribute 1 (vertex color) to the same vertex data buffer.

**Numbered Steps**

1.  **Initialization**:
    -   Set `isLoaded = false`.
    -   Allocate `sModelName` (1024 chars).
    -   Set `width`, `height`, `numCuts = 1`.
    -   Set `stepSize = 0.001f`, `threshold = 0.15f`.
    -   Initialize texture/buffer IDs to 0.
    -   Initialize `transferFunc` array with 256 entries: `transferFunc[i] = vec4((float)i / 255.f)`.
    -   Call `setupVolumeShaders()`, `createTransferFunctionTexture()`, and `stf.loadStyles()`.

2.  **Volume Loading**:
    -   Call `createVertexBuffer()` to set up VBO/VAO for cube rendering.
    -   Call `loadVolumeFromFile8()` or `loadVolumeFromFile16()` to read raw data into `dataScalars` (normalized floats).
    -   Call `createBackFaceTexture()` for exit point storage.
    -   Call `createFrameBuffer()` for FBO setup.
    -   Set dimensions. Compute `cubeSizes = vec3(width/max, height/max, numCuts/max)`.
    -   Scale model, position at `(-cubeSizes.x/2, -cubeSizes.y/2, 0)`.
    -   Compute projection (45 deg FOV, aspect ratio, 0.1-500 near/far), view (`lookAt(0,0,-1, 0,0,0, 0,1,0)`), model, normalMatrix, viewProjection, and modelViewProjection matrices.

3.  **Two-Pass Rendering**:
    -   **Pass 1: Back Face Rendering**: Render the back face of the volume bounding box to the FBO. This captures the exit points of rays traversing the volume.
    -   **Pass 2: Ray Casting with Style Transfer**:
        -   Bind the default framebuffer.
        -   Set viewport to window size.
        -   Use the ray casting shader.
        -   Set uniforms: `MVP` (modelViewProjection), `ViewMatrix`, `NormalMatrix`, `StepSize`, `Threshold`, `ScreenSize`.
        -   Bind textures:
            -   Unit 1: `transferFunctionTexture` (1D).
            -   Unit 2: `indexFunctionTexture` (1D).
            -   Unit 3: `styleTransferTexture` (2D Array).
        -   For each pixel, cast a ray through the volume.
        -   Sample the volume scalar value $V(\mathbf{p}_i)$ at each step.
        -   Map $V(\mathbf{p}_i)$ to a material index $i$ via the Index Function Texture.
        -   Map $i$ to a discrete style index $k$.
        -   Compute the surface normal $\mathbf{n}_i$ from the volume gradient.
        -   Map $\mathbf{n}_i$ to UV coordinates: $\mathbf{uv} = (\mathbf{n}_i.x, -\mathbf{n}_i.y) \cdot 0.5 + 0.5$.
        -   Sample the Style Transfer Texture at $(\mathbf{uv}, k)$ to get the base color $\mathbf{c}_{base}$.
        -   Combine $\mathbf{c}_{base}$ with opacity $\alpha_i$ and composite using the standard front-to-back equation.

**Formulas**
-   **Normal-to-UV Mapping**:
    $$
    \mathbf{uv} = (\mathbf{n}_i.x, -\mathbf{n}_i.y) \cdot 0.5 + 0.5
    $$
    This maps the hemisphere of normals to the unit square.
-   **Transfer Function Initialization**:
    $$
    \text{transferFunc}[i] = \text{vec4}\left(\frac{i}{255.0}, \frac{i}{255.0}, \frac{i}{255.0}, \frac{i}{255.0}\right)
    $$
    for $i \in [0, 255]$.

**Parameters**
-   `stepSize`: Ray casting step size. Default: `0.001f`.
-   `threshold`: Ray casting threshold. Default: `0.15f`.

**Edge Cases and Pitfalls**
-   **Precision**: The Back Face Texture uses `GL_RGBA32F` to ensure sufficient precision for storing exit points.
-   **Boundaries**: The volume texture uses `GL_CLAMP` wrap modes to prevent sampling artifacts at the volume boundaries.
-   **Degenerate Input**: If the volume data is empty or incorrectly loaded, `isLoaded` remains false, and rendering is skipped.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_volume_with_style_transfer(volume_texture, transfer_func_texture, index_func_texture, style_texture, camera_matrices, window_size):
    # Pass 1: Render back face to FBO to get exit points
    fbo = create_fbo(window_size, format='RGBA32F')
    bind_fbo(fbo)
    render_back_face(volume_texture, camera_matrices)
    unbind_fbo()

    # Pass 2: Ray casting with style transfer
    bind_default_framebuffer()
    set_viewport(window_size)
    use_shader(ray_cast_shader)
    set_uniform(ray_cast_shader, 'MVP', camera_matrices['modelViewProjection'])
    set_uniform(ray_cast_shader, 'ViewMatrix', camera_matrices['view'])
    set_uniform(ray_cast_shader, 'NormalMatrix', camera_matrices['normal'])
    set_uniform(ray_cast_shader, 'StepSize', 0.001f)
    set_uniform(ray_cast_shader, 'Threshold', 0.15f)
    set_uniform(ray_cast_shader, 'ScreenSize', window_size)

    bind_texture(1, transfer_func_texture, target='GL_TEXTURE_1D')
    set_uniform(ray_cast_shader, 'transferFunctionTexture', 1)

    bind_texture(2, index_func_texture, target='GL_TEXTURE_1D')
    set_uniform(ray_cast_shader, 'indexFunctionTexture', 2)

    bind_texture(3, style_texture, target='GL_TEXTURE_2D_ARRAY')
    set_uniform(ray_cast_shader, 'styleTransferTexture', 3)

    for y in range(window_size.y):
        for x in range(window_size.x):
            ray_origin, ray_direction = compute_ray(x, y, camera_matrices)
            exit_point = get_exit_point_from_fbo(x, y, fbo)
            color = vec4(0.0)
            alpha = 0.0

            t = 0.0
            while t < exit_point.z: # Assuming z is depth along ray
                sample_point = ray_origin + ray_direction * t
                scalar_value = sample_3d_texture(volume_texture, sample_point)
                
                # Map scalar to index
                material_index = sample_1d_texture(index_func_texture, scalar_value)
                
                # Map index to style
                style_index = map_index_to_style(material_index)
                
                # Compute normal
                normal = compute_gradient(volume_texture, sample_point)
                normal = normalize(normal)
                
                # Map normal to UV
                uv = vec2(normal.x, -normal.y) * 0.5 + 0.5
                
                # Sample style
                base_color = sample_2d_array_texture(style_texture, uv, style_index)
                
                # Get opacity from transfer function
                opacity = sample_1d_texture(transfer_func_texture, scalar_value).a
                
                # Composite
                color = color + base_color * opacity * (1.0 - alpha)
                alpha = alpha + opacity * (1.0 - alpha)
                
                t += step_size
                
                if alpha > 0.99:
                    break
            
            set_pixel_color(x, y, color)
```

## Subtopics

### Raycasting with Style Transfer and Contour Lines
Adds contour lines to the style transfer rendering. Choose when line-based features are needed. [Raycasting with Style Transfer and Contour Lines](11409-raycasting-with-style-transfer-and-contour-lines.md)

## Operating characteristics
-   **Time Complexity**: $O(W \cdot H \cdot N)$, where $W, H$ are image dimensions and $N$ is the number of steps per ray.
-   **Memory Complexity**: $O(W \cdot H \cdot D)$ for the volume texture, plus $O(W_{win} \cdot H_{win})$ for the back face texture.
-   **Measured Performance**: Not given in the sources.
-   **Strengths**: Allows interactive rotation and zoom with artistic styles. Supports saving and loading transfer functions with assigned styles.
-   **Limitations**: Performance depends on the number of steps and the complexity of the style textures.
-   **When to use**: When non-photorealistic visualization of volumetric data is required.
-   **When not to use**: When photorealistic rendering is needed.

## Background
This method is based on the academic paper "Style Transfer for Volume Rendering" by Bruckner et al. (2007) [S1, unofficial]. It was implemented as a homework project for a college course, extending the concepts of style transfer to volumetric data. The implementation uses OpenGL and SFML for rendering and user interface.

## Sources
[S1] Illustrative Volume Rendering with Style Transfer, https://github.com/jose-villegas/IllustrativeVolumeRendering, fetched 2026-10-26 (unofficial)
