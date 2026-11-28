# Visualization-Driven Volume Data Construction

Also called: On-the-fly Stitching and Resampling, Visualization-Driven Volume Construction

## Overview
This method constructs 3D volume blocks on-demand only when requested by a ray-caster for visible blocks at a specific resolution level, making the construction process independent of the total data size [S1, unofficial][S2]. Unlike octree-based systems that require lower or higher resolutions to be available, this approach allows direct resampling to the requested grid resolution without intermediate up-sampling or down-sampling steps [S2]. It solves the problem of efficiently visualizing petascale microscopy data streams by decoupling data construction from the visualization stage, ensuring that only visible and necessary data are processed [S1, unofficial][S2].

## How it works
**Inputs:**
*   A 3D target block definition (position and resolution level) issued by the ray-caster.
*   A visualization archive containing 2D image tiles (sub-tiles) at various mipmap levels.
*   An index structure based on Morton order traversal for efficient sub-tile retrieval.
*   Alignment matrices associated with each image tile.

**Outputs:**
*   A 3D volume block at the requested resolution, transmitted to the visualization stage.

**Data Structures:**
*   **Morton Index:** A compact index structure fitting in main memory, mapping spatial coordinates to 2D sub-tiles. It enables $O(\log_2 N)$ retrieval, where $N$ is the number of slices [S2].
*   **Alignment Matrix:** A transformation matrix associated with each image tile, used to determine the correct spatial positioning during stitching [S1, unofficial].

**Steps:**
1.  **Request Validation:** The ray-caster issues a 3D block construction request only if the block is visible on screen and cannot be fulfilled from existing caches [S1, unofficial][S2].
2.  **Sub-tile Determination:** Determine all 2D sub-tiles that intersect the 3D target block. This reduces the 3D intersection problem to a 2D problem using the Morton index [S2].
3.  **Resolution Selection:** For each identified sub-tile, choose the mipmap level that matches the desired resampling resolution [S2].
4.  **Fetch:** Retrieve the 2D image sub-tiles from the visualization archive using the compact index structure [S1, unofficial].
5.  **Stitching and Resampling:** Stitch and resample the 2D sub-tiles directly into the 3D target grid.
    *   For anisotropic data (e.g., aspect ratio 1:10), this is simplified by stitching sub-tiles in 2D without actual 3D filtering, storing results into the correct 3D location [S1, unofficial].
    *   For each 2D slice comprising the 3D target block, the corresponding 2D sub-tile images are resampled directly into the corresponding location in the 3D target grid [S2].
    *   The alignment matrix is used to determine the precise stitching coordinates [S1, unofficial].
    *   Reconstruction filters (e.g., GPU bi-linear filtering or higher-order filters in fragment shaders) are applied during resampling [S1, unofficial].
6.  **Transmission:** Transmit the resampled 3D block to the visualization stage [S2].
7.  **Empty Block Handling:** If the requested data has not yet been scanned by the microscope, report the block as empty and skip construction [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_volume_block(target_block, visualization_archive, morton_index):
    """
    Constructs a 3D volume block on-demand.
    
    Args:
        target_block: Object containing position (x, y, z) and resolution level.
        visualization_archive: Storage for 2D image tiles with mipmap levels.
        morton_index: Compact index structure for sub-tile retrieval.
        
    Returns:
        3D volume block at requested resolution, or None if empty/unavailable.
    """
    # Step 1: Check visibility and cache (handled by caller, but conceptually here)
    if not is_visible(target_block) or is_in_cache(target_block):
        return None

    # Step 2: Determine intersecting 2D sub-tiles
    # Reduces 3D problem to 2D using Morton index
    intersecting_subtiles = morton_index.find_intersecting(target_block)
    
    if not intersecting_subtiles:
        # Step 7: Handle unscanned/empty data
        return None

    # Initialize empty 3D grid for the target block
    target_grid = initialize_3d_grid(target_block.resolution)

    # Step 3 & 4: Fetch and Select Mipmap Level
    for subt in intersecting_subtiles:
        # Choose mipmap level matching target resolution
        mipmap_level = select_mipmap(subt, target_block.resolution)
        
        # Fetch the 2D sub-tile image
        image_tile = visualization_archive.fetch(subt.id, mipmap_level)
        
        if image_tile is None:
            continue

        # Step 5: Stitch and Resample
        # Use alignment matrix to determine position
        alignment = subt.alignment_matrix
        
        # For anisotropic data, simplify to 2D stitching into 3D location
        # Resample 2D sub-tile into the corresponding 2D slice of the 3D target grid
        # Apply reconstruction filter (e.g., bi-linear)
        resampled_slice = resample_2d_into_3d_slice(
            image_tile, 
            alignment, 
            target_grid, 
            target_block.z_position,
            filter_type="bilinear" # or higher-order shader
        )
        
        # Store result in correct 3D location
        target_grid.update_slice(resampled_slice)

    # Step 6: Transmit
    return target_grid
```

**Parameters:**
*   **Reconstruction Filter:** GPU bi-linear filtering or higher-order filters implemented in the fragment shader [S1, unofficial].
*   **Index Structure:** Morton order traversal [S1, unofficial][S2].

**Edge Cases and Pitfalls:**
*   **Unscanned Data:** If the requested data has not been scanned, the block must be reported as empty to avoid blocking [S1, unofficial].
*   **Anisotropy:** For highly anisotropic data (e.g., 1:10 aspect ratio), 3D filtering is simplified to 2D stitching to maintain performance [S1, unofficial].
*   **Cache Misses:** Requests originate from cache misses in the visualization stage; efficient handling prevents rendering delays [S2].

## Operating characteristics
*   **Time Complexity:** Sub-tile retrieval is $O(\log_2 N)$, where $N$ is the number of slices [S2]. Registration on a 256x256 reference grid takes approximately 3 ms on an NVIDIA Fermi GPU (GTX 580) [S1, unofficial].
*   **Memory Complexity:** The index structure is compact and fits in main memory [S1, unofficial][S2]. Construction is independent of total data size, relying only on visible blocks [S1, unofficial].
*   **Strengths:** Decouples construction from data size; allows direct resampling to requested resolution; efficient for petascale data streams [S1, unofficial][S2].
*   **Limitations:** Requires a robust visualization archive and index structure; performance depends on GPU texture mapping and shader efficiency [S1, unofficial].
*   **When to Use:** Ideal for interactive visualization of large, streaming microscopy data where only a fraction of the volume is visible at any time [S1, unofficial][S2].
*   **When Not to Use:** When full-volume pre-computation is feasible and latency is not a concern, or when isotropic data benefits from full 3D filtering [S1, unofficial].

## Background
This method was developed for the petascale visualization of microscopy data streams, specifically for exploring the connectome [S1, unofficial][S2]. It addresses the limitations of octree-based systems by eliminating the need for multi-resolution hierarchies in the storage format, allowing for direct on-the-fly construction [S2]. The approach was implemented in a system designed to handle real-time visualization of massive datasets, leveraging GPU-based stitching and resampling [S1, unofficial].

## Sources
[S1] Petascale Volume Visualization of Microscopy Data Streams, https://vcg.seas.harvard.edu/publications/20130101-exploring-the-connectome-petascale-volume-visualization-of-microscopy-data-streams/paper, fetched 2026-10-27 (unofficial)
[S2] PDF Interactive Volume Exploration of Petascale Microscopy Data Streams ..., https://vccvisualization.org/publications/2012_hadwiger_petascale.pdf, fetched 2026-10-28
