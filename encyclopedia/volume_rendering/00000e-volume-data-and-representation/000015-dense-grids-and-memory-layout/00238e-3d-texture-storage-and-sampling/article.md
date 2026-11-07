# 3D Texture Storage and Sampling

Also called: 3D image storage, GPU 3D texture mapping

## Overview
3D Texture Storage and Sampling maps a discrete volumetric field $f(\mathbf{x})$ to GPU hardware-accelerated 3D texture resources, enabling efficient spatial locality and hardware tri-linear interpolation for volume rendering [S1, S2, unofficial]. It solves the problem of memory bandwidth bottlenecks in voxel ray traversal by leveraging GPU-specific memory structures that are more sophisticated than software-optimized Z-order curves [S1, unofficial]. Unlike general dense grid layouts that may require manual cache management or complex indexing, this method offloads data locality and interpolation to the graphics hardware, often yielding significant performance improvements (e.g., 10x FPS gains) without altering the traversal algorithm itself [S1, unofficial].

## How it works
**Inputs:**
*   Volumetric data field $V$ (scalar values, gradients, or pre-computed color/opacity).
*   Grid dimensions $(N_x, N_y, N_z)$.
*   Optional metadata: segmentation masks, ambient occlusion factors, or bent normals.

**Outputs:**
*   A GPU 3D texture object (or texture array) containing the voxel data.
*   Texture coordinates $(u, v, w)$ normalized to $[0, 1]$ for sampling.

**Data Structures:**
*   **3D Texture Resource:** A hardware-managed memory block. The logical layout maps 3D indices $(i, j, k)$ to physical memory addresses. The exact physical layout (e.g., row-major, tiled, or hardware-specific interleaving) is abstracted by the GPU driver but is optimized for spatial coherence [S1, S2, unofficial].
*   **Cube Tiling Layout (Specific Variant):** In example-based illustration systems, cube textures are grouped into a grid of size $4C \times 4C \times MC$, where $M$ is the number of cube sets and $C$ is the cube side length. Each cube is stored as a sub-volume to support tri-linear interpolation [S2, unofficial].

**Steps:**

1.  **Allocation and Upload:** Allocate a 3D texture object on the GPU with dimensions matching the volume resolution (or a power-of-two approximation). Upload the voxel data from host memory to the GPU texture memory [S2, S4, S5, S6, unofficial].
2.  **Coordinate Normalization:** For a query point $\mathbf{p}$ in world space, transform it to texture space coordinates $(u, v, w) \in [0, 1]^3$. This typically involves subtracting the volume origin and dividing by the volume dimensions, accounting for any specific texture coordinate system (e.g., OpenGL vs. DirectX conventions) [G].
3.  **Sampling:** Use the GPU hardware sampler to retrieve the value at $(u, v, w)$. The hardware performs tri-linear interpolation between the 8 surrounding voxels using the texture filtering mode (e.g., `GL_LINEAR`) [S2, S3, unofficial].
    *   Formula for tri-linear interpolation $V(\mathbf{p})$:
        $$ V(\mathbf{p}) = \sum_{i=0}^{1} \sum_{j=0}^{1} \sum_{k=0}^{1} w_i w_j w_k V_{ijk} $$
        where $w_i, w_j, w_k$ are the fractional distances to the grid nodes, and $V_{ijk}$ are the voxel values at the corners of the cell containing $\mathbf{p}$ [S1, S3, unofficial].
4.  **Advanced Coordinate Calculation (Cube Tiling Variant):** In systems using cube tiling for illustration:
    *   Retrieve the render scale $S$ for the current object using the segmentation mask [S2, unofficial].
    *   Calculate the cube index from the cube tiling using the voxel center position and $S$ [S2, unofficial].
    *   Calculate the relative position within the cube using the world position and $S$ [S2, unofficial].
    *   Retrieve the color from the cube texture using the relative position [S2, unofficial].
5.  **Post-Sampling Processing:** Apply transfer functions, lighting effects (e.g., silhouette enhancement), or composite the sampled values to produce the final fragment color [S2, unofficial].

**Parameters:**
*   **Filtering Mode:** `GL_LINEAR` (tri-linear) is standard for smooth interpolation. `GL_NEAREST` is used for sharp, discrete voxel rendering [S1, S3, unofficial].
*   **Render Scale:** A parameter for each cube set corresponding to the voxel numbers in each direction, used to scale cube textures for visibility [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Sampling near the volume boundaries requires correct handling of texture wrap modes (e.g., `GL_CLAMP_TO_EDGE`) to avoid reading invalid memory or wrapping to the opposite side [G].
*   **Resolution Limits:** Standard 3D textures have maximum dimension limits (e.g., 2048 voxels per dimension on older hardware). Large volumes may require sparse techniques or texture arrays [S1, unofficial].
*   **Precision:** The precision of the stored texture format (e.g., 8-bit, 16-bit float, 32-bit float) affects dynamic range and quantization error [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for standard 3D texture sampling
def sample_3d_texture(texture, volume_origin, volume_dimensions, world_point):
    """
    Samples a 3D texture at a given world point.
    
    Args:
        texture: GPU 3D texture object
        volume_origin: (x0, y0, z0) world coordinates of the volume start
        volume_dimensions: (Nx, Ny, Nz) number of voxels in each direction
        world_point: (x, y, z) world coordinates to sample
        
    Returns:
        Interpolated value from the texture
    """
    # 1. Transform world coordinates to normalized texture coordinates [0, 1]
    # Note: Exact mapping depends on API (OpenGL vs DirectX). 
    # Assuming OpenGL convention where (0,0,0) is the corner.
    u = (world_point.x - volume_origin.x) / volume_dimensions.x
    v = (world_point.y - volume_origin.y) / volume_dimensions.y
    w = (world_point.z - volume_origin.z) / volume_dimensions.z
    
    # 2. Clamp coordinates to [0, 1] to handle boundary conditions
    u = clamp(u, 0.0, 1.0)
    v = clamp(v, 0.0, 1.0)
    w = clamp(w, 0.0, 1.0)
    
    # 3. Hardware tri-linear interpolation
    # The actual sampling is performed by the GPU hardware sampler
    # value = texture3D(texture, vec3(u, v, w))
    value = hardware_texture_sample_3d(texture, u, v, w)
    
    return value

# Pseudocode for Cube Tiling Variant (from S2)
def sample_cube_tiling_texture(texture_tiling, texture_cubes, segmentation_mask, world_point, render_scale):
    """
    Samples a cube tiling texture for volume illustration.
    
    Args:
        texture_tiling: 3D texture containing the cube grid layout
        texture_cubes: 3D texture containing the cube data volumes
        segmentation_mask: Mask to determine object type and render scale
        world_point: (x, y, z) world coordinates
        render_scale: Scale factor for the current object
        
    Returns:
        Color value from the cube texture
    """
    # 1. Retrieve render scale from segmentation mask
    # (Assuming mask provides a lookup for the scale)
    scale = segmentation_mask.get_scale(world_point)
    
    # 2. Calculate cube index from cube tiling
    # Using voxel center position and render scale
    cube_index = calculate_cube_index(world_point, scale)
    
    # 3. Calculate relative position within the cube
    relative_pos = calculate_relative_position(world_point, cube_index, scale)
    
    # 4. Retrieve color from the cube texture
    # Using relative position and the cube index to locate the sub-volume
    color = hardware_texture_sample_3d(texture_cubes, relative_pos)
    
    return color
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
*   **Time Complexity:** Sampling is $O(1)$ per query, performed by hardware. The cost is dominated by memory bandwidth and cache hit rates [S1, unofficial].
*   **Memory Complexity:** $O(N_x N_y N_z)$ for dense storage. The physical memory footprint depends on the texture format (e.g., 1 byte per voxel for 8-bit, 4 bytes for 32-bit float) [S1, S2, unofficial].
*   **Measured Performance:** Storing voxels in a 3D texture on the GPU can achieve 121 FPS at a $2048^3$ grid resolution and $1000 \times 1000$ screen resolution, a 10x improvement over the original 12 FPS achieved with Z-order curves [S1, unofficial].
*   **Strengths:** Hardware-accelerated tri-linear interpolation, optimized data locality, reduced memory bandwidth usage due to cache efficiency [S1, unofficial].
*   **Limitations:** Maximum texture dimension limits on some hardware, potential precision loss with lower-bit formats, less flexibility for irregular grids compared to sparse structures [S1, S2, unofficial].
*   **When to Use:** Real-time volume rendering on GPUs, especially when the volume fits within texture dimension limits and hardware interpolation is desired [S1, S2, unofficial].
*   **When Not to Use:** When the volume exceeds hardware texture limits and sparse techniques are required, or when non-trilinear interpolation is needed [S1, unofficial].

## Background
The use of 3D textures for volume rendering on GPUs became prominent with the advent of hardware supporting 3D texture sampling and tri-linear interpolation [S1, S2, unofficial]. Early implementations focused on direct volume rendering using slicing techniques, where 3D textures were used to store volume data and segmentation masks [S2, unofficial]. The method has been further refined to support advanced features such as example-based illustrations, ambient occlusion baking, and sparse volume representations [S1, S2, S3, unofficial].

## Sources
[S1] This Tiny Algorithm Can Render BILLIONS of Voxels in Real Time, https://www.youtube.com/watch?v=ztkh1r1ioZo, fetched 2026-10-25 (unofficial)
[S2] Example-based volume illustrations, https://ieeexplore.ieee.org/abstract/document/1532854/, fetched 2026-10-31 (unofficial)
[S3] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
