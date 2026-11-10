# 3D Texture Storage and Sampling

Also called: GPU 3D Texture Volume Storage, Hardware 3D Texture Sampling

## Overview
3D Texture Storage and Sampling is a method for storing volumetric voxel data in GPU-accessible 3D image resources to leverage hardware-optimized memory locality and sampling units [S1, unofficial]. It solves the problem of inefficient CPU-side or software-managed memory layouts by utilizing GPU-specific structures that provide superior data locality compared to software-implemented Z-order curves [S1, unofficial]. Unlike [Evaluation Cache](874-evaluation-cache.md), which caches computed values, or [Sparse Volume Texture (SVT)](7011-sparse-volume-texture-svt.md), which uses indirection for sparse data, this method stores dense grids directly in hardware textures, often achieving significant performance gains (e.g., 10x FPS improvement) by reducing memory latency during ray traversal [S1, unofficial].

## How it works
**Inputs:**
- Volumetric data: A dense 3D grid of voxel values (scalar, vector, or multi-channel).
- Grid dimensions: $(D_x, D_y, D_z)$, typically powers of two or within hardware limits (e.g., 2048^3) [S1, unofficial].
- Texture format: Internal format (e.g., `R16UI`, `R8UI`, `RGBA8`) and data type (`UNSIGNED_BYTE`, `UNSIGNED_SHORT`) [S4, unofficial].

**Outputs:**
- GPU 3D Texture Resource: A bound texture object ready for sampling in shaders.
- Rendered Image: Final pixel colors derived from sampling the texture along rays [S3, unofficial].

**Data Structures:**
1. **3D Texture Object:**
   - Type: `GL_TEXTURE_3D` (OpenGL) or equivalent Vulkan/DirectX 3D texture.
   - Dimensions: $(D_x, D_y, D_z)$.
   - Internal Format: Matches data precision (e.g., `GL_R16UI` for 16-bit unsigned integer density) [S4, unofficial].
   - Storage Layout: Hardware-defined, optimized for 3D locality (often more sophisticated than Z-order) [S1, unofficial].
   - Sampling Mode: Typically `GL_LINEAR` (tri-linear interpolation) for smooth shading or `GL_NEAREST` for discrete classification [S3, unofficial].

2. **Auxiliary Textures (Optional):**
   - Segmentation Mask: 3D texture storing classification labels [S2, unofficial].
   - Transfer Function: 1D or 3D texture mapping density/color to RGBA [S2, unofficial].

**Numbered Steps:**

1. **Texture Allocation and Initialization:**
   - Generate a 3D texture object ID.
   - Allocate memory with dimensions $(D_x, D_y, D_z)$ and specified internal format [S4, unofficial].
   - Set sampling parameters: `GL_TEXTURE_MIN_FILTER`, `GL_TEXTURE_MAG_FILTER`, and `GL_TEXTURE_WRAP_MODE` (usually `GL_CLAMP_TO_EDGE`) [S3, unofficial].

2. **Data Upload:**
   - Copy host-side voxel data into the GPU texture memory using `glTexSubImage3D` or equivalent [S4, unofficial].
   - Ensure data layout matches the expected memory order (hardware-dependent) [S1, unofficial].

3. **Ray Traversal and Sampling (Shader Side):**
   - For each pixel, cast a ray through the volume.
   - Calculate sample positions in world space, then transform to texture coordinates $(u, v, w) \in [0, 1]^3$ [S3, unofficial].
   - Sample the 3D texture at the calculated coordinates using tri-linear interpolation to retrieve voxel values [S2, unofficial].
   - If using segmentation masks, sample the mask texture to determine rendering properties (e.g., render scale, classification) [S2, unofficial].

4. **Color Computation:**
   - Apply transfer functions to map sampled values to color and opacity [S2, unofficial].
   - Accumulate color and opacity along the ray using volume integration (e.g., front-to-back compositing) [S3, unofficial].
   - Apply lighting effects (e.g., silhouette enhancement, gradient-based shading) if required [S2, unofficial].

5. **Output:**
   - Write the final accumulated color to the framebuffer or output texture [S3, unofficial].

**Formulas:**
- Texture Coordinate Mapping:
  $$ u = \frac{x_{world} - x_{min}}{x_{max} - x_{min}}, \quad v = \frac{y_{world} - y_{min}}{y_{max} - y_{min}}, \quad w = \frac{z_{world} - z_{min}}{z_{max} - z_{min}} $$
  where $(x_{world}, y_{world}, z_{world})$ is the sample position in world space, and $(x_{min}, y_{min}, z_{min})$ to $(x_{max}, y_{max}, z_{max})$ is the volume bounding box [S3, unofficial].

- Tri-linear Interpolation:
  $$ V(u,v,w) = \sum_{i=0}^{1} \sum_{j=0}^{1} \sum_{k=0}^{1} w_i w_j w_k V(x_i, y_j, z_k) $$
  where $w_i, w_j, w_k$ are interpolation weights based on fractional parts of coordinates, and $V(x_i, y_j, z_k)$ are the 8 neighboring voxel values [S3, unofficial].

**Parameters:**
- **Rendering Scale:** A parameter used in some implementations to scale cube textures, corresponding to voxel numbers in each direction, allowing details to be visible even if cube size is small [S2, unofficial].
- **Sampling Distance:** Distance between sample points along a ray; higher distance reduces sample count but increases approximation error [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Hardware Limits:** 3D textures have maximum dimension limits (e.g., 2048 voxels per dimension). Datasets exceeding this require [Sparse Volume Texture (SVT)](7011-sparse-volume-texture-svt.md) or [Brick-based storage and caching](4138-brick-based-storage-and-caching.md) [S1, unofficial].
- **Memory Bandwidth:** Dense 3D textures consume significant VRAM. Large volumes may exceed available memory [S1, unofficial].
- **Precision:** Use appropriate internal formats (e.g., `R16UI` vs `R8UI`) to avoid quantization artifacts in high-precision data [S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# GPU Shader Pseudocode (GLSL-like)

uniform sampler3D volumeTexture;
uniform sampler3D maskTexture;
uniform vec3 volumeBoundsMin;
uniform vec3 volumeBoundsMax;
uniform float samplingDistance;

vec3 rayOrigin;
vec3 rayDirection;

void main() {
    vec2 uv = gl_FragCoord.xy / vec2(textureSize(screenTexture, 0));
    // Calculate ray origin and direction from camera setup [G]
    
    vec3 pos = rayOrigin;
    vec3 color = vec3(0.0);
    float alpha = 0.0;
    
    // Ray marching loop
    while (pos.x < volumeBoundsMax.x && pos.y < volumeBoundsMax.y && pos.z < volumeBoundsMax.z && alpha < 1.0) {
        // Map world position to texture coordinates [0,1]
        vec3 texCoords = (pos - volumeBoundsMin) / (volumeBoundsMax - volumeBoundsMin);
        
        // Sample volume density
        float density = texture(volumeTexture, texCoords).r;
        
        // Sample mask if available [S2, unofficial]
        float maskVal = texture(maskTexture, texCoords).r;
        
        // Apply transfer function and lighting [S2, unofficial]
        vec3 voxelColor = applyTransferFunction(density, maskVal);
        float voxelAlpha = computeOpacity(density);
        
        // Front-to-back compositing
        color += voxelColor * voxelAlpha * (1.0 - alpha);
        alpha += voxelAlpha * (1.0 - alpha);
        
        // Advance along ray
        pos += rayDirection * samplingDistance;
    }
    
    gl_FragColor = vec4(color, alpha);
}
```

## Subtopics

### [Volume Texture Creation](1857-volume-texture-creation.md)
Allocates GPU memory and loads volumetric data into texture resources, handling allocation, data transfer, and lifecycle for dense grids.

### [Texture Uniting (Multi-Channel Packing)](2470-texture-uniting-multi-channel-packing.md)
Combines multiple 3D textures (e.g., noise, detail) into a single RGBA 3D texture to optimize bandwidth.

### [PixelFlow Volume Rendering](3443-pixelflow-volume-rendering.md)
Uses pixel processors to manipulate 3D texture values directly, enabling on-the-fly Phong lighting.

### [Texture Array Sampling with Alpha Discard](3993-texture-array-sampling-with-alpha-discard.md)
Samples 3D texture arrays and discards low-alpha fragments to avoid transparent regions.

### [Ambient Occlusion Volume](4341-ambient-occlusion-volume.md)
Stores precomputed AO factors in 3D textures to avoid runtime computation.

### [Hardware Volume Rendering Pipeline with Pixel Textures](5641-hardware-volume-rendering-pipeline-with-pixel-textures.md)
Uses fixed-function pixel textures for classification and shading on older hardware.

### [Sparse Volume Texture (SVT)](7011-sparse-volume-texture-svt.md)
Uses chunking and page tables to render large sparse volumes within texture dimension limits.

### [Memory Organization and Layout](7154-memory-organization-and-layout.md)
Defines strategies for mapping linear coordinates to physical memory to optimize cache locality.

### [3D Volume Atlas Packing](7158-3d-volume-atlas-packing.md)
Consolidates multiple discrete volumes into a single 3D texture resource.

### [Image Texture Atlas Packing](11896-image-texture-atlas-packing.md)
Emulates 3D textures using 2D atlases for WebGL 1.0 compatibility.

### OpenGL Texture Output
A method to output rendering results using OpenGL texture objects for efficiency [S3, unofficial]. It initializes texture objects with format `GL_RGB` and texel format `GL_UNSIGNED_BYTE` in a `DrawModel` class, then defines a `GL_QUAD` polygon to map the texture onto the screen display plane [S3, unofficial].

### Original Volume Rendering
A rendering mode that loads the entire volume dataset into a single 3D texture, bypassing page tables and masking [S4, unofficial]. It creates a `TEXTURE_3D` with dimensions matching the source data and internal format `R16UI` or `R8UI`, simplifying shader logic but requiring the entire volume to fit in GPU memory [S4, unofficial].

## Operating characteristics
- **Time Complexity:** Sampling is $O(1)$ per voxel access, but total time depends on ray length and sample count. Hardware sampling is highly optimized [S1, unofficial].
- **Memory Complexity:** $O(D_x \times D_y \times D_z)$ for dense storage. Requires sufficient VRAM [S1, unofficial].
- **Performance:** Achieves 121 FPS at 2048^3 resolution and 1000x1000 screen resolution, a 10x improvement over software-managed layouts (12 FPS) [S1, unofficial].
- **Strengths:** High data locality, hardware-accelerated interpolation, simple API usage [S1, unofficial].
- **Limitations:** Fixed maximum dimensions, high memory usage for large dense volumes, less flexible than sparse methods [S1, unofficial].
- **When to Use:** Dense volumes fitting within hardware limits, real-time rendering requirements [S1, unofficial].
- **When Not to Use:** Very large sparse volumes, memory-constrained environments [S1, unofficial].

## Background
Originated with the advent of GPU-based volume rendering systems that utilized 3D texture mapping extensions (e.g., SGI V-series, NVIDIA GeForce3) [S2, unofficial]. Early implementations used Z-order curves for locality, but were superseded by native 3D textures which offer more sophisticated hardware-level optimization [S1, unofficial]. The method is widely used in medical imaging, scientific visualization, and real-time rendering applications [S2, unofficial].

## Sources
[S1] This Tiny Algorithm Can Render BILLIONS of Voxels in Real Time, https://www.youtube.com/watch?v=ztkh1r1ioZo, fetched 2026-10-25 (unofficial)
[S2] Example-based volume illustrations, https://ieeexplore.ieee.org/abstract/document/1532854/, fetched 2026-10-31 (unofficial)
[S3] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
[S4] Winson-L/VolumeSegmenter, https://github.com/Winson-L/VolumeSegmenter, fetched 2026-10-27 (unofficial)
