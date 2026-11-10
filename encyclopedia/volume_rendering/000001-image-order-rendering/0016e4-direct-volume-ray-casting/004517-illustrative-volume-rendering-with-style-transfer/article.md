# Illustrative Volume Rendering with Style Transfer

Also called: Style Transfer Function (STF) Volume Rendering

## Overview
Illustrative Volume Rendering with Style Transfer combines standard direct volume ray casting with a Style Transfer Function (STF) to apply artistic styles to volumetric data [S1, unofficial]. It allows users to load a volume, save and load transfer functions with assigned styles, and interactively rotate or zoom around the data [S1, unofficial]. It sets itself apart from standard ray casting by introducing a dedicated texture-based mechanism to map scalar values not just to optical properties, but to artistic style attributes, enabling non-photorealistic visualization [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume dataset (raw data), camera parameters (view, projection, model matrices), and a set of style transfer functions.
*   **Output:** A 2D image with artistic styling applied to the volumetric structures.

**Data Structures and Layouts:**
The implementation relies on specific OpenGL textures and buffers to manage the ray casting and style transfer [S1, unofficial]:

1.  **Volume Texture:** A 3D texture (`GL_TEXTURE_3D`) with dimensions `width`, `height`, and `numCuts`. Internal format is `GL_INTENSITY`. Wrap modes are `GL_CLAMP` for S, T, and R coordinates. Min/Mag filters are `GL_LINEAR`. Data source is a float array `dataScalars` containing normalized volume values [S1, unofficial].
2.  **Transfer Function Texture:** A 1D texture (`GL_TEXTURE_1D`) of size 256. Internal format is `GL_RGBA8`. Wrap mode is `GL_REPEAT`. Min/Mag filters are `GL_NEAREST`. Pixel unpack alignment is 1. Data source is an array `transferFunc` of 256 `glm::vec4` entries, where `transferFunc[i] = glm::vec4((float)i / 255.f)` by default [S1, unofficial].
3.  **Index Function Texture:** A 1D texture (`GL_TEXTURE_1D`) used by the STF. Bound to texture unit 2 [S1, unofficial].
4.  **Style Function Texture:** A 2D array texture (`GL_TEXTURE_2D_ARRAY`) used to store multiple artistic styles. Bound to texture unit 3 [S1, unofficial].
5.  **Back Face Texture:** A 2D texture (`GL_TEXTURE_2D`) with size equal to the window resolution (`MainData::rootWindow->getSize().x` by `y`). Internal format is `GL_RGBA32F`. Wrap modes are `GL_REPEAT`. Min/Mag filters are `GL_NEAREST`. Used to store exit points for ray length calculation [S1, unofficial].
6.  **Vertex Buffer Object (VBO):** Defines 8 vertices of a unit cube (0.0-1.0 range) using 24 floats. Indices are 36 unsigned ints defining 12 triangles (2 per face, counter-clockwise). The Vertex Array Object (VAO) binds attribute 0 (vertex location) and attribute 1 (vertex color) to the same vertex data buffer. The index buffer is bound to `GL_ELEMENT_ARRAY_BUFFER` [S1, unofficial].
7.  **Frame Buffer Object (FBO):** Contains a depth renderbuffer (`GL_DEPTH_COMPONENT`) and a color attachment (`backFaceTexture`, `GL_TEXTURE_2D`). Depth testing is enabled [S1, unofficial].

**Parameters:**
*   `stepSize`: Ray casting step size. Default value: `0.001f` [S1, unofficial].
*   `threshold`: Ray casting threshold. Default value: `0.15f` [S1, unofficial].
*   `width`, `height`, `numCuts`: Dimensions of the volume data [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Initialize `RawDataModel`: set `isLoaded = false`, allocate `sModelName` (1024 chars), set `width`, `height`, `numCuts = 1`, `stepSize = 0.001f`, `threshold = 0.15f`, and texture/buffer IDs to 0.
    *   Initialize `transferFunc` array with 256 entries where `transferFunc[i] = glm::vec4((float)i / 255.f)`.
    *   Call `setupVolumeShaders()`, `createTransferFunctionTexture()`, and `stf.loadStyles()` [S1, unofficial].

2.  **Volume Loading:**
    *   Call `createVertexBuffer()` to set up VBO/VAO for cube rendering.
    *   Call `loadVolumeFromFile8()` or `loadVolumeFromFile16()` to read raw data into `dataScalars` (normalized floats).
    *   Call `createBackFaceTexture()` for exit point storage.
    *   Call `createFrameBuffer()` for FBO setup.
    *   Set dimensions, compute `cubeSizes = glm::vec3(width/max, height/max, numCuts/max)`, scale model, and position at `(-cubeSizes.x/2, -cubeSizes.y/2, 0)`.
    *   Compute projection (45 deg FOV, aspect ratio, 0.1-500 near/far), view (`lookAt(0,0,-1, 0,0,0, 0,1,0)`), model, normalMatrix, viewProjection, and modelViewProjection matrices [S1, unofficial].

3.  **Rendering (Two-Pass Approach):**
    *   **Pass 1: Back Face Rendering:** Render the back face of the volume bounding box to the FBO (`backFaceTexture`) to capture exit points. This defines the ray length for the subsequent ray casting pass [S1, unofficial].
    *   **Pass 2: Ray Casting with Style Transfer:**
        *   Bind the default framebuffer.
        *   Set viewport to window size.
        *   Use the `rayCastShader`.
        *   Set uniforms: `MVP` (modelViewProjection), `ViewMatrix`, `NormalMatrix`, `StepSize`, `Threshold`, `ScreenSize` [S1, unofficial].
        *   Bind textures:
            *   Texture Unit 1: `transferFunctionTexture` (1D, size 256).
            *   Texture Unit 2: `indexFunctionTexture` (1D).
            *   Texture Unit 3: `styleTransferTexture` (2D Array) [S1, unofficial].
        *   Execute the ray casting shader, which samples the volume texture, uses the transfer function for optical properties, and applies the style transfer function for artistic rendering [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_raw_data_model():
    """Initialize the RawDataModel structure and default values."""
    model = {
        'isLoaded': False,
        'sModelName': '\0' * 1024,
        'width': 0,
        'height': 0,
        'numCuts': 1,
        'stepSize': 0.001f,
        'threshold': 0.15f,
        'textureIds': [0, 0, 0, 0], # Placeholder for texture IDs
        'bufferIds': [0, 0, 0, 0]   # Placeholder for buffer IDs
    }
    # Initialize transfer function with default identity mapping
    model['transferFunc'] = [glm.vec4(i / 255.0) for i in range(256)]
    return model

def load_volume_data(model, file_path):
    """Load volume data from file and setup OpenGL resources."""
    # Load raw data
    if file_path.endswith('.8'):
        model['dataScalars'] = load_volume_from_file_8(file_path)
    else:
        model['dataScalars'] = load_volume_from_file_16(file_path)
    
    # Normalize dataScalars to [0, 1] if not already done by loader
    
    # Setup VBO/VAO for cube
    create_vertex_buffer(model)
    
    # Setup Back Face Texture for exit points
    create_back_face_texture(model)
    
    # Setup Frame Buffer Object
    create_frame_buffer(model)
    
    # Compute dimensions and matrices
    max_dim = max(model['width'], model['height'], model['numCuts'])
    model['cubeSizes'] = glm.vec3(model['width']/max_dim, model['height']/max_dim, model['numCuts']/max_dim)
    
    # Position model
    model['position'] = glm.vec3(-model['cubeSizes'].x/2, -model['cubeSizes'].y/2, 0)
    
    # Compute matrices
    aspect = model['width'] / model['height']
    model['projection'] = glm.perspective(glm.radians(45.0), aspect, 0.1, 500.0)
    model['view'] = glm.lookAt(glm.vec3(0, 0, -1), glm.vec3(0, 0, 0), glm.vec3(0, 1, 0))
    model['model'] = glm.translate(glm.mat4(1.0), model['position'])
    # Scale model by cubeSizes would be applied here if needed
    model['normalMatrix'] = glm.mat3(glm.inverseTranspose(model['model']))
    model['viewProjection'] = model['projection'] * model['view']
    model['modelViewProjection'] = model['viewProjection'] * model['model']
    
    # Create textures
    create_transfer_function_texture(model)
    model['stf'].load_styles() # Load style transfer functions
    
    model['isLoaded'] = True

def render(model):
    """Main rendering loop."""
    if not model['isLoaded']:
        return
    
    # Pass 1: Render back face to FBO to get exit points
    render_back_face(model)
    
    # Pass 2: Ray casting with style transfer
    render_volume_ray_casting(model)

def render_back_face(model):
    """Render the back face of the volume to the back face texture."""
    glBindFramebuffer(GL_FRAMEBUFFER, model['fboId'])
    glViewport(0, 0, model['windowWidth'], model['windowHeight'])
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT)
    
    # Bind back face shader and render cube
    # ... (Shader setup and draw calls) ...
    
    glBindFramebuffer(GL_FRAMEBUFFER, 0)

def render_volume_ray_casting(model):
    """Render the volume using ray casting and style transfer."""
    glBindFramebuffer(GL_FRAMEBUFFER, 0)
    glViewport(0, 0, model['windowWidth'], model['windowHeight'])
    
    # Use ray casting shader
    model['rayCastShader'].use()
    
    # Set uniforms
    model['rayCastShader'].set_uniform("MVP", model['modelViewProjection'])
    model['rayCastShader'].set_uniform("ViewMatrix", model['view'])
    model['rayCastShader'].set_uniform("NormalMatrix", model['normalMatrix'])
    model['rayCastShader'].set_uniform("StepSize", model['stepSize'])
    model['rayCastShader'].set_uniform("Threshold", model['threshold'])
    model['rayCastShader'].set_uniform("ScreenSize", model['windowWidth'], model['windowHeight'])
    
    # Bind Transfer Function Texture (Unit 1)
    glActiveTexture(GL_TEXTURE1)
    glBindTexture(GL_TEXTURE_1D, model['transferFunctionTextureId'])
    model['rayCastShader'].set_uniform("transferFunctionTexture", 1)
    
    # Bind Index Function Texture (Unit 2)
    glActiveTexture(GL_TEXTURE2)
    glBindTexture(GL_TEXTURE_1D, model['stf']['indexFunctionTextureId'])
    model['rayCastShader'].set_uniform("indexFunctionTexture", 2)
    
    # Bind Style Transfer Texture (Unit 3)
    glActiveTexture(GL_TEXTURE3)
    glBindTexture(GL_TEXTURE_2D_ARRAY, model['stf']['styleTransferTextureId'])
    model['rayCastShader'].set_uniform("styleTransferTexture", 3)
    
    # Bind Volume Texture (Unit 0, assumed)
    glActiveTexture(GL_TEXTURE0)
    glBindTexture(GL_TEXTURE_3D, model['volumeTextureId'])
    model['rayCastShader'].set_uniform("volumeTexture", 0)
    
    # Draw cube
    # ... (Draw calls using VAO) ...

def create_transfer_function_texture(model):
    """Create the 1D transfer function texture."""
    model['transferFunctionTextureId'] = glGenTextures(1)
    glBindTexture(GL_TEXTURE_1D, model['transferFunctionTextureId'])
    glTexImage1D(GL_TEXTURE_1D, 0, GL_RGBA8, 256, 0, GL_RGBA, GL_FLOAT, model['transferFunc'])
    glTexParameteri(GL_TEXTURE_1D, GL_TEXTURE_WRAP_S, GL_REPEAT)
    glTexParameteri(GL_TEXTURE_1D, GL_TEXTURE_MIN_FILTER, GL_NEAREST)
    glTexParameteri(GL_TEXTURE_1D, GL_TEXTURE_MAG_FILTER, GL_NEAREST)
    glPixelStorei(GL_UNPACK_ALIGNMENT, 1)

def create_back_face_texture(model):
    """Create the 2D back face texture."""
    model['backFaceTextureId'] = glGenTextures(1)
    glBindTexture(GL_TEXTURE_2D, model['backFaceTextureId'])
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA32F, model['windowWidth'], model['windowHeight'], 0, GL_RGBA, GL_FLOAT, None)
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT)
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT)
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST)
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST)

def create_frame_buffer(model):
    """Create the Frame Buffer Object."""
    model['fboId'] = glGenFramebuffers(1)
    glBindFramebuffer(GL_FRAMEBUFFER, model['fboId'])
    
    # Depth renderbuffer
    model['depthBufferId'] = glGenRenderbuffers(1)
    glBindRenderbuffer(GL_RENDERBUFFER, model['depthBufferId'])
    glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH_COMPONENT, model['windowWidth'], model['windowHeight'])
    glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_RENDERBUFFER, model['depthBufferId'])
    
    # Color attachment (back face texture)
    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, model['backFaceTextureId'], 0)
    
    # Check completeness
    if glCheckFramebufferStatus(GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE:
        raise Exception("Framebuffer is not complete!")
    
    glBindFramebuffer(GL_FRAMEBUFFER, 0)

def create_vertex_buffer(model):
    """Create VBO/VAO for the unit cube."""
    # Define 8 vertices of a unit cube (0.0-1.0)
    vertices = [
        # x, y, z, r, g, b, a
        0.0, 0.0, 0.0, 1.0, 1.0, 1.0, 1.0,
        1.0, 0.0, 0.0, 1.0, 1.0, 1.0, 1.0,
        1.0, 1.0, 0.0, 1.0, 1.0, 1.0, 1.0,
        0.0, 1.0, 0.0, 1.0, 1.0, 1.0, 1.0,
        0.0, 0.0, 1.0, 1.0, 1.0, 1.0, 1.0,
        1.0, 0.0, 1.0, 1.0, 1.0, 1.0, 1.0,
        1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0,
        0.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0
    ]
    indices = [
        0, 1, 2, 0, 2, 3, # Front
        4, 6, 5, 4, 7, 6, # Back
        0, 4, 5, 0, 5, 1, # Bottom
        3, 7, 6, 3, 6, 2, # Top
        3, 2, 6, 3, 6, 7, # Right
        0, 5, 1, 0, 4, 5  # Left
    ]
    
    model['vboId'] = glGenBuffers(1)
    glBindBuffer(GL_ARRAY_BUFFER, model['vboId'])
    glBufferData(GL_ARRAY_BUFFER, len(vertices) * 4, vertices, GL_STATIC_DRAW) # Assuming 4 bytes per float
    
    model['vaoId'] = glGenVertexArrays(1)
    glBindVertexArray(model['vaoId'])
    
    # Attribute 0: Vertex Location
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 28, ctypes.c_void_p(0))
    glEnableVertexAttribArray(0)
    
    # Attribute 1: Vertex Color
    glVertexAttribPointer(1, 4, GL_FLOAT, GL_FALSE, 28, ctypes.c_void_p(12))
    glEnableVertexAttribArray(1)
    
    model['eboId'] = glGenBuffers(1)
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, model['eboId'])
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, len(indices) * 4, indices, GL_STATIC_DRAW) # Assuming 4 bytes per int

def create_volume_texture(model):
    """Create the 3D volume texture."""
    model['volumeTextureId'] = glGenTextures(1)
    glBindTexture(GL_TEXTURE_3D, model['volumeTextureId'])
    glTexImage3D(GL_TEXTURE_3D, 0, GL_INTENSITY, model['width'], model['height'], model['numCuts'], 0, GL_RED, GL_FLOAT, model['dataScalars'])
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_WRAP_S, GL_CLAMP)
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_WRAP_T, GL_CLAMP)
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_WRAP_R, GL_CLAMP)
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_MIN_FILTER, GL_LINEAR)
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_MAG_FILTER, GL_LINEAR)
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels and the number of samples per ray ($O(W \cdot H \cdot N)$), where $W$ and $H$ are image dimensions and $N$ is the number of samples. The two-pass approach adds a constant overhead for the back face rendering.
*   **Memory Complexity:** Linear with respect to the volume size ($O(W_{vol} \cdot H_{vol} \cdot D_{vol})$) for the volume data, plus constant overhead for textures and buffers.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Performance scales with resolution and volume size. The use of GPU-accelerated OpenGL (VBOs, FBOs, Textures) suggests good scalability on modern hardware.
*   **Strengths:** Interactive rotation and zoom; ability to save/load transfer functions with styles; non-photorealistic visualization.
*   **Limitations:** Specific to the implementation details provided; relies on OpenGL 4.0 features; no explicit mention of parallelization beyond standard GPU rendering.
*   **When to use:** When artistic or illustrative visualization of volumetric data is desired, and interactive exploration is needed.
*   **When not to use:** When photorealistic rendering is required, or when hardware does not support OpenGL 4.0.

## Background
The method is based on the academic paper "Volume Rendering and Style Transfer Function" by Bruckner et al. (2007) [S1, unofficial]. It was implemented as a homework project for a college course [S1, unofficial]. The technique extends standard volume rendering by incorporating style transfer functions, allowing for artistic manipulation of the rendered output [S1, unofficial].

## Sources
[S1] jose-villegas/IllustrativeVolumeRendering, https://github.com/jose-villegas/IllustrativeVolumeRendering, fetched 2026-10-26 (unofficial)
