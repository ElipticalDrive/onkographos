# Multiresolution Volume Sampling: Intrablock

Also called: Block-local Sampling, Intrablock Sampling

## Overview
Intrablock sampling is a method for accessing volumetric data within a single block of a multiresolution hierarchy, specifically handling boundary conditions and resolution mismatches between adjacent blocks [S1, unofficial]. It differs from Interblock sampling, which interpolates across block boundaries, by restricting the lookup to a single spatial region to avoid discontinuities or complex weight calculations at interfaces [S1, unofficial]. This method is chosen when rendering requires consistent sampling within a defined spatial partition without blending data from neighboring partitions.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D sampling position $p$ in world or texture space; a multiresolution volume structure composed of blocks with varying resolutions.
*   **Output:** The interpolated scalar value $V$ at position $p$ from the appropriate block.

**Data Structures:**
1.  **Index Texture:** A 2D or 3D texture storing metadata for each block. Fields typically include:
    *   Block ID or pointer.
    *   Block dimensions (resolution $N_x, N_y, N_z$).
    *   World-space bounding box or origin coordinates.
2.  **Packed Texture:** The actual volumetric data, organized into mipmaps or separate textures per block, storing the scalar values.

**Algorithm Steps:**
1.  **Determine Closest Block:** Identify the block containing or closest to the sampling position $p$. This is often referred to as "nearest block sampling" [S1, unofficial].
2.  **Lookup Block Metadata:** Access the Index Texture using the block identifier to retrieve the block's size and spatial location (bounding box) [S1, unofficial].
3.  **Clamp Sampling Position:** Transform $p$ into the local coordinate system of the identified block. Clamp this local position to the block's sample boundaries to ensure the lookup remains within valid data [S1, unofficial].
4.  **Lookup Interpolated Sample:** Perform trilinear (or higher-order) interpolation on the Packed Texture using the clamped local coordinates to retrieve the final value [S1, unofficial].

**Formulas and Parameters:**
*   **Boundary Handling:** To handle resolution differences or gaps, an inset $\delta$ is applied from the block boundary during mean-value downsampling [S1, unofficial]. This prevents sampling from invalid or undefined regions at the edges of lower-resolution blocks.
*   **Downsampling Methods:** The method supports various downsampling strategies for generating coarser levels, including:
    *   Boundary aligned
    *   Centered
    *   Sample replication
    *   Average value
    *   Nearest neighbor
    *   Wavelets [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def intrablock_sample(p_world, index_texture, packed_texture):
    # Step 1: Determine closest block
    # Find the block ID that contains or is nearest to p_world
    block_id = find_closest_block(p_world, index_texture)
    
    # Step 2: Lookup block size and location in index texture
    block_meta = index_texture.read(block_id)
    block_origin = block_meta.origin
    block_size = block_meta.resolution # (nx, ny, nz)
    block_extent = block_meta.extent   # (dx, dy, dz)
    
    # Step 3: Clamp sampling position to the block’s sample boundary
    # Convert world position to local block coordinates
    p_local = (p_world - block_origin) / block_extent
    
    # Clamp to [0, 1] range to stay within block boundaries
    p_local = clamp(p_local, 0.0, 1.0)
    
    # Note: If using mean-value downsampling with inset delta,
    # the effective sampling range might be [delta, 1-delta].
    # For standard intrablock, clamping to [0,1] is primary.
    
    # Step 4: Lookup interpolated sample in the packed texture
    # The packed texture may be structured as a 3D texture or a 2D array
    # where each slice corresponds to a block or mipmap level.
    value = packed_texture.sample_trilinear(block_id, p_local)
    
    return value
```

## Subtopics

### Multiresolution Volume Sampling: Interblock
Interblock sampling handles interpolation between adjacent blocks in a multiresolution volume using edge weights based on distance and boundary splits [S1, unofficial]. It is chosen when continuity across block boundaries is required, unlike Intrablock which isolates sampling. It calculates weights using three methods: Minimum distance edge weight, Boundary split edge weight, and Maximum distance edge weight [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample, assuming constant-time block lookup and texture sampling.
*   **Memory Complexity:** Depends on the multiresolution hierarchy storage (e.g., octree or brick structure).
*   **Strengths:** Simple implementation; avoids artifacts at block boundaries by restricting sampling; efficient for GPU raycasting where fragment shaders can conditionally select blocks [S1, unofficial].
*   **Limitations:** Potential discontinuities at block boundaries if not combined with Interblock methods; requires careful handling of resolution mismatches via insets or downsampling strategies [S1, unofficial].
*   **When to Use:** When rendering within a single resolution level or when block boundaries are not visually critical; when hardware constraints favor simple, localized lookups.
*   **When Not to Use:** When high visual continuity across varying resolution blocks is required; in such cases, Interblock sampling is preferred.

## Background
This method originates from research into GPU-based volume raycasting, specifically in the context of advanced illumination techniques. It was developed by Markus Hadwiger at the VRVIS Research Center, Vienna, Austria, as part of work on efficient volume rendering on graphics hardware [S1, unofficial]. The approach leverages the flexibility of fragment shader conditionals and deferred shading to manage complex filtering and shading at high frame rates [S1, unofficial].

## Sources
[S1] GPU-Based Volume Ray-Casting with Advanced Illumination, https://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/viscom/2008/advanced_illumination.pdf, fetched 2026-10-29 (unofficial)
