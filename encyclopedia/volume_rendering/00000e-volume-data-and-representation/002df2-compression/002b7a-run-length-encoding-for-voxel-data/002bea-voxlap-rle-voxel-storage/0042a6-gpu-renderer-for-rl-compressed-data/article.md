# GPU Renderer for RL-compressed data

Also called: GPU Renderer for Run-Length Compressed Data, VRE Volume Rendering Engine Renderer

## Overview
This method is a GPU-based volume rendering technique designed for data compressed via run-length encoding (RLE), specifically targeting voxelized models with sharp or small details [S1]. It solves the problem of efficiently accessing massive voxel grids (1000³ and larger) by storing only surface voxels uncompressed while compressing inside and outside voxels, achieving compression ratios up to 1:20 for noise-free data [S1]. Unlike methods that require full decompression of the volume, this approach distinguishes between uncompressed surface voxels and compressed interior/exterior regions on-the-fly during ray casting, avoiding the overhead of reconstructing the full volume in memory [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A compressed volume dataset represented as a 2D array of compressed rows, where off-surface areas are RLE-encoded and surface areas are uncompressed runs [S1].
*   **Input:** Three GPU textures: an Index Texture, a Line Texture, and a Surface Texture [S1].
*   **Input:** Ray origin and direction for each sample point in the ray casting process [S1].
*   **Output:** The scalar value (or color/opacity) of the voxel at the requested 3D position $[x, y, z]$ [S1].

**Data Structures and Layout**
The compressed volume is organized into three specific textures to facilitate fast access without full decompression [S1]:
1.  **Index Texture:** A 2D texture storing offsets. For a given $y, z$ coordinate, it provides the offset into the Line Texture where the compressed $yz$-line data begins [S1].
2.  **Line Texture:** Stores the compressed $yz$-lines. Each line consists of a sequence of count-offset pairs representing RLE runs along the $x$-axis [S1].
3.  **Surface Texture:** Stores the uncompressed values of surface voxels. It also stores special values for inside and outside regions [S1].

**Numbered Steps**
To retrieve the value of a voxel at position $[x, y, z]$, the fragment shader executes the following steps [S1]:

1.  **Locate Line:** The shader uses the $y$ and $z$ coordinates to look up the Index Texture. This returns an offset pointing to the start of the compressed $yz$-line in the Line Texture [S1].
2.  **Find Run:** The shader reads count-offset pairs from the Line Texture sequentially until it finds the pair corresponding to the requested $x$ coordinate. A pair $(count, offset)$ is the correct match if the cumulative sum of previous counts satisfies:
    $$ \sum_{i=0}^{k-1} count_i \le x < \sum_{i=0}^{k} count_i $$
    where $k$ is the index of the current pair [S1].
3.  **Determine Voxel Type:** The shader examines the $offset$ value of the found pair to determine the voxel type [S1]:
    *   **Inside/Outside:** If the voxel belongs to the interior or exterior of the object, the offset points to a special value in the Surface Texture. These special values are distinct from surface voxel values: $O$ denotes an outside voxel and $I$ denotes an inside voxel [S1]. The shader returns the value stored at the offset directly [S1].
    *   **Surface:** If the voxel is part of the surface area, the offset indicates the starting position of the uncompressed run in the Surface Texture [S1].
4.  **Retrieve Surface Value:** If the voxel is in the surface area (indicated by $offset > 1$), the shader calculates the final position in the Surface Texture. It adds the position of the voxel within the specific uncompressed run (the local index within the run) to the base offset retrieved from the pair [S1]. The value is then retrieved from the Surface Texture at this calculated position [S1].

**Pseudocode**
The following Python-like pseudocode implements the voxel lookup logic described in the source [S1]. It assumes access to the three textures via helper functions `lookup_index`, `read_line_texture`, and `read_surface_texture`.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_voxel_value(x, y, z, index_texture, line_texture, surface_texture):
    """
    Retrieves the value of a voxel at [x, y, z] from RL-compressed data.
    
    Args:
        x, y, z: Integer coordinates of the voxel.
        index_texture: 2D array mapping (y, z) to an offset in line_texture.
        line_texture: 1D array of count-offset pairs for compressed lines.
        surface_texture: 1D array storing uncompressed surface values and special I/O markers.
        
    Returns:
        The voxel value (scalar, color, etc.).
    """
    
    # Step 1: Look up the offset to the line texture for the requested y, z
    line_offset = lookup_index(index_texture, y, z)
    
    # Step 2: Read count-offset pairs from the line texture to find the run containing x
    current_x = 0
    pair_index = 0
    
    while True:
        # Read the next count-offset pair from the line texture
        # Assuming pairs are stored sequentially starting at line_offset
        count = read_line_texture(line_texture, line_offset + pair_index * 2)
        offset = read_line_texture(line_texture, line_offset + pair_index * 2 + 1)
        
        # Check if x falls within this run
        if current_x <= x < current_x + count:
            # Found the correct run
            break
        
        current_x += count
        pair_index += 1
        
    # Step 3 & 4: Determine voxel type and retrieve value
    if offset == 0 or offset == 1:
        # Special values for Inside (I) or Outside (O)
        # The offset itself might indicate the type, or the value at that offset
        # According to source: "offset points to value common for all inside and outside voxels"
        # We assume the value at 'offset' in surface texture is the special marker
        value = read_surface_texture(surface_texture, offset)
    else:
        # Surface voxel
        # Calculate position within the uncompressed run
        local_index = x - current_x
        
        # Final position in surface texture
        final_position = offset + local_index
        
        value = read_surface_texture(surface_texture, final_position)
        
    return value
```

**Parameters**
*   **Compression Ratio:** Up to 1:20 for noise-free voxelisation data [S1].
*   **Data Scale:** Grids of $1000^3$ and more voxels [S1].

**Edge Cases and Pitfalls**
*   **Special Values:** The values $O$ (outside) and $I$ (inside) are distinct from the value $C$ of requested surface voxels. Care must be taken to distinguish between an offset pointing to a surface run and an offset pointing to a special interior/exterior value [S1].
*   **Offset Interpretation:** An offset value of 1 or 0 may indicate special handling for inside/outside voxels, whereas offsets $>1$ indicate surface runs [S1].
*   **Boundary Conditions:** The search for the correct count-offset pair must handle the case where $x$ is at the very end of a run or the volume. The condition $current\_x \le x < current\_x + count$ ensures correct range checking [S1].

## Operating characteristics
*   **Time Complexity:** Lookup time is proportional to the number of RLE runs along the $x$-axis for a given $y, z$ line. In the worst case (no compression), this is $O(N_x)$, but typically much faster due to compression [S1].
*   **Memory Complexity:** Significantly reduced compared to full volume storage. Off-surface areas are compressed, achieving up to 1:20 compression ratio [S1].
*   **Performance:** Suitable for large grids ($1000^3+$) where only surface details are of interest [S1].
*   **Strengths:** Avoids full decompression overhead; fast access to surface voxels; high compression for noise-free data [S1].
*   **Limitations:** Only effective for data with distinct inside/outside regions and sharp surfaces; less effective for noisy or diffuse volumetric data [S1].
*   **When to Use:** Volumetric data obtained by voxelization of complex models defined by implicit functions with sharp or small details [S1].
*   **When Not to Use:** Data without clear surface boundaries or with significant noise, where compression ratios would be poor [S1].

## Background
The motivation for this renderer stems from the domain of volumetric data obtained by voxelization of complex models defined by implicit functions [S1]. Such models, especially those with sharp or small details, require dense sampling, resulting in huge grids of $1000^3$ and more voxels [S1]. However, since only the surfaces of the objects are typically of interest, it is sufficient to fully represent only voxels close to the surfaces, while compressing the rest [S1]. The technique builds on RLE compression methods described by Novotný [S1]. It was implemented as part of the VRE volume rendering engine, detailed in a paper by Parulek et al. [S1].

## Sources
[S1] The VRE volume rendering engine, https://doi.org/10.1145/1925059.1925072, fetched 2026-11-05
