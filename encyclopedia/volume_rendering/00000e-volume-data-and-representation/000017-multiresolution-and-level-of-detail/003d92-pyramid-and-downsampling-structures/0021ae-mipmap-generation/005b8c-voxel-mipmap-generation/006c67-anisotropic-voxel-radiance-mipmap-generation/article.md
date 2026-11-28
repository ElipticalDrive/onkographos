# Anisotropic Voxel Radiance Mipmap Generation

Also called: Anisotropic Voxel Mipmap Generation

## Overview
Anisotropic Voxel Radiance Mipmap Generation creates a hierarchy of anisotropic mipmaps for voxel radiance textures to support efficient multi-resolution rendering. It distinguishes itself from standard isotropic mipmap generation by processing data in specific face directions, allowing for anisotropic filtering characteristics in the radiance transfer. It builds on [Voxel Mipmap Generation](12422-voxel-mipmap-generation.md) by introducing a two-phase compute shader approach (leaf and interior passes) to handle the directional dependencies of the radiance data.

## How it works
**Inputs and Outputs:**
*   **Input:** The base voxel radiance texture (dimension $m\_VoxelDim$).
*   **Output:** A hierarchy of $s\_AnisotropicMipmapCount$ anisotropic mipmap textures.

**Data Structures:**
*   **Voxel Radiance Texture:** 3D texture, base dimension $m\_VoxelDim$.
*   **Mipmap Textures:** 3D textures with internal format `GL_RGBA8`, format `GL_RGBA`, type `GL_FLOAT`, and trilinear filtering.
    *   **Leaf Level:** Dimensions $D_{leaf} = \lceil m\_VoxelDim / 2 \rceil$.
    *   **Interior Levels:** Dimensions halved at each step.

**Parameters:**
*   $m\_VoxelDim$: Base dimension of the voxel grid.
*   $m\_AnisoMipmapGroupSize$: Compute shader group size (block size) for the leaf mipmap pass.
*   $m\_BoxMipmapGroupSize$: Compute shader group size for interior mipmap passes.
*   $s\_AnisotropicMipmapCount$: Total number of anisotropic mipmap textures allocated.

**Steps:**
1.  **Initialization:**
    *   Create and link the leaf compute shader program from `anisotropic_mipmap_start_comp.glsl` [S1, unofficial].
    *   Allocate $s\_AnisotropicMipmapCount$ 3D textures for the mipmap hierarchy. The leaf level textures have dimensions $half\_voxel\_dim \times half\_voxel\_dim \times half\_voxel\_dim$, where $half\_voxel\_dim = \lfloor m\_VoxelDim / 2 \rfloor$ [S1, unofficial].
2.  **Leaf Pass:**
    *   Bind the base voxel radiance texture to texture unit 0.
    *   Bind the leaf mipmap textures (level 0 of the hierarchy) to image units $1$ through $N$ as `WRITE_ONLY`.
    *   Calculate the dispatch grid dimensions for the leaf pass: $GridDim = \lceil (m\_VoxelDim / 2) / m\_AnisoMipmapGroupSize \rceil$.
    *   Dispatch the leaf compute shader with $GridDim$.
    *   Issue a `GL_SHADER_IMAGE_ACCESS_BARRIER_BIT` to ensure memory consistency.
3.  **Interior Passes:**
    *   Calculate the total number of mipmap levels by halving $dim = m\_VoxelDim / 2$ until $dim == 0$, incrementing $level\_count$ at each step.
    *   For each anisotropic mipmap texture, initialize $dim = m\_VoxelDim / 4$.
    *   Loop for $level$ from $0$ to $level\_count - 2$:
        *   Bind the current mipmap level (source) as a read-only texture to unit 0.
        *   Bind the next mipmap level (destination, $level + 1$) as a write-only image to unit 1.
        *   Set the uniform `ChildMipLevel` to the current $level$.
        *   Calculate the dispatch grid dimensions: $GridDim = \lceil dim / m\_BoxMipmapGroupSize \rceil$.
        *   Dispatch the interior compute shader with $GridDim$.
        *   Issue a `GL_SHADER_IMAGE_ACCESS_BARRIER_BIT`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_anisotropic_mipmaps(voxel_radiance_texture, m_VoxelDim, m_AnisoMipmapGroupSize, m_BoxMipmapGroupSize, s_AnisotropicMipmapCount):
    # 1. Initialization
    leaf_shader = compile_shader("anisotropic_mipmap_start_comp.glsl") # [S1, unofficial]
    
    half_voxel_dim = m_VoxelDim // 2
    mipmap_textures = []
    for i in range(s_AnisotropicMipmapCount):
        tex = create_3d_texture(
            width=half_voxel_dim,
            height=half_voxel_dim,
            depth=half_voxel_dim,
            internal_format="GL_RGBA8",
            format="GL_RGBA",
            type="GL_FLOAT",
            filter="trilinear"
        )
        mipmap_textures.append(tex)

    # 2. Leaf Pass
    bind_texture(0, voxel_radiance_texture)
    for i in range(s_AnisotropicMipmapCount):
        bind_image(i + 1, mipmap_textures[i], access="WRITE_ONLY")
        
    leaf_grid_x = ceil((m_VoxelDim / 2) / m_AnisoMipmapGroupSize)
    leaf_grid_y = leaf_grid_x
    leaf_grid_z = leaf_grid_x
    dispatch_compute(leaf_shader, leaf_grid_x, leaf_grid_y, leaf_grid_z)
    gl_memory_barrier(GL_SHADER_IMAGE_ACCESS_BARRIER_BIT)

    # 3. Interior Passes
    # Calculate total levels
    dim = m_VoxelDim / 2
    level_count = 0
    while dim > 0:
        level_count += 1
        dim = dim / 2
    
    # Process interior levels
    # Note: The source specifies starting dim for interior as m_VoxelDim / 4
    # and looping level from 0 to level_count - 2.
    # This implies the interior passes start from the second mipmap level (index 1)
    # and generate subsequent levels.
    
    current_dim = m_VoxelDim / 4
    
    for level in range(0, level_count - 1): # level_count - 2 in source, but range end is exclusive, so -1 to match 0..N-2
        if level == 0:
            # First interior pass reads from leaf level (index 0) and writes to level 1
            source_tex = mipmap_textures[0]
            dest_tex = mipmap_textures[1]
        else:
            # Subsequent passes read from previous level and write to next
            source_tex = mipmap_textures[level]
            dest_tex = mipmap_textures[level + 1]
            
        bind_texture(0, source_tex, access="READ_ONLY")
        bind_image(1, dest_tex, access="WRITE_ONLY")
        set_uniform("ChildMipLevel", level)
        
        grid_dim = ceil(current_dim / m_BoxMipmapGroupSize)
        dispatch_compute(interior_shader, grid_dim, grid_dim, grid_dim)
        gl_memory_barrier(GL_SHADER_IMAGE_ACCESS_BARRIER_BIT)
        
        current_dim = current_dim / 2
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating point precision in `GL_FLOAT` textures may accumulate errors during multiple downsampling passes.
*   **Boundaries:** The leaf pass uses $m\_VoxelDim / 2$, while interior passes start with $m\_VoxelDim / 4$. Ensure the base dimension is divisible by 4 to avoid alignment issues in interior passes.
*   **Degenerate Input:** If $m\_VoxelDim$ is too small, the loop for interior passes may not execute or may produce empty textures.

## Subtopics

### Anisotropic Mipmap Texture Setup
Initializes the compute shader programs and 3D textures required for anisotropic mipmap generation across six face directions. It involves creating the leaf shader from `anisotropic_mipmap_start_comp.glsl` and allocating 3D textures with dimensions $half\_voxel\_dim \times half\_voxel\_dim \times half\_voxel\_dim$ using `GL_RGBA8` internal format, `GL_RGBA` format, `GL_FLOAT` type, and trilinear filtering [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels and the number of mipmap levels. The leaf pass processes $O((m\_VoxelDim/2)^3)$ voxels, and each interior pass processes half the volume of the previous level.
*   **Memory Complexity:** $O(s\_AnisotropicMipmapCount \times (m\_VoxelDim/2)^3)$ for storing the mipmap hierarchy.
*   **Performance:** Depends on the efficiency of the compute shaders and the hardware's support for 3D texture sampling and image stores. The use of barriers ensures correctness but may introduce synchronization overhead.
*   **Strengths:** Supports anisotropic filtering for radiance data, which can improve visual quality at oblique viewing angles.
*   **Limitations:** Requires significant memory for the mipmap hierarchy. The two-phase approach adds complexity to the implementation.
*   **When to use:** When anisotropic filtering is required for voxel radiance and the hardware supports efficient compute shader-based mipmap generation.
*   **When not to use:** For isotropic filtering or when memory bandwidth is a critical bottleneck.

## Background
This method is implemented in the HarshLight project, an unofficial graphics engine/repository [S1, unofficial]. The specific implementation details are derived from the source code of this project.

## Sources
[S1] MangoSister/HarshLight, https://github.com/MangoSister/HarshLight, fetched 2026-10-26 (unofficial)
