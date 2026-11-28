# 3D Texture Upload

Also called: 3D Texture Upload for Volume Data, 3D Texture Upload for Volumetric Display, 3D Volume Texture Upload, GPU Volume Texture Upload, Volume Texture Upload

## Overview
3D Texture Upload transfers volumetric voxel data from host memory into GPU texture storage (`GL_TEXTURE_3D`) to enable hardware-accelerated ray marching or volume rendering [S1, S2, S3, S4, S5, S6, S7, unofficial]. It maps host-side volume buffers into the contiguous memory layout required by OpenGL texture upload functions [S1, S2, unofficial]. It differs from its parent [Volume Texture Setup](4055-volume-texture-setup.md) by handling the actual data transformation and final buffer upload step, and from [3D Texture Initialization](13158-3d-texture-initialization.md) by focusing on data layout and content population rather than texture object creation and lifecycle management [S1, S2, S3, S4, S5, S6, S7, unofficial].

## How it works
The method involves preparing the host-side data buffer, configuring OpenGL pixel storage parameters, and invoking the texture upload command with appropriate format specifications [S4, S8, unofficial].

### Inputs and Outputs
*   **Input:** A 3D array of voxel data (dimensions $Z \times Y \times X$), typically stored as a nested array or flat buffer in host memory [S1, unofficial].
*   **Output:** A populated `GL_TEXTURE_3D` object in GPU memory containing the voxel data [S1, S2, S3, unofficial].

### Data Structures and Layout
The host-side data must be contiguous in memory [S2, unofficial]. The standard layout for OpenGL `glTexImage3D` places the fastest-varying index first (X), followed by Y, then Z [S2, S5, unofficial].
*   **Host Buffer:** A flat array of type `float` (32-bit) or `unsigned char` (8-bit), depending on the target format [S2, S3, S7, unofficial].
*   **GPU Texture:** `GL_TEXTURE_3D` with an internal format matching the data precision (e.g., `GL_R32F`, `GL_R8`, `GL_RGBA`) [S7, unofficial].

### Parameters
*   **Dimensions:** $Width$ (X), $Height$ (Y), $Depth$ (Z) [S2, S3, S7, unofficial].
*   **Internal Format:** The format stored in GPU memory (e.g., `GL_R32F`, `GL_R8`, `GL_RGBA`) [S2, S3, S7, unofficial].
*   **Format:** The layout of the input data (e.g., `GL_RED`, `GL_RGBA`) [S2, S3, S7, unofficial].
*   **Type:** The data type of the input buffer (e.g., `GL_FLOAT`, `GL_UNSIGNED_BYTE`) [S2, S3, S7, unofficial].
*   **Pixel Store Alignment:** Must be set to 1 to match host array byte alignment [S2, S8, unofficial].

### Steps
1.  **Ensure Data Contiguity:** Convert the input volume data to a contiguous array of the target type (e.g., `float32`) [S2, unofficial]. If the source is a nested array, flatten it such that the index order is $Z, Y, X$ [S2, S5, unofficial].
2.  **Set Pixel Store Alignment:** Call `glPixelStorei(GL_UNPACK_ALIGNMENT, 1)` to prevent padding issues during upload [S2, S8, unofficial].
3.  **Generate and Bind Texture:** Generate a texture ID and bind it to `GL_TEXTURE_3D` [S1, S3, S5, unofficial].
4.  **Configure Parameters:** Set wrapping modes (e.g., `GL_CLAMP_TO_EDGE` or `GL_CLAMP`) and filtering modes (`GL_NEAREST` for discrete voxels, `GL_LINEAR` for smooth interpolation) [S1, S2, S3, S7, S8, unofficial].
5.  **Upload Data:** Call `glTexImage3D` with the prepared buffer [S2, S3, S7, unofficial]. Alternatively, use `glTexSubImage3D` for updates [S6, unofficial].
6.  **Generate Mipmaps (Optional):** If mipmapping is enabled, call `glGenerateMipmap` [S4, S5, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def upload_3d_texture(gl, volume_data, width, height, depth, internal_format, format, gl_type):
    """
    Uploads a 3D volume to a GL_TEXTURE_3D.
    
    Args:
        gl: OpenGL context object.
        volume_data: Host-side 3D array (Z, Y, X) or flat buffer.
        width: X dimension.
        height: Y dimension.
        depth: Z dimension.
        internal_format: GPU internal format (e.g., GL_R32F, GL_R8).
        format: Input data format (e.g., GL_RED, GL_RGBA).
        gl_type: Input data type (e.g., GL_FLOAT, GL_UNSIGNED_BYTE).
    """
    # Step 1: Ensure data is contiguous and of the correct type
    # Note: volume_data must be flattened to 1D in Z-Y-X order
    # If using numpy: data = np.ascontiguousarray(volume_data, dtype=np.float32)
    data_ptr = volume_data # Pointer to contiguous buffer
    
    # Step 2: Set pixel store alignment to 1
    gl.glPixelStorei(gl.GL_UNPACK_ALIGNMENT, 1)
    
    # Step 3: Generate and bind texture
    tex_id = gl.glGenTextures(1)
    gl.glBindTexture(gl.GL_TEXTURE_3D, tex_id)
    
    # Step 4: Set parameters
    gl.glTexParameteri(gl.GL_TEXTURE_3D, gl.GL_TEXTURE_WRAP_S, gl.GL_CLAMP_TO_EDGE)
    gl.glTexParameteri(gl.GL_TEXTURE_3D, gl.GL_TEXTURE_WRAP_T, gl.GL_CLAMP_TO_EDGE)
    gl.glTexParameteri(gl.GL_TEXTURE_3D, gl.GL_TEXTURE_WRAP_R, gl.GL_CLAMP_TO_EDGE)
    
    # Filtering: NEAREST for discrete voxels, LINEAR for smooth
    gl.glTexParameteri(gl.GL_TEXTURE_3D, gl.GL_TEXTURE_MIN_FILTER, gl.GL_NEAREST)
    gl.glTexParameteri(gl.GL_TEXTURE_3D, gl.GL_TEXTURE_MAG_FILTER, gl.GL_NEAREST)
    
    # Step 5: Upload data
    # glTexImage3D(target, level, internalformat, width, height, depth, border, format, type, pixels)
    gl.glTexImage3D(
        gl.GL_TEXTURE_3D,
        0,                  # level
        internal_format,    # internalformat
        width,              # width
        height,             # height
        depth,              # depth
        0,                  # border
        format,             # format
        gl_type,            # type
        data_ptr            # pixels
    )
    
    # Step 6: Generate mipmaps if needed (optional)
    # gl.glGenerateMipmap(gl.GL_TEXTURE_3D)
    
    return tex_id
```

### Edge Cases and Pitfalls
*   **Alignment:** Failure to set `GL_UNPACK_ALIGNMENT` to 1 can cause misalignment errors, especially for non-byte-aligned types like `float` [S2, S8, unofficial].
*   **Data Layout:** The host memory layout must match the expected OpenGL layout (X fastest, then Y, then Z) [S2, S5, unofficial].
*   **Format Mismatch:** The `format` and `type` arguments must match the host data. For example, uploading `float` data requires `GL_FLOAT` type and `GL_RED` format for single-channel volumes [S2, S3, unofficial].
*   **RGB Volumes:** On some platforms (e.g., Windows D3D11/ANGLE), using a second sampler3D pointing to an uninitialized texture can cause draw-call failures. RGB volumes are stored as 3 separate R/G/B greyscale channels (single-channel render path) to avoid this [S2, unofficial].
*   **Normalization:** If uploading integer data (e.g., `GL_R8`) but the shader expects normalized floats, ensure the shader performs the normalization (e.g., `value / 255.0`) or use normalized integer types if supported [S1, S3, unofficial].

## Subtopics

### 3D Texture Upload for Volume Data, GL_TEXTURE_3D volume upload, Volume Texture Binding
This variant converts a 3D image stack from a nested array structure into a flat 1D buffer suitable for GPU upload via OpenGL. It solves the problem of mapping discrete volumetric data, which is often stored in host-memory structures with varying strides, into a contiguous memory layout required by `glTexImage3D` or `glTexSubImage3D` [S1, unofficial]. Unlike sparse storage methods that suffer from pointer overhead or out-of-core streaming that incurs decoding latency, this method packs data into dense 3D texture slices, trading memory efficiency for hardware compatibility and sampling speed [S1, S2, unofficial]. It differs from its parent [3D Texture Upload](4419-3d-texture-upload.md) by handling the data transformation (format conversion, normalization) prior to the final buffer upload step, and from [3D Texture Initialization](13158-3d-texture-initialization.md) by focusing on the data layout rather than texture object creation [S1, S2, S3, S4, S5, S6, S7, unofficial].

### [Volume Atlas Texture Update](7157-volume-atlas-texture-update.md)
A method to populate a 3D volume atlas texture with sampled data values, tracking the minimum and maximum values encountered during the update process. It maps a sequence of timesteps into a multi-volume 3D texture array, allowing temporal data to be stored in a single GPU texture object for efficient rendering.

### [3D Texture Initialization](13158-3d-texture-initialization.md)
Allocates GPU memory for a 3D texture, configures sampling parameters (wrapping, filtering, mipmapping), and uploads initial volume data. It sets up the `GL_TEXTURE_3D` object required for subsequent volume rendering passes. It differs from [3D Texture Upload](4419-3d-texture-upload.md) by handling the full lifecycle of texture creation, storage allocation, and mipmap generation, whereas the parent focuses on binding and parameter setting for existing data.

### [Volume Texture Ping-Pong Swap](13951-volume-texture-ping-pong-swap.md)
Manages dynamic 3D scalar fields in GPU memory using a double-buffered ping-pong swap to avoid full re-uploads on field updates. It differs from [3D Texture Initialization](13158-3d-texture-initialization.md) by handling subsequent updates efficiently, and from [Volume Atlas Texture Update](7157-volume-atlas-texture-update.md) by operating on single 3D textures rather than atlas layouts.

### Brick Grid Texture Upload
This variant uploads multiple associated textures for a brick-based volume rendering system. It creates an indirection texture (`GL_RGB10_A2UI`), a range texture (`GL_RG16F`), and an atlas texture (`GL_COMPRESSED_RED`), each with specific dimensions and data sources [S9, unofficial]. It also generates mipmaps for the range texture. Wrap modes are set to `GL_CLAMP_TO_BORDER` and filters to `GL_NEAREST` [S9, unofficial].

### 3D Volume Texture Population
This variant populates a 3D volume texture with procedural data, specifically for cloud rendering. The R channel stores noise, the G channel stores local AO, the B channel stores noise-driven height-based baseline albedo, and the A channel stores procedural cloud physical density [S10, unofficial]. This allows for on-the-fly procedural 3D noise and shape modulation during rendering [S10, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels, as each voxel must be copied from host to device memory [G].
*   **Memory Complexity:** $O(N)$ for the GPU texture storage [G].
*   **Performance:** Upload speed is limited by PCIe bandwidth. For large volumes, this can be a bottleneck [S6, unofficial]. Using `glTexSubImage3D` for partial updates can reduce transfer size if only a subset of the volume changes [S6, unofficial].
*   **Strengths:** Simple to implement, hardware-accelerated sampling, compatible with standard OpenGL pipelines [G].
*   **Limitations:** Requires contiguous memory. Standard formats do not include compression, except for variants like Brick Grid [S9, unofficial].
*   **When to Use:** For static or infrequently updated volumes, or when hardware ray marching is required [G].
*   **When Not to Use:** For very large volumes that exceed texture size limits, or for highly dynamic volumes where update frequency is high and PCIe bandwidth is a bottleneck (consider compute buffer approaches or sparse voxel octrees) [S1, S5, unofficial].

## Background
The method originates from standard OpenGL volume rendering techniques, where 3D textures are used to store volumetric data for ray marching [S2, S3, S4, S6, S7, unofficial]. The use of `GL_TEXTURE_3D` for volume data is a well-established practice in computer graphics, allowing for efficient interpolation and filtering using hardware texture units [S2, S3, S7, unofficial].

## Sources
[S1] Multivox: Volumetric Display, https://github.com/AncientJames/multivox, fetched 2026-10-26 (unofficial)
[S2] xwghua/clearvolume-plus, https://github.com/xwghua/clearvolume-plus, fetched 2026-10-26 (unofficial)
[S3] jaanonim/volume-rendering, https://github.com/jaanonim/volume-rendering, fetched 2026-10-26 (unofficial)
[S4] jonathonb21/volume4d-viewer, https://github.com/jonathonb21/volume4d-viewer, fetched 2026-10-26 (unofficial)
[S5] Dy111111/volumeRenderingGPU, https://github.com/Dy111111/volumeRenderingGPU, fetched 2026-10-26 (unofficial)
[S6] arigo/CascatedVoxelConeTracing, https://github.com/arigo/CascatedVoxelConeTracing, fetched 2026-10-26 (unofficial)
[S7] termijn/webgl-volumerendering, https://github.com/termijn/webgl-volumerendering, fetched 2026-10-27 (unofficial)
[S8] GitHub - m-pilia/volume-raycasting: Simple visualiser based on GPU ..., https://github.com/m-pilia/volume-raycasting, fetched 2026-10-27 (unofficial)
[S9] nihofm/volren: Simple OpenGL-based renderer for ..., https://github.com/nihofm/volren, fetched 2026-10-29 (unofficial)
[S10] Optimize cloud rendering with eroded Voronoi weather maps and stable linear raymarching, https://github.com/ricecake/boidsish/pull/679, fetched 2026-10-28 (unofficial)
[S11] marwan-abdellah/cuFVR, https://github.com/marwan-abdellah/cuFVR, fetched 2026-10-26 (unofficial)
