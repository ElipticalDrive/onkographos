# 3D Block Stitching and Resampling

Also called: Stitching and resampling for 3D block construction

## Overview
This method constructs 3D volumetric blocks from 2D image tiles by stitching them into a 3D target grid and applying a 3D pre-filter to avoid aliasing during resampling [S1]. It is distinguished from sibling methods by its explicit separation of 2D pre-filtering (mipmap generation) from the visualization-driven 3D stitching, utilizing GPU-based Voronoi-like depth testing to resolve overlapping tile contributions without blending [S1].

## How it works
**Inputs and Outputs**
*   **Input**: A set of $N$ 2D source image tiles $I_k^{2d}$ with associated affine alignment matrices $\tilde{T}_k$.
*   **Output**: A 3D volume block $I^{3d}$ of requested resolution, typically extracted from a larger constructed block.

**Data Structures**
*   **2D Mipmap Archive**: For each source tile $k$, a hierarchy of pre-filtered 2D images $\tilde{I}_k^{2d}$ is stored. Each level $l$ contains images filtered by $W_{x,y}$.
*   **Texture Pool**: A pool of 2D GPU textures holding the sub-tiles fetched from the mipmap archive for the current resampling operation.
*   **Off-screen Buffer**: A render target used to store the depth values for the Voronoi selection during stitching.

**Parameters**
*   $W_{x,y}$: 2D filter kernel in the $(x, y)$ plane.
*   $W_z$: 1D filter kernel along the $z$ axis. In the cited implementation, this is a box filter of size one (nearest-neighbor) [S1].
*   $S^{2d}$: The stitching operator that maps 2D tiles into the 3D grid.
*   $\pi_z$: Projection onto the $z$-coordinate.
*   Block Size: Constructed blocks are $512 \times 512 \times 32$ voxels to optimize disk I/O, from which $32^3$ blocks are extracted [S1].
*   Resolution Level $m$: The threshold where the strategy switches from stitching to down-sampling. Defined such that it corresponds to the resolution of one sub-tile times the number of image tiles comprising the full volume [S1].

**Algorithm Steps**

1.  **Raw Tile Pre-processing (Mipmap Generation)**
    For each source image tile $I_k^{2d}$, compute the pre-filtered 2D images $\tilde{I}_k^{2d}$ by applying the 2D filter $W_{x,y}$:
    $$
    \tilde{I}_k^{2d}(x) = \sum_i W_{x,y}(x - x_i) I_k^{2d}(x_i)
    $$
    Store these levels in the visualization archive. Note that this step approximates the full 3D pre-filtering by applying $W_{x,y}$ before stitching [S1].

2.  **Tile Fetching**
    Upon a 3D block request at resolution level $l$:
    *   If $l \le m$: Fetch the required sub-tiles from the correct resolution level of the 2D image tile mipmaps.
    *   If $l > m$: Compute the entire level $l=m$ via stitching/resampling first. Then, compute the block for $l > m$ via simple down-sampling of the level $m$ result [S1].

3.  **GPU Stitching and Resampling**
    Perform stitching and resampling directly on the 3D target grid using texture mapping and fragment shaders:
    a.  Download the fetched sub-tiles into the pool of 2D texture maps.
    b.  Render all contributing sub-tiles to an off-screen buffer. Use the alignment matrix $\tilde{T}_k$ as the texture matrix.
    c.  For each fragment, set the depth value to the Euclidean distance from the fragment's position to the center of its respective source tile.
    d.  Perform a depth test (minimum distance) to select the pixel with the minimum distance to its tile center. This implements the stitching operator $S^{2d}$, selecting one measurement value over a blend [S1].
    e.  Sample the selected texture using a reconstruction filter (GPU bi-linear or a higher-order filter in the fragment shader).

4.  **3D Volume Assembly**
    The resampled 3D volume $I^{3d}(x)$ is computed by applying the 1D filter $W_z$ to the stitched 2D data:
    $$
    I^{3d}(x) = \sum_i W_z(\pi_z(x - x_i)) \cdot S^{2d}_{(\tilde{T}_1, \tilde{I}_1^{2d}), \dots, (\tilde{T}_N, \tilde{I}_N^{2d})}(x_i)
    $$
    where $x_i$ ranges over a 1D neighborhood of $x$ in the $z$-direction [S1].

5.  **Caching and Extraction**
    Extract the requested $32^3$ block from the constructed $512 \times 512 \times 32$ block. If construction and visualization are on separate nodes, cache the resampled 3D blocks in memory and optionally on fast local SSD storage [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_3d_block(requested_block_origin, requested_resolution_level):
    # Determine if we use stitching or down-sampling
    if requested_resolution_level > m:
        # Compute level m first if not cached
        if level_m_not_cached():
            full_level_m = stitch_and_resample(requested_resolution_level=m)
            cache_level_m(full_level_m)
        # Down-sample from level m
        return downsample(level_m_data, requested_resolution_level)
    
    # Fetch sub-tiles from 2D mipmap archive
    sub_tiles = fetch_subtiles_from_mipmaps(requested_resolution_level)
    
    # GPU Stitching and Resampling
    # 1. Upload sub-tiles to GPU texture pool
    gpu_textures = upload_to_gpu(sub_tiles)
    
    # 2. Render to off-screen buffer for Voronoi selection
    #    Use alignment matrices as texture matrices
    #    Set fragment depth to distance from tile center
    offscreen_buffer = render_voronoi_selection(gpu_textures)
    
    # 3. Sample the selected pixels with reconstruction filter
    #    Apply 1D filter Wz along z-axis
    resampled_block = sample_and_filter_1d(offscreen_buffer, W_z)
    
    # 4. Extract requested block from larger constructed block
    #    Constructed block is 512x512x32, extract 32x32x32
    final_block = extract_block(resampled_block, requested_block_origin)
    
    # 5. Cache if necessary
    if separate_nodes:
        cache_in_memory(final_block)
        cache_on_ssd(final_block)
        
    return final_block

def stitch_and_resample(resolution_level):
    # Full volume stitching at level m
    # Fetch all necessary tiles for the full volume at this level
    all_tiles = fetch_all_tiles(resolution_level)
    
    # Render all tiles to off-screen buffer
    # Depth = distance to tile center
    full_voronoi_buffer = render_voronoi_selection(all_tiles)
    
    # Sample and filter
    full_volume = sample_and_filter_1d(full_voronoi_buffer, W_z)
    
    return full_volume
```

**Edge Cases and Pitfalls**
*   **Approximation Error**: The method approximates the ideal 3D pre-filter because $W_{x,y}$ is applied before stitching, not afterward. This decouples stitching from pre-filtering and sub-sampling [S1].
*   **Anisotropy**: Using a box filter for $W_z$ (nearest-neighbor) may cause quality degradation with highly anisotropic data. The framework allows integrating other $W_z$ filters, such as interpolating intermediate slices, if higher quality is required [S1].
*   **Tile Overlap**: The Voronoi selection heuristic (minimum distance to tile center) is used to resolve overlaps. This prefers a single measurement value over a blend, which may be visually distinct from averaging methods [S1].

## Operating characteristics
*   **Time Complexity**: The stitching and resampling step is performed on the GPU using fragment shaders, making it efficient for visualization-driven processing [S1]. The complexity is proportional to the number of visible 3D blocks and the number of contributing tiles per block.
*   **Memory Complexity**: Requires storing 2D mipmaps for all source tiles. Additionally, large $512 \times 512 \times 32$ blocks are constructed in memory/SSD to avoid inefficient disk accesses of small $32^3$ blocks [S1].
*   **Strengths**: Avoids aliasing through pre-filtering; decouples 2D filtering from 3D stitching for performance; handles overlapping tiles efficiently via GPU depth testing; supports multi-resolution hierarchies.
*   **Limitations**: Approximates the ideal 3D pre-filter; relies on GPU hardware for efficient stitching; requires significant memory for large constructed blocks and mipmap archives.
*   **When to use**: When visualizing petascale microscopy data streams where 2D image tiles need to be converted to 3D volumes on-the-fly with minimal aliasing and efficient disk I/O [S1].
*   **When not to use**: When exact 3D pre-filtering is required and the approximation error is unacceptable; when GPU resources are unavailable.

## Background
This method was developed for the interactive exploration of petascale microscopy data streams [S1]. It addresses the challenge of converting 2D image tiles from electron microscopy (EM) into 3D volumes suitable for real-time visualization, handling the large data volumes and multi-resolution requirements typical of such datasets [S1]. The approach is inspired by techniques such as Hoff et al.’s Voronoi computation for tile selection [S1] and clipmaps for multi-resolution management [S1].

## Sources
[S1] Interactive Volume Exploration of Petascale Microscopy Data Streams, https://vccvisualization.org/publications/2012_hadwiger_petascale.pdf, fetched 2026-10-28
