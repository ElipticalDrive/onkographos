# Illustrative Volume Rendering with Style Transfer

Also called: Style Transfer Function (STF) Volume Rendering

## Overview
Illustrative Volume Rendering with Style Transfer combines direct volume ray casting with a Style Transfer Function (STF) to apply artistic, non-photorealistic styles to volumetric data [S1, unofficial]. It solves the problem of visualizing scalar volumes with aesthetic qualities (e.g., lit sphere shading, matcap textures) rather than standard lighting models. It is distinguished from its sibling, [Raycasting with Style Transfer and Contour Lines](11409-raycasting-with-style-transfer-and-contour-lines.md), by its specific two-pass architecture using a back-face exit point texture to drive ray length, whereas the sibling may integrate contour lines directly into the style mapping or use different boundary handling [S1, unofficial].

## How it works
The method uses a two-pass rendering approach. The first pass renders the back face of the volume bounding box to a Frame Buffer Object (FBO) to capture exit points. The second pass performs volume ray casting, using the exit points to determine ray length and sampling style textures to determine color [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   Scalar volume data (normalized floats).
*   Camera matrices: Model-View-Projection (MVP), View, Normal.
*   Style Transfer Function (STF) textures:
    *   Transfer Function Texture (1D).
    *   Index Function Texture (1D).
    *   Style Function Texture (2D Array).
*   Parameters: `stepSize`, `threshold`, `ScreenSize` (width, height).

**Outputs:**
*   Final color image with illustrative style applied.

### Data Structures and Layouts
The implementation relies on specific GPU resource layouts [S1, unofficial]:

1.  **Volume Texture:**
    *   Type: `GL_TEXTURE_3D`.
    *   Dimensions: $W \times H \times D$ (width, height, numCuts).
    *   Internal Format: `GL_INTENSITY`.
    *   Wrap Modes: `GL_CLAMP` for S, T, R.
    *   Filters: `GL_LINEAR` Min/Mag.
    *   Data: Normalized float array `dataScalars`.

2.  **Transfer Function Texture:**
    *   Type: `GL_TEXTURE_1D`.
    *   Size: 256.
    *   Internal Format: `GL_RGBA8`.
    *   Wrap Mode: `GL_REPEAT`.
    *   Filters: `GL_NEAREST` Min/Mag.
    *   Data: Array of `glm::vec4` (color/opacity).

3.  **Index Function Texture:**
    *   Type: `GL_TEXTURE_1D`.
    *   Size: Not explicitly given, but implied to be associated with the STF.
    *   Internal Format: Not given.
    *   Data: Not given.

4.  **Style Function Texture:**
    *   Type: `GL_TEXTURE_2D_ARRAY`.
    *   Dimensions: Not given.
    *   Internal Format: Not given.
    *   Data: Style images (e.g., lit spheres).

5.  **Back Face Texture:**
    *   Type: `GL_TEXTURE_2D`.
    *   Size: Window width $\times$ Window height.
    *   Internal Format: `GL_RGBA32F`.
    *   Wrap Modes: `GL_REPEAT`.
    *   Filters: `GL_NEAREST` Min/Mag.
    *   Purpose: Stores exit points from the first pass.

6.  **Frame Buffer Object (FBO):**
    *   Attachments: Depth renderbuffer (`GL_DEPTH_COMPONENT`) and Color attachment (`backFaceTexture`).
    *   State: Depth test enabled.

7.  **Geometry (Unit Cube):**
    *   VBO: 24 floats defining 8 vertices (range 0.0-1.0).
    *   EBO: 36 unsigned ints defining 12 triangles (2 per face, counter-clockwise).
    *   VAO: Binds attribute 0 (vertex location) and attribute 1 (vertex color) to the VBO.

### Parameters
*   `stepSize`: Ray casting step size. Default: $0.001$ [S1, unofficial].
*   `threshold`: Ray casting threshold. Default: $0.15$ [S1, unofficial].
*   `ScreenSize`: Window dimensions $(W, H)$ [S1, unofficial].

### Algorithm Steps

1.  **Initialization:**
    *   Set `isLoaded = false`.
    *   Allocate model name buffer (1024 chars).
    *   Set dimensions: `width`, `height`, `numCuts = 1`.
    *   Set `stepSize = 0.001f`, `threshold = 0.15f`.
    *   Initialize `transferFunc` array with 256 entries: `transferFunc[i] = vec4(i/255.0, i/255.0, i/255.0, 1.0)` [S1, unofficial].
    *   Load styles via `stf.loadStyles()` [S1, unofficial].

2.  **Volume Loading:**
    *   Create Vertex Buffer Object (VBO) and Index Buffer Object (IBO) for the unit cube.
    *   Load volume data from file (8-bit or 16-bit) into `dataScalars` (normalized floats).
    *   Create `backFaceTexture` (2D, RGBA32F).
    *   Create `transferFunctionTexture` (1D, RGBA8, size 256).
    *   Create `styleFunctionTexture` (2D Array).
    *   Create FBO with depth buffer and `backFaceTexture` attachment.
    *   Compute geometry scaling: `cubeSizes = vec3(width/max, height/max, numCuts/max)`.
    *   Position model at `(-cubeSizes.x/2, -cubeSizes.y/2, 0)`.
    *   Compute matrices:
        *   Projection: Perspective, 45 deg FOV, aspect ratio, near 0.1, far 500.
        *   View: `lookAt(eye=vec3(0,0,-1), center=vec3(0,0,0), up=vec3(0,1,0))`.
        *   Model, Normal, ViewProjection, ModelViewProjection.
    *   Set `isLoaded = true`.

3.  **Rendering (Two-Pass):**
    *   **Pass 1: Back Face Rendering**
        *   Bind FBO.
        *   Render the back face of the volume bounding box.
        *   Store exit points in `backFaceTexture`.
    *   **Pass 2: Volume Ray Casting**
        *   Bind default framebuffer.
        *   Set viewport to window size.
        *   Activate `rayCastShader`.
        *   Set uniforms: `MVP`, `ViewMatrix`, `NormalMatrix`, `StepSize`, `Threshold`, `ScreenSize`.
        *   Bind textures to units:
            *   Unit 1: `transferFunctionTexture`.
            *   Unit 2: `indexFunctionTexture`.
            *   Unit 3: `styleTransferTexture`.
        *   Render full-screen quad (or volume geometry) to trigger ray casting.
        *   The ray caster uses the exit points from `backFaceTexture` to determine ray length and samples the style textures to compute final color.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
class RawDataModel:
    def __init__(self):
        self.isLoaded = False
        self.sModelName = "" # 1024 chars
        self.width = 0
        self.height = 0
        self.numCuts = 1
        self.stepSize = 0.001
        self.threshold = 0.15
        self.textureIds = [0, 0, 0, 0] # Placeholder for IDs
        self.transferFunc = [vec4(i/255.0, i/255.0, i/255.0, 1.0) for i in range(256)]
        
    def setup(self):
        self.setupVolumeShaders()
        self.createTransferFunctionTexture()
        self.stf.loadStyles()

    def load_volume(self, file_path):
        # Load raw data into dataScalars (normalized floats)
        self.dataScalars = load_volume_file(file_path)
        
        # Create VBO/VAO for unit cube
        self.create_vertex_buffer()
        
        # Create Back Face Texture (2D, RGBA32F, window size)
        self.backFaceTexture = create_texture_2d(
            width=window_width, 
            height=window_height, 
            internal_format=GL_RGBA32F, 
            wrap=GL_REPEAT, 
            filter=GL_NEAREST
        )
        
        # Create FBO
        self.fbo = create_framebuffer(
            color_attachment=self.backFaceTexture,
            depth_attachment=create_renderbuffer(GL_DEPTH_COMPONENT)
        )
        
        # Compute Geometry and Matrices
        max_dim = max(self.width, self.height, self.numCuts)
        self.cubeSizes = vec3(self.width/max_dim, self.height/max_dim, self.numCuts/max_dim)
        self.position = vec3(-self.cubeSizes.x/2, -self.cubeSizes.y/2, 0)
        
        self.projection = perspective(45.0, self.width/self.height, 0.1, 500.0)
        self.view = lookAt(vec3(0, 0, -1), vec3(0, 0, 0), vec3(0, 1, 0))
        self.model = translate(mat4(1.0), self.position) * scale(mat4(1.0), self.cubeSizes)
        self.normalMatrix = inverse(transpose(self.model))
        self.viewProjection = self.projection * self.view
        self.modelViewProjection = self.viewProjection * self.model
        
        self.isLoaded = True

    def render(self):
        if self.isLoaded:
            self.render_back_face()
            self.render_volume_ray_casting()

    def render_back_face(self):
        bind_framebuffer(self.fbo)
        enable_depth_test()
        # Render back face geometry to capture exit points in backFaceTexture
        draw_back_face_geometry(self.model, self.viewProjection)
        bind_framebuffer(0) # Reset to default

    def render_volume_ray_casting(self):
        bind_framebuffer(0)
        set_viewport(0, 0, window_width, window_height)
        
        use_shader(self.rayCastShader)
        set_uniform(self.rayCastShader, "MVP", self.modelViewProjection)
        set_uniform(self.rayCastShader, "ViewMatrix", self.view)
        set_uniform(self.rayCastShader, "NormalMatrix", self.normalMatrix)
        set_uniform(self.rayCastShader, "StepSize", self.stepSize)
        set_uniform(self.rayCastShader, "Threshold", self.threshold)
        set_uniform(self.rayCastShader, "ScreenSize", vec2(window_width, window_height))
        
        # Bind Transfer Function
        active_texture(GL_TEXTURE1)
        bind_texture(GL_TEXTURE_1D, self.transferFunctionTexture)
        set_uniform(self.rayCastShader, "transferFunctionTexture", 1)
        
        # Bind Index Function
        active_texture(GL_TEXTURE2)
        bind_texture(GL_TEXTURE_1D, self.stf.indexFunctionTexture)
        set_uniform(self.rayCastShader, "indexFunctionTexture", 2)
        
        # Bind Style Function
        active_texture(GL_TEXTURE3)
        bind_texture(GL_TEXTURE_2D_ARRAY, self.stf.styleFunctionTexture)
        set_uniform(self.rayCastShader, "styleTransferTexture", 3)
        
        # Render volume (ray casting)
        draw_volume_geometry(self.model, self.viewProjection, self.backFaceTexture)
```

### Edge Cases and Pitfalls
*   **Precision:** The back face texture uses `GL_RGBA32F` to store exit points, ensuring high precision for ray length calculation [S1, unofficial].
*   **Boundaries:** The volume is rendered as a unit cube scaled to data dimensions. Ray casting must correctly map screen coordinates to ray origins and directions in world space [S1, unofficial].
*   **Threshold:** The `threshold` parameter (default 0.15) likely filters volume values during ray casting. Values below this threshold may be treated as transparent or ignored [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels and the number of samples per ray (determined by `stepSize` and volume depth). The two-pass approach adds a constant overhead for the back-face pass.
*   **Memory Complexity:** Proportional to the volume data size ($W \times H \times D$), the back-face texture size ($W_{screen} \times H_{screen}$), and the style textures.
*   **Strengths:** Allows interactive rotation and zoom with artistic styles. Supports loading/saving transfer functions with assigned styles [S1, unofficial].
*   **Limitations:** Performance depends on the number of ray casting steps. The two-pass approach requires sufficient GPU memory for the FBO and textures.
*   **When to use:** When illustrative, artistic visualization of volumes is desired, and interactive exploration is needed.
*   **When not to use:** When photorealistic lighting is required, or when memory bandwidth is a critical bottleneck.

## Background
The method is based on the "Style Transfer Function" (STF) concept, originally proposed by Bruckner et al. in 2007 [S1, unofficial]. The implementation described here is a homework project based on this academic paper, available on GitHub [S1, unofficial]. It extends standard direct volume ray casting by incorporating style transfer textures, allowing the volume's appearance to mimic various artistic styles (e.g., lit spheres) rather than relying solely on gradient-based lighting.

## Sources
[S1] jose-villegas/IllustrativeVolumeRendering, https://github.com/jose-villegas/IllustrativeVolumeRendering, fetched 2026-10-26 (unofficial)
