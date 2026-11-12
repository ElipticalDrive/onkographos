# Voxel Volume Initialization & Binding

Also called: VoxelVolume Setup

## Overview
This method initializes the GPU-side 3D texture storage and binds it for simultaneous read and write access, preparing the voxel grid for subsequent data injection. It differs from [3D Texture Upload](4419-3d-texture-upload.md) by focusing on the creation of empty, mipmapped storage and image load/store binding rather than uploading pre-existing host memory, and from [Sparse Volume Texture Setup](7432-sparse-volume-texture-setup.md) by using a dense `GL_TEXTURE_3D` layout. It serves as the prerequisite for [Voxel Radiance Injection](12433-voxel-radiance-injection.md).

## How it works
**Inputs:** `_volumeSize` (integer, dimension of the cubic volume), `volumeMipmapLevel` (integer, from settings).
**Outputs:** Bound 3D texture object, bound Framebuffer Object (FBO), image load/store binding point.

**Data Structures:**
*   `_volumeTexture`: OpenGL texture object ID.
*   `_volumeSize`: `int` or `glm::ivec3` representing the side length $N$.
*   Pipeline Uniforms/Attributes:
    *   `volumeTexture`: Sampler index 10.
    *   `minVertex`: `vec3` (min bounding box corner).
    *   `maxVertex`: `vec3` (max bounding box corner).
    *   `volumeSize`: `ivec3` (dimensions $N \times N \times N$).
    *   `volumeMipmapLevel`: `int`.

**Steps:**
1.  **Clear and Store Size:** Clear any existing volume data associated with the volume object and store the new `_volumeSize` [S1, unofficial].
2.  **Generate 3D Texture:** Create a 3D texture object (`GL_TEXTURE_3D`) with internal format `GL_RGBA8`, dimensions $N \times N \times N$, and border 0 [S1, unofficial].
3.  **Set Parameters:** Configure texture parameters:
    *   Min filter: `GL_LINEAR_MIPMAP_LINEAR`
    *   Mag filter: `GL_LINEAR`
    *   Wrap S, T, R: `GL_CLAMP_TO_BORDER` [S1, unofficial].
4.  **Generate Mipmaps:** Generate mipmaps for the 3D texture [S1, unofficial].
5.  **Create and Attach FBO:** Create a Framebuffer Object (FBO), bind it, and attach the 3D texture to `GL_COLOR_ATTACHMENT0` [S1, unofficial].
6.  **Bind for Writing (Image Load/Store):** Bind the texture for writing using image load/store at binding point 0:
    ```cpp
    GL::BindImageTexture(0, _volumeTexture, 0, GL_FALSE, 0, GL_WRITE_ONLY, GL_RGBA8);
    ```
    [S1, unofficial].
7.  **Bind for Reading:** Activate texture unit 10 and bind the 3D texture to this unit for sampling [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_voxel_volume(volume_size: int, volume_mipmap_level: int):
    # 1. Clear existing data and store size
    clear_existing_volume_data()
    _volume_size = volume_size

    # 2. Generate 3D texture
    _volume_texture = glCreateTexture(GL_TEXTURE_3D)
    glTextureStorage3D(_volume_texture, 
                       generate_mipmap_levels(volume_size), 
                       GL_RGBA8, 
                       volume_size, volume_size, volume_size)

    # 3. Set parameters
    glTextureParameteri(_volume_texture, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR)
    glTextureParameteri(_volume_texture, GL_TEXTURE_MAG_FILTER, GL_LINEAR)
    glTextureParameteri(_volume_texture, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_BORDER)
    glTextureParameteri(_volume_texture, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_BORDER)
    glTextureParameteri(_volume_texture, GL_TEXTURE_WRAP_R, GL_CLAMP_TO_BORDER)

    # 4. Generate mipmaps
    glGenerateTextureMipmap(_volume_texture)

    # 5. Create and attach FBO
    fbo = glCreateFramebuffer()
    glBindFramebuffer(GL_FRAMEBUFFER, fbo)
    glFramebufferTexture3D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_3D, _volume_texture, 0, 0)

    # 6. Bind for writing (Image Load/Store)
    glBindImageTexture(0, _volume_texture, 0, GL_FALSE, 0, GL_WRITE_ONLY, GL_RGBA8)

    # 7. Bind for reading (Texture Unit 10)
    glActiveTexture(GL_TEXTURE10)
    glBindTexture(GL_TEXTURE_3D, _volume_texture)

    # Set pipeline uniforms
    set_uniform_1i("volumeTexture", 10)
    set_uniform_3i("volumeSize", volume_size, volume_size, volume_size)
    set_uniform_1i("volumeMipmapLevel", volume_mipmap_level)
    # minVertex and maxVertex must be set by caller based on scene bounds [S1, unofficial]

def generate_mipmap_levels(n: int) -> int:
    import math
    return int(math.floor(math.log2(n))) + 1
```

**Edge Cases and Pitfalls:**
*   **Precision:** `GL_RGBA8` provides 8-bit per channel precision. For high-dynamic-range radiance or density, this may saturate. [G] Ensure `GL_CLAMP_TO_BORDER` is used to prevent wrapping artifacts at volume boundaries during sampling.
*   **Boundaries:** The `GL_CLAMP_TO_BORDER` wrap mode requires a border color to be defined if accessed outside bounds. [G] Set border color to `(0,0,0,0)` to ensure out-of-bounds reads return zero.
*   **Mipmap Levels:** The number of mipmap levels is determined by $\lfloor \log_2(N) \rfloor + 1$. If `volumeMipmapLevel` exceeds this, sampling may fail or return undefined values.

## Subtopics

### Voxel Border Pass
A compute shader-based post-initialization step that fills the borders of each mipmap level. It iterates from mip level 0 to $MIPMAP\_LEVELS - 1$. For each level, it dispatches a compute shader with work groups calculated as $\lceil \frac{dstMipRes}{4.0} \rceil$ in X, Y, and Z. It uses a source mip level uniform and a destination mip resolution uniform. Memory barriers are inserted before and after the dispatch to ensure coherence between texture fetches and image writes. The destination mip resolution is halved for the next iteration. This ensures that border voxels are correctly propagated or initialized across the mipmap chain [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ for texture creation and binding. Mipmap generation is $O(N^3)$ where $N$ is the volume dimension. The Voxel Border Pass is $O(N^3)$ total across all mip levels.
*   **Memory Complexity:** $O(N^3)$ for the 3D texture storage (`GL_RGBA8`).
*   **Performance:** Dependent on GPU memory bandwidth for mipmap generation. Binding is instantaneous.
*   **Strengths:** Provides a unified read/write interface via image load/store, essential for voxel cone tracing or ray marching where voxels are updated dynamically.
*   **Limitations:** Dense storage is memory-intensive for large $N$. `GL_RGBA8` may lack precision for certain physical quantities.
*   **When to use:** When dynamic voxel updates are required via compute shaders and mipmapped sampling is needed.
*   **When not to use:** For static volumes where upload-only is sufficient, or when memory is constrained and sparse representations are viable.

## Background
This method is derived from the Voxel Cone Tracing implementation by maritim [S1, unofficial]. It reflects standard OpenGL practices for setting up 3D textures for compute shader access, combining FBO attachment for rendering compatibility with image load/store for compute access.

## Sources
[S1] maritim/Voxel-Cone-Tracing, https://github.com/maritim/Voxel-Cone-Tracing, fetched 2026-10-26 (unofficial)
