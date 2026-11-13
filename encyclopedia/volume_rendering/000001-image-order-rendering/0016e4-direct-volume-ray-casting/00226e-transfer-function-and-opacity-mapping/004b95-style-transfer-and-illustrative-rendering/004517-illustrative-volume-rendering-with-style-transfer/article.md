# Illustrative Volume Rendering with Style Transfer

Also called: Style Transfer Function (STF) Volume Rendering

## Overview
Illustrative Volume Rendering with Style Transfer applies artistic styles to volumetric data by replacing standard color/opacity lookups with a **Style Transfer Function (STF)** [S1, unofficial]. It maps volume density to a material index, then to a style index, and finally retrieves base colors from a 2D array texture based on surface normals, enabling interactive rotation, zoom, and style assignment [S1, unofficial]. It differs from standard ray casting by decoupling material identification from artistic appearance via a two-stage indexing process and lit-sphere mapping [11409-raycasting-with-style-transfer-and-contour-lines, 11445-illustrative-volume-rendering-with-style-transfer].

## How it works
**Inputs and Outputs:**
- **Input:** Volumetric scalar data (density), a transfer function texture mapping density to material indices, an index function texture mapping material indices to style indices, a 2D array texture storing artistic styles, and camera/view matrices.
- **Output:** A 2D image with stylized volumetric rendering.

**Data Structures:**
1.  **Volume Texture:** 3D texture (`GL_TEXTURE_3D`) with dimensions $W \times H \times D$ (width, height, numCuts). Internal format `GL_INTENSITY`. Wrap modes `GL_CLAMP` for S, T, R. Min/Mag filter `GL_LINEAR`. Data source is normalized float array `dataScalars` [S1, unofficial].
2.  **Transfer Function Texture:** 1D texture (`GL_TEXTURE_1D`) of size 256. Internal format `GL_RGBA8`. Wrap mode `GL_REPEAT`. Min/Mag filter `GL_NEAREST`. Pixel unpack alignment 1. Data source is `transferFunc` array of `glm::vec4` [S1, unofficial].
3.  **Index Function Texture:** 1D texture (`GL_TEXTURE_1D`) mapping material indices to style indices. Internal format not explicitly specified in source, but implied to be integer or float index [11409-raycasting-with-style-transfer-and-contour-lines, 11445-illustrative-volume-rendering-with-style-transfer].
4.  **Style Transfer Texture:** 2D array texture (`GL_TEXTURE_2D_ARRAY`). Dimensions correspond to lit-sphere coordinates (width, height) and number of styles (layers). Internal format not explicitly specified, but stores base colors $\mathbf{c}_{base}$ [11409-raycasting-with-style-transfer-and-contour-lines, 11445-illustrative-volume-rendering-with-style-transfer].
5.  **Back Face Texture:** 2D texture (`GL_TEXTURE_2D`) with size $W_{screen} \times H_{screen}$. Internal format `GL_RGBA32F`. Wrap modes `GL_REPEAT`. Min/Mag filter `GL_NEAREST`. Used to store exit points of rays through the volume [S1, unofficial].
6.  **Vertex Buffer Object (VBO):** 24 floats defining 8 vertices of a unit cube (0.0-1.0 range). Indices: 36 unsigned ints defining 12 triangles (2 per face, counter-clockwise). VAO binds attribute 0 (vertex location) and attribute 1 (vertex color) to the same vertex data buffer. Index buffer bound to `GL_ELEMENT_ARRAY_BUFFER` [S1, unofficial].
7.  **Frame Buffer Object (FBO):** Contains a depth renderbuffer (`GL_DEPTH_COMPONENT`) and a color attachment (`backFaceTexture`, `GL_TEXTURE_2D`). Depth test enabled [S1, unofficial].

**Numbered Steps:**
1.  **Initialization:**
    -   Set `isLoaded = false`.
    -   Allocate `sModelName` (1024 chars).
    -   Set `width`, `height`, `numCuts = 1`.
    -   Set `stepSize = 0.001f`, `threshold = 0.15f`.
    -   Set texture/buffer IDs to 0.
    -   Initialize `transferFunc` array with 256 entries where `transferFunc[i] = glm::vec4((float)i / 255.f)` [S1, unofficial].
    -   Call `setupVolumeShaders()`, `createTransferFunctionTexture()`, and `stf.loadStyles()` [S1, unofficial].

2.  **Volume Loading:**
    -   Call `createVertexBuffer()` to set up VBO/VAO for cube rendering [S1, unofficial].
    -   Call `loadVolumeFromFile8()` or `loadVolumeFromFile16()` to read raw data into `dataScalars` (normalized floats) [S1, unofficial].
    -   Call `createBackFaceTexture()` for exit point storage [S1, unofficial].
    -   Call `createFrameBuffer()` for FBO setup [S1, unofficial].
    -   Set dimensions, compute `cubeSizes = glm::vec3(width/max, height/max, numCuts/max)`.
    -   Scale model, position at `(-cubeSizes.x/2, -cubeSizes.y/2, 0)`.
    -   Compute projection (45 deg FOV, aspect ratio, 0.1-500 near/far), view (`lookAt(0,0,-1, 0,0,0, 0,1,0)`), model, normalMatrix, viewProjection, and modelViewProjection matrices [S1, unofficial].

3.  **Back Face Rendering (Exit Point Capture):**
    -   Render the back face of the volume bounding box into the FBO (`backFaceTexture`) to capture exit points [S1, unofficial]. This facilitates efficient ray marching by determining ray length [11409-raycasting-with-style-transfer-and-contour-lines, 11445-illustrative-volume-rendering-with-style-transfer].

4.  **Ray Casting with Style Transfer:**
    -   Bind FBO to 0 (default framebuffer).
    -   Set viewport to screen size.
    -   Use ray cast shader.
    -   Set uniforms: `MVP`, `ViewMatrix`, `NormalMatrix`, `StepSize`, `Threshold`, `ScreenSize` [S1, unofficial].
    -   Bind textures:
        -   `transferFunctionTexture` (1D, unit 1) [S1, unofficial].
        -   `indexFunctionTexture` (1D, unit 2) [S1, unofficial].
        -   `styleTransferTexture` (2D Array, unit 3) [S1, unofficial].
    -   For each pixel, cast a ray through the volume using the exit point from the back face texture to determine ray length.
    -   Sample volume data along the ray.
    -   Map density $V$ to material index $idx$ using `transferFunctionTexture` [11409-raycasting-with-style-transfer-and-contour-lines, 11445-illustrative-volume-rendering-with-style-transfer].
    -   Map material index $idx$ to style index $s$ using `indexFunctionTexture` [11409-raycasting-with-style-transfer-and-contour-lines, 11445-illustrative-volume-rendering-with-style-transfer].
    -   Compute surface normal $\mathbf{n}$ at the sample point.
    -   Project $\mathbf{n}$ onto lit sphere coordinates $\mathbf{u}$:
        $$ \mathbf{u} = \begin{bmatrix} n_x \\ -n_y \end{bmatrix} \cdot 0.5 + 0.5 $$
        [11409-raycasting-with-style-transfer-and-contour-lines]
    -   Retrieve base color $\mathbf{c}_{base}$ from `styleTransferTexture` using coordinates $\mathbf{u}$ and layer $s$ [11409-raycasting-with-style-transfer-and-contour-lines, 11445-illustrative-volume-rendering-with-style-transfer].
    -   Composite colors and opacities along the ray to produce the final pixel color [S1, unofficial].

**Parameters:**
-   `stepSize`: Ray casting step size, default `0.001f` [S1, unofficial].
-   `threshold`: Ray casting threshold, default `0.15f` [S1, unofficial].

**Edge Cases and Pitfalls:**
-   Precision: Use `GL_RGBA32F` for back face texture to ensure accurate exit point storage [S1, unofficial].
-   Boundaries: Clamp volume texture coordinates to avoid wrapping artifacts [S1, unofficial].
-   Degenerate Input: Handle empty or invalid volume data by checking `isLoaded` flag [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_raw_data_model():
    is_loaded = False
    s_model_name = allocate_string(1024)
    width = 0
    height = 0
    num_cuts = 1
    step_size = 0.001f
    threshold = 0.15f
    texture_ids = [0] * 10
    buffer_ids = [0] * 10
    
    transfer_func = array_of_vec4(256)
    for i in range(256):
        transfer_func[i] = vec4(float(i) / 255.0f)
        
    setup_volume_shaders()
    create_transfer_function_texture(transfer_func)
    stf_load_styles()

def load_volume_data(raw_data_path):
    create_vertex_buffer()
    data_scalars = load_volume_file(raw_data_path) # Returns normalized floats
    create_back_face_texture()
    create_frame_buffer()
    
    # Set dimensions from loaded data
    width, height, num_cuts = get_volume_dimensions(data_scalars)
    
    cube_sizes = vec3(width / max_dim, height / max_dim, num_cuts / max_dim)
    scale_model(cube_sizes)
    position_model(vec3(-cube_sizes.x / 2, -cube_sizes.y / 2, 0))
    
    projection_matrix = compute_projection(fov=45, aspect=width/height, near=0.1, far=500)
    view_matrix = look_at(vec3(0, 0, -1), vec3(0, 0, 0), vec3(0, 1, 0))
    model_matrix = compute_model_matrix()
    normal_matrix = compute_normal_matrix(model_matrix)
    view_projection_matrix = projection_matrix * view_matrix
    model_view_projection_matrix = view_projection_matrix * model_matrix
    
    is_loaded = True

def render():
    if not is_loaded:
        return
    render_back_face()
    render_volume_ray_casting()

def render_back_face():
    bind_frame_buffer(back_face_fbo)
    clear_depth()
    draw_cube_back_face()
    unbind_frame_buffer()

def render_volume_ray_casting():
    bind_frame_buffer(0)
    set_viewport(0, 0, screen_width, screen_height)
    use_ray_cast_shader()
    
    set_uniform("MVP", model_view_projection_matrix)
    set_uniform("ViewMatrix", view_matrix)
    set_uniform("NormalMatrix", normal_matrix)
    set_uniform("StepSize", step_size)
    set_uniform("Threshold", threshold)
    set_uniform("ScreenSize", vec2(screen_width, screen_height))
    
    active_texture(1)
    bind_texture_1d(1, transfer_function_texture)
    set_uniform("transferFunctionTexture", 1)
    
    active_texture(2)
    bind_texture_1d(2, index_function_texture)
    set_uniform("indexFunctionTexture", 2)
    
    active_texture(3)
    bind_texture_2d_array(3, style_transfer_texture)
    set_uniform("styleTransferTexture", 3)
    
    draw_full_screen_quad()

def ray_cast_shader_main():
    screen_uv = get_screen_uv()
    ray_origin = unproject(screen_uv, view_matrix, projection_matrix)
    ray_direction = normalize(unproject(screen_uv + vec2(1.0, 0.0), view_matrix, projection_matrix) - ray_origin)
    
    # Get exit point from back face texture
    exit_point = texture(back_face_texture, screen_uv).xyz
    
    # Calculate ray length
    ray_length = length(exit_point - ray_origin)
    
    # Ray marching
    accumulated_color = vec4(0.0)
    accumulated_opacity = 0.0
    
    for t in range(0, int(ray_length / step_size)):
        sample_pos = ray_origin + ray_direction * t * step_size
        density = texture(volume_texture, sample_pos).r
        
        if density < threshold:
            continue
            
        # Map density to material index
        material_index = texture(transfer_function_texture, vec2(density, 0.0)).r
        
        # Map material index to style index
        style_index = texture(index_function_texture, vec2(material_index, 0.0)).r
        
        # Compute normal (simplified, assuming gradient or precomputed)
        normal = compute_normal(sample_pos)
        
        # Map normal to lit sphere coordinates
        u = vec2(normal.x, -normal.y) * 0.5 + 0.5
        
        # Retrieve base color from style transfer texture
        base_color = texture(style_transfer_texture, vec3(u, style_index)).rgb
        
        # Composite (simplified)
        alpha = 1.0 - exp(-density * step_size * 10.0) # [G] Typical opacity mapping
        accumulated_color = accumulated_color * (1.0 - alpha) + base_color * alpha
        accumulated_opacity += alpha * (1.0 - accumulated_opacity)
        
        if accumulated_opacity > 0.99:
            break
            
    set_fragment_color(accumulated_color)
```

## Subtopics

### [Raycasting with Style Transfer and Contour Lines](11409-raycasting-with-style-transfer-and-contour-lines.md)
Adds contour lines to the stylized rendering. Choose when edge emphasis is needed alongside style.

### [Style Transfer and Illustrative Rendering](13031-style-transfer-and-illustrative-rendering.md)
General family of methods using STF. Choose for broader artistic control beyond basic style indexing.

## Operating characteristics
-   **Time Complexity:** $O(N \cdot M)$ where $N$ is the number of pixels and $M$ is the number of samples per ray. $M$ is determined by ray length and `stepSize` [S1, unofficial].
-   **Memory Complexity:** $O(W \cdot H \cdot D)$ for volume data, $O(W \cdot H)$ for back face texture, and $O(S \cdot W_{lit} \cdot H_{lit})$ for style transfer texture, where $S$ is the number of styles [S1, unofficial].
-   **Measured Performance:** Not given in the sources.
-   **Scaling:** Performance scales linearly with resolution and volume size. `stepSize` controls accuracy vs. speed [S1, unofficial].
-   **Strengths:** Interactive style assignment, efficient ray length calculation via back face texture, decoupling of material and style [S1, unofficial].
-   **Limitations:** Requires pre-computation of back face texture, limited to discrete styles, potential aliasing with large `stepSize` [S1, unofficial].
-   **When to use:** When artistic stylization of volumetric data is required with interactive control.
-   **When not to use:** When photorealistic rendering is the primary goal, or when real-time performance on low-end hardware is critical without style complexity.

## Background
The method is based on the academic paper "Volume Rendering and Style Transfer Function" by Bruckner et al. (2007) [S1, unofficial]. It was implemented as a homework project for a college course, demonstrating the integration of style transfer into volume rendering pipelines [S1, unofficial].

## Sources
[S1] jose-villegas/IllustrativeVolumeRendering, https://github.com/jose-villegas/IllustrativeVolumeRendering, fetched 2026-10-26 (unofficial)
