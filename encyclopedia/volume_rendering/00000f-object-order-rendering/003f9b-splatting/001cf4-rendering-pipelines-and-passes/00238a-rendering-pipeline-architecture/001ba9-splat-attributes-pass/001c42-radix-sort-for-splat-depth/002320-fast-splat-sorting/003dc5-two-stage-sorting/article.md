# Two-Stage Sorting

Also called: Schutz et al. sorting, Separate Sorting, Two-pass sorting

## Overview
Two-Stage Sorting is a sorting optimization for volume rendering pipelines that splits the generation of depth-sorted per-tile splat lists into two separate radix sort routines to reduce VRAM usage and computational complexity [S1, unofficial]. It sets itself apart from single-stage approaches by decoupling depth ordering from tile assignment, allowing the use of smaller key sizes (32-bit and 16-bit) instead of a single large key [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $n$ visible Gaussians, each with a depth value $\hat{\mu}_z$ and a spatial position determining its tile assignment.
*   **Output:** Depth-sorted lists of splats for each tile, ready for rasterization.

**Data Structures:**
1.  **Compacted Depth Buffer:** Stores indices of visible Gaussians and their 32-bit depth values.
2.  **Intermediate Gaussian Values:** Stores attributes needed for rasterization and blending. These are **not** compacted to avoid indirection during gradient computation [S1, unofficial].
3.  **Tile Index Buffer:** Stores the tile index for each Gaussian.

**Parameters:**
*   **Tile Size:** $16 \times 16$ pixels [S1, unofficial].
*   **Max Resolution for 16-bit Tile Indices:** $2^{16} \times 256$ (roughly 16 megapixels). Note that this limit is slightly lower when image dimensions are not a multiple of 16 [S1, unofficial].
*   **Sorting Algorithm:** Must be stable (e.g., radix sort) to maintain the order established in the first stage [S1, unofficial].

**Steps:**
1.  **Pre-processing:** Identify indices of all visible Gaussians and their depth $\hat{\mu}_z$. Write these to compacted buffers using an atomic counter [S1, unofficial].
2.  **First Sort (Depth):** Perform a radix sort using 32-bit keys to establish depth ordering of the Gaussians [S1, unofficial].
3.  **Second Sort (Tile):** Perform a radix sort using 16-bit keys (or 32-bit for very high resolutions) to sort splats into per-tile lists [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def two_stage_sort(gaussians, tile_size=16):
    """
    Implements Two-Stage Sorting for Gaussian Splatting.
    gaussians: List of Gaussian objects with attributes:
               - index: int
               - depth: float (32-bit compatible)
               - pos: vec3 (for tile calculation)
    """
    # Step 1: Pre-processing
    # Identify visible Gaussians and write to compacted buffers
    # Use atomic counter to manage buffer writes
    visible_gaussians = []
    depth_buffer = []
    gaussian_indices = []
    
    # [G] Implementation detail: Visibility check depends on the specific 
    # culling method (e.g., bounding box intersection with tile).
    # Here we assume all input gaussians are pre-filtered as visible.
    
    for g in gaussians:
        gaussian_indices.append(g.index)
        depth_buffer.append(g.depth)
        visible_gaussians.append(g)
        
    # Note: Intermediate Gaussian values (color, transmittance, etc.) 
    # are kept in original order/structure and NOT compacted to avoid 
    # indirection during gradient computation [S1, unofficial].

    # Step 2: First Sort - Depth Sorting
    # Sort visible_gaussians by depth using 32-bit keys
    # Stability is required
    depth_sorted_indices = stable_radix_sort_32bit(depth_buffer, gaussian_indices)
    
    # Reorder visible_gaussians based on depth sort
    depth_sorted_gaussians = [visible_gaussians[i] for i in depth_sorted_indices]
    
    # Step 3: Second Sort - Tile Sorting
    # Calculate tile indices for each Gaussian
    tile_indices = []
    for g in depth_sorted_gaussians:
        # [G] Tile calculation: floor(pos.x / tile_size) * num_tiles_y + floor(pos.y / tile_size)
        # Assuming 2D projection for simplicity, actual 3DGS uses screen-space projection
        tile_idx = calculate_tile_index(g.pos, tile_size)
        tile_indices.append(tile_idx)
        
    # Sort by tile index using 16-bit keys (or 32-bit if resolution > 16MP)
    # Stability preserves depth order within tiles
    tile_sorted_indices = stable_radix_sort_16bit(tile_indices, range(len(depth_sorted_gaussians)))
    
    # Construct per-tile lists
    # [G] Implementation detail: This typically involves creating a list of lists 
    # or a flat buffer with offsets. The final output is a structure allowing 
    # efficient iteration over splats per tile.
    per_tile_lists = build_per_tile_lists(depth_sorted_gaussians, tile_sorted_indices, tile_indices)
    
    return per_tile_lists

def stable_radix_sort_32bit(keys, values):
    """
    Performs a stable radix sort on 32-bit integer keys.
    Returns indices sorted by keys.
    """
    # [G] Standard LSD radix sort implementation for 32-bit integers.
    # Must be stable.
    pass

def stable_radix_sort_16bit(keys, values):
    """
    Performs a stable radix sort on 16-bit integer keys.
    Returns indices sorted by keys.
    """
    # [G] Standard LSD radix sort implementation for 16-bit integers.
    # Must be stable.
    pass

def calculate_tile_index(pos, tile_size):
    """
    Calculates the tile index for a given position.
    """
    # [G] Placeholder for specific projection logic.
    # In 3DGS, pos is screen-space coordinate.
    tile_x = int(pos.x) // tile_size
    tile_y = int(pos.y) // tile_size
    # Assuming width is known or calculated globally
    return tile_y * num_tiles_x + tile_x
```

**Edge Cases and Pitfalls:**
*   **Resolution Limit:** Tile indices must fit in 16 bits. For resolutions exceeding roughly 16 megapixels, 32-bit keys must be used for the second sort [S1, unofficial].
*   **Non-Multiples of 16:** If image dimensions are not a multiple of 16, the effective max resolution for 16-bit tile indices is slightly lower [S1, unofficial].
*   **Gradient Computation:** Compacting intermediate Gaussian values needed for rasterization and blending should be avoided to prevent additional indirection during the backward pass [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Original 3DGS:** Uses a single 64-bit key sort. Assuming a 48-bit effective key size and an average of 8 tiles per Gaussian, the complexity is $O(8 \cdot (16 + 32)n) = O(384n)$ [S1, unofficial].
*   **Two-Stage Sorting:**
    *   First Sort (Depth): $O(32n)$ [S1, unofficial].
    *   Second Sort (Tile): $O(8 \cdot 16n) = O(128n)$ [S1, unofficial].
    *   Combined Complexity: $O(32n) + O(128n) = O(160n)$ [S1, unofficial].
*   **VRAM Usage:** Reduced because it eliminates the need for 64-bit data types to store sorting keys [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Significant reduction in sorting time and VRAM usage compared to single-stage sorting, with benefits increasing as the number of Gaussians increases [S1, unofficial].
*   **Limitations:** Requires a stable sorting algorithm. The 16-bit tile index limit restricts maximum resolution unless 32-bit keys are used for the second stage [S1, unofficial].

**When to Use:**
*   Use when VRAM is constrained or when sorting performance is a bottleneck.
*   Suitable for resolutions up to ~16 megapixels with 16-bit tile keys.

**When Not to Use:**
*   For very high resolutions (>16MP) where 32-bit keys for the second stage negate some VRAM benefits.
*   If the rendering pipeline does not support stable radix sorts.

## Background
Two-Stage Sorting was introduced by Schutz et al. [73] as an optimization for Gaussian Splatting rendering pipelines [S1, unofficial]. It addresses the computational inefficiency of single-stage radix sorts that use large keys to encode both depth and tile information [S1, unofficial]. The method is implemented in Faster-GS [S1, unofficial].

## Sources
[S1] Faster-GS: Analyzing and Improving Gaussian Splatting Optimization, https://arxiv.org/abs/2602.09999v1, fetched 2026-10-26 (unofficial)
