# 3D Texture Upload

Also called: 3D Texture Upload for Volume Data, 3D Texture Upload for Volumetric Display, 3D Volume Texture Upload, Volume Texture Upload

## Overview
This method uploads dense 3D voxel data from host memory to GPU memory as a `GL_TEXTURE_3D` object, preparing it for ray marching or volume rendering [S1, S2, S3, S4, unofficial]. It solves the problem of mapping discrete volumetric data, which often resides in host-memory structures with varying strides or non-contiguous layouts, into the contiguous memory layout required by OpenGL texture upload functions like `glTexImage3D` or `glTexSubImage3D` [S1, S2, unofficial]. Unlike sparse storage methods that suffer from pointer overhead or out-of-core streaming that incurs decoding latency, this method packs data into dense 3D texture slices, trading memory efficiency for hardware compatibility and sampling speed [S1, unofficial]. It differs from its parent [Volume Texture Setup](4055-volume-texture-setup.md) by handling the actual data transformation and buffer transfer, and from [3D Texture Initialization](13158-3d-texture-initialization.md) by focusing on the data layout rather than the full lifecycle of texture object creation and mipmap generation [S1, S5, S7, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel volume buffer (host memory), typically a nested array or flat buffer. Dimensions are $Z \times Y \times X$ (slices, rows, columns).
*   **Output:** A bound `GL_TEXTURE_3D` object on the GPU containing the volume data.

**Data Structures and Layout:**
The host data must be contiguous in memory. If the source is a nested array (e.g., `data[z][y][x]`), it must be flattened or copied into a contiguous buffer. The standard layout for OpenGL 3D textures is Z-slices, where each slice is a 2D image with rows (Y) and columns (X) [S2, S7, unofficial].
*   **Pixel Store Alignment:** Must be set to 1 byte (`GL_UNPACK_ALIGNMENT = 1`) to ensure that each row of pixels starts at a byte boundary, regardless of the data type width [S2, S7, unofficial].
*   **Formats:** Common internal formats include `GL_R8` (8-bit unsigned integer), `GL_R16F` (16-bit float), `GL_R32F` (32-bit float), or `GL_RGBA` for classified color data [S3, S4, S6, S7, unofficial].

**Numbered Steps:**

1.  **Prepare Host Data:** Ensure the volume data is contiguous and in the correct data type (e.g., `float32` or `uint8`). If the data is not contiguous, copy it to a new contiguous buffer [S2, unofficial].
    *   *Pitfall:* On Windows D3D11/ANGLE, using a second sampler3D pointing to an uninitialized texture can cause draw-call failures. Ensure all channels are populated or use single-channel render paths for RGB volumes [S2, unofficial].

2.  **Generate and Bind Texture:** Generate a texture ID and bind it to `GL_TEXTURE_3D` [S1, S3, S4, unofficial].

3.  **Set Pixel Store Parameters:** Set the unpack alignment to 1 to match the host array byte alignment [S2, S7, unofficial].
    ```python
    gl.glPixelStorei(gl.GL_UNPACK_ALIGNMENT, 1)
    ```

4.  **Configure Texture Parameters:** Set wrapping modes and filtering.
    *   **Wrapping:** Set `GL_TEXTURE_WRAP_S`, `GL_TEXTURE_WRAP_T`, and `GL_TEXTURE_WRAP_R` to `GL_CLAMP_TO_EDGE` or `GL_CLAMP` to prevent sampling artifacts at volume boundaries [S4, S6, S7, unofficial].
    *   **Filtering:** Set `GL_TEXTURE_MIN_FILTER` and `GL_TEXTURE_MAG_FILTER`. `GL_NEAREST` is common for voxel-based rendering to preserve sharp edges, while `GL_LINEAR` provides smooth interpolation [S1, S2, S6, S7, unofficial].
    *   **Mipmapping:** If mipmaps are required, set the base and max levels and generate them after upload [S4, unofficial].

5.  **Upload Data:** Call `glTexImage3D` (for initial upload) or `glTexSubImage3D` (for updates).
    *   **Syntax:** `glTexImage3D(target, level, internalFormat, width, height, depth, border, format, type, data)`
    *   **Parameters:**
        *   `target`: `GL_TEXTURE_3D`
        *   `level`: 0 (base level)
        *   `internalFormat`: e.g., `GL_R8`, `GL_R32F`, `GL_RGBA`
        *   `width`: $X$ dimension
        *   `height`: $Y$ dimension
        *   `depth`: $Z$ dimension (number of slices)
        *   `border`: 0
        *   `format`: e.g., `GL_RED`, `GL_RGBA`
        *   `type`: e.g., `GL_UNSIGNED_BYTE`, `GL_FLOAT`, `GL_HALF_FLOAT`
        *   `data`: Pointer to the contiguous host memory buffer [S3, S6, S7, unofficial].

    *   *Example (8-bit unsigned):*
        ```python
        gl.glTexImage3D(
            gl.GL_TEXTURE_3D, 0, gl.GL_R8,
            width, height, depth, 0,
            gl.GL_RED, gl.GL_UNSIGNED_BYTE,
            data_buffer
        )
        ```

    *   *Example (32-bit float):*
        ```python
        gl.glTexImage3D(
            gl.GL_TEXTURE_3D, 0, gl.GL_R32F,
            width, height, depth, 0,
            gl.GL_RED, gl.GL_FLOAT,
            data_buffer
        )
        ```

6.  **Bind to Shader:** Bind the texture to a specific texture unit (e.g., unit 0) and pass the uniform location to the shader [S6, unofficial].

**Formulas:**
*   **Linear Index Calculation (for Compute Shader Repacking):** If repacking from a flat buffer in a compute shader, the linear index `src_index` for a 3D coordinate $(x, y, z)$ in a grid of resolution $R$ is:
    $$ \text{src\_index} = (z \cdot R + y) \cdot R + x $$
    where $R$ is the grid resolution (assuming a cubic grid) [S5, unofficial].

**Parameters with Typical Values:**
*   `GL_UNPACK_ALIGNMENT`: 1 [S2, S7, unofficial]
*   `GL_TEXTURE_WRAP_S/T/R`: `GL_CLAMP_TO_EDGE` [S2, S3, S7, unofficial]
*   `GL_TEXTURE_MIN/MAG_FILTER`: `GL_NEAREST` or `GL_LINEAR` [S1, S2, S6, S7, unofficial]
*   `HIGH_COLOUR` (Compile-time flag): If defined, use `GL_RGB565` (`GL_UNSIGNED_SHORT_5_6_5`) with internal format `GL_RGB`; otherwise, use `GL_R8UI` (`GL_UNSIGNED_BYTE`) with internal format `GL_RED_INTEGER` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-Contiguous Data:** Passing a non-contiguous array to `glTexImage3D` can lead to memory corruption or incorrect rendering. Always ensure data is contiguous [S2, unofficial].
*   **Alignment:** Failure to set `GL_UNPACK_ALIGNMENT` to 1 can cause misalignment errors, especially with data types larger than 1 byte [S7, unofficial].
*   **Format Mismatch:** Ensure the `format` and `type` arguments match the host data layout. For example, `GL_RED` format with `GL_FLOAT` type expects 32-bit floats [S3, S6, unofficial].
*   **Windows D3D11/ANGLE:** RGB volumes stored as 3 separate R/G/B greyscale channels may cause issues if a second sampler3D points to an uninitialized texture. Use single-channel render paths or ensure all textures are initialized [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def upload_3d_texture(gl, data, width, height, depth, internal_format, format, data_type):
    """
    Uploads a 3D volume buffer to a GL_TEXTURE_3D.
    
    Args:
        gl: OpenGL context object.
        data: Contiguous numpy array or buffer of volume data.
        width: X dimension.
        height: Y dimension.
        depth: Z dimension.
        internal_format: OpenGL internal format (e.g., GL_R8, GL_R32F).
        format: OpenGL format (e.g., GL_RED, GL_RGBA).
        data_type: OpenGL data type (e.g., GL_UNSIGNED_BYTE, GL_FLOAT).
    """
    # Step 1: Ensure data is contiguous
    if not data.flags['C_CONTIGUOUS']:
        data = np.ascontiguousarray(data, dtype=data.dtype)
    
    # Step 2: Generate and bind texture
    tex_id = gl.glGenTextures(1)
    gl.glBindTexture(gl.GL_TEXTURE_3D, tex_id)
    
    # Step 3: Set pixel store alignment
    gl.glPixelStorei(gl.GL_UNPACK_ALIGNMENT, 1)
    
    # Step 4: Set texture parameters
    gl.glTexParameteri(gl.GL_TEXTURE_3D, gl.GL_TEXTURE_WRAP_S, gl.GL_CLAMP_TO_EDGE)
    gl.glTexParameteri(gl.GL_TEXTURE_3D, gl.GL_TEXTURE_WRAP_T, gl.GL_CLAMP_TO_EDGE)
    gl.glTexParameteri(gl.GL_TEXTURE_3D, gl.GL_TEXTURE_WRAP_R, gl.GL_CLAMP_TO_EDGE)
    
    # Choose filter based on application needs (Nearest for voxels, Linear for smooth)
    gl.glTexParameteri(gl.GL_TEXTURE_3D, gl.GL_TEXTURE_MIN_FILTER, gl.GL_LINEAR)
    gl.glTexParameteri(gl.GL_TEXTURE_3D, gl.GL_TEXTURE_MAG_FILTER, gl.GL_LINEAR)
    
    # Step 5: Upload data
    gl.glTexImage3D(
        gl.GL_TEXTURE_3D,
        0,                  # level
        internal_format,    # internalformat
        width,              # width
        height,             # height
        depth,              # depth
        0,                  # border
        format,             # format
        data_type,          # type
        data                # data
    )
    
    # Step 6: Generate mipmaps if needed
    # gl.glGenerateMipmap(gl.GL_TEXTURE_3D)
    
    return tex_id
```

## Subtopics

### [3D Texture Upload — 3D Texture Upload for Volume Data, GL_TEXTURE_3D volume upload, Volume Texture Binding](4890-3d-texture-upload-3d-texture-upload-for-volume-data-gl-textu.md)
This method converts a 3D image stack from a nested array structure into a flat 1D buffer suitable for GPU upload via OpenGL. It solves the problem of mapping discrete volumetric data, which is often stored in host-memory structures with varying strides, into a contiguous memory layout required by `glTexImage3D` or `glTexSubImage3D`. Unlike sparse storage methods that suffer from pointer overhead or out-of-core streaming that incurs decoding latency, this method packs data into dense 3D texture slices, trading memory efficiency for hardware compatibility and sampling speed. It differs from its parent [3D Texture Upload](4419-3d-texture-upload.md) by handling the data transformation (format conversion, normalization) prior to the final buffer upload step, and from [3D Texture Initialization](13158-3d-texture-initialization.md) by focusing on the data layout rather than texture object creation.

### [Volume Atlas Texture Update](7157-volume-atlas-texture-update.md)
A method to populate a 3D volume atlas texture with sampled data values, tracking the minimum and maximum values encountered during the update process. It maps a sequence of timesteps into a multi-volume 3D texture array, allowing temporal data to be stored in a single GPU texture object for efficient rendering.

### [3D Texture Initialization](13158-3d-texture-initialization.md)
Allocates GPU memory for a 3D texture, configures sampling parameters (wrapping, filtering, mipmapping), and uploads initial volume data. It sets up the `GL_TEXTURE_3D` object required for subsequent volume rendering passes. It differs from [3D Texture Upload](4419-3d-texture-upload.md) by handling the full lifecycle of texture creation, storage allocation, and mipmap generation, whereas the parent focuses on binding and parameter setting for existing data.

### [Volume Texture Ping-Pong Swap](13951-volume-texture-ping-pong-swap.md)
Manages dynamic 3D scalar fields in GPU memory using a double-buffered ping-pong swap to avoid full re-uploads on field updates.

### Brick Grid Texture Upload
Creates multiple 3D textures for a brick grid structure: an indirection texture (`GL_RGB10_A2UI`), a range texture (`GL_RG16F`), and an atlas texture (`GL_COMPRESSED_RED`). It sets wrap modes to `GL_CLAMP_TO_BORDER` and filters to `GL_NEAREST`. Mipmaps are generated for the range texture by iterating through levels and calling `glTexImage3D` for each [S8, unofficial].

### 3D Volume Texture Population
Populates a 3D volume texture with procedural data: R channel stores noise, G channel stores local AO, B channel stores noise-driven height-based baseline albedo, and A channel stores procedural cloud physical density. This allows for consistent retrieval of density and albedo during raymarching [S9, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the total number of voxels ($X \times Y \times Z$), as each voxel must be copied from host to device memory.
*   **Memory Complexity:** $O(N)$ for the GPU texture storage. The host memory requirement depends on whether a contiguous copy is made.
*   **Measured Performance:** Upload speed is limited by PCIe bandwidth. For a $256^3$ volume with 32-bit floats, the data size is ~512 MB. On a typical PCIe 3.0 x16 link (theoretical 16 GB/s), the upload takes approximately 32 ms, though actual performance may vary due to driver overhead and system load [S2, S7, unofficial].
*   **Scaling:** Performance scales linearly with volume size. For very large volumes, consider chunking or streaming [S1, S8, unofficial].
*   **Strengths:** Simple to implement, hardware-accelerated sampling, compatible with standard OpenGL pipelines.
*   **Limitations:** Inefficient for sparse data, limited by GPU memory capacity, upload latency can be significant for large volumes.
*   **When to Use:** Dense volumes with moderate size, static or infrequently updated data.
*   **When Not to Use:** Sparse volumes (use sparse textures or grid-based methods), extremely large volumes requiring out-of-core streaming, or real-time updates with high-frequency changes (consider ping-pong or compute shader repacking).

## Background
The method originates from standard OpenGL volume rendering techniques, where 3D textures are used to store volumetric data for ray marching. Early implementations used `GL_TEXTURE_3D` with `GL_LUMINANCE` or `GL_RED` formats. Modern implementations leverage `GL_R8`, `GL_R16F`, and `GL_R32F` for better precision and performance. The technique is widely used in medical imaging, scientific visualization, and real-time volume rendering applications [S1, S2, S3, S4, S7, unofficial].

## Sources
[S1] Multivox: Volumetric Display, https://github.com/AncientJames/multivox, fetched 2026-10-26 (unofficial)
[S2] xwghua/clearvolume-plus, https://github.com/xwghua/clearvolume-plus, fetched 2026-10-26 (unofficial)
[S3] jaanonim/volume-rendering, https://github.com/jaanonim/volume-rendering, fetched 2026-10-26 (unofficial)
[S4] Dy111111/volumeRenderingGPU, https://github.com/Dy111111/volumeRenderingGPU, fetched 2026-10-26 (unofficial)
[S5] arigo/CascatedVoxelConeTracing, https://github.com/arigo/CascatedVoxelConeTracing, fetched 2026-10-26 (unofficial)
[S6] termijn/webgl-volumerendering, https://github.com/termijn/webgl-volumerendering, fetched 2026-10-27 (unofficial)
[S7] GitHub - m-pilia/volume-raycasting: Simple visualiser based on GPU ..., https://github.com/m-pilia/volume-raycasting, fetched 2026-10-27 (unofficial)
[S8] nihofm/volren: Simple OpenGL-based renderer for ..., https://github.com/nihofm/volren, fetched 2026-10-29 (unofficial)
[S9] Optimize cloud rendering with eroded Voronoi weather maps and stable linear raymarching, https://github.com/ricecake/boidsish/pull/679, fetched 2026-10-28 (unofficial)
