# Voxel Volume Initialization & Binding

Also called: VoxelVolume Setup

## Overview
Voxel Volume Initialization & Binding prepares a dense 3D texture for voxel cone tracing by allocating GPU memory, configuring sampling parameters, and establishing read/write bindings for compute shaders. It solves the problem of transitioning from raw scene data to a structured, mipmapped voxel grid ready for lighting injection. Unlike [3D Texture Upload](4419-3d-texture-upload.md) which focuses on data transfer, this method handles the OpenGL resource lifecycle and binding state. It differs from [Sparse Volume Texture Setup](7432-sparse-volume-texture-setup.md) by using a dense `GL_TEXTURE_3D` with fixed dimensions rather than sparse binding extensions.

## How it works
**Inputs:**
- `_volumeSize`: Integer defining the cubic resolution (N x N x N).
- `volumeMipmapLevel`: Integer specifying the target mip level for subsequent operations.
- Shader uniforms: `minVertex`, `maxVertex` (bounding box coordinates).

**Outputs:**
- Bound 3D texture resource (`_volumeTexture`).
- Framebuffer Object (FBO) with the texture attached.
- Image load/store binding for write access.
- Texture unit binding for read access.

**Data Structures:**
- `_volumeTexture`: OpenGL handle for `GL_TEXTURE_3D`.
- Internal format: `GL_RGBA8` (8-bit unsigned integer per channel, normalized).
- Dimensions: `volumeSize` x `volumeSize` x `volumeSize`.
- Border: 0.

**Steps:**

1.  **Clear and Resize:** Clear any existing volume data and update the internal `_volumeSize` variable to the new resolution.
2.  **Texture Generation:** Generate a new 3D texture object. Bind it as `GL_TEXTURE_3D`. Set dimensions to `volumeSize` for all three axes. Set internal format to `GL_RGBA8`.
3.  **Parameter Configuration:**
    -   Minification filter: `GL_LINEAR_MIPMAP_LINEAR`.
    -   Magnification filter: `GL_LINEAR`.
    -   Wrap modes (S, T, R): `GL_CLAMP_TO_BORDER`.
4.  **Mipmap Generation:** Generate mipmaps for the bound 3D texture.
5.  **Framebuffer Setup:** Create a Framebuffer Object (FBO). Bind the FBO. Attach the generated 3D texture to attachment point `GL_COLOR_ATTACHMENT0`.
6.  **Write Binding:** Bind the texture for image load/store operations.
    -   Target: `GL_TEXTURE_3D`.
    -   Unit: 0.
    -   Access: `GL_WRITE_ONLY`.
    -   Format: `GL_RGBA8`.
    -   Call: `glBindImageTexture(0, _volumeTexture, 0, GL_FALSE, 0, GL_WRITE_ONLY, GL_RGBA8)`.
7.  **Read Binding:** Activate texture unit 10. Bind the 3D texture to this unit for sampling.
8.  **Shader Uniforms:** Pass the following attributes to the shader pipeline:
    -   `volumeTexture`: Sampler index 10 (`ATTR_1I`, value 10).
    -   `minVertex`: Bounding box minimum (`ATTR_3F`).
    -   `maxVertex`: Bounding box maximum (`ATTR_3F`).
    -   `volumeSize`: Resolution vector (`ATTR_3I`).
    -   `volumeMipmapLevel`: Target mip level (`ATTR_1I`).

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_voxel_volume(volume_size, volume_mipmap_level, min_vertex, max_vertex):
    # 1. Clear existing data
    clear_existing_volume_data()
    _volume_size = volume_size

    # 2. Generate 3D Texture
    _volume_texture = glCreateTexture(GL_TEXTURE_3D)
    glBindTexture(GL_TEXTURE_3D, _volume_texture)
    glTexImage3D(
        GL_TEXTURE_3D,
        0,                  # Level
        GL_RGBA8,           # Internal format
        volume_size,        # Width
        volume_size,        # Height
        volume_size,        # Depth
        0,                  # Border
        GL_RGBA,            # Format (unused for internal format storage but required by API)
        GL_UNSIGNED_BYTE,   # Type
        None                # Data (null for allocation)
    )

    # 3. Set Parameters
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR)
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_MAG_FILTER, GL_LINEAR)
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_BORDER)
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_BORDER)
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_WRAP_R, GL_CLAMP_TO_BORDER)

    # 4. Generate Mipmaps
    glGenerateMipmap(GL_TEXTURE_3D)

    # 5. FBO Setup
    fbo = create_fbo()
    bind_fbo(fbo)
    glFramebufferTexture3D(
        GL_FRAMEBUFFER,
        GL_COLOR_ATTACHMENT0,
        GL_TEXTURE_3D,
        _volume_texture,
        0  # Mipmap level
    )

    # 6. Bind for Writing (Image Load/Store)
    glBindImageTexture(
        0,                  # Binding point
        _volume_texture,    # Texture ID
        0,                  # Mipmap level
        GL_FALSE,           # Layered
        0,                  # Layer
        GL_WRITE_ONLY,      # Access
        GL_RGBA8            # Format
    )

    # 7. Bind for Reading
    glActiveTexture(GL_TEXTURE10)
    glBindTexture(GL_TEXTURE_3D, _volume_texture)

    # 8. Set Shader Uniforms
    set_uniform_1i("volumeTexture", 10)
    set_uniform_3f("minVertex", min_vertex.x, min_vertex.y, min_vertex.z)
    set_uniform_3f("maxVertex", max_vertex.x, max_vertex.y, max_vertex.z)
    set_uniform_3i("volumeSize", volume_size, volume_size, volume_size)
    set_uniform_1i("volumeMipmapLevel", volume_mipmap_level)

    unbind_fbo()
```

**Edge Cases and Pitfalls:**
-   **Precision:** `GL_RGBA8` limits color precision to 256 levels per channel. For high-dynamic-range lighting, this may cause banding. [G] Ensure the consumer shaders interpret the values correctly (normalized vs. raw).
-   **Boundaries:** `GL_CLAMP_TO_BORDER` requires a border color to be set if sampling outside the volume. [G] Set border color to `(0,0,0,0)` to ensure out-of-bounds samples return black/transparent.
-   **Mipmap Levels:** If `volumeMipmapLevel` is requested but mipmaps are not generated, sampling may fail or return undefined data.

## Subtopics

### Voxel Border Pass
A compute-shader-based post-processing step that refines the voxel volume boundaries across mip levels. It iterates from mip level 0 to `MIPMAP_LEVELS - 1`. For each level, it dispatches a compute shader (`VOXEL_BORDER_PASS_COMPUTE_SHADER`) with work groups calculated as `ceil(dstMipRes / 4.0)` for X, Y, and Z. It sets source mip level and destination resolution uniforms. Memory barriers are inserted after dispatch and after the loop to ensure coherence between image writes and subsequent reads. This ensures smooth transitions and correct boundary representation in lower-resolution mip levels.

## Operating characteristics
-   **Time Complexity:** O(N^3) for texture allocation and mipmap generation, where N is `volumeSize`. The border pass adds O(N^3) work across all mip levels.
-   **Memory Complexity:** O(N^3) for the dense 3D texture storage. `GL_RGBA8` uses 1 byte per channel, 4 bytes per voxel.
-   **Performance:** Dependent on GPU memory bandwidth for texture allocation. Mipmap generation is typically hardware-accelerated.
-   **Strengths:** Simple API usage; standard OpenGL compatibility; efficient for dense volumes.
-   **Limitations:** High memory usage for large N; no support for sparse data structures; fixed resolution.
-   **When to use:** When the scene fits within memory limits and dense voxelization is required for cone tracing.
-   **When not to use:** For very large scenes where memory is constrained; consider [Sparse Volume Texture Setup](7432-sparse-volume-texture-setup.md) instead.

## Background
Originated in the context of real-time voxel cone tracing implementations, specifically within the Voxel-Cone-Tracing project. It provides the foundational GPU resource management for voxel-based global illumination.

## Sources
[S1] maritim/Voxel-Cone-Tracing, https://github.com/maritim/Voxel-Cone-Tracing, fetched 2026-10-26 (unofficial)
