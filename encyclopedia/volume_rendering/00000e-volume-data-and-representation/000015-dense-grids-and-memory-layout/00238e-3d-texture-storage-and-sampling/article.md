# 3D Texture Storage and Sampling

Also called: GPU 3D Texture Volume Storage, Hardware-Accelerated Volume Sampling

## Overview
3D Texture Storage and Sampling is a method for storing volumetric data in GPU 3D image resources (3D textures) to leverage hardware-optimized memory layouts and sampling units for direct volume rendering [S1, unofficial][S2, unofficial]. It solves the problem of inefficient memory access patterns in CPU-side or naive GPU storage by utilizing sophisticated data locality optimizations inherent in graphics hardware, which outperform software-managed layouts like Z-order curves [S1, unofficial]. Unlike [Dense grids and memory layout](3-dense-grids-and-memory-layout.md) which discusses general memory organization, this method specifically targets the GPU texture API and hardware sampling pipeline [S2, unofficial]. It differs from [3D Texture Representation (CPU/GPU Indexing)](2393-3d-texture-representation-cpu-gpu-indexing.md) by focusing on the runtime sampling mechanics and hardware acceleration rather than the logical indexing scheme [S2, unofficial].

## How it works
**Inputs:**
- Volumetric scalar field data (e.g., density, segmentation masks).
- Rendering parameters: resolution, rendering scale (optional, for illustrative purposes), and sampling distance.

**Outputs:**
- A GPU 3D texture object containing the voxel data.
- Rendered image or intermediate texture buffers.

**Data Structures:**
- **3D Texture Object:** A GPU resource with dimensions $W \times H \times D$. The internal memory layout is managed by the GPU driver/hardware to optimize for tri-linear interpolation and cache locality [S1, unofficial].
- **Cube Tiling Structure (Specific Implementation):** In some illustrative rendering systems, cube textures are grouped into a grid of size $4C \times 4C \times MC$, where $M$ is the number of cube sets and $C$ is the cube side length [S2, unofficial]. Each cube is stored as a volume to enable tri-linear interpolation [S2, unofficial].

**Steps:**
1. **Texture Allocation and Upload:** Allocate a 3D texture on the GPU with dimensions matching the volume resolution. Upload the voxel data from host memory to the GPU texture memory [S1, unofficial][S2, unofficial].
2. **Coordinate Calculation:** For each sample point along a ray (or fragment position in slicing):
   - Determine the voxel center position relative to the volume bounds [S2, unofficial].
   - If using a cube tiling scheme, calculate the cube index using the voxel center position and a rendering scale parameter [S2, unofficial].
   - Calculate the relative position within the cube/voxel using the rendering scale [S2, unofficial].
3. **Sampling:** Use hardware tri-linear interpolation to sample the 3D texture at the calculated coordinates [S1, unofficial][S2, unofficial]. This retrieves the scalar value (e.g., density) and potentially associated attributes (e.g., color, segmentation mask) [S2, unofficial].
4. **Shading and Compositing:** Apply transfer functions, lighting effects (e.g., silhouette enhancement), and compositing (e.g., ray marching accumulation) to the sampled values to compute the final pixel color [S2, unofficial].

**Formulas:**
- **Cube Index Calculation (Illustrative System):**
  $$ \text{CubeIndex} = f(\text{VoxelCenter}, \text{RenderScale}) $$
  Where $\text{RenderScale}$ corresponds to the voxel numbers in each direction for a cube set [S2, unofficial].
- **Relative Position:**
  $$ \text{RelativePos} = f(\text{WorldPos}, \text{RenderScale}) $$
  Used to retrieve the color/value within the specific cube/voxel [S2, unofficial].

**Parameters:**
- **Rendering Scale:** A parameter for each cube set corresponding to the voxel numbers in each direction, used to scale cube textures for better visibility of details [S2, unofficial].
- **Sampling Distance:** The distance between succeeding sample points along a ray in world coordinates. Higher sampling distance reduces the number of sample points, increasing approximation error [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Resolution Limits:** Standard 3D textures have maximum dimension limits (e.g., 2048 voxels per dimension). Large volumes may require techniques like [Sparse Volume Texture (SVT)](7011-sparse-volume-texture-svt.md) or [Brick-based storage and caching](4138-brick-based-storage-and-caching.md) [S1, unofficial].
- **Precision:** Hardware interpolation may introduce precision errors compared to exact analytical traversal, though this is generally negligible for visualization [S3, unofficial].
- **Boundary Conditions:** Sampling near volume boundaries requires proper clamping or border color handling to avoid artifacts [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for GPU-side sampling in a volume rendering shader
# Assumes a 3D texture 'volumeTex' is bound and accessible

def sample_volume_texture(u, v, w):
    """
    Samples a 3D texture at normalized coordinates (u, v, w).
    Returns the interpolated scalar value (e.g., density).
    """
    # Hardware performs tri-linear interpolation
    # Coordinates are typically in [0, 1] range
    density = texture(volumeTex, vec3(u, v, w)).r
    return density

def render_slice(volumeTex, slice_z, resolution, render_scale=None):
    """
    Renders a slice through the volume at z-coordinate slice_z.
    """
    # Initialize output texture if needed
    # For each pixel (x, y) in the output image:
    for x in range(resolution):
        for y in range(resolution):
            # Calculate normalized coordinates
            u = x / resolution
            v = y / resolution
            w = slice_z / resolution
            
            # Sample the volume
            density = sample_volume_texture(u, v, w)
            
            # Apply transfer function and shading
            color = apply_transfer_function(density)
            
            # Write to output buffer
            write_pixel(x, y, color)
```

## Subtopics

### [3D Texture Proxy Geometry](870-3d-texture-proxy-geometry.md)
3D Texture Proxy Geometry represents a rectilinear volume as a stack of parallel planar polygons (slabs) textured with 3D texture maps, leveraging hardware-accelerated tri-linear interpolation to sample volume data at arbitrary locations. It solves the problem of efficient hardware-accelerated volume rendering with directional lighting by pre-computing color and opacity per voxel and storing them in 3D textures, allowing the graphics pipeline to handle interpolation and compositing. Unlike shear-warp methods which fix plane counts to volume resolution, this method allows the number of rendering planes to be adjusted at runtime independent of volume resolution, and unlike coherent projection, it supports perspective views without a time penalty, though perspective projection causes irregular sampling along rays.

### [Volume Texture Creation](1857-volume-texture-creation.md)
Volume Texture Creation allocates GPU memory and populates it with volumetric data (scalar fields, gradients, transfer functions) and rendering targets to enable hardware-accelerated raycasting. It solves the problem of transferring host-side volume representations into GPU-accessible texture objects with specific memory layouts, filtering modes, and coordinate systems required by volume rendering shaders. It differs from [3D Texture Storage and Sampling](3064-3d-texture-storage-and-sampling.md) by focusing on the API-level allocation and data upload steps rather than the sampling logic.

### [Texture Uniting (Multi-Channel Packing)](2470-texture-uniting-multi-channel-packing.md)
Texture Uniting is a technique to combine four separate 3D textures (noise, detail, light, mask) into a single RGBA 3D texture to optimize memory bandwidth and sampling. It uses linear clamping sampling with normalized coordinates to handle resolution differences between source textures.

### [PixelFlow Volume Rendering](3443-pixelflow-volume-rendering.md)
PixelFlow Volume Rendering is a polygon resampling method designed for the PixelFlow graphics system that overcomes the lighting and data size limitations of the RealityEngine by allowing pixel processors to manipulate 3D texture values directly. It differs from its siblings by replacing precomputed shading coefficients with on-the-fly Phong lighting calculations derived from interpolated volume gradients, enabling general image-space lighting.

### [Texture Array Sampling with Alpha Discard](3993-texture-array-sampling-with-alpha-discard.md)
This method implements a fragment shader that samples a 3D texture array to retrieve voxel color and opacity. It discards fragments with alpha below a specific threshold to avoid rendering transparent regions. It relies on hardware texture sampling to retrieve voxel data rather than generating proxy geometry for occupied voxels.

### [Ambient Occlusion Volume](4341-ambient-occlusion-volume.md)
Ambient Occlusion Volume (AOV) is a method for storing precomputed near-field ambient occlusion factors directly in a 3D texture, optionally alongside bent normals, to avoid runtime reconstruction computation. It approximates the near-field ambient occlusion problem by computing an analytic shadow-volume-like enclosure around each polygon and applying a tunable occlusion function within that region. Unlike Kontkanen and Laine’s approach, which requires runtime reconstruction of occlusion factors, AOV eliminates this cost by baking the factors into the texture. It differs from disk-based methods (e.g., Bunnell, Hoberock/Jia) and Shopf et al.’s bounding cube method by using tight dynamic bounding volumes and partial coverage to avoid over-estimating occlusion and double-occlusion artifacts.

### [Hardware Volume Rendering Pipeline with Pixel Textures](5641-hardware-volume-rendering-pipeline-with-pixel-textures.md)
This method implements direct volume rendering on hardware supporting 3D texture mapping (e.g., SGI V-series, NVIDIA GeForce3) by using pixel textures (dependent textures) to perform classification and shading lookups. It solves the problem of applying complex, multi-dimensional transfer functions and lighting models efficiently by decoupling data storage from shading computation. Unlike [3D Texture Proxy Geometry](870-3d-proxy-geometry.md), which rasterizes geometry, this method rasterizes slabs of parallel planes and uses hardware texture interpolation to sample pre-computed color/opacity values. It differs from [PixelFlow Volume Rendering](3443-pixelflow-volume-rendering.md) by relying on fixed-function pixel texture extensions rather than custom fragment shaders.

### [Sparse Volume Texture (SVT)](7011-sparse-volume-texture-svt.md)
Sparse Volume Texture (SVT) is a technique that trades rendering speed for data compression by chunking voxel data into 3D tiles and storing only non-empty tiles in a physical texture, using a second volume texture as a page table for lookup. It solves the problem of rendering large volume datasets that exceed the maximum dimension limits of standard 3D textures (e.g., 2048 voxels per dimension) while managing memory footprint. Unlike dense grids, SVT circumvents API limits by mapping a large logical coordinate space onto a smaller physical texture containing only active tiles, though it incurs overhead from padding and mipmap hierarchies.

### [Memory Organization and Layout](7154-memory-organization-and-layout.md)
This article defines the memory organization strategies for storing dense voxel grids in GPU texture resources, focusing on how linear voxel coordinates map to physical memory addresses to optimize cache locality and bandwidth. It distinguishes between standard row-major storage, which relies on hardware caching for spatial coherence, and hardware-specific interleaving schemes designed to ensure that all voxels required for a single tri-linear interpolation fetch reside in distinct memory modules, allowing the entire cell to be retrieved in a single memory transaction.

### [3D Volume Atlas Packing](7158-3d-volume-atlas-packing.md)
3D Volume Atlas Packing consolidates multiple discrete volumes (such as animation timesteps or separate datasets) into a single contiguous 3D texture resource. This method solves the problem of managing numerous individual texture objects by reducing GPU state changes and memory fragmentation. It differs from [Texture Uniting (Multi-Channel Packing)](2470-texture-uniting-multi-channel-packing.md), which packs different data channels into one volume, by packing identical channel structures across the depth dimension. It builds on [3D Texture Storage and Sampling](3064-3d-texture-storage-and-sampling.md) by defining the specific layout and allocation strategy for these multi-volume containers.

### OpenGL Texture Output
A method to output rendering results using OpenGL texture objects for efficiency [S3, unofficial]. It initializes texture objects in the DrawModel class with format GL_RGB and texel format GL_UNSIGNED_BYTE, and defines a polygon (GL_QUAD) to map the texture onto the screen display plane [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Sampling is $O(1)$ per voxel access due to hardware interpolation [S1, unofficial]. The overall rendering time depends on the number of samples per ray and the number of rays (pixels) [S3, unofficial].
- **Memory Complexity:** $O(N)$ where $N$ is the number of voxels, stored in GPU texture memory [S1, unofficial].
- **Measured Performance:** Storing voxels in a 3D texture improved performance from 12 FPS to 121 FPS at 2048^3 grid resolution and 1000x1000 screen resolution, a 10x improvement over Z-order curve storage [S1, unofficial].
- **Strengths:** High data locality, hardware-accelerated interpolation, efficient memory bandwidth usage [S1, unofficial].
- **Limitations:** Limited by maximum texture dimensions, potential precision issues with large coordinates, overhead for sparse data [S1, unofficial].
- **When to Use:** When rendering dense volumes on modern GPUs with hardware 3D texture support [S1, unofficial].
- **When Not to Use:** For very sparse volumes where [Sparse Volume Texture (SVT)](7011-sparse-volume-texture-svt.md) or [VDB Data Loading & 3D Texture Setup](4880-vdb-data-loading-3d-texture-setup.md) might be more efficient [S1, unofficial].

## Background
The method originated with the advent of GPU hardware supporting 3D texture mapping, such as the SGI V-series and NVIDIA GeForce3 [S1, unofficial][S2, unofficial][S3, unofficial]. It was developed to overcome the limitations of CPU-based volume rendering and early GPU techniques that lacked efficient memory access patterns [S1, unofficial]. The use of 3D textures for volume rendering became a standard technique in direct volume rendering pipelines, enabling real-time visualization of large datasets [S1, unofficial].

## Sources
[S1] This Tiny Algorithm Can Render BILLIONS of Voxels in Real Time, https://www.youtube.com/watch?v=ztkh1r1ioZo, fetched 2026-10-25 (unofficial)
[S2] Example-based volume illustrations, https://ieeexplore.ieee.org/abstract/document/1532854/, fetched 2026-10-31 (unofficial)
[S3] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
