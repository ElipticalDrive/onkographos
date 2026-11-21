# Adaptive Rendering

Also called: Adaptive voxel reconstruction

## Overview
Adaptive rendering is a volume visualization technique that improves time efficiency by performing ray tracing on coarse-resolution data first, then incrementally refining the result by incorporating information from higher-resolution levels [S1, unofficial]. It functions as an intelligent masking mechanism that accelerates rendering by skipping empty or non-useful voxels identified in the low-resolution domain before calculating new voxels from higher resolutions for the integration equation [S1, unofficial]. Unlike octree-based methods, it avoids the extra memory cost of auxiliary tree structures by leveraging the compressed nature of the multiresolution pyramid itself [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A multiresolution pyramid of volume data consisting of $M+1$ levels, indexed $m=0 \dots M$. Level $m=0$ is the lowest resolution (LLL), and level $m=M$ is the highest resolution. The dataset has $N$ voxels on a side, where $N = 2^M + 1$ for some integer $M$ [S1, unofficial].
*   **Output:** A rendered image with adaptive detail, where only necessary high-resolution voxels are computed.

**Data Structures**
The method relies on a pyramid of binary volumes (or scalar volumes) indexed by level number $m$.
*   Volume $V_m$ measures $(N-1)/2^m$ cells on a side [S1, unofficial].
*   Cells are indexed by a level number $m$ and a vector $\mathbf{i} = (i, j, k)$ where $i, j, k = 1, \dots, (N-1)/2^m + 1$ [S1, unofficial].
*   The value contained in cell $\mathbf{i}$ on level $m$ is denoted $V_m(\mathbf{i})$ [S1, unofficial].

**Parameters**
*   **Resolution Levels:** The number of levels to use in the pyramid. Experiments suggest that using 1 to 3 levels is suitable [S1, unofficial].
*   **Constraint:** Going to too low a level (very coarse) is counter-productive if the volume data does not contain blocks of the associated size that are completely zero [S1, unofficial].

**Algorithm Steps**
1.  **Initialize:** Start ray tracing at the lowest resolution level (LLL, $m=0$).
2.  **March:** Advance along the ray through the current level's voxels.
3.  **Check Value:**
    *   If a voxel with a zero value is encountered, skip it immediately [S1, unofficial].
    *   If a non-zero voxel is encountered, invoke the block reconstruction function to obtain the corresponding block of voxels at the next higher resolution level ($m+1$) [S1, unofficial].
4.  **Switch Level:** Upon invoking block reconstruction, switch the ray tracing context to the higher resolution level. The ray continues marching through the newly reconstructed high-resolution block [S1, unofficial].
5.  **Integrate:** For useful voxels identified in the low-resolution domain, calculate the new voxel from the higher resolution for the integration equation along the ray [S1, unofficial].
6.  **Cache (Optional):** Since voxels reconstructed along one projection ray are likely used by adjacent rays, use a cache memory to temporarily retain reconstructed voxels to reduce redundant reconstruction [S1, unofficial].
7.  **Repeat:** Continue until the ray exits the volume or the highest resolution level is reached.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_render_ray(ray_origin, ray_direction, pyramid, cache):
    """
    ray_origin: 3D vector
    ray_direction: 3D vector
    pyramid: List of volumes [V_0, V_1, ..., V_M]
    cache: Dictionary mapping (level, voxel_coords) -> value
    """
    # Start at the lowest level (LLL)
    current_level = 0
    current_voxel = pyramid[current_level].get_voxel_at(ray_origin)
    
    # Initialize integration variables
    accumulated_color = 0.0
    accumulated_opacity = 0.0
    
    while current_level < len(pyramid):
        # Check cache first
        cached_val = cache.get((current_level, current_voxel))
        if cached_val is not None:
            voxel_value = cached_val
        else:
            voxel_value = pyramid[current_level].get_value(current_voxel)
            # Store in cache for adjacent rays
            cache[(current_level, current_voxel)] = voxel_value
            
        if voxel_value == 0:
            # Skip empty space efficiently
            current_voxel = pyramid[current_level].next_voxel_along(ray_direction, current_voxel)
            continue
            
        # Non-zero voxel found: reconstruct higher resolution
        # This step implies switching to the next level
        if current_level + 1 < len(pyramid):
            # Reconstruct block at higher level
            high_res_block = reconstruct_block(pyramid[current_level], current_voxel, pyramid[current_level + 1])
            
            # Switch to higher resolution level
            current_level += 1
            
            # Re-evaluate the current position in the new higher resolution
            # The ray origin/position must be mapped to the new grid
            current_voxel = pyramid[current_level].get_voxel_at(ray_origin)
            
            # Re-check cache for the new high-res voxel
            cached_val = cache.get((current_level, current_voxel))
            if cached_val is not None:
                voxel_value = cached_val
            else:
                voxel_value = pyramid[current_level].get_value(current_voxel)
                cache[(current_level, current_voxel)] = voxel_value
        
        # Perform integration step for the current voxel
        # Note: Specific integration equation (e.g., volume rendering integral) 
        # is assumed to be handled by the renderer's core loop.
        # Here we just accumulate based on the adaptive voxel value.
        accumulated_color += voxel_value
        accumulated_opacity += 1.0 # Simplified opacity model
        
        # Advance ray
        current_voxel = pyramid[current_level].next_voxel_along(ray_direction, current_voxel)
        
        # If we are at the highest level and still have non-zero, continue marching
        # If we are at a level where we just switched, we continue in the new level
        if current_level == len(pyramid) - 1 and voxel_value == 0:
            break
            
    return accumulated_color
```

**Edge Cases and Pitfalls**
*   **Too Coarse Levels:** Using a level that is too low (coarse) is counter-productive if the data lacks blocks of that size containing completely zero voxels, as the skipping benefit is lost [S1, unofficial].
*   **Precision:** Mapping ray positions between different resolution grids requires careful coordinate transformation to avoid sampling errors [G].
*   **Cache Management:** The cache size must be managed to prevent memory overhead from negating the performance gains, especially for large volumes [S1, unofficial].

## Subtopics

### Voxel Mipmap Generation via Clipmap
Uses a clipmap structure instead of a full pyramid, offering different memory access patterns and suitability for dynamic data. [Voxel Mipmap Generation via Clipmap](13505-voxel-mipmap-generation-via-clipmap.md)

## Operating characteristics
*   **Time Complexity:** Improves time efficiency by skipping empty voxels at low resolution. The cost is dominated by the number of non-zero voxels reconstructed at higher levels [S1, unofficial].
*   **Memory Complexity:** Saves space compared to octree methods by not requiring an auxiliary tree structure; it uses the compressed multiresolution pyramid data directly [S1, unofficial].
*   **Strengths:** Fast elimination of empty space; no extra memory for tree structures; suitable for compressed data [S1, unofficial].
*   **Limitations:** Performance gain depends on the sparsity of the data (presence of zero blocks); choosing the wrong number of levels can degrade performance [S1, unofficial].
*   **When to Use:** When volume data has significant empty regions and a multiresolution pyramid is already available or generated.
*   **When Not to Use:** When the data is dense with no large zero blocks, or when memory is extremely constrained and caching overhead becomes significant.

## Background
The method was developed to facilitate properties of multiresolution representation, such as adaptive rendering, progressive rendering, and error bound checking [S1, unofficial]. It was motivated by the desire to eliminate the extra memory cost associated with auxiliary octree structures while maintaining the benefits of hierarchical rendering [S1, unofficial]. The approach is conceptually equivalent to octree techniques but operates directly on the pyramid of binary/scalar volumes [S1, unofficial].

## Sources
[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
