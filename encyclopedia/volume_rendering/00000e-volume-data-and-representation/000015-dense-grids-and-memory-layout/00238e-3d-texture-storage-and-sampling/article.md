# 3D Texture Storage and Sampling

Also called: GPU 3D Texture Mapping, Hardware Volume Texture Sampling

## Overview
3D Texture Storage and Sampling is a method for storing volumetric data in GPU 3D image resources and sampling them during rendering. It solves the problem of inefficient memory access patterns inherent in linear array storage by leveraging hardware-optimized data locality mechanisms [S1, unofficial]. Unlike [Dense grids and memory layout](3-dense-grids-and-memory-layout.md) which focuses on CPU-side or generic linear indexing, this method utilizes GPU-specific structures that provide superior cache coherence for ray traversal compared to Z-order curves [S1, unofficial]. It differs from [Sparse Volume Texture (SVT)](7011-sparse-volume-texture-svt.md) by operating on dense grids without page tables, and from [Brick-based storage and caching](4138-brick-based-storage-and-caching.md) by relying on the GPU's internal texture unit logic rather than explicit software-managed brick hierarchies.

## How it works
**Inputs:**
- Volumetric data array $V$ of dimensions $D_x \times D_y \times D_z$.
- GPU context supporting 3D texture objects (e.g., OpenGL `GL_TEXTURE_3D`, Vulkan `VK_IMAGE_TYPE_3D`).

**Outputs:**
- A GPU-accessible 3D texture resource $T$.
- Sampled values $V(\mathbf{p})$ for any query point $\mathbf{p}$ within the domain.

**Data Structures:**
- **3D Texture Object:** A hardware resource defined by width $W$, height $H$, and depth $D$.
- **Internal Format:** Determines the number of channels and bit depth (e.g., `GL_RGB`, `GL_R16UI`, `GL_R8UI`) [S3, unofficial][S4, unofficial].
- **Data Type:** The underlying storage type (e.g., `GL_UNSIGNED_BYTE`, `GL_UNSIGNED_SHORT`, `GL_FLOAT`) [S3, unofficial][S4, unofficial].

**Steps:**

1.  **Texture Allocation:** Create a 3D texture object on the GPU with dimensions matching the volume resolution $(W, H, D)$ [S4, unofficial].
2.  **Data Upload:** Transfer the host-side volumetric data array to the GPU texture memory. The data is typically stored in row-major order, but the GPU hardware manages the physical memory layout for optimal locality [S1, unofficial].
3.  **Parameter Configuration:** Set texture parameters:
    -   **Filtering:** Enable `GL_LINEAR` (trilinear interpolation) for smooth sampling [S3, unofficial].
    -   **Wrapping:** Set to `GL_CLAMP_TO_EDGE` to prevent sampling outside the volume bounds [G].
    -   **Mipmapping:** Typically disabled for volume rendering to avoid blurring during ray marching, unless specific multi-resolution techniques are employed [G].
4.  **Sampling:** During rendering, the fragment shader or compute shader samples the texture at normalized coordinates $(u, v, w) \in [0, 1]^3$. The hardware performs trilinear interpolation:
    $$ V(\mathbf{p}) = \sum_{i,j,k \in \{0,1\}} w_i w_j w_k V_{ijk} $$
    where $w_i, w_j, w_k$ are the interpolation weights derived from the fractional part of the query position [G].

**Parameters:**
-   **Resolution:** Must fit within GPU texture limits (e.g., 2048 or 4096 per dimension on modern hardware) [S1, unofficial].
-   **Format:** `GL_RGB` for color data, `GL_R8UI`/`GL_R16UI` for scalar fields [S3, unofficial][S4, unofficial].

**Edge Cases and Pitfalls:**
-   **Memory Limits:** Large volumes ($> 2048^3$) may exceed maximum texture dimensions, requiring [Sparse Volume Texture (SVT)](7011-sparse-volume-texture-svt.md) or [Brick-based storage](4138-brick-based-storage-and-caching.md) [S1, unofficial].
-   **Precision:** 8-bit formats may introduce banding in smooth gradients; 16-bit or 32-bit floats are preferred for scientific data [S2, unofficial][S4, unofficial].
-   **Boundary Artifacts:** Without `CLAMP_TO_EDGE`, wrapping modes can cause data leakage from opposite sides of the volume [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_3d_texture(volume_data, width, height, depth, internal_format, data_type):
    """
    Allocates and uploads volumetric data to a GPU 3D texture.
    
    Args:
        volume_data: numpy array of shape (depth, height, width) or (width, height, depth) depending on layout.
        width, height, depth: Dimensions of the volume.
        internal_format: OpenGL internal format string (e.g., 'GL_RGB', 'GL_R16UI').
        data_type: OpenGL data type string (e.g., 'GL_UNSIGNED_BYTE', 'GL_FLOAT').
        
    Returns:
        texture_id: Integer handle to the GPU texture object.
    """
    # 1. Generate texture name
    texture_id = glGenTextures(1)
    
    # 2. Bind the texture
    glBindTexture(GL_TEXTURE_3D, texture_id)
    
    # 3. Set parameters
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_MIN_FILTER, GL_LINEAR)
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_MAG_FILTER, GL_LINEAR)
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE)
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE)
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_WRAP_R, GL_CLAMP_TO_EDGE)
    
    # 4. Allocate and upload data
    # Note: Data layout (row-major vs column-major) must match the source array's memory layout.
    # OpenGL typically expects data in row-major order (x varies fastest, then y, then z).
    glTexImage3D(
        GL_TEXTURE_3D, 
        0,                  # Mipmap level
        internal_format,    # Internal format
        width, height, depth, # Dimensions
        0,                  # Border
        internal_format,    # Format (must match internal_format for single-channel or RGB)
        data_type,          # Type
        volume_data         # Pointer to host data
    )
    
    return texture_id

def sample_3d_texture(texture_id, u, v, w):
    """
    Samples a 3D texture at normalized coordinates.
    
    Args:
        texture_id: Handle to the GPU texture.
        u, v, w: Normalized coordinates in [0, 1].
        
    Returns:
        color: Sampled value (vec3 or float).
    """
    # In a shader, this is done via texture() or tex3D()
    # In CPU-side emulation or debug:
    # 1. Convert normalized coords to voxel indices
    # 2. Perform trilinear interpolation
    pass
```

## Subtopics

### [Volume Texture Creation](1857-volume-texture-creation.md)
Handles the high-level allocation, data transfer, and object lifecycle of GPU texture resources. It translates host-side representations into hardware-ready objects with correct memory layouts and filtering modes, differing from [Volume Texture Setup](4055-volume-texture-setup.md) which configures sampling states.

### [PixelFlow Volume Rendering](3443-pixelflow-volume-rendering.md)
A polygon resampling method for the PixelFlow system that allows pixel processors to manipulate 3D texture values directly, enabling on-the-fly Phong lighting calculations derived from interpolated volume gradients.

### [Texture Array Sampling with Alpha Discard](3993-texture-array-sampling-with-alpha-discard.md)
Implements fragment shader sampling of a 3D texture array to retrieve voxel color and opacity, discarding fragments with alpha below a threshold to avoid rendering transparent regions.

### [Ambient Occlusion Volume](4341-ambient-occlusion-volume.md)
Stores precomputed near-field ambient occlusion factors directly in a 3D texture, optionally alongside bent normals, to avoid runtime reconstruction computation.

### [Hardware Volume Rendering Pipeline with Pixel Textures](5641-hardware-volume-rendering-pipeline-with-pixel-textures.md)
Implements direct volume rendering on hardware supporting 3D texture mapping (e.g., SGI V-series, NVIDIA GeForce3) by using pixel textures (dependent textures) to perform classification and shading lookups, decoupling data storage from shading computation.

### [Sparse Volume Texture (SVT)](7011-sparse-volume-texture-svt.md)
Trades rendering speed for data compression by chunking voxel data into 3D tiles and storing only non-empty tiles in a physical texture, using a second volume texture as a page table for lookup.

### [Memory Organization and Layout](7154-memory-organization-and-layout.md)
Defines memory organization strategies for storing dense voxel grids in GPU texture resources, mapping linear voxel coordinates to physical memory addresses to optimize cache locality and bandwidth, distinguishing between standard row-major storage and hardware-specific interleaving schemes.

### [Texture Packing and Atlas Organization](11925-texture-packing-and-atlas-organization.md)
Consolidates multiple volumetric data sources or semantic channels into fewer, larger texture resources to optimize memory bandwidth, reduce state changes, and emulate 3D texture support in constrained environments.

### OpenGL Texture Output
A method to output rendering results using OpenGL texture objects for efficiency. It initializes texture objects in the `DrawModel` class with format `GL_RGB` and texel format `GL_UNSIGNED_BYTE`, then defines a `GL_QUAD` polygon to map the texture onto the screen display plane [S3, unofficial].

### Original Volume Rendering
A rendering mode that loads the entire volume dataset into a single 3D texture, bypassing page tables and masking. It creates a `TEXTURE_3D` with dimensions `source.dimensions_xyz`, internal format `R16UI` or `R8UI`, and type `UNSIGNED_SHORT` or `UNSIGNED_BYTE` [S4, unofficial]. Unlike compact mode, it requires the entire volume to fit in GPU memory but simplifies shader logic [S4, unofficial].

## Operating characteristics
-   **Time Complexity:** Sampling is $O(1)$ per voxel access due to hardware interpolation [G].
-   **Memory Complexity:** $O(N)$ where $N$ is the total number of voxels, stored contiguously in GPU VRAM [S1, unofficial].
-   **Performance:** Storing voxels in a 3D texture can yield a 10x improvement over linear array storage (e.g., 12 FPS to 121 FPS at $2048^3$ resolution) due to optimized data locality and cache coherence [S1, unofficial].
-   **Strengths:** High performance for dense volumes; hardware-accelerated trilinear interpolation; simple API usage [S1, unofficial][S3, unofficial].
-   **Limitations:** Limited by maximum texture dimensions (e.g., 2048 or 4096 per axis); inefficient for sparse data compared to [Sparse Volume Texture (SVT)](7011-sparse-volume-texture-svt.md) [S1, unofficial].
-   **When to Use:** Dense volumetric datasets that fit within GPU texture limits; real-time rendering applications requiring high throughput [S1, unofficial].
-   **When Not to Use:** Sparse datasets; volumes exceeding maximum texture dimensions; scenarios requiring complex data structures like [VDB](4880-vdb-data-loading-3d-texture-setup.md) [S1, unofficial][S7154-memory-organization-and-layout.md].

## Background
The method originated with the advent of GPU hardware supporting 3D texture mapping, such as the SGI V-series and NVIDIA GeForce3 [S1, unofficial][S2, unofficial][S3, unofficial]. It evolved as a solution to the memory bandwidth bottlenecks of linear array storage, leveraging the GPU's specialized texture units for data locality [S1, unofficial]. Early implementations used pixel textures (dependent textures) for classification and shading lookups [S1, unofficial][S2, unofficial][S3, unofficial]. Modern implementations continue to use 3D textures for dense volume rendering, often combined with techniques like [Brick-based storage](4138-brick-based-storage-and-caching.md) or [Sparse Volume Texture (SVT)](7011-sparse-volume-texture-svt.md) for larger datasets [S1, unofficial][S7154-memory-organization-and-layout.md].

## Sources
[S1] This Tiny Algorithm Can Render BILLIONS of Voxels in Real Time, https://www.youtube.com/watch?v=ztkh1r1ioZo, fetched 2026-10-25 (unofficial)
[S2] Example-based volume illustrations, https://ieeexplore.ieee.org/abstract/document/1532854/, fetched 2026-10-31 (unofficial)
[S3] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
[S4] Winson-L/VolumeSegmenter, https://github.com/Winson-L/VolumeSegmenter, fetched 2026-10-27 (unofficial)
