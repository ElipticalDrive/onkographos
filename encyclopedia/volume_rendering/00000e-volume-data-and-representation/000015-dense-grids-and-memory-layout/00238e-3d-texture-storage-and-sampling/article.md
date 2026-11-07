# 3D Texture Storage and Sampling

Also called: GPU 3D Texture Volume Storage, Hardware 3D Texture Sampling

## Overview
3D Texture Storage and Sampling is a method for storing volumetric data in GPU 3D texture resources and sampling them using hardware-accelerated tri-linear interpolation [S1, S2, unofficial]. It solves the problem of efficient memory access patterns for volume rendering by leveraging GPU-specific hardware optimizations that provide superior data locality compared to software-managed layouts like Z-order curves [S1, unofficial]. Unlike [Bricking](914-bricking.md) which manages memory hierarchies explicitly, this method relies on the graphics API and hardware to handle caching and filtering, offering significant performance gains for dense grids [S1, unofficial].

## How it works
**Inputs:**
*   Volumetric data (scalar fields, RGB values, or multi-channel attributes).
*   Grid dimensions $(X, Y, Z)$.
*   Texture format (e.g., `GL_RGBA32F`, `GL_R8`).

**Outputs:**
*   A GPU 3D texture object bound to a texture unit.
*   Sampled values $V(\mathbf{x})$ at arbitrary 3D coordinates $\mathbf{x} \in [0,1]^3$ via hardware interpolation.

**Data Structures:**
The data is stored in a 3D texture object. The logical layout maps 3D indices $(i, j, k)$ to texture coordinates $(u, v, w)$.
*   **Dimensions:** The texture dimensions $D_u, D_v, D_w$ correspond to the grid dimensions $X, Y, Z$.
*   **Format:** Each texel stores one or more channels (e.g., 1 for density, 4 for RGBA).
*   **Filtering:** Tri-linear interpolation is enabled. This hardware feature computes the weighted average of the 8 surrounding voxels based on the fractional part of the texture coordinates [S2, unofficial].

**Steps:**
1.  **Allocation:** Allocate a 3D texture on the GPU with dimensions matching the volume resolution. Set filtering mode to `LINEAR` (tri-linear) [S2, unofficial].
2.  **Upload:** Transfer the host-side voxel data to the GPU texture memory using API calls (e.g., `glTexImage3D`) [S2, S4, S7, unofficial].
3.  **Sampling:** In the shader, compute normalized texture coordinates $(u, v, w)$ from the world position $\mathbf{x}$ and the volume's origin and scale.
    $$ u = \frac{x - x_0}{\Delta x \cdot X}, \quad v = \frac{y - y_0}{\Delta y \cdot Y}, \quad w = \frac{z - z_0}{\Delta z \cdot Z} $$
4.  **Fetch:** Sample the texture using the hardware tri-linear interpolation kernel:
    $$ V(P) = \sum_{i,j,k \in \{0,1\}} w_i w_j w_k V_{ijk} $$
    where $w_i, w_j, w_k$ are the fractional distances to the grid nodes [G].

**Parameters:**
*   **Filtering Mode:** `LINEAR` for tri-linear interpolation, `NEAREST` for point sampling [G].
*   **Wrapping Mode:** Typically `CLAMP_TO_EDGE` to prevent wrapping artifacts at volume boundaries [G].

**Edge Cases and Pitfalls:**
*   **Resolution Limits:** GPUs have maximum 3D texture dimensions (e.g., 2048 or 4096). Volumes exceeding these limits require [Bricking](914-bricking.md) or [Sparse Volume Texture](7011-sparse-volume-texture-svt.md) [S1, unofficial].
*   **Precision:** 8-bit textures may introduce banding in smooth gradients; 16-bit or 32-bit floats are preferred for scientific data [G].
*   **Coordinate System:** Ensure the texture coordinate system matches the shader's expectation (e.g., OpenGL vs. DirectX Y-axis direction) [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for setting up and sampling a 3D texture volume
# Assumes a graphics API context is initialized

def setup_volume_texture(grid_dims, data):
    """
    Allocates and uploads volumetric data to a GPU 3D texture.
    grid_dims: tuple (X, Y, Z)
    data: numpy array of shape (Z, Y, X, channels) or similar, depending on API
    """
    # 1. Generate texture ID
    tex_id = create_3d_texture()
    
    # 2. Set parameters
    set_texture_parameter(tex_id, FILTER, LINEAR) # Tri-linear interpolation
    set_texture_parameter(tex_id, WRAP_S, CLAMP_TO_EDGE)
    set_texture_parameter(tex_id, WRAP_T, CLAMP_TO_EDGE)
    set_texture_parameter(tex_id, WRAP_R, CLAMP_TO_EDGE)
    
    # 3. Upload data
    # Note: Data layout (row-major vs column-major) depends on the specific API
    upload_3d_texture_data(tex_id, grid_dims, data)
    
    return tex_id

def sample_volume(tex_id, world_pos, volume_origin, voxel_size):
    """
    Samples the volume at a given world position.
    """
    # 1. Convert world position to normalized texture coordinates [0, 1]
    u = (world_pos.x - volume_origin.x) / (voxel_size.x * grid_dims.x)
    v = (world_pos.y - volume_origin.y) / (voxel_size.y * grid_dims.y)
    w = (world_pos.z - volume_origin.z) / (voxel_size.z * grid_dims.z)
    
    # 2. Clamp coordinates to prevent wrapping (if not handled by API)
    u = clamp(u, 0.0, 1.0)
    v = clamp(v, 0.0, 1.0)
    w = clamp(w, 0.0, 1.0)
    
    # 3. Hardware tri-linear interpolation
    # This is performed by the GPU hardware when sampling with LINEAR filter
    value = texture_3d(tex_id, u, v, w)
    
    return value
```

## Subtopics

### [3D Texture Proxy Geometry](870-3d-texture-proxy-geometry.md)
A direct volume rendering method that represents a rectilinear volume as a stack of parallel planar polygons (slabs) textured with 3D texture maps, leveraging hardware-accelerated tri-linear interpolation to sample volume data at arbitrary locations. It solves the problem of efficient hardware-accelerated volume rendering with directional lighting by pre-computing color and opacity per voxel and storing them in 3D textures, allowing the graphics pipeline to handle interpolation and compositing. Unlike shear-warp methods which fix plane counts to volume resolution, this method allows the number of rendering planes to be adjusted at runtime independent of volume resolution, and unlike coherent projection, it supports perspective views without a time penalty, though perspective projection causes irregular sampling along rays.

### [Volume Texture Creation](1857-volume-texture-creation.md)
Volume Texture Creation allocates GPU memory and populates it with volumetric data (scalar fields, gradients, transfer functions) and rendering targets to enable hardware-accelerated raycasting. It solves the problem of transferring host-side volume representations into GPU-accessible texture objects with specific memory layouts, filtering modes, and coordinate systems required by volume rendering shaders. It differs from [3D Texture Storage and Sampling](3064-3d-texture-storage-and-sampling.md) by focusing on the API-level allocation and data upload steps rather than the sampling logic.

### [Texture Uniting (Multi-Channel Packing)](2470-texture-uniting-multi-channel-packing.md)
Texture Uniting is a technique to combine four separate 3D textures (noise, detail, light, mask) into a single RGBA 3D texture to optimize memory bandwidth and sampling. It solves the problem of excessive GPU memory consumption and redundant sampling overhead when rendering high-resolution volumes with multiple attribute channels by avoiding the duplication of raw voxel data and enabling simultaneous access to related attributes. Unlike [3D Volume Atlas Packing](7158-3d-volume-atlas-packing.md), which arranges distinct datasets in a spatial atlas to minimize texture binding switches, this method interleaves attributes into a single volume's color channels to optimize bandwidth for simultaneous access.

### [PixelFlow Volume Rendering](3443-pixelflow-volume-rendering.md)
PixelFlow Volume Rendering is a polygon resampling method designed for the PixelFlow graphics system that overcomes the lighting and data size limitations of the RealityEngine by allowing pixel processors to manipulate 3D texture values directly. It differs from its siblings by replacing precomputed shading coefficients with on-the-fly Phong lighting calculations derived from interpolated volume gradients, enabling general image-space lighting.

### [Texture Array Sampling with Alpha Discard](3993-texture-array-sampling-with-alpha-discard.md)
This method implements a fragment shader that samples a 3D texture array to retrieve voxel color and opacity, discarding fragments with alpha below a specific threshold to optimize rendering performance by avoiding overdraw for transparent regions. It differs from proxy geometry approaches by relying entirely on hardware texture sampling and early-Z rejection, rather than generating geometry for occupied voxels.

### [Hardware Volume Rendering Pipeline with Pixel Textures](5641-hardware-volume-rendering-pipeline-with-pixel-textures.md)
This method implements volume rendering on hardware that supports the OpenGL pixel texture (dependent texture) extension, specifically targeting architectures like the SGI Octane 2 with V-series graphics cards. It solves the problem of applying multi-dimensional transfer functions (classification and shading) by using color fragments from a rendered slice to generate texture coordinates for a 3D lookup texture, effectively replacing the fragment color with the result of the transfer function evaluation. It differs from per-fragment dependent texture reads (e.g., GeForce3) by requiring an auxiliary scratch buffer for intermediate storage, as the pixel texture operation does not occur directly during rasterization on the target hardware.

### [Sparse Volume Texture (SVT)](7011-sparse-volume-texture-svt.md)
Sparse Volume Texture (SVT) is a technique that trades rendering speed for data compression by chunking voxel data into 3D tiles and storing only non-empty tiles in a physical texture, using a second volume texture as a page table for lookup. It solves the problem of rendering large volume datasets that exceed the maximum dimension limits of standard 3D textures (e.g., 2048 voxels per dimension) while managing memory footprint. Unlike dense grids, SVT circumvents API limits by mapping a large logical coordinate space onto a smaller physical texture containing only active tiles, though it incurs overhead from padding and mipmap hierarchies.

### [Splat Texture Layout & Mapping](7089-splat-texture-layout-mapping.md)
Splat Texture Layout & Mapping organizes per-primitive splat data into 3D texture structures to enable efficient GPU-based rendering of Gaussian splats. It solves the problem of mapping arbitrary numbers of splats into a fixed-size hardware texture format by calculating specific texture coordinates and managing memory alignment. It sets itself apart from proxy geometry methods by keeping data in texture memory for hardware filtering, and from sparse volume textures by using a dense, pre-allocated 3D texture layout for the splat attributes themselves.

### [Memory Organization and Layout](7154-memory-organization-and-layout.md)
This article defines the memory organization strategies for storing dense voxel grids in GPU texture resources, focusing on how linear voxel coordinates map to physical memory addresses to optimize cache locality and bandwidth. It distinguishes between standard row-major storage, which relies on hardware caching for spatial coherence, and hardware-specific interleaving schemes designed to ensure that all voxels required for a single tri-linear interpolation fetch reside in distinct memory modules, allowing the entire cell to be retrieved in a single memory transaction.

### [3D Volume Atlas Packing](7158-3d-volume-atlas-packing.md)
3D Volume Atlas Packing consolidates multiple discrete volumes (such as animation timesteps or separate datasets) into a single contiguous 3D texture resource. This method solves the problem of managing numerous individual texture objects by reducing GPU state changes and memory fragmentation. It differs from [Texture Uniting (Multi-Channel Packing)](2470-texture-uniting-multi-channel-packing.md), which packs different data channels into one volume, by packing identical channel structures across the depth dimension. It builds on [3D Texture Storage and Sampling](3064-3d-texture-storage-and-sampling.md) by defining the specific layout and allocation strategy for these multi-volume containers.

### Ambient Occlusion Volume
A storage method where ambient occlusion factors are stored directly in a 3D texture, optionally alongside bent normals. This avoids runtime reconstruction computation. Data structure: 3D texture storing ambient occlusion factors. Optional field: bent normals. Differs from Kontkanen and Laine's approach by storing precomputed factors directly, eliminating the need for reconstruction and reducing runtime cost [S3, unofficial].

## Operating characteristics
**Time Complexity:** Sampling is $O(1)$ per voxel access due to hardware interpolation [S1, unofficial].
**Memory Complexity:** $O(X \cdot Y \cdot Z \cdot C)$ where $C$ is the number of channels [G].
**Performance:** Storing voxels in a 3D texture can yield a 10x performance improvement over software-managed Z-order curves, achieving 121 FPS at $2048^3$ resolution and $1000 \times 1000$ screen resolution, compared to 12 FPS [S1, unofficial].
**Strengths:** Hardware-accelerated tri-linear interpolation; optimized memory access patterns; simple API usage [S1, S2, unofficial].
**Limitations:** Limited by maximum 3D texture dimensions; less control over memory layout compared to software structures; potential precision loss with low-bit formats [S1, unofficial].
**When to use:** Dense volumes within GPU texture size limits; real-time rendering requiring smooth interpolation [S1, S2, unofficial].
**When not to use:** Volumes exceeding texture size limits; sparse volumes where memory efficiency is critical [S1, unofficial].

## Background
Originating with the advent of GPU-based volume rendering, this method leverages the graphics hardware's native support for 3D textures. Early implementations used software-managed memory layouts, but the shift to 3D textures provided significant performance benefits due to hardware caching and interpolation [S1, unofficial]. It is widely used in medical imaging, scientific visualization, and real-time rendering engines [S2, unofficial].

## Sources
[S1] This Tiny Algorithm Can Render BILLIONS of Voxels in Real Time, https://www.youtube.com/watch?v=ztkh1r1ioZo, fetched 2026-10-25 (unofficial)
[S2] Example-based volume illustrations, https://ieeexplore.ieee.org/abstract/document/1532854/, fetched 2026-10-31 (unofficial)
[S3] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
