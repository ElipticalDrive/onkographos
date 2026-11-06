# Memory Organization and Layout

Also called: Texture Memory Layout, Voxel Storage Layout

## Overview
This article defines how dense voxel grids are mapped to GPU memory resources to optimize cache locality and bandwidth during ray traversal. It distinguishes between linear byte buffers and hardware-accelerated 3D texture resources, and details specific interleaving schemes that align memory accesses with hardware interpolation requirements.

## Shared principles
The core problem is that ray traversal accesses spatially adjacent voxels, but naive linear storage (row-major or column-major) places spatially adjacent voxels far apart in memory, causing cache misses.

1.  **Resource Type**: Modern GPUs provide `Texture3D` resources that support hardware-accelerated coordinate-based sampling and caching. These are preferred over linear buffers (`ByteAddressBuffer`) for volume rendering because the hardware manages cache lines for 3D spatial locality [S2, unofficial].
2.  **Access Pattern**: The primary access pattern is fetching a $2 \times 2 \times 2$ cell of voxels for tri-linear interpolation. Memory layouts should ensure that these 8 voxels are accessible with minimal bank conflicts or cache line spills.
3.  **Precision Trade-off**: Storage precision (e.g., 16-bit vs 32-bit) affects bandwidth but not necessarily arithmetic precision if intermediate calculations are performed in higher precision [S1, unofficial].

## Subtopics

### 16-Bit Texture Storage
*   **Definition**: Using 16-bit floating-point textures (e.g., `R16F`) for storage to reduce memory bandwidth usage [S1, unofficial].
*   **Mechanism**: Data is stored in 16-bit format but arithmetic operations are performed in 32-bit floating point. Results are truncated upon writing back to the 16-bit texture [S1, unofficial].
*   **Performance**: 16-bit textures yield roughly half the performance penalty (i.e., twice the performance) compared to 32-bit textures, with little visually discernible degradation [S1, unofficial].
*   **When to choose**: When memory bandwidth is the bottleneck and the data range/precision requirements allow for 16-bit storage without significant visual artifacts [S1, unofficial].

### 3D Texture Storage
*   **Definition**: Storing the voxel grid in a `Texture3D` resource instead of a linear byte buffer to improve cache locality [S2, unofficial].
*   **Implementation**:
    *   Resource declaration: `Texture3D<uint> voxels : register(t0, space2);` [S2, unofficial].
    *   Access: `voxels.Load(int4(p, 0)).r` [S2, unofficial].
*   **Comparison**: Compared to `ByteAddressBuffer`, 3D texture reduces GPU walltime by 14.2% and increases FPS by 10.3% on a MacBook Air M2, due to better cache utilization for spatially adjacent data access [S2, unofficial].
*   **When to choose**: When using modern GPU APIs (DirectX 11/12, Vulkan, Metal) and the grid dimensions are known at compile time or upload time [S2, unofficial].

### 8-way Interleaved Texture Memory
*   **Definition**: A memory organization technique that distributes voxels across 8 memory modules to allow fetching any tri-linear cell in one memory access [S3].
*   **Formula**: For a voxel at address $a = [zyx]$, the memory module index $k$ and local index $i$ are:
    $$k = (x \mod 2) + 2 (y \mod 2) + 4 (z \mod 2)$$
    $$i = \lfloor x/2 \rfloor + 2 \lfloor y/2 \rfloor + 4 \lfloor z/2 \rfloor$$
    [S3].
*   **Parameter**: The number of memory modules is 8, determined by the 3-bit index $k$ derived from the parity of the voxel coordinates [S3].
*   **Comparison**: This interleaving scheme is identical in organization to the 8-way interleaved memory system used in the VIRIM geometry unit [S3].
*   **When to choose**: When implementing custom hardware or low-level software renderers that require explicit control over memory bank conflicts for tri-linear interpolation [S3].

### 3D Texture Proxy Geometry
[3D Texture Proxy Geometry](870-3d-texture-proxy-geometry.md)

### AR mapping to 3D texture
[AR mapping to 3D texture](884-ar-mapping-to-3d-texture.md)

### Volume Texture Creation
[Volume Texture Creation](1857-volume-texture-creation.md)

### Texture Uniting (Multi-Channel Packing)
[Texture Uniting (Multi-Channel Packing)](2470-texture-uniting-multi-channel-packing.md)

### PixelFlow Volume Rendering
[PixelFlow Volume Rendering](3443-pixelflow-volume-rendering.md)

### Hardware Volume Rendering Pipeline with Pixel Textures
[Hardware Volume Rendering Pipeline with Pixel Textures](5641-hardware-volume-rendering-pipeline-with-pixel-textures.md)

### 3D Transfer Function Texture Layout
[3D Transfer Function Texture Layout](5646-3d-transfer-function-texture-layout.md)

### 3D Volume Atlas Packing
[3D Volume Atlas Packing](7158-3d-volume-atlas-packing.md)

## Background
The concept of 8-way interleaved memory for 3D textures originated in early hardware volume rendering systems. The SGI Reality Engine used this technique to optimize tri-linear interpolation [S3]. The VIRIM (Volume Rendering Image Machine) architecture also employed an identical 8-way interleaved memory system for its geometry unit [S3].

In modern software rendering, the shift from linear buffers to 3D texture resources is driven by the need to leverage hardware cache hierarchies. While linear buffers require manual index calculation and byte-shifting, 3D textures provide hardware-accelerated sampling [S2, unofficial]. The use of 16-bit textures for fluid simulation and other scientific visualization tasks was popularized by techniques described in GPU Gems, highlighting the bandwidth benefits of reduced precision storage [S1, unofficial].

## Sources
[S1] Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S2] Voxel Raytracer Devlog // My thought repository, https://nelari.us/post/voxel-ray-tracing/, fetched 2026-10-24 (unofficial)
[S3] Architectures for real-time volume rendering, https://doi.org/10.1016/s0167-739x(98)00051-x, fetched 2026-11-01
