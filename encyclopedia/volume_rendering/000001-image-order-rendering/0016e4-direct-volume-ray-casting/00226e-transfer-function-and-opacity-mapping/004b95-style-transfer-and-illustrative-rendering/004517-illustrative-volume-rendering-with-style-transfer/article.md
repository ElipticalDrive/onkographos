# Illustrative Volume Rendering with Style Transfer

Also called: Style Transfer Function (STF) Volume Rendering

## Overview
Illustrative Volume Rendering with Style Transfer combines direct volume ray casting with a Style Transfer Function (STF) to apply artistic styles to volumetric data [S1, unofficial]. It allows users to load a volume, save and load transfer functions with assigned styles, and interactively rotate or zoom around the data [S1, unofficial]. The method is distinguished by its two-pass rendering approach: first capturing exit points via back-face rendering, then performing ray casting that utilizes these exit points alongside style transfer textures to generate the final image [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   Scalar volume data (8-bit or 16-bit raw data, normalized to floats) [S1, unofficial].
*   Camera parameters: View matrix, Model matrix, Normal matrix, View-Projection matrix, Model-View-Projection (MVP) matrix [S1, unofficial].
*   Rendering parameters: Step size (default 0.001f), Threshold (default 0.15f), Screen resolution [S1, unofficial].
*   Textures:
    *   Volume Texture: 3D texture (`GL_TEXTURE_3D`) containing the scalar data [S1, unofficial].
    *   Transfer Function Texture: 1D texture (`GL_TEXTURE_1D`) of size 256, format `GL_RGBA8`, storing color and opacity mappings [S1, unofficial].
    *   Index Function Texture: 1D texture (`GL_TEXTURE_1D`) [S1, unofficial].
    *   Style Transfer Texture: 2D array texture (`GL_TEXTURE_2D_ARRAY`) [S1, unofficial].
    *   Back Face Texture: 2D texture (`GL_TEXTURE_2D`) storing exit points [S1, unofficial].

**Outputs:**
*   Final rendered image with artistic style applied [S1, unofficial].

### Data Structures
**RawDataModel Initialization:**
*   `isLoaded`: Boolean, initialized to `false` [S1, unofficial].
*   `sModelName`: String, 1024 characters [S1, unofficial].
*   `width`, `height`, `numCuts`: Integers, initialized to 1 [S1, unofficial].
*   `stepSize`: Float, default 0.001f [S1, unofficial].
*   `threshold`: Float, default 0.15f [S1, unofficial].
*   `texture/buffer IDs`: Integers, initialized to 0 [S1, unofficial].
*   `transferFunc`: Array of 256 `glm::vec4` entries. Initialized such that `transferFunc[i] = glm::vec4((float)i / 255.f)` [S1, unofficial].

**Vertex Buffer Object (VBO) Layout:**
*   Vertices: 24 floats defining 8 vertices of a unit cube (range 0.0-1.0) [S1, unofficial].
*   Indices: 36 unsigned ints defining 12 triangles (2 per face, counter-clockwise) [S1, unofficial].
*   Vertex Array Object (VAO): Binds attribute 0 (vertex location) and attribute 1 (vertex color) to the same vertex data buffer. Index buffer bound to `GL_ELEMENT_ARRAY_BUFFER` [S1, unofficial].

**Texture Layouts:**
*   **Back Face Texture:** 2D texture (`GL_TEXTURE_2D`) with dimensions equal to the window size. Internal format `GL_RGBA32F`. Wrap modes `GL_REPEAT`. Min/Mag filter `GL_NEAREST` [S1, unofficial].
*   **Transfer Function Texture:** 1D texture (`GL_TEXTURE_1D`) of size 256. Internal format `GL_RGBA8`. Wrap mode `GL_REPEAT`. Min/Mag filter `GL_NEAREST`. Pixel unpack alignment 1 [S1, unofficial].
*   **Volume Texture:** 3D texture (`GL_TEXTURE_3D`) with dimensions `width`, `height`, `numCuts`. Internal format `GL_INTENSITY`. Wrap modes `GL_CLAMP` for S, T, R. Min/Mag filter `GL_LINEAR` [S1, unofficial].

**Frame Buffer Object (FBO):**
*   Contains a depth renderbuffer (`GL_DEPTH_COMPONENT`) and a color attachment (`backFaceTexture`, `GL_TEXTURE_2D`). Depth test enabled [S1, unofficial].

### Steps
1.  **Initialize RawDataModel:** Set `isLoaded=false`, allocate `sModelName` (1024 chars), set `width/height/numCuts=1`, `stepSize=0.001f`, `threshold=0.15f`, and texture/buffer IDs to 0. Initialize `transferFunc` array with 256 entries where `transferFunc[i] = glm::vec4((float)i / 255.f)`. Call `setupVolumeShaders()`, `createTransferFunctionTexture()`, and `stf.loadStyles()` [S1, unofficial].
2.  **Load Volume Data:**
    *   Call `createVertexBuffer()` to set up VBO/VAO for cube rendering [S1, unofficial].
    *   Call `loadVolumeFromFile8()` or `loadVolumeFromFile16()` to read raw data into `dataScalars` (normalized floats) [S1, unofficial].
    *   Call `createBackFaceTexture()` for exit point storage [S1, unofficial].
    *   Call `createFrameBuffer()` for FBO setup [S1, unofficial].
    *   Set dimensions, compute `cubeSizes = glm::vec3(width/max, height/max, numCuts/max)`, scale model, position at `(-cubeSizes.x/2, -cubeSizes.y/2, 0)` [S1, unofficial].
    *   Compute projection (45 deg FOV, aspect ratio, 0.1-500 near/far), view (`lookAt(0,0,-1, 0,0,0, 0,1,0)`), model, normalMatrix, viewProjection, and modelViewProjection matrices [S1, unofficial].
3.  **Render Back Face:** Render the back face of the volume to the FBO to capture exit points [S1, unofficial].
4.  **Render Volume Ray Casting:**
    *   Bind framebuffer to 0 (default) [S1, unofficial].
    *   Set viewport to window size [S1, unofficial].
    *   Use ray cast shader [S1, unofficial].
    *   Set uniforms: `MVP`, `ViewMatrix`, `NormalMatrix`, `StepSize`, `Threshold`, `ScreenSize` [S1, unofficial].
    *   Bind Transfer Function Texture to texture unit 1 (`transferFunctionTexture`) [S1, unofficial].
    *   Bind Index Function Texture to texture unit 2 (`indexFunctionTexture`) [S1, unofficial].
    *   Bind Style Transfer Texture to texture unit 3 (`styleTransferTexture`) [S1, unofficial] [S1, unofficial].
    *   Execute ray casting pass [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_raw_data_model():
    is_loaded = False
    s_model_name = "" # 1024 chars
    width = 1
    height = 1
    num_cuts = 1
    step_size = 0.001f
    threshold = 0.15f
    texture_ids = [0, 0, 0, 0] # texture/buffer IDs
    
    # Initialize transfer function
    transfer_func = array[256] of vec4
    for i from 0 to 255:
        transfer_func[i] = vec4(float(i) / 255.0f)
        
    setup_volume_shaders()
    create_transfer_function_texture()
    stf.load_styles()

def load_volume_data(volume_file):
    create_vertex_buffer() # Sets up VBO/VAO for cube
    
    # Load data
    if volume_file.bits == 8:
        data_scalars = load_volume_from_file_8(volume_file)
    else:
        data_scalars = load_volume_from_file_16(volume_file)
        
    # Normalize data_scalars to floats
    
    create_back_face_texture()
    create_frame_buffer()
    
    # Compute dimensions and matrices
    cube_sizes = vec3(width / max_dim, height / max_dim, num_cuts / max_dim)
    scale_model(cube_sizes)
    position_model(vec3(-cube_sizes.x / 2, -cube_sizes.y / 2, 0))
    
    projection_matrix = perspective(45.0f, aspect_ratio, 0.1f, 500.0f)
    view_matrix = look_at(vec3(0, 0, -1), vec3(0, 0, 0), vec3(0, 1, 0))
    model_matrix = compute_model_matrix()
    normal_matrix = compute_normal_matrix(model_matrix)
    view_projection_matrix = projection_matrix * view_matrix
    model_view_projection_matrix = view_projection_matrix * model_matrix

def render():
    if not is_loaded:
        return
        
    render_back_face()
    render_volume_ray_casting()

def render_back_face():
    bind_framebuffer(fbo)
    # Render back face to capture exit points
    unbind_framebuffer()

def render_volume_ray_casting():
    bind_framebuffer(0)
    set_viewport(0, 0, window_width, window_height)
    
    use_shader(ray_cast_shader)
    
    set_uniform(ray_cast_shader, "MVP", model_view_projection_matrix)
    set_uniform(ray_cast_shader, "ViewMatrix", view_matrix)
    set_uniform(ray_cast_shader, "NormalMatrix", normal_matrix)
    set_uniform(ray_cast_shader, "StepSize", step_size)
    set_uniform(ray_cast_shader, "Threshold", threshold)
    set_uniform(ray_cast_shader, "ScreenSize", vec2(window_width, window_height))
    
    # Bind textures
    active_texture(GL_TEXTURE1)
    bind_texture(GL_TEXTURE_1D, stf.transfer_function_texture)
    set_uniform(ray_cast_shader, "transferFunctionTexture", 1)
    
    active_texture(GL_TEXTURE2)
    bind_texture(GL_TEXTURE_1D, stf.index_function_texture)
    set_uniform(ray_cast_shader, "indexFunctionTexture", 2)
    
    active_texture(GL_TEXTURE3)
    bind_texture(GL_TEXTURE_2D_ARRAY, stf.style_transfer_texture)
    set_uniform(ray_cast_shader, "styleTransferTexture", 3)
    
    # Execute ray casting
    draw_cube()
```

### Parameters
*   `stepSize`: Ray casting step size, default 0.001f [S1, unofficial].
*   `threshold`: Ray casting threshold, default 0.15f [S1, unofficial].

### Edge Cases and Pitfalls
*   **Precision:** The Back Face Texture uses `GL_RGBA32F` for exit point storage, ensuring high precision for ray length calculations [S1, unofficial].
*   **Boundaries:** Volume texture wrap modes are `GL_CLAMP` to prevent sampling artifacts at volume boundaries [S1, unofficial].
*   **Initialization:** The transfer function must be initialized before rendering to avoid undefined behavior [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Allows interactive rotation and zoom; supports saving and loading transfer functions with assigned styles [S1, unofficial].
*   **Limitations:** Not given in the sources.
*   **When to use:** When artistic style transfer is desired for volumetric data with interactive exploration [S1, unofficial].
*   **When not to use:** When high performance or scientific accuracy is prioritized over artistic style [S1, unofficial].

## Background
The method is based on the academic paper "Style Transfer Function" by Bruckner et al. (2007) [S1, unofficial]. It was implemented as a homework project for college [S1, unofficial].

## Sources
[S1] jose-villegas/IllustrativeVolumeRendering, https://github.com/jose-villegas/IllustrativeVolumeRendering, fetched 2026-10-26 (unofficial)
