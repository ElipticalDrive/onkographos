# 3D Texture Storage and Sampling


## Overview
3D Texture Storage and Sampling stores volumetric data in GPU 3D image resources and samples them during rendering [S1, unofficial]. It solves the problem of inefficient memory access patterns inherent in linear array storage by leveraging hardware-optimized data locality mechanisms [S1, unofficial]. Unlike [Dense grids and memory layout](3-dense-grids-and-memory-layout.md) which focuses on CPU-side or generic linear indexing, this method utilizes GPU-specific structures that provide superior cache coherence for ray traversal compared to Z-order curves [S1, unofficial].

## How it works
**Inputs:**
- Volumetric data array $V$ of dimensions $D_x \times D_y \times D_z$ [G].
- GPU context supporting 3D texture objects [G].

**Outputs:**
- A GPU-accessible 3D texture resource $T$ [G].
- Sampled values $V(\mathbf{p})$ for any query point $\mathbf{p}$ within the domain [G].

**Data Structures:**
- **3D Texture Object:** A hardware resource defined by width $W$, height $H$, and depth $D$ [G].
- **Internal Format:** Determines the number of channels and bit depth (e.g., `GL_RGB`, `GL_R16UI`, `GL_R8UI`) [S3, unofficial][S4, unofficial].

**Steps:**

1.  **Texture Allocation:** Create a 3D texture object on the GPU with dimensions matching the volume resolution $(W, H, D)$ [S4, unofficial].
2.  **Data Upload:** Transfer the host-side volumetric data array to the GPU texture memory. The data is typically stored in row-major order, but the GPU hardware manages the physical memory layout for optimal locality [S1, unofficial].
3.  **Parameter Configuration:** Set texture parameters:
    -   **Filtering:** Enable `GL_LINEAR` (trilinear interpolation) for smooth sampling [S3, unofficial].
    -   **Wrapping:** Set to `GL_CLAMP_TO_EDGE` to prevent sampling outside the volume bounds [G].
    -   **Mipmapping:** Typically disabled for volume rendering to avoid blurring during ray marching [G].
4.  **Sampling:** During rendering, the fragment shader or compute shader samples the texture at normalized coordinates $(u, v, w) \in [0, 1]^3$. The hardware performs trilinear interpolation:
    $$ V(\mathbf{p}) = \sum_{i,j,k \in \{0,1\}} w_i w_j w_k V_{ijk} $$
    where $w_i, w_j, w_k$ are the interpolation weights derived from the fractional part of the query position [G].

**Parameters:**
-   **Format:** `GL_RGB` for color data, `GL_R8UI`/`GL_R16UI` for scalar fields [S3, unofficial][S4, unofficial].
-   **Resolution:** Not given in the sources.

**Edge Cases and Pitfalls:**
-   **Boundary Artifacts:** Without `CLAMP_TO_EDGE`, wrapping modes can cause data leakage from opposite sides of the volume [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def allocate_3d_texture(width, height, depth, internal_format):
    """Simulates GPU texture allocation and parameter setup."""
    texture = {
        'dimensions': (width, height, depth),
        'internal_format': internal_format,
        'filter_min': 'LINEAR',
        'filter_mag': 'LINEAR',
        'wrap_s': 'CLAMP_TO_EDGE',
        'wrap_t': 'CLAMP_TO_EDGE',
        'wrap_r': 'CLAMP_TO_EDGE',
        'data': None
    }
    return texture

def upload_data(texture, host_array):
    """Transfers host-side volumetric data to the texture buffer."""
    # Assumes host_array is contiguous and matches the expected layout
    texture['data'] = host_array.copy()
    return texture

def sample_3d_texture(texture, u, v, w):
    """Samples the texture at normalized coordinates [0, 1]^3 using trilinear interpolation."""
    w_dim, h_dim, d_dim = texture['dimensions']
    # Convert normalized coordinates to voxel indices
    x = u * (w_dim - 1)
    y = v * (h_dim - 1)
    z = w * (d_dim - 1)
    
    # Clamp to valid range to handle boundary cases
    x = max(0.0, min(x, w_dim - 1))
    y = max(0.0, min(y, h_dim - 1))
    z = max(0.0, min(z, d_dim - 1))
    
    # Get integer and fractional parts
    x0, y0, z0 = int(x), int(y), int(z)
    dx, dy, dz = x - x0, y - y0, z - z0
    
    # Handle boundary indices for interpolation neighbors
    x1 = min(x0 + 1, w_dim - 1)
    y1 = min(y0 + 1, h_dim - 1)
    z1 = min(z0 + 1, d_dim - 1)
    
    data = texture['data']
    # Extract 8 neighboring voxels
    v000 = data[z0, y0, x0]
    v001 = data[z0, y0, x1]
    v010 = data[z0, y1, x0]
    v011 = data[z0, y1, x1]
    v100 = data[z1, y0, x0]
    v101 = data[z1, y0, x1]
    v110 = data[z1, y1, x0]
    v111 = data[z1, y1, x1]
    
    # Trilinear interpolation weights
    wx, wy, wz = dx, dy, dz
    
    # Interpolate along x
    v00 = v000 * (1 - wx) + v001 * wx
    v01 = v010 * (1 - wx) + v011 * wx
    v10 = v100 * (1 - wx) + v101 * wx
    v11 = v110 * (1 - wx) + v111 * wx
    
    # Interpolate along y
    v0 = v00 * (1 - wy) + v01 * wy
    v1 = v10 * (1 - wy) + v11 * wy
    
    # Interpolate along z
    return v0 * (1 - wz) + v1 * wz
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
-   **Memory Complexity:** $O(N)$ where $N$ is the total number of voxels, stored contiguously in GPU VRAM [G].
-   **Performance:** Storing voxels in a 3D texture can yield a 10x improvement over linear array storage (e.g., 12 FPS to 121 FPS at $2048^3$ resolution) due to optimized data locality and cache coherence [S1, unofficial].
-   **Strengths:** High performance for dense volumes; hardware-accelerated trilinear interpolation; simple API usage [S1, unofficial][S3, unofficial].
-   **Limitations:** Not given in the sources.
-   **When to Use:** Dense volumetric datasets that fit within GPU texture limits; real-time rendering applications requiring high throughput [S1, unofficial].
-   **When Not to Use:** Sparse datasets; volumes exceeding maximum texture dimensions; scenarios requiring complex data structures like [VDB](4880-vdb-data-loading-3d-texture-setup.md) [G].

## Background
The method evolved as a solution to the memory bandwidth bottlenecks of linear array storage, leveraging the GPU's specialized texture units for data locality [S1, unofficial].

## Sources
[S1] This Tiny Algorithm Can Render BILLIONS of Voxels in Real Time, https://www.youtube.com/watch?v=ztkh1r1ioZo, fetched 2026-10-25 (unofficial)
[S2] Example-based volume illustrations, https://ieeexplore.ieee.org/abstract/document/1532854/, fetched 2026-10-31 (unofficial)
[S3] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
[S4] Winson-L/VolumeSegmenter, https://github.com/Winson-L/VolumeSegmenter, fetched 2026-10-27 (unofficial)
