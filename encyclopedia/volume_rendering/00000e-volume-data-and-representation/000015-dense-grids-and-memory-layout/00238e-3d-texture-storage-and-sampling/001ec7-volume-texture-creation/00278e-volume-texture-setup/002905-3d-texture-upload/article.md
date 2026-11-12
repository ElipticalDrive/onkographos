# 3D Texture Upload

Also called: 3D Texture Upload for Volume Data, 3D Volume Texture Upload, GL_TEXTURE_3D volume upload, Volume Texture Binding, Volume Texture Upload

## Overview
This method handles the transfer of preprocessed scalar field data from CPU memory to GPU memory via an OpenGL `GL_TEXTURE_3D` object [S1, unofficial]. It solves the problem of bridging the gap between host-side data structures (often nested arrays or raw buffers) and the GPU's texture sampling hardware [S1, unofficial]. It is distinct from [Volume Texture Generation](7099-volume-texture-generation.md), which prepares the data format, and [3D Texture Upload](4419-3d-texture-upload.md), which manages the final RGBA classified data upload with specific filtering parameters [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A contiguous array of volume data (scalar field) and its dimensions $(W, H, D)$ [S1, unofficial]. Optionally, a 2D transfer function texture [S1, unofficial].
*   **Output:** A bound OpenGL `GL_TEXTURE_3D` object containing the volume data, configured with specific internal formats, wrapping, and filtering parameters [S4, unofficial].

**Data Structures:**
The volume data is typically stored in a 1D contiguous buffer representing a 3D grid. The layout follows row-major or column-major ordering depending on the host implementation, but the upload call maps this linear memory to 3D coordinates $(x, y, z)$ where $x$ varies fastest [S2, unofficial].

**Parameters:**
*   `internalFormat`: The format of the texture in GPU memory. Common values are `GL_R8` (8-bit unsigned integer) or `GL_R16F` (16-bit floating point) [S1, unofficial][S3, unofficial][S4, unofficial].
*   `format`: The format of the data in the host buffer. Typically `GL_RED` for single-channel data [S1, unofficial][S4, unofficial].
*   `type`: The data type of the host buffer. Typically `GL_UNSIGNED_BYTE` for `R8` or `GL_HALF_FLOAT` for `R16F` [S1, unofficial][S3, unofficial][S4, unofficial].
*   `wrapMode`: Set to `GL_CLAMP_TO_EDGE` to prevent sampling artifacts at boundaries [S1, unofficial][S3, unofficial][S4, unofficial].
*   `minFilter` / `magFilter`: Set to `GL_LINEAR` for trilinear interpolation [S1, unofficial][S3, unofficial][S4, unofficial]. Some implementations use `GL_LINEAR_MIPMAP_LINEAR` with mipmaps generated [S2, unofficial].

**Steps:**
1.  **Generate and Bind:** Generate a texture ID and bind it to `GL_TEXTURE_3D` [S1, unofficial][S2, unofficial].
2.  **Allocate and Upload:** Call `glTexImage3D` to allocate storage and upload the data.
    *   `target`: `GL_TEXTURE_3D`
    *   `level`: 0 (base level)
    *   `internalformat`: e.g., `GL_R8` or `GL_R16F`
    *   `width`, `height`, `depth`: The dimensions of the volume $(W, H, D)$
    *   `border`: 0
    *   `format`: `GL_RED`
    *   `type`: `GL_UNSIGNED_BYTE` or `GL_HALF_FLOAT`
    *   `data`: Pointer to the host memory buffer
3.  **Set Parameters:**
    *   Set `GL_TEXTURE_WRAP_S`, `GL_TEXTURE_WRAP_T`, and `GL_TEXTURE_WRAP_R` to `GL_CLAMP_TO_EDGE` [S1, unofficial][S3, unofficial][S4, unofficial].
    *   Set `GL_TEXTURE_MIN_FILTER` and `GL_TEXTURE_MAG_FILTER` to `GL_LINEAR` [S1, unofficial][S3, unofficial][S4, unofficial].
    *   (Optional) If using mipmaps, set `GL_TEXTURE_MAX_LEVEL` and generate mipmaps using `glGenerateMipmap` [S2, unofficial].
4.  **Bind to Unit:** Bind the texture to a specific texture unit (e.g., unit 0) and update the shader uniform location [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def upload_3d_texture(volume_data, width, height, depth, internal_format, data_type):
    """
    Uploads 3D volume data to an OpenGL texture.
    
    Args:
        volume_data: bytes or array-like, contiguous host memory.
        width, height, depth: int, dimensions of the volume.
        internal_format: str, e.g., 'GL_R8', 'GL_R16F'.
        data_type: str, e.g., 'GL_UNSIGNED_BYTE', 'GL_HALF_FLOAT'.
    """
    # 1. Generate and bind texture
    texture_id = gl.glGenTextures(1)
    gl.glBindTexture(gl.GL_TEXTURE_3D, texture_id)
    
    # 2. Allocate and upload data
    # glTexImage3D(target, level, internalformat, width, height, depth, border, format, type, data)
    gl.glTexImage3D(
        target=gl.GL_TEXTURE_3D,
        level=0,
        internalformat=internal_format,
        width=width,
        height=height,
        depth=depth,
        border=0,
        format=gl.GL_RED,
        type=data_type,
        data=volume_data
    )
    
    # 3. Set wrapping parameters
    gl.glTexParameteri(gl.GL_TEXTURE_3D, gl.GL_TEXTURE_WRAP_S, gl.GL_CLAMP_TO_EDGE)
    gl.glTexParameteri(gl.GL_TEXTURE_3D, gl.GL_TEXTURE_WRAP_T, gl.GL_CLAMP_TO_EDGE)
    gl.glTexParameteri(gl.GL_TEXTURE_3D, gl.GL_TEXTURE_WRAP_R, gl.GL_CLAMP_TO_EDGE)
    
    # 4. Set filtering parameters
    gl.glTexParameteri(gl.GL_TEXTURE_3D, gl.GL_TEXTURE_MIN_FILTER, gl.GL_LINEAR)
    gl.glTexParameteri(gl.GL_TEXTURE_3D, gl.GL_TEXTURE_MAG_FILTER, gl.GL_LINEAR)
    
    # 5. (Optional) Generate mipmaps if min_filter requires it
    # gl.glGenerateMipmap(gl.GL_TEXTURE_3D)
    
    return texture_id
```

**Edge Cases and Pitfalls:**
*   **Alignment:** Ensure `GL_UNPACK_ALIGNMENT` is set correctly (typically 1 for tightly packed data) to avoid stride errors [S4, unofficial].
*   **Data Type Mismatch:** The `type` parameter must match the actual data type in the host buffer. Uploading `GL_UNSIGNED_BYTE` data with `GL_HALF_FLOAT` type will result in garbage values [S1, unofficial][S3, unofficial].
*   **Boundary Sampling:** Without `GL_CLAMP_TO_EDGE`, sampling at the exact edge (coordinate 1.0) may access undefined memory or wrap around, causing visual artifacts [S1, unofficial][S4, unofficial].

## Subtopics

### Brick Grid Texture Upload
This variant handles the upload of multiple auxiliary textures for a brick-grid based volume rendering system, including indirection, range, and atlas textures [S5, unofficial]. It differs from standard scalar upload by managing multiple texture objects with different internal formats and mipmap chains [S5, unofficial].
*   **Indirection Texture:** Created with `GL_RGB10_A2UI` format, `GL_UNSIGNED_INT_10_10_10_2` type, and `GL_CLAMP_TO_BORDER` wrapping [S5, unofficial].
*   **Range Texture:** Created with `GL_RG16F` format, `GL_HALF_FLOAT` type, and mipmaps generated for each level [S5, unofficial].
*   **Atlas Texture:** Created with `GL_COMPRESSED_RED` format, `GL_UNSIGNED_BYTE` type, and `GL_CLAMP_TO_BORDER` wrapping [S5, unofficial].
*   **Filtering:** All textures use `GL_NEAREST` filtering [S5, unofficial].

### 3D Volume Texture Population
This variant focuses on populating specific channels of a 3D volume texture with procedural data, such as cloud rendering data [S6, unofficial]. It differs by storing multiple semantic values in different channels (R, G, B, A) rather than a single scalar field [S6, unofficial].
*   **Channel Layout:** R channel stores noise, G channel stores local AO, B channel stores noise-driven height-based baseline albedo, and A channel stores procedural cloud physical density [S6, unofficial].
*   **Usage:** This allows for efficient retrieval of density and albedo during raymarching, ensuring rendering consistency [S6, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels, as the entire data set must be transferred from CPU to GPU memory [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for the GPU texture storage, plus $O(N)$ for the host buffer [S1, unofficial].
*   **Performance:** The upload speed is limited by the PCIe bus bandwidth. For large volumes, this can be a bottleneck. Using `GL_R16F` doubles the memory bandwidth requirement compared to `GL_R8` [S1, unofficial][S3, unofficial].
*   **Strengths:** Simple API, hardware-accelerated sampling, supports mipmaps for LOD [S1, unofficial][S2, unofficial].
*   **Limitations:** Fixed memory layout (dense grid), not suitable for highly sparse data without compression [S1, unofficial].

## Background
This method is a standard technique in GPU-based volume rendering, relying on OpenGL's `glTexImage3D` function. It is widely used in educational examples and simple raymarchers [S1, unofficial][S2, unofficial][S3, unofficial]. The approach has been adapted for various specialized rendering techniques, such as brick-grid rendering [S5, unofficial] and procedural cloud generation [S6, unofficial].

## Sources
[S1] jaanonim/volume-rendering, https://github.com/jaanonim/volume-rendering, fetched 2026-10-26 (unofficial)
[S2] Dy111111/volumeRenderingGPU, https://github.com/Dy111111/volumeRenderingGPU, fetched 2026-10-26 (unofficial)
[S3] termijn/webgl-volumerendering, https://github.com/termijn/webgl-volumerendering, fetched 2026-10-27 (unofficial)
[S4] GitHub - m-pilia/volume-raycasting: Simple visualiser based on GPU ..., https://github.com/m-pilia/volume-raycasting, fetched 2026-10-27 (unofficial)
[S5] nihofm/volren: Simple OpenGL-based renderer for ..., https://github.com/nihofm/volren, fetched 2026-10-29 (unofficial)
[S6] Optimize cloud rendering with eroded Voronoi weather maps and stable linear raymarching, https://github.com/ricecake/boidsish/pull/679, fetched 2026-10-28 (unofficial)
