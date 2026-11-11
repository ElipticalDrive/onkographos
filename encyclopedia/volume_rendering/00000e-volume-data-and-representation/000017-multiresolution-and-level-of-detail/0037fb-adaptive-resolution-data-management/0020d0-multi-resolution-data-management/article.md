# Multi-resolution Data Management

Also called: Flat Blocking Multi-resolution, Flat Blocking Multi-resolution Data Management, Flat Multiresolution Blocking, Multi-resolution Data Management for Light, Multiresolution Blocking for LAO, Multiresolution volume management

## Overview
Multi-resolution Data Management is a data structure and processing approach designed to reduce computation and storage in volume rendering by avoiding superfluous sampling in empty or homogeneous spaces [S1, S2]. It solves the problem of high memory bandwidth and processing costs for large-scale volumes by organizing data into small blocks assigned a Level-of-Detail (LOD) based on their content in the transfer function (TF) domain [S1, S2]. Unlike spatially packed structures, this method organizes blocks by their LOD rather than spatial location, allowing for significant texture size reduction (at least a factor of 8) and skipping calculations for empty regions [S2]. It sets itself apart from sibling methods by using a "flat" blocking scheme where resolution selection is driven by image quality preservation in the TF domain, enabling graceful data reduction [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A high-resolution volume dataset (e.g., $512^3$ voxels) and a transfer function (TF) defining opacity and color mapping [S1, S2].
*   **Output:** A multi-resolution data structure consisting of a packed 3D texture containing blocks of varying resolutions, an index texture specifying block locations/resolutions, and a reverse index map for coordinate translation [S1].

**Data Structures:**
1.  **Packed 3D Texture:** Contains all blocks of different resolutions. Blocks are not organized by spatial location but by their LOD [S1, S2].
2.  **Index Texture:** Specifies the location and resolution of each block within the packed texture [S1].
3.  **Reverse Index Map ($P \to V$):** Maps packed coordinates ($P$) to volume coordinates ($V$). This map holds volume coordinates for all non-empty blocks [S1].

**Parameters:**
*   **Block Size:** Each block consists of $16^3$ voxels [S1, S2].
*   **Resolution Levels:** Discrete numbers in the range $[0, \lambda_h]$, where $\lambda_h$ is the highest resolution level. In the standard implementation, $\lambda_h = 4$ [S1].
*   **Minimum Resolution:** Blocks with the two lowest resolution levels ($1^3$ and $2^3$) are ignored in the reverse index map to reduce its size [S1]. All non-empty blocks have a minimum resolution of $4^3$ voxels [S1].

**Steps:**
1.  **Blocking:** Divide the volume into small cubes of $16^3$ voxels [S1, S2].
2.  **LOD Assignment:** Assign a Level-of-Detail (LOD) to each block. This determination is based on optimizing image quality through the block's content in the TF domain [S1, S2].
    *   Empty or non-contributing blocks are assigned the lowest resolution or ignored entirely [S1, S2].
    *   Remaining blocks are assigned higher resolutions, with a minimum of $4^3$ voxels [S1].
3.  **Packing:** Organize the blocks into the multi-resolution structure by their LOD, not by spatial location [S1, S2]. Pack all blocks into a single 3D texture [S1].
4.  **Indexing:** Create an index texture that specifies the location and resolution of each block in the packed texture [S1].
5.  **Reverse Mapping:** Generate a reverse index map ($P \to V$) for all non-empty blocks with resolution $\ge 4^3$. Blocks with resolution $1^3$ and $2^3$ are excluded from this map to reduce its size by a factor of 64 [S1].
6.  **Computation:** Perform light computations (e.g., Local Ambient Occlusion) directly in the coordinates of the multi-resolution data structure [S2].
    *   Skip calculations for empty blocks [S2].
    *   This allows the method to skip calculations for regions that are empty, reducing the number of render passes [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_resolution_data_management(volume_data, transfer_function, block_size=16, lambda_h=4):
    """
    volume_data: 3D array of voxel values (e.g., 512x512x512)
    transfer_function: function mapping voxel value to (opacity, color)
    block_size: integer, size of each block in voxels (default 16)
    lambda_h: integer, highest resolution level (default 4)
    
    Returns:
    packed_texture: 3D array containing blocks of varying resolutions
    index_texture: 3D array specifying location and resolution of blocks
    reverse_map: dictionary mapping packed coordinates to volume coordinates
    """
    
    # Step 1: Blocking
    # Divide volume into 16^3 voxel blocks
    blocks = []
    num_blocks_x = volume_data.shape[0] // block_size
    num_blocks_y = volume_data.shape[1] // block_size
    num_blocks_z = volume_data.shape[2] // block_size
    
    for x in range(num_blocks_x):
        for y in range(num_blocks_y):
            for z in range(num_blocks_z):
                block_data = volume_data[
                    x*block_size:(x+1)*block_size,
                    y*block_size:(y+1)*block_size,
                    z*block_size:(z+1)*block_size
                ]
                blocks.append({
                    'volume_coords': (x, y, z),
                    'data': block_data,
                    'resolution': None
                })
    
    # Step 2: LOD Assignment
    # Assign LOD based on TF domain content
    for block in blocks:
        # Calculate contribution in TF domain
        # If block is empty or non-contributing, assign lowest LOD or ignore
        opacity_sum = sum(transfer_function(v)[0] for v in block['data'].flatten())
        
        if opacity_sum == 0:
            block['resolution'] = 0 # Ignored
        else:
            # Determine resolution level based on image quality optimization
            # In implementation, lambda_h = 4
            # Minimum resolution for non-empty blocks is 4^3
            # Resolution levels are discrete numbers in [0..4]
            # For simplicity, assume a heuristic based on opacity_sum
            # In practice, this might involve more complex TF analysis
            if opacity_sum > threshold_high:
                block['resolution'] = lambda_h
            elif opacity_sum > threshold_low:
                block['resolution'] = 3
            else:
                block['resolution'] = 2 # Minimum 4^3 voxels implies resolution >= 2 if base is 16^3?
                # Note: The source says minimum resolution is 4^3 voxels.
                # If base block is 16^3, resolution levels might correspond to:
                # Level 4: 16^3, Level 3: 8^3, Level 2: 4^3, Level 1: 2^3, Level 0: 1^3
                # So resolution 2 corresponds to 4^3 voxels.
    
    # Step 3: Packing
    # Organize blocks by LOD, not spatial location
    # Pack into a single 3D texture
    packed_texture = []
    index_texture = []
    
    # Group blocks by resolution
    blocks_by_res = {i: [] for i in range(lambda_h + 1)}
    for block in blocks:
        if block['resolution'] > 0: # Ignore empty/lowest res for packing if needed
            blocks_by_res[block['resolution']].append(block)
    
    # Pack blocks into texture
    # The exact packing algorithm is not specified, but blocks are stored in a 3D texture
    # Index texture specifies location and resolution
    current_packed_pos = (0, 0, 0)
    for res in range(lambda_h, -1, -1):
        for block in blocks_by_res[res]:
            # Determine resolution size in voxels
            # Assuming resolution level r corresponds to (16 / 2^(4-r))^3 ?
            # Source says min resolution is 4^3 for non-empty blocks.
            # Let's assume resolution level r maps to size s_r
            # Level 4: 16^3, Level 3: 8^3, Level 2: 4^3
            # Level 1: 2^3, Level 0: 1^3
            if res == 4: size = 16
            elif res == 3: size = 8
            elif res == 2: size = 4
            elif res == 1: size = 2
            else: size = 1
            
            # Store block data in packed texture
            # This is a simplified representation
            packed_texture.append((block['data'], current_packed_pos, res))
            
            # Add to index texture
            index_texture.append({
                'packed_pos': current_packed_pos,
                'resolution': res,
                'volume_coords': block['volume_coords']
            })
            
            # Update packed position
            # This is a placeholder for actual packing logic
            current_packed_pos = (current_packed_pos[0] + 1, current_packed_pos[1], current_packed_pos[2])
            if current_packed_pos[0] >= texture_width:
                current_packed_pos = (0, current_packed_pos[1] + 1, current_packed_pos[2])
            if current_packed_pos[1] >= texture_height:
                current_packed_pos = (0, 0, current_packed_pos[2] + 1)
    
    # Step 4: Reverse Mapping
    # Create reverse index map (P->V) for non-empty blocks with resolution >= 4^3
    reverse_map = {}
    for entry in index_texture:
        if entry['resolution'] >= 2: # 4^3 voxels
            # Map packed coordinates to volume coordinates
            # Packed coordinates are in the packed texture space
            # Volume coordinates are in the original volume space
            reverse_map[entry['packed_pos']] = entry['volume_coords']
            
    return packed_texture, index_texture, reverse_map
```

**Edge Cases and Pitfalls:**
*   **Interpolation Artifacts:** Since blocks are not packed spatially, interpolation can cause artifacts. Solution: Clamp sample locations to a proper distance from block borders [S2]. This is sufficient for $\alpha_{lp}$ and $I_{final}$ but not for global opacity $\alpha_g$ [S2].
*   **Reverse Map Size:** A block represented by a single voxel would require a reverse index map of the same size as the packed texture. This is avoided by ignoring blocks below a certain minimum resolution level ($1^3$ and $2^3$) [S1].

## Operating characteristics
*   **Time Complexity:** The method reduces the number of voxels processed, leading to a speed-up in LAO processing [S1, S2]. The complexity depends on the number of non-empty blocks and their assigned resolutions.
*   **Memory Complexity:** Can reduce texture sizes by at least a factor of 8 compared to full resolution volumes [S2]. For a $512^3$ volume, it can be stored using $256 \times 256 \times 128$ voxels without significant loss of information [S2].
*   **Strengths:** Significant reduction in storage and computation for light volumes by skipping empty regions and using lower resolution blocks [S2]. Allows for interactive frame rates during illumination updates [S2].
*   **Limitations:** Interpolation artifacts due to non-spatial packing [S2]. Not suitable for computing global opacity $\alpha_g$ without clamping, which is not a possible solution due to large data reduction [S2].
*   **When to use:** When rendering large-scale volumes with significant empty or homogeneous space, and when interactive frame rates are required for illumination updates [S1, S2].
*   **When not to use:** When global opacity calculations are critical and cannot tolerate the artifacts from non-spatial packing [S2].

## Background
*   **Origin:** The method was introduced by Hernell et al. in 2004 [LLYM04] and further detailed in 2006 [LLY06] and 2007 [HLY07] [S1, S2].
*   **History:** It was developed to support efficient ambient and emissive tissue illumination in volume rendering, specifically for the Local Ambient Occlusion (LAO) pipeline [S1].
*   **Usage:** It is used in medical volume rendering to enhance perception of shapes and tissue properties [S2].

## Sources
[S1] Efficient Ambient and Emissive Tissue Illumination using Local Occlusion in Multiresolution Volume Rendering, https://doi.org/10.2312/vg/vg07/001-008, fetched 2026-10-26
[S2] Interactive Global Light Propagation in Direct Volume Rendering using Local Piecewise Integration, https://doi.org/10.2312/vg/vg-pbg08/105-112, fetched 2026-10-26
