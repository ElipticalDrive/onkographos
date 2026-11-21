# Volume Texture Setup


## Overview
Volume Texture Setup is the method for configuring GPU-accessible 3D texture resources with specific memory layouts, sampling parameters, and lifecycle states to prepare volumetric data for rendering. It solves the problem of translating discrete, host-side volumetric representations into optimized GPU textures that support efficient ray marching or voxel cone tracing. It differs from [3D Texture Upload](4419-3d-texture-upload.md) by focusing on the configuration of texture objects (filters, wrap modes, mipmaps) rather than the raw data transfer, and from [Sparse Volume Texture Setup](7432-sparse-volume-texture-setup.md) by targeting dense grid textures rather than sparse tile-based structures.

## How it works
The method involves creating texture objects, defining their memory formats, configuring sampling behavior, and initializing them with data or parameters. The specific steps vary by API and data type, but generally follow this sequence:

1.  **Texture Object Creation**: Generate a texture ID or handle using the graphics API (e.g., `glGenTextures`, `device.createTexture`).
2.  **Binding**: Bind the texture to a target (e.g., `GL_TEXTURE_3D`) and/or a specific texture unit (e.g., `GL_TEXTURE0`).
3.  **Format and Dimension Allocation**: Define the internal format (e.g., `GL_RGBA32F`, `PF_A32B32G32R32F`, `depth32float`) and dimensions (width, height, depth).
4.  **Parameter Configuration**: Set sampling filters (minification, magnification), wrap modes, and mipmap generation settings.
5.  **Data Population**: Upload host data to the GPU texture memory using appropriate functions (e.g., `glTexImage3D`, `queue.copyExternalImageToTexture`).
6.  **Lifecycle Management**: Disable features not needed (e.g., SRGB, compression, streaming) and update the resource state.

### Inputs and Outputs
*   **Inputs**: Volumetric data (host memory buffer), metadata (dimensions, min/max values), rendering requirements (filtering mode, format).
*   **Outputs**: Configured GPU texture object(s) ready for sampling in shaders.

### Data Structures and Layouts
The layout depends on the specific variant and API. Common layouts include:

*   **Dense 3D Texture**:
    *   Format: `GL_RGBA32F` (32-bit floating point per channel) or `GL_RED` (for single-channel data).
    *   Dimensions: $W \times H \times D$ voxels.
    *   Memory: Contiguous block allocated by the GPU driver.
*   **VRDF Label-Weighted**:
    *   Label Texture: `Texture3D`, point filtered.
    *   Weight Texture: `Texture3D`, bilinear filtered.
    *   LUT: `Texture2D` for transfer function.
    *   Control Texture: `256x1 RGBAFloat` for dynamic UI control.

### Numbered Steps (Generalized Implementation)
1.  **Generate Texture ID**:
    ```python
    texture_id = gl_create_textures(1)
    ```
2.  **Bind Texture**:
    ```python
    gl_bind_texture(GL_TEXTURE_3D, texture_id)
    ```
3.  **Set Parameters**:
    *   **Wrap Mode**: Set S, T, R coordinates to `GL_REPEAT` or `GL_CLAMP_TO_EDGE` depending on boundary handling requirements [S2, S3, unofficial].
    *   **Filtering**: Set minification and magnification filters to `GL_LINEAR` for smooth interpolation or `GL_NEAREST` for discrete labels [S2, S3, S4, unofficial].
    *   **Mipmaps**: Disable mipmap generation if not required (`TMGS_NoMipmaps`) [S1, unofficial].
    *   **SRGB**: Disable sRGB conversion for linear data (`SRGB = false`) [S1, unofficial].
    *   **Compression**: Disable texture compression (`CompressionNone = true`) to preserve data integrity [S1, unofficial].
    *   **Streaming**: Prevent texture streaming (`NeverStream = true`) to ensure data residency [S1, unofficial].
4.  **Allocate and Upload Data**:
    *   For OpenGL:
        ```python
        gl_tex_image_3d(
            target=GL_TEXTURE_3D,
            level=0,
            internal_format=GL_RED, # or GL_RGBA32F
            width=width,
            height=height,
            depth=depth,
            border=0,
            format=GL_RED, # or GL_RGBA
            type=GL_FLOAT,
            data=host_data_buffer
        )
        ```
    *   For WebGPU:
        ```python
        texture = device.create_texture(
            size=[width, height, depth],
            format="rgba32float",
            usage=GPUTextureUsage.TEXTURE_BINDING | GPUTextureUsage.COPY_DST
        )
        device.queue.write_texture(
            texture=texture,
            data=host_data_buffer,
            texture_layout=TextureDataLayout(offset=0, bytes_per_row=width*4, rows_per_image=height)
        )
        ```
5.  **Finalize**:
    *   Update resource state on the rendering thread (`UpdateResource()`) [S1, unofficial].
    *   Bind to texture unit for shader access (`gl_active_texture(GL_TEXTURE0)`, `gl_bind_texture(...)`) [S2, unofficial].

### Formulas
*   **Data Normalization** (if required before upload):
    $$
    v_{norm} = \frac{v - v_{min}}{v_{max} - v_{min}}
    $$
    where $v$ is the voxel value, $v_{min}$ and $v_{max}$ are the global minimum and maximum values [S3, unofficial].

### Parameters and Typical Values
*   **Format**: `GL_RGBA32F` for high-precision scientific data; `GL_RED` for single-channel density; `depth32float` for shadow maps [S2, S3, S4, unofficial].
*   **Filter**: `GL_LINEAR` for smooth noise or density; `GL_NEAREST` for discrete labels [S2, S4, unofficial].
*   **Wrap**: `GL_REPEAT` for periodic noise; `GL_CLAMP_TO_EDGE` for bounded volumes [S2, S3, unofficial].
*   **Dimensions**: Often cubic ($N \times N \times N$) for noise textures; arbitrary for medical/scientific data [S2, S3, unofficial].

### Edge Cases and Pitfalls
*   **Precision**: Using integer formats for continuous data causes quantization artifacts. Use `GL_FLOAT` or `GL_RGBA32F` [S2, unofficial].
*   **Boundaries**: Incorrect wrap modes can cause artifacts at volume edges. Use `GL_CLAMP_TO_EDGE` for bounded volumes [S3, unofficial].
*   **Memory**: Large volumes may exceed GPU memory limits. Consider sparse textures or chunking for very large datasets [S1, S3, unofficial].
*   **Data Layout**: Ensure host data is contiguous and correctly aligned for GPU upload. Non-contiguous layouts require padding or copying [S1, S2, unofficial].
*   **SRGB**: Enabling sRGB conversion for linear data (e.g., density) causes incorrect lighting calculations. Disable sRGB for linear data [S1, unofficial].

## Subtopics

### [3D Texture Upload](4419-3d-texture-upload.md)
This method uploads a 3D voxel volume buffer from host memory to GPU memory as a `GL_TEXTURE_3D` object, preparing it for ray marching or volume rendering. It solves the problem of mapping discrete volumetric data, which often resides in host-memory structures with varying strides or non-contiguous layouts, into the contiguous memory layout required by OpenGL texture upload functions like `glTexImage3D` or `glTexSubImage3D`. Unlike sparse storage methods that suffer from pointer overhead or out-of-core streaming that incurs decoding latency, this method packs data into dense 3D texture slices, trading memory efficiency for hardware compatibility and sampling speed. It differs from its parent [Volume Texture Setup](4055-volume-texture-setup.md) by handling the actual data transformation and buffer upload, and from [3D Texture Initialization](13158-3d-texture-initialization.md) by focusing on the data layout and content population rather than the lifecycle of texture object creation and parameter configuration.

### [VRDF Texture Construction for Label-Weighted Volumes](6959-vrdf-texture-construction-for-label-weighted-volumes.md)
This method constructs GPU-accessible 3D textures for Unity volume rendering when the input data is in 'anatomy_label_weighted' mode with 2 channels. It separates discrete labels and continuous weights into distinct 3D textures with specific filtering modes to support weighted raymarching. It serves as the runtime counterpart to the VRDF file export logic, ensuring that the label and weight data are sampled correctly for opacity modulation.

### [Sparse Volume Texture Setup](7432-sparse-volume-texture-setup.md)
Sparse Volume Texture Setup is a method for importing and configuring large, sparse volumetric datasets (up to ~6 gigavoxels) into Unreal Engine 5 using Sparse Volume Textures (SVT). It solves the problem of rendering massive volumetric data without chunking artifacts by storing data in sparse tiles within a single large volume texture, thereby circumventing the DirectX 12 API limit of 2048 voxels per dimension. Unlike debug viewers or fog-specific renderers, it supports full lighting, emission, and color, making it suitable for scientific data visualization. It builds on [Volume Texture Setup](4055-volume-texture-setup.md) by defining the specific material and actor workflow required to visualize the data.

### [Voxel Volume Initialization & Binding](12429-voxel-volume-initialization-binding.md)
Voxel Volume Initialization & Binding prepares a dense 3D texture for voxel cone tracing by allocating GPU memory, configuring sampling parameters, and establishing read/write bindings for compute shaders. It solves the problem of transitioning from raw scene data to a structured, mipmapped voxel grid ready for lighting injection. Unlike [3D Texture Upload](4419-3d-texture-upload.md) which focuses on data transfer, this method handles the OpenGL resource lifecycle and binding state. It differs from [Sparse Volume Texture Setup](7432-sparse-volume-texture-setup.md) by using a dense `GL_TEXTURE_3D` with fixed dimensions rather than sparse binding extensions.

### [Volume Texture Mapping with ROI](12595-volume-texture-mapping-with-roi.md)
Volume Texture Mapping with ROI is a technique for rendering volumetric data by mapping a Data Region of Interest (ROI) within the source memory buffer to a Geometry ROI in world space, enabling the rendering of sub-volumes without copying data. It differs from full-volume rendering methods by allowing efficient access to specific data subsets via coordinate mapping and clipping, and from sparse methods by operating on dense grid textures managed by the OpenGL pipeline.

### Texture Setup
This variant focuses on creating 2D textures for specific purposes like shadow maps or noise lookup tables. It involves loading texture data from files (e.g., PNG) and configuring samplers with specific address modes and filters [S4, unofficial].

### VRDF Runtime Volume Rendering Setup
This variant handles the runtime behavior of the VolumeDVR component, parsing VRDF files, building GPU textures, and setting up shader uniforms for real-time 3D raymarched volume rendering with weight-driven transparency [S5, unofficial]. It involves reading the VRDF magic header, building label and weight textures, and injecting shader uniforms for dynamic control [S5, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of voxels, for data upload. Configuration is $O(1)$.
*   **Memory Complexity**: $O(N)$ for dense textures. Sparse textures reduce this to $O(K)$ where $K$ is the number of occupied voxels/tiles.
*   **Performance**: Dense textures offer fast, hardware-accelerated sampling. Sparse textures reduce memory bandwidth but may incur overhead in tile management.
*   **Strengths**: Hardware compatibility, fast sampling, support for mipmaps (if enabled).
*   **Limitations**: High memory usage for large dense volumes, fixed resolution limits.
*   **When to Use**: Small to medium volumes, real-time rendering, when hardware sampling is preferred.
*   **When Not to Use**: Very large sparse volumes (use sparse textures), when memory is extremely constrained.

## Background
Volume texture setup is a fundamental technique in GPU-based volume rendering, evolving with the capabilities of graphics APIs like OpenGL, DirectX, and WebGPU. Early implementations relied on OpenGL's `glTexImage3D`, while modern approaches utilize more flexible APIs like WebGPU's `queue.write_texture` or Vulkan's buffer-to-texture copies. The method has been adapted for various domains, including medical imaging, scientific visualization, and real-time game engines, with specific variants emerging to handle sparse data, label-weighted volumes, and regions of interest.

## Sources
[S1] JI20/unreal-splat, https://github.com/JI20/unreal-splat, fetched 2026-10-27 (unofficial)
[S2] Depersonalizc/volumetric-clouds, https://github.com/Depersonalizc/volumetric-clouds, fetched 2026-10-27 (unofficial)
[S3] GitHub - priscilla100/medical_volume_renderer, https://github.com/priscilla100/medical_volume_renderer/, fetched 2026-10-29 (unofficial)
[S4] Ne0nWinds/WebGPU-Volumetrics-Demo, https://github.com/Ne0nWinds/WebGPU-Volumetrics-Demo, fetched 2026-10-27 (unofficial)
[S5] guillaume-schneider/vrdf-sdk, https://github.com/guillaume-schneider/vrdf-sdk, fetched 2026-10-26 (unofficial)
