# Texture Format Optimization for Rendering

Also called: GL LUMINANCE/INTENSITY Optimization, Optimal Texture Format Selection, Precision Pitfall: Texture Format for Position Data

## Overview
This method defines the optimal OpenGL texture formats and brick dimensions for volume rendering to maximize throughput and minimize artifacts. It addresses the trade-off between data transfer speed, rendering performance, and numerical precision. It sets itself apart from general compression methods by focusing on the specific hardware cache behaviors and driver overheads of direct volume rendering pipelines rather than storage efficiency [S1].

## How it works
**Inputs:** Volume data (scalar fields), rendering hardware capabilities (texture cache size), and application requirements (precision needs).
**Outputs:** Selected `GL_TEXTURE_3D` internal format, external format, and brick dimensions.

**Data Structures:**
The volume is partitioned into 3D bricks. Each brick is stored in a single 3D texture object.
*   **Scalar Field Bricks:** Stored in `GL_INTENSITY` internal format.
*   **Position/Geometry Bricks:** Stored in `R32G32B32A32_Float` internal format.

**Steps:**
1.  **Select Format for Scalar Data:** Use `GL_LUMINANCE` external format with `GL_INTENSITY` internal format [S1].
2.  **Determine Brick Size for Scalar Data:** Set brick dimensions to $64 \times 32 \times 32$ voxels [S1].
    *   This results in a 64 KB texture size [S1].
3.  **Select Format for Position Data:** Use `R32G32B32A32_Float` for textures storing front/back face positions [S2, unofficial].
4.  **Avoid Suboptimal Formats:** Do not use `GL_RGBA8` for scalar data if performance is critical, as it is significantly slower [S1]. Do not use `B8G8R8A8_UNorm` for position data [S2, unofficial].

**Formulas:**
*   Texture Size (Scalar): $64 \times 32 \times 32 \times 1 \text{ byte} = 64 \text{ KB}$ [S1].
*   Texture Size (RGBA8 Comparison): $32 \times 32 \times 32 \times 4 \text{ bytes} = 128 \text{ KB}$ per brick, but optimal rendering uses interleaved bricks resulting in 64 KB effective pages [S1].

**Parameters:**
*   **Scalar Brick Dimensions:** $64 \times 32 \times 32$ [S1].
*   **Position Format:** `R32G32B32A32_Float` [S2, unofficial].
*   **Scalar Format:** `GL_INTENSITY` internal [S1].

**Edge Cases and Pitfalls:**
*   **CPU Bound Rendering:** Using bricks smaller than $64 \times 32 \times 32$ with `GL_INTENSITY` causes the rendering to become CPU-bound due to volume slicing overhead [S1].
*   **Cache Misses:** Using bricks larger than $64 \times 32 \times 32$ with `GL_INTENSITY` causes textures to exceed the texture cache, degrading performance [S1].
*   **Precision Artifacts:** Using `B8G8R8A8_UNorm` for position textures (front/back faces) causes insufficient precision. This leads to miscalculated ray directions and pixelated artifacts when zooming in [S2, unofficial].
*   **RGBA8 Performance:** `GL_RGBA8` rendering is twice as slow as `GL_INTENSITY` rendering [S1]. The optimal brick size for `GL_RGBA8` is $32 \times 32 \times 32$, but this is still slower than the optimal `GL_INTENSITY` configuration [S1].
*   **Data Download:** While `GL_BGRA` with `GL_RGBA8` internal format is faster for downloading data to the GPU, `GL_INTENSITY` is faster for rendering [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def setup_volume_textures(volume_data, position_data):
    # Scalar field setup
    # Use GL_LUMINANCE external format with GL_INTENSITY internal format
    scalar_format = GL_INTENSITY
    scalar_ext_format = GL_LUMINANCE
    brick_size = (64, 32, 32)
    
    # Validate brick size to avoid CPU bound or cache miss issues
    if brick_size != (64, 32, 32):
        raise ValueError("Brick size must be 64x32x32 for optimal GL_INTENSITY performance")
        
    # Create 3D texture for scalar data
    scalar_tex = create_3d_texture(scalar_format, brick_size)
    upload_data(scalar_tex, volume_data)
    
    # Position data setup
    # Use R32G32B32A32_Float to avoid precision artifacts
    position_format = GL_RGBA32F
    position_tex = create_3d_texture(position_format, position_data.shape)
    upload_data(position_tex, position_data)
    
    return scalar_tex, position_tex
```

## Subtopics

### BC5 Compression
Compresses normal maps or dual-channel data using block compression. Choose when storage size is critical and precision requirements are lower than float. [BC5 Compression](5013-bc5-compression.md)

### Texture Map Coding (TMC)
A general-purpose texture compression algorithm. Choose when dealing with 2D texture maps rather than 3D volume bricks. [Texture Map Coding (TMC)](18716-texture-map-coding-tmc.md)

## Operating characteristics
*   **Time Complexity:** Rendering performance is optimized by the $64 \times 32 \times 32$ brick size. Smaller bricks increase CPU overhead; larger bricks increase cache misses [S1].
*   **Memory Complexity:** Scalar bricks are 64 KB each. RGBA8 bricks are larger per voxel but can be interleaved to fit similar cache lines [S1].
*   **Measured Performance:** On an NVIDIA GeForce 6800 Ultra, `GL_INTENSITY` with $64 \times 32 \times 32$ bricks yields the best frame rate [S1]. `GL_RGBA8` is twice as slow [S1].
*   **Strengths:** Maximizes rendering throughput for scalar volumes. Avoids precision loss for position data.
*   **Limitations:** Specific to OpenGL. Requires careful brick sizing to balance CPU and GPU loads. Not suitable for high-precision position data if lower precision formats are used [S2, unofficial].
*   **When to Use:** When rendering large volumes on hardware with limited texture cache and when scalar data precision is sufficient for 8-bit intensity.
*   **When Not to Use:** When position data requires high precision (use `R32G32B32A32_Float`) or when data download speed is the primary bottleneck (consider `GL_BGRA`/`GL_RGBA8` for upload, though rendering will be slower) [S1].

## Background
This method originates from research on distributed shared memory systems for roaming large volumes, specifically targeting NVIDIA hardware in the mid-2000s [S1]. It highlights the importance of hardware-specific optimizations in volume rendering pipelines. The precision pitfall regarding position data is a common issue in 3D graphics programming, often encountered when mapping geometry to textures [S2, unofficial].

## Sources
[S1] Distributed Shared Memory for Roaming Large Volumes, https://doi.org/10.1109/tvcg.2006.135, fetched 2026-10-26
[S2] 3D Texture Only interpolates on Y-Axis, https://gamedev.stackexchange.com/questions/166205, fetched 2026-10-29 (unofficial)
