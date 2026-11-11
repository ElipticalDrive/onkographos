# GPU Data Layout


## Overview
GPU Data Layout defines the host-side memory structures and OpenGL resource bindings required to map logical volumetric data to GPU-accessible textures and framebuffers for raycasting [S1, unofficial]. It solves the problem of organizing scalar fields, transfer functions, and rendering targets into a unified state object that ensures compatibility with shader pipelines [S1, unofficial]. It differs from [ClearView Unified Buffer Construction](2475-clearview-unified-buffer-construction.md) by focusing on standard 3D/2D texture arrays for dense grids rather than hexahedral mesh topology mapping [S1, unofficial].

## How it works
The method organizes volumetric rendering state into a host-side structure containing OpenGL resource IDs and metadata. This structure is initialized with validated data types and bounds before being passed to the rendering kernel [S1, unofficial].

### Inputs and Outputs
*   **Input**: Volumetric scalar data (1-component or 4-component RGBA), transfer function data (color/opacity), and rendering context (renderer, camera).
*   **Output**: A populated `TextureObjects` array and associated framebuffer/renderbuffer IDs ready for shader binding.

### Data Structures
The core layout is defined by the following fields [S1, unofficial]:

1.  **TextureObjects**: An array of 5 `unsigned int` OpenGL texture IDs.
    *   Index 0: 3D scalar texture (volume data).
    *   Index 1: 1D color table texture.
    *   Index 2: 1D opacity table texture.
    *   Index 3: 2D grabbed depth buffer.
    *   Index 4: 2D color buffer.
2.  **MaxValueFrameBuffer** and **MaxValueFrameBuffer2**: Two `unsigned int` framebuffer IDs used for ping-pong accumulation in Maximum Intensity Projection (MIP) mode [S1, unofficial].
3.  **FrameBufferObject**: An `unsigned int` ID for the primary offscreen framebuffer object [S1, unofficial].
4.  **DepthRenderBufferObject**: An `unsigned int` ID for the depth renderbuffer attached to the framebuffer [S1, unofficial].
5.  **BoundingBox**: A `double` array of size $8 \times 3$ storing the world coordinates of the 8 corners of the dataset bounding box [S1, unofficial].
6.  **NoiseTexture**: A `float` array storing procedural noise data for speckle reduction [S1, unofficial].
7.  **NoiseTextureSize**: An `int` specifying the 1D dimension size of the noise texture [S1, unofficial].
8.  **NoiseTextureId**: An `unsigned int` OpenGL ID for the noise texture [S1, unofficial].
9.  **TextureCoord_1toN**: An array of `vtkTransform` pointers mapping texture coordinates from the primary volume to additional volumes [S1, unofficial].

### Validation Constraints
Before initialization, the data must satisfy specific constraints [S1, unofficial]:
1.  **Component Count**: Data must have 1 component or 4 components. If 4 components, `IndependentComponents` must be 0.
2.  **Data Type**: If 4 components, the data type must be `VTK_UNSIGNED_CHAR` (RGBA).
3.  **Blend Mode**: Additive blend mode is restricted to 1-component scalars only.

### Initialization Steps
1.  **Validate Data**: Check component count and data type against constraints [S1, unofficial].
2.  **Create Resources**: Generate OpenGL texture and framebuffer IDs for the 5 textures and 2 ping-pong framebuffers [S1, unofficial].
3.  **Load Data**: Transfer scalar data to the 3D texture and transfer function data to 1D textures [S2, unofficial].
4.  **Setup Framebuffer**: Attach color and depth renderbuffers to the primary `FrameBufferObject` [S1, unofficial].
5.  **Compute Bounds**: Calculate the 8 corner coordinates of the volume bounding box and store in `BoundingBox` [S1, unofficial].
6.  **Initialize Noise**: Generate procedural noise data, store in `NoiseTexture`, and create `NoiseTextureId` [S1, unofficial].
7.  **Set Transforms**: Initialize `TextureCoord_1toN` for multi-volume alignment if applicable [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_gpu_data_layout(volume_data, transfer_function, renderer):
    # 1. Validate inputs
    num_components = volume_data.get_num_components()
    data_type = volume_data.get_data_type()
    
    if num_components not in (1, 4):
        raise ValueError("Only 1 or 4 component scalars supported")
    
    if num_components == 4 and data_type != VTK_UNSIGNED_CHAR:
        raise ValueError("4-component data must be VTK_UNSIGNED_CHAR")
        
    if volume_data.get_blend_mode() == ADDITIVE_BLEND and num_components != 1:
        raise ValueError("Additive mode requires 1-component scalars")

    # 2. Allocate host-side structure
    layout = {
        "TextureObjects": [0] * 5,
        "MaxValueFrameBuffer": 0,
        "MaxValueFrameBuffer2": 0,
        "FrameBufferObject": 0,
        "DepthRenderBufferObject": 0,
        "BoundingBox": [[0.0] * 3 for _ in range(8)],
        "NoiseTexture": [],
        "NoiseTextureSize": 0,
        "NoiseTextureId": 0,
        "TextureCoord_1toN": []
    }

    # 3. Create OpenGL resources
    layout["TextureObjects"][0] = glCreateTexture(GL_TEXTURE_3D)
    layout["TextureObjects"][1] = glCreateTexture(GL_TEXTURE_1D)
    layout["TextureObjects"][2] = glCreateTexture(GL_TEXTURE_1D)
    layout["TextureObjects"][3] = glCreateTexture(GL_TEXTURE_2D)
    layout["TextureObjects"][4] = glCreateTexture(GL_TEXTURE_2D)
    
    layout["MaxValueFrameBuffer"] = glCreateFramebuffer()
    layout["MaxValueFrameBuffer2"] = glCreateFramebuffer()
    layout["FrameBufferObject"] = glCreateFramebuffer()
    layout["DepthRenderBufferObject"] = glCreateRenderbuffer()

    # 4. Load data into textures
    # Note: Specific loading logic delegated to GPU Texture Loading [S2, unofficial]
    load_3d_texture(layout["TextureObjects"][0], volume_data)
    load_1d_texture(layout["TextureObjects"][1], transfer_function.color)
    load_1d_texture(layout["TextureObjects"][2], transfer_function.opacity)

    # 5. Setup Framebuffer attachments
    bind_framebuffer(layout["FrameBufferObject"])
    attach_renderbuffer(layout["DepthRenderBufferObject"], GL_DEPTH_ATTACHMENT)
    attach_texture(layout["TextureObjects"][4], GL_COLOR_ATTACHMENT0)
    unbind_framebuffer()

    # 6. Compute Bounding Box
    bounds = volume_data.get_bounds() # Returns [min_x, max_x, min_y, max_y, min_z, max_z]
    # Calculate 8 corners from bounds
    layout["BoundingBox"] = compute_8_corners(bounds)

    # 7. Initialize Noise Texture
    noise_size = 64 # Typical size for speckle reduction [G]
    layout["NoiseTextureSize"] = noise_size
    layout["NoiseTexture"] = generate_procedural_noise(noise_size)
    layout["NoiseTextureId"] = glCreateTexture(GL_TEXTURE_1D)
    load_1d_texture(layout["NoiseTextureId"], layout["NoiseTexture"])

    # 8. Initialize Transforms
    layout["TextureCoord_1toN"] = [None] # Placeholder for additional volume transforms [S1, unofficial]

    return layout
```

## Subtopics

### [GPU Texture Loading](2473-gpu-texture-loading.md)
Transfers volumetric scalar data from host memory to GPU 3D textures. It differs by handling the actual data transfer and memory allocation for the 3D texture, whereas GPU Data Layout manages the overall resource state and validation.

### [Tiled 2D Texture Storage for 3D Fields](2740-tiled-2d-texture-storage-for-3d-fields.md)
Stores 3D fields in 2D textures using a tiling scheme. It differs by using 2D texture objects and coordinate mapping instead of native 3D textures.

## Operating characteristics
*   **Time Complexity**: Initialization is $O(1)$ relative to volume size for structure setup, but $O(N)$ for data transfer to GPU textures, where $N$ is the number of voxels [S2, unofficial].
*   **Memory Complexity**: Requires host memory for the layout structure and GPU memory for 3D/2D textures and framebuffers. The 3D texture size is proportional to the volume resolution [S2, unofficial].
*   **Strengths**: Standard OpenGL compatibility; supports multiple volume alignment via transforms; efficient ping-pong accumulation for MIP [S1, unofficial].
*   **Limitations**: Restricted to 1 or 4 component scalars; additive blending limited to 1 component; requires explicit validation of data types [S1, unofficial].
*   **When to use**: Standard dense volume raycasting with OpenGL; multi-volume alignment.
*   **When not to use**: When using CUDA unified memory or non-standard mesh topologies (use [ClearView](2475-clearview-unified-buffer-construction.md)) [S1, unofficial].

## Background
The layout originates from the VTK (Visualization Toolkit) GPU volume raycasting mappers, specifically `vtkGPUMultiVolumeRayCastMapper` [S1, unofficial]. It was developed to support multi-volume rendering and canonical view generation within the VTK framework [S1, unofficial].

## Sources
[S1] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S2] GPU Texture Loading, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
