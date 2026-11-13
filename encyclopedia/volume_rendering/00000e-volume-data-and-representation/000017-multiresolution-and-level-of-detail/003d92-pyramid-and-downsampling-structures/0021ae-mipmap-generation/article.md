# Mipmap Generation
Also called: 3D Texture Mipmap Generation, Compute Shader Mipmap Generation, Volume Mipmapping

## Overview
Mipmap generation creates a hierarchy of downscaled volume data representations to enable multi-resolution rendering and faster traversal [S2, unofficial]. It solves the problem of aliasing and performance degradation when viewing high-resolution volumes from a distance by allowing the renderer to sample from lower-resolution levels [S4, unofficial]. Unlike [VoxelGrid Down Sample](1014-voxelgrid-down-sample.md) which focuses on structural grid reduction, or [Streaming MIP Implementation (Basic)](1803-streaming-mip-implementation-basic.md) which handles memory bandwidth, this method specifically constructs the filtered texture pyramid itself, often using compute shaders for parallel generation [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A base volume texture (Level 0) at resolution $N \times N \times N$.
- Output: A mip-chain of textures where Level $L$ has resolution $\lfloor N / 2^L \rfloor \times \lfloor N / 2^L \rfloor \times \lfloor N / 2^L \rfloor$ [S2, unofficial].

**Data Structures:**
The mipmap level is managed via a structure containing:
- `sizeX_`, `sizeY_`, `sizeZ_`: Integer dimensions of the current level.
- `dataCpu_`: Pointer to CPU-side memory.
- `dataGpu_`: CUDA array or GPU buffer handle.
- `dataTex_`: CUDA texture object or OpenGL/Vulkan texture handle.
- `cpuDataCounter_`, `gpuDataCounter_`: Usage counters for synchronization.
- `parent_`: Reference to the parent Volume object [S2, unofficial].

**Parameters:**
- `MipmapFilterMode::AVERAGE`: Applies average filtering (box filter) when generating the mipmap level [S2, unofficial].
- `MipmapFilterMode::HALTON`: Uses Halton sampling to select voxels for downsampling, reducing bias compared to simple averaging [S2, unofficial].
- Typical downsampling factor: 2x per level (Level $n$ is $(n+1)$x downsampling relative to base? No, standard is $2^n$. The source [S2, unofficial] states "level 1 is 2x downsampling, level 2 is 3x downsampling, and level n is (n+1)x downsampling". This contradicts standard power-of-two mipmaps. However, standard practice and [S4, unofficial] imply sequential lower resolutions. We will assume standard power-of-two halving for the algorithm steps, noting the source's specific "n+1" claim as a potential variant or error, but implementing the standard halving loop described in [S1, unofficial] which says "halve the current dimension"). *Correction*: [S1, unofficial] explicitly states "halve the current dimension". [S2, unofficial] says "level 2 is 3x downsampling". This is a contradiction. Standard mipmap generation halves dimensions. We follow the explicit step "halve the current dimension" [S1, unofficial] as it is the operational instruction.

**Algorithm Steps:**
1. **Initialization**: Set `current_dim` to the base resolution $N$. Set `mip_level` to 0.
2. **Loop Condition**: While `current_dim` $\ge 1$:
    a. **Check Existence**: Call `mipmapCheckOrCreate` to ensure the target level exists. If not, allocate memory for the new level [S2, unofficial].
    b. **Setup Dispatch**: Calculate work group size as $\lceil \text{current\_dim} / 8 \rceil$ in all three dimensions [S1, unofficial].
    c. **Bind Target**: Bind the 3D texture face (or the specific 3D texture) as a write-only image at `mip_level + 1` using RGBA8 format [S1, unofficial].
    d. **Dispatch Compute**: Execute the compute shader dispatch.
    e. **Barrier**: Insert memory barriers for texture fetch and shader image access to ensure coherence [S1, unofficial].
    f. **Increment**: Increment `mip_level` by 1.
    g. **Downsample**: Set `current_dim` = `current_dim` / 2 (integer division) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_mip_chain(base_texture, filter_mode):
    current_dim = base_texture.resolution
    mip_level = 0
    
    # Level 0 is always the original data
    levels = [base_texture]
    
    while current_dim >= 1:
        # Check if level exists, create if not
        next_level = mipmap_check_or_create(mip_level + 1, current_dim // 2)
        
        # Bind next level as write-only target
        bind_write_only_image(next_level.texture_handle, mip_level + 1)
        
        # Calculate work groups
        work_group_size = 8
        num_groups_x = ceil(current_dim / work_group_size)
        num_groups_y = ceil(current_dim / work_group_size)
        num_groups_z = ceil(current_dim / work_group_size)
        
        # Dispatch compute shader
        # The shader reads from mip_level and writes to mip_level + 1
        dispatch_compute(num_groups_x, num_groups_y, num_groups_z, 
                         base_texture.texture_handle, 
                         next_level.texture_handle, 
                         filter_mode)
        
        # Memory barrier to ensure write completion before next read
        insert_memory_barrier()
        
        levels.append(next_level)
        mip_level += 1
        current_dim = current_dim // 2

    return levels
```

**Compute Shader Logic (Internal):**
The compute shader for each thread (representing a voxel in the *new* lower-resolution level) must determine which voxels from the *previous* higher-resolution level contribute to it.
- If `filter_mode` is `AVERAGE`:
    - Identify the $2 \times 2 \times 2$ block of voxels in the parent level corresponding to the current thread's coordinates.
    - Sum the values of these 8 voxels.
    - Divide by 8.0.
- If `filter_mode` is `HALTON`:
    - Use Halton sequence indices to select a subset of voxels from the parent block to reduce computational cost or bias [S2, unofficial]. *Note: Specific Halton implementation details are not provided in sources, so this step is marked as requiring implementation based on the mode flag.*

**Edge Cases and Pitfalls:**
- **Boundary Conditions**: When `current_dim` is odd, the last block may not be a full $2 \times 2 \times 2$. The shader must handle partial blocks by averaging only the existing voxels or padding with zeros.
- **Precision**: Using `RGBA8` may lose precision for floating-point volume data. If high dynamic range is required, use `RGBA16F` or `RGBA32F` [S1, unofficial].
- **Degenerate Input**: If the base resolution is 1, the loop terminates immediately after Level 0.

## Subtopics

### [Atlas Mipmapping](2639-atlas-mipmapping.md)
Atlas Mipmapping addresses undersampling artifacts that occur when a distant surfel covers few pixels but retains its full-resolution texture tile, causing a single pixel to span multiple texels during a bilinear fetch. It focuses on storage layout optimization within an atlas rather than the generation algorithm itself.

### [Voxel Pre-integration](2956-voxel-pre-integration.md)
Voxel Pre-integration generates mipmaps for a voxelized 3D texture using bilinear filtering to create a volumetric sampling function that yields average color and occlusion for cone tracing. It differs from standard mipmap generation by specifically targeting the creation of a "volumetric sampling function" for indirect lighting approximation via cone tracing, rather than general texture compression. Trade-offs include potential inaccuracies in appearance when input voxels are very opaque due to the loss of masking and shadowing effects.

### [Mipmap Filtering](4541-mipmap-filtering.md)
Mipmap Filtering is a pre-filtering technique that avoids aliasing artifacts when a projected texture's resolution exceeds the sample resolution, and mitigates block-like discontinuities when the projected resolution is lower than the target resolution. It sets itself apart by explicitly managing the transition between different levels of detail (LOD) during the sampling phase, whereas other methods focus on storage or creation [G].

### [GPU 3D Mip Chain Generation](5067-gpu-3d-mip-chain-generation.md)
`generateMipChain3d` is a GPU primitive that fills a 3D texture's mip chain by executing per-(level, z-slice) render passes, targeting specific Z-slices via `GPURenderPassColorAttachment.depthSlice`. Unlike standard mipmaps which store averaged values, this method supports building a per-field max pyramid in deviation space to enable transfer-function-adaptive skipping during volume ray marching, allowing dynamic region skipping based on live contrast settings without rebuilding the structure.

### [Shadow Volume Mipmap Generation](5724-shadow-volume-mipmap-generation.md)
Shadow Volume Mipmap Generation constructs a 3-level mipmap chain for a shadow volume texture using bitwise OR accumulation to preserve shadow presence across downsampled resolutions. It sets itself apart from standard 3D mipmap generation by using logical OR instead of averaging or min/max filtering.

### [Voxel Mipmap Generation](12422-voxel-mipmap-generation.md)
Not given in the sources.

### [Clipmap Voxel Storage with Mipmap Generation](13504-clipmap-voxel-storage-with-mipmap-generation.md)
A voxel storage structure using clipmaps that generates mipmaps for inner clipmaps and the largest clipmap via compute shaders, copying regions between textures to handle axis-aligned downsampling.

### Multi-Frequency Texture Sampling with LOD
A technique to simulate high-frequency noise details using a small 3D texture by sampling it at different frequencies (Fractal Brownian Motion levels), reducing the number of texture lookups for distant objects [S3, unofficial].
- **Steps**: Limit the number of frequency levels (LODs) sampled for far-away primary samples to minimize texture lookups, which are a bottleneck. Blend between different LODs to ensure smooth transitions in visual quality [S3, unofficial].
- **Parameters**: Texture resolution: 128x128x128. This size is small enough for fast web loading but large enough for 2-3 FBM levels [S3, unofficial].

### Volume Rendering with Mipmapping
Applies mipmapping to the retained texture to improve rendering performance [S4, unofficial].
- **Steps**: Upload the final 3D texture atlas and lookup tables to the GPU. Render the volume slice-by-slice by drawing a proxy polygon for each slice. Use a fragment shader to sample the correct texels from the packed texture using the lookup tables [S4, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N^3)$ for the entire chain generation, as the total number of voxels in a power-of-two mip chain is $N^3 (1 + 1/8 + 1/64 + \dots) \approx 1.14 N^3$. Each level is processed in parallel via compute shaders [S1, unofficial].
- **Memory Complexity**: $O(N^3)$ total storage for the full mip chain. For 3D textures, this can be significant. Using `RGBA8` reduces memory footprint compared to `RGBA32F` [S1, unofficial].
- **Measured Performance**: Not given in the sources.
- **Strengths**: Reduces aliasing artifacts and improves rendering speed by sampling appropriate resolution levels [S4, unofficial]. Enables efficient traversal and skipping in volume ray marching when combined with max-value pyramids [S1, unofficial].
- **Limitations**: Increases memory usage due to storing multiple levels. Halton sampling may introduce bias if not implemented correctly [S2, unofficial].
- **When to use**: When rendering high-resolution volumes at varying distances or when using techniques like cone tracing that benefit from pre-filtered volumetric data [S4, unofficial].
- **When not to use**: When memory bandwidth is extremely constrained and the volume is viewed at a single fixed resolution, or when the volume is too small to benefit from multi-resolution sampling.

## Background
Mipmap generation for volume rendering originated from the need to handle texture aliasing in 3D graphics. The concept of mipmaps was introduced for 2D textures by Lance Williams [LaM99] (cited in [S4, unofficial]). In volume rendering, it was adapted to handle the dense sampling requirements of ray marching and cone tracing. The specific compute shader dispatch method described in [S1, unofficial] and the adaptive sampling integration in [S2, unofficial] represent modern GPU-accelerated implementations. The use of mipmaps in hybrid CPU-GPU pipelines for medical imaging was detailed in [S4, unofficial], highlighting their role in memory-efficient rendering.

## Sources
[S1] latencyhiding/voxel_cone_tracing, https://github.com/latencyhiding/voxel_cone_tracing, fetched 2026-10-26 (unofficial)
[S2] Learning adaptive sampling and reconstruction for Volume Visualization, https://github.com/shamanDevel/AdaptiveSampling, fetched 2026-10-27 (unofficial)
[S3] Efficient volumetric clouds - Resources - three.js forum, https://discourse.threejs.org/t/efficient-volumetric-clouds/66067, fetched 2026-10-29 (unofficial)
[S4] A Hybrid CPU-GPU Approach for Memory-Efficient Volume ..., https://www.researchgate.net/publication/408581180_A_Hybrid_CPU-GPU_Approach_for_Memory-Efficient_Volume_Rendering_of_Medical_Image_Data, fetched 2026-10-26 (unofficial)
