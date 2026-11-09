# Mipmap Generation

Also called: 3D Texture Mipmap Generation, Compute Shader Mipmap Generation, Volume Mipmapping

## Overview
Mipmap Generation creates a hierarchy of downscaled volume data to enable multi-resolution rendering and faster traversal [S2, unofficial]. It sets itself apart from sibling methods like [VoxelGrid Down Sample](1014-voxelgrid-down-sample.md) by explicitly organizing data into a structured mip-chain accessible via texture sampling, and from [Level-of-Detail Volume Rendering via 3D Textures](1706-level-of-detail-volume-rendering-via-3d-textures.md) by focusing on the generation pipeline itself rather than the LOD selection logic [G].

## How it works
**Inputs and Outputs:**
*   **Input:** A high-resolution 3D volume texture (Level 0).
*   **Output:** A mip-chain where Level $n$ is a $(n+1)$-times downscaled version of the original data [S2, unofficial].

**Data Structures:**
The method relies on a `MipmapLevel` structure to track state and memory [S2, unofficial]:
*   `sizeX_`, `sizeY_`, `sizeZ_`: Integer dimensions of the current level [S2, unofficial].
*   `dataCpu_`: Pointer to CPU-side memory.
*   `dataGpu_`: Pointer to CUDA array or GPU memory.
*   `dataTex_`: CUDA texture object or GPU texture handle.
*   `cpuDataCounter_`, `gpuDataCounter_`: Usage counters [S2, unofficial].
*   `parent_`: Reference to the parent Volume object.

**Algorithm Steps:**
1.  **Initialization:** Set `current_dimension` to the grid resolution and `mip_level` to 0 [S1, unofficial].
2.  **Check Existence:** Call `mipmapCheckOrCreate` to verify if the current mip level already exists; if not, allocate it [S2, unofficial].
3.  **Loop Condition:** While `current_dimension` $\ge 1$:
    a.  Set shader uniforms for `resolution` and `mip_level` [S1, unofficial].
    b.  Bind the 3D texture face as a write-only image texture at the next mip level using RGBA8 format [S1, unofficial].
    c.  Dispatch a compute shader work group sized to $\lceil \text{current\_dimension} / 8 \rceil$ in all three dimensions [S1, unofficial].
    d.  Insert memory barriers to ensure texture fetch completion and shader image access synchronization [S1, unofficial].
    e.  Increment `mip_level` by 1.
    f.  Halve `current_dimension` for the next iteration [S1, unofficial].

**Filtering Modes:**
Two filter modes are supported during downsampling [S2, unofficial]:
1.  **AVERAGE:** Applies average filtering when generating the mipmap level [S2, unofficial]. Each output voxel is the arithmetic mean of the input voxels in the sampling kernel.
2.  **HALTON:** Uses Halton sampling to select voxels when generating the mipmap level [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_mip_chain(volume: Volume, filter_mode: str):
    """
    Generates a 3D mipmap chain for a volume.
    filter_mode: 'AVERAGE' or 'HALTON'
    """
    # Assume volume has a base level (Level 0) with dimensions (dim, dim, dim)
    current_dim = volume.level[0].sizeX_
    mip_level = 0
    
    # Check if Level 0 exists (it should)
    if not mipmap_check_or_create(volume, mip_level):
        raise ValueError("Base level missing")

    while current_dim >= 1:
        # Set uniforms for the compute shader
        set_uniform("resolution", current_dim)
        set_uniform("mip_level", mip_level)
        set_uniform("filter_mode", filter_mode)

        # Bind the next mip level as a write-only image
        # In a real implementation, this binds volume.level[mip_level].dataTex_
        bind_write_only_image(volume.level[mip_level].dataTex_)

        # Calculate work group size (8x8x8 threads per group)
        num_groups_x = (current_dim + 7) // 8
        num_groups_y = (current_dim + 7) // 8
        num_groups_z = (current_dim + 7) // 8

        # Dispatch compute shader
        # The shader reads from mip_level-1 (or 0 if mip_level=1) and writes to mip_level
        dispatch_compute(num_groups_x, num_groups_y, num_groups_z)

        # Memory barrier to ensure writes are visible
        memory_barrier_texture_and_image()

        # Prepare for next level
        mip_level += 1
        current_dim = current_dim // 2
        
        # Check/create next level if dimension > 0
        if current_dim > 0:
            mipmap_check_or_create(volume, mip_level)

def mipmap_check_or_create(volume: Volume, level: int) -> bool:
    """Checks if the mip level exists, creates it if not."""
    if level >= len(volume.levels):
        # The sources do not specify the exact allocation logic for mip levels.
        pass
    return True

def set_uniform(name: str, value):
    """Sets a shader uniform variable."""
    pass

def bind_write_only_image(texture_handle):
    """Binds a texture handle as a write-only image for the compute shader."""
    pass

def dispatch_compute(groups_x, groups_y, groups_z):
    """Dispatches a compute shader with the specified number of work groups."""
    pass

def memory_barrier_texture_and_image():
    """Inserts a memory barrier to synchronize texture fetches and image writes."""
    pass
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** When downsampling, ensure the sampling kernel (e.g., 2x2x2 for average) does not exceed the bounds of the source level [G].
*   **Precision:** Using `RGBA8` for intermediate levels may cause quantization artifacts if the volume data has high dynamic range [G]. For scientific volumes, `RGBA16F` or `RGBA32F` is recommended to preserve precision during averaging.
*   **Degenerate Input:** If the initial resolution is not a power of two, the loop terminates when `current_dim` < 1 [S1, unofficial]. The last level may be smaller than half the previous level [G].

## Subtopics

### [Atlas Mipmapping](2639-atlas-mipmapping.md)
Atlas Mipmapping addresses the undersampling artifacts that occur when a distant surfel covers few pixels but retains its full-resolution texture tile, causing a single pixel to span multiple texels during a bilinear fetch. It sets itself apart from sibling methods like [GPU 3D Mip Chain Generation](5067-gpu-3d-mip-chain-generation.md) by operating on a pre-compressed 2D texture atlas rather than a volumetric grid, and from [Voxel Pre-integration](2956-voxel-pre-integration.md) by maintaining the surfel-based rendering pipeline while adding multi-resolution support.

### [Voxel Pre-integration](2956-voxel-pre-integration.md)
Voxel Pre-integration is a method to generate mipmaps for a voxelized 3D texture using bilinear filtering to create a volumetric sampling function that yields average color and occlusion for cone tracing. It solves the problem of approximating light from areas of various sizes by sampling a 4D texture (3D space + mip levels) using linear interpolation. It differs from sibling methods like Atlas Mipmapping and GPU 3D Mip Chain Generation by specifically targeting the creation of a "volumetric sampling function" for indirect lighting approximation via cone tracing, rather than general texture compression or shadow volume optimization. Trade-offs include potential inaccuracies in appearance when input voxels are very opaque due to the loss of masking and shadowing effects.

### [GPU 3D Mip Chain Generation](5067-gpu-3d-mip-chain-generation.md)
`generateMipChain3d` is a GPU primitive that fills a 3D texture's mip chain by generating each level via per-(level, z-slice) render passes, using `GPURenderPassColorAttachment.depthSlice` to target specific Z-slices. It builds on [Mipmap Generation](2602-mipmap-generation.md) by extending the 2D slice-based approach to 3D volumes, enabling efficient downsampling for acceleration structures like transfer-function-adaptive skipping pyramids. It differs from [Atlas Mipmapping](2639-atlas-mipmapping.md) by operating on contiguous 3D volumes rather than 2D atlases, and from [Voxel Pre-integration](2956-voxel-pre-integration.md) by focusing on spatial resolution reduction rather than radiometric integration.

### [Shadow Volume Mipmap Generation](5724-shadow-volume-mipmap-generation.md)
Shadow Volume Mipmap Generation creates a multiresolution hierarchy for 3D shadow volume textures to enable efficient rendering at varying distances or resolutions. It solves the problem of preserving shadow presence information during downsampling by using bitwise OR accumulation, ensuring that if any voxel in a parent block is shadowed, the corresponding voxel in the child block remains shadowed. This method is distinct from [Atlas Mipmapping](2639-atlas-mipmapping.md) which handles 2D texture atlases, [Voxel Pre-integration](2956-voxel-pre-integration.md) which averages density, and [GPU 3D Mip Chain Generation](5067-gpu-3d-mip-chain-generation.md) which typically uses fragment shaders for downsampling; this method is a CPU-side bitwise accumulation approach.

### [Mipmap Filtering](4541-mipmap-filtering.md)
Mipmap Filtering is a pre-filtering technique that avoids aliasing artifacts when a projected texture's resolution exceeds the sample resolution, and mitigates block-like discontinuities when the projected resolution is lower than the target resolution. It sets itself apart from its siblings by explicitly managing the transition between different levels of detail (LOD) in the mip-chain during the sampling phase, whereas [Atlas Mipmapping](2639-atlas-mipmapping.md) focuses on storage layout, [Voxel Pre-integration](2956-voxel-pre-integration.md) on radiative transfer approximation, and [GPU 3D Mip Chain Generation](5067-gpu-3d-mip-chain-generation.md) on the creation process itself.

## Operating characteristics
*   **Generation Method:** Typically performed on the GPU using compute shaders for parallelism [S1, unofficial].
*   **Format:** Uses `RGBA8` format for write-only image textures at each mip level [S1, unofficial].
*   **When to use:** When rendering large volumes at varying distances where uniform downsampling is sufficient.
*   **When not to use:** When the volume has highly anisotropic features or when memory bandwidth is extremely constrained and adaptive methods are preferred.

## Background
In volume rendering, mipmap generation was adapted to handle 3D scalar fields, often using compute shaders or fragment shaders for downsampling. The specific implementation details described here, such as the `MipmapLevel` struct and Halton sampling, are derived from modern GPU-based volume rendering frameworks [S2, unofficial].

## Sources
[S1] latencyhiding/voxel_cone_tracing, https://github.com/latencyhiding/voxel_cone_tracing, fetched 2026-10-26 (unofficial)
[S2] Learning adaptive sampling and reconstruction for Volume Visualization, https://github.com/shamanDevel/AdaptiveSampling, fetched 2026-10-27 (unofficial)
[S3] Efficient volumetric clouds - Resources - three.js forum, https://discourse.threejs.org/t/efficient-volumetric-clouds/66067, fetched 2026-10-29 (unofficial)
