# 3D Texture Storage and Sampling

Also called: 3D image storage, GPU 3D texture mapping

## Overview
3D Texture Storage and Sampling maps a discrete volumetric field $f(\mathbf{x})$ to GPU hardware-accelerated 3D texture resources, enabling efficient spatial queries via hardware tri-linear interpolation [S1, S2, unofficial]. It solves the problem of memory bandwidth bottlenecks in volume ray marching by leveraging GPU-specific cache optimizations that are superior to software-managed layouts like Z-order curves [S1, unofficial]. Unlike [Bricking](914-bricking.md) which manages memory tiling explicitly, this method relies on the graphics driver to handle locality, trading explicit control for significant performance gains in dense grid rendering [S1, unofficial].

## How it works
**Inputs:**
*   Volumetric data array $V$ of dimensions $D_x \times D_y \times D_z$.
*   Data type (e.g., 8-bit unsigned integer, 16-bit float, 32-bit float) [S1, unofficial].
*   Coordinate system parameters (world-to-texture transformation) [G].

**Outputs:**
*   GPU 3D texture object $T_{3D}$ containing the voxel data.
*   Interpolated scalar or vector values $V(P)$ at arbitrary query points $P$ [S2, unofficial].

**Data Structures:**
The data is stored in a hardware 3D texture object. The logical layout is a regular grid. While the internal memory layout is managed by the GPU driver (often using sophisticated locality optimizations beyond simple Z-order), the logical access pattern assumes a 3D coordinate space $(u, v, w)$ normalized to $[0, 1]$ or integer indices $[0, D_x-1], [0, D_y-1], [0, D_z-1]$ [S1, S2, unofficial].

**Steps:**

1.  **Texture Allocation:** Allocate a 3D texture resource on the GPU with dimensions matching the volume resolution $(D_x, D_y, D_z)$ and the appropriate internal format (e.g., `GL_RGBA32F` for high precision, `GL_R8` for scalar fields) [S1, S2, unofficial].
2.  **Data Upload:** Transfer the host-side voxel array $V$ to the GPU texture memory. This step maps the linear memory buffer to the 3D texture storage [S2, unofficial].
3.  **Coordinate Normalization:** For a query point $\mathbf{x}$ in world space, compute the normalized texture coordinates $(u, v, w)$:
    $$
    u = \frac{x - x_{min}}{x_{max} - x_{min}}, \quad v = \frac{y - y_{min}}{y_{max} - y_{min}}, \quad w = \frac{z - z_{min}}{z_{max} - z_{min}}
    $$
    where $[x_{min}, x_{max}]$ etc. define the volume bounds [G].
4.  **Hardware Sampling:** Invoke the GPU texture sampler with tri-linear interpolation enabled. The hardware calculates the weighted average of the 8 neighboring voxels based on the fractional part of $(u, v, w)$ [S2, unofficial].
    $$
    V(P) = \text{tex3D}(T_{3D}, u, v, w)
    $$
5.  **Boundary Handling:** Define behavior for coordinates outside $[0, 1]$. Common modes are `CLAMP_TO_EDGE` (repeating the boundary value) or `REPEAT` [G].

**Parameters:**
*   **Filtering Mode:** Tri-linear interpolation is the standard for smooth rendering. Nearest-neighbor is used for sharp, voxel-art styles [S1, S2, unofficial].
*   **Mipmap Generation:** Optional. If enabled, lower-resolution versions of the texture are generated for distant sampling, reducing aliasing and memory bandwidth [G].

**Edge Cases and Pitfalls:**
*   **Resolution Limits:** Standard 3D textures have maximum dimension limits (e.g., 2048 or 4096 voxels per side depending on hardware). Volumes exceeding these limits require [Sparse Volume Texture (SVT)](7011-sparse-volume-texture-svt.md) or [Bricking](914-bricking.md) [S1, unofficial].
*   **Precision:** 8-bit storage may introduce banding artifacts in smooth gradients. 16-bit or 32-bit floats are preferred for scientific visualization [S1, unofficial].
*   **Coordinate System:** OpenGL textures typically use $(s, t, r)$ or $(u, v, w)$ with $z$ pointing towards the viewer, while DirectX may differ. Mismatched coordinate systems cause inverted volumes [G].

## Subtopics

### [3D Texture Proxy Geometry](870-3d-texture-proxy-geometry.md)
Represents a rectilinear volume as a stack of parallel planar polygons (slabs) textured with 3D texture maps, leveraging hardware-accelerated tri-linear interpolation. It allows the number of rendering planes to be adjusted at runtime independent of volume resolution and supports perspective views without a time penalty, though perspective projection causes irregular sampling along rays.

### [Volume Texture Creation](1857-volume-texture-creation.md)
Focuses on the API-level allocation and data upload steps to populate GPU memory with volumetric data (scalar fields, gradients, transfer functions) and rendering targets. It differs from the sampling logic by handling resource generation rather than query execution.

### [Texture Uniting (Multi-Channel Packing)](2470-texture-uniting-multi-channel-packing.md)
Combines four separate 3D textures (noise, detail, light, mask) into a single RGBA 3D texture to optimize memory bandwidth and sampling. It interleaves attributes into a single volume's color channels to optimize bandwidth for simultaneous access, unlike [3D Volume Atlas Packing](7158-3d-volume-atlas-packing.md) which arranges distinct datasets spatially.

### [PixelFlow Volume Rendering](3443-pixelflow-volume-rendering.md)
A polygon resampling method for the PixelFlow graphics system that allows pixel processors to manipulate 3D texture values directly. It replaces precomputed shading coefficients with on-the-fly Phong lighting calculations derived from interpolated volume gradients, enabling general image-space lighting.

### [Texture Array Sampling with Alpha Discard](3993-texture-array-sampling-with-alpha-discard.md)
Implements a fragment shader that samples a 3D texture array to retrieve voxel color and opacity, discarding fragments with alpha below a specific threshold to optimize rendering performance by avoiding overdraw for transparent regions. It relies on hardware texture sampling and early-Z rejection rather than generating geometry.

### [Hardware Volume Rendering Pipeline with Pixel Textures](5641-hardware-volume-rendering-pipeline-with-pixel-textures.md)
Implements volume rendering on hardware supporting the OpenGL pixel texture (dependent texture) extension, targeting architectures like the SGI Octane 2. It uses color fragments from a rendered slice to generate texture coordinates for a 3D lookup texture, effectively replacing the fragment color with the result of the transfer function evaluation. It requires an auxiliary scratch buffer for intermediate storage.

### [3D Transfer Function Texture Layout](5646-3d-transfer-function-texture-layout.md)
Defines the memory layout for hardware-based volume rendering using multi-dimensional transfer functions (MTFs). It addresses the need to pre-compute classification variables (such as gradient magnitude) at each voxel sample point, imposing specific alignment constraints on the texture memory layout to support auxiliary data volumes like normals.

### [Sparse Volume Texture (SVT)](7011-sparse-volume-texture-svt.md)
Trades rendering speed for data compression by chunking voxel data into 3D tiles and storing only non-empty tiles in a physical texture, using a second volume texture as a page table for lookup. It circumvents API limits on maximum texture dimensions by mapping a large logical coordinate space onto a smaller physical texture, though it incurs overhead from padding and mipmap hierarchies.

### [Splat Texture Layout & Mapping](7089-splat-texture-layout-mapping.md)
Stores and accesses 3D Gaussian splat data within GPU 3D textures by mapping an unstructured set of splats into a fixed-size, hardware-sampled 3D texture volume. It uses a dense, pre-allocated texture structure with specific bit-packing for coordinates and layer indices, differing from sparse volume textures by its dense allocation.

### [Memory Organization and Layout](7154-memory-organization-and-layout.md)
Defines how dense voxel grids are mapped to GPU memory resources to optimize cache locality and bandwidth during ray traversal. It distinguishes between linear byte buffers and hardware-accelerated 3D texture resources, detailing specific interleaving schemes that align memory accesses with hardware interpolation requirements.

### [3D Volume Atlas Packing](7158-3d-volume-atlas-packing.md)
Consolidates multiple distinct 3D volumes (such as timesteps in a time-series dataset) into a single 3D texture resource. It solves the problem of limited texture unit availability by spatially arranging volumes within a larger 3D grid, differing from [3D Texture Proxy Geometry](870-3d-texture-proxy-geometry.md) by storing actual voxel data rather than geometric approximations.

### Ambient Occlusion Volume
A storage method where ambient occlusion factors are stored directly in a 3D texture, optionally alongside bent normals [S3, unofficial]. This avoids runtime reconstruction computation, differing from Kontkanen and Laine's approach by storing precomputed factors directly, eliminating the need for reconstruction and reducing runtime cost [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Sampling is $O(1)$ per query, performed in hardware. The cost is dominated by memory bandwidth and cache hit rates [S1, unofficial].
*   **Memory Complexity:** $O(N)$ where $N$ is the number of voxels, plus overhead for mipmaps if generated. The memory footprint is fixed by the maximum texture dimensions [S1, unofficial].
*   **Measured Performance:** Storing voxels in a 3D image brings performance to 121 FPS at a $2048^3$ grid resolution and $1000 \times 1000$ screen resolution, a 10x improvement over the original 12 FPS achieved with Z-order curves [S1, unofficial].
*   **Strengths:** High data locality due to GPU-specific cache optimizations; hardware-accelerated tri-linear interpolation; simple API usage [S1, S2, unofficial].
*   **Limitations:** Limited by maximum texture dimensions (e.g., 2048/4096 voxels per side); less control over memory layout compared to software-managed structures like [Bricking](914-bricking.md) [S1, unofficial].
*   **When to Use:** Ideal for dense volumes that fit within hardware texture limits and require smooth interpolation.
*   **When Not to Use:** For volumes exceeding texture dimension limits (use [Sparse Volume Texture (SVT)](7011-sparse-volume-texture-svt.md)) or when explicit memory layout control is required for non-standard traversal patterns [S1, unofficial].

## Background
The use of 3D textures for volume rendering emerged with the advent of hardware-accelerated graphics pipelines supporting 3D texture mapping. Early implementations utilized 3D textures to store voxel data, leveraging the hardware's built-in interpolation capabilities [S1, S2, unofficial]. The approach gained prominence as a solution to the memory bandwidth bottlenecks of software-managed voxel traversal, demonstrating significant performance improvements over Z-order curve optimizations [S1, unofficial]. It is widely used in real-time volume rendering applications, medical visualization, and scientific computing where hardware acceleration is critical [S1, S2, unofficial].

## Sources
[S1] This Tiny Algorithm Can Render BILLIONS of Voxels in Real Time, https://www.youtube.com/watch?v=ztkh1r1ioZo, fetched 2026-10-25 (unofficial)
[S2] Example-based volume illustrations, https://ieeexplore.ieee.org/abstract/document/1532854/, fetched 2026-10-31 (unofficial)
[S3] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
