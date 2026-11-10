# XBrickMap
Also called: eXtendedBrickMap
## Overview
XBrickMap is a space-partitioning ray traversal algorithm for 3-level hierarchical voxel grids that uses 4³ occupancy bitmasks to skip empty regions via bitwise tests [S1, unofficial]. It improves upon simpler skipping methods by using tiled bitmasks to perform hierarchical skips of 4³, 2³, and 1³ cuboids, while left-packing empty bricks within sectors to save memory [S1, unofficial]. It is distinguished from its sibling [Hierarchical Grid Traversal](2560-hierarchical-grid-traversal.md) by its specific use of compacted bitmasks for random access and its support for arbitrary in-place edits within bricks [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$; a hierarchical voxel grid consisting of sectors, bricks, and voxels.
- Output: The intersection point and material properties of the first non-transparent voxel hit by the ray.

**Data Structures:**
The hierarchy consists of three levels [S1, unofficial]:
1.  **Sector:** The top-level container.
2.  **Brick:** A $4^3$ (64 voxel) block within a sector.
3.  **Voxel:** The leaf element, stored in $8^3$ chunks within the brick structure (Note: Source [S1, unofficial] states "each brick contains 8³ voxels" but also "4³ occupancy bitmasks" and "4³ bricks" in sectors. The table indicates "8³ bits per brick". We interpret the structure as: Sectors contain bricks. Bricks are $4^3$ logical units for skipping, but physically store $8^3$ voxels. The bitmask is $4^3$ bits).
    -   *Correction based on [S1, unofficial] table:* The table lists "8³ bits per brick". The text says "4³ occupancy bitmasks". This implies the bitmask tracks $4^3$ sub-regions or the brick is logically $4^3$ for skipping but physically $8^3$ for storage. Let's stick to the explicit text: "4³ occupancy bitmasks" and "each brick contains 8³ voxels".
    -   *Layout:*
        -   **Sector:** Contains a list of bricks. Empty bricks are omitted via left-packing. The sector stores metadata to map indices.
        -   **Brick:** Contains an $4^3$ (64-bit) occupancy bitmask. If a bit is 0, the corresponding sub-region is empty. The brick also contains the $8^3$ voxel data.
        -   **Voxel:** 8-bit value (0 is transparent).

**Parameters:**
-   Sector size: Not explicitly given in units, but contains $4^3$ bricks [S1, unofficial].
-   Brick size: $8^3$ voxels [S1, unofficial].
-   Bitmask size: $4^3$ bits per brick [S1, unofficial].

**Steps:**
1.  **Initialize Ray Marching:** Start at the ray origin. Compute initial intersection with the three front-facing planes of the root voxel/sector [S1, unofficial].
2.  **Sector Traversal:** Determine which sector the ray is currently in.
3.  **Brick Skipping (Hierarchical):**
    -   Query the sector's data structure to find the bricks in the current path.
    -   Use the $4^3$ occupancy bitmask of each brick.
    -   Perform bitwise tests on the bitmask to identify empty sub-cuboids ($4^3, 2^3, 1^3$) [S1, unofficial].
    -   If a sub-cuboid is empty, skip it entirely.
    -   Use `popcnt` (population count) instructions to count preceding entries for random access to the next non-empty brick or voxel [S1, unofficial].
4.  **Voxel Traversal:**
    -   Once a non-empty brick/sub-region is identified, march through the $8^3$ voxels.
    -   Intersect the ray with the three front-facing cube planes of the current voxel to compute the step size [S1, unofficial].
    -   Query the voxel data. If non-zero, return the hit.
5.  **Iteration:** Repeat steps 2-4 until the ray exits the grid or hits a surface.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_xbrickmap(ray_origin, ray_direction, grid):
    # Initialize ray marching parameters
    t = 0.0
    # Bias distance to avoid floating-point infinite backtracking [S1, unofficial]
    t += BIAS 
    
    current_sector = get_sector_at(ray_origin, grid)
    
    while t < grid.max_distance:
        # Determine current brick within the sector
        # Use ray-plane intersection for front-facing planes [S1, unofficial]
        voxel_pos = compute_voxel_position(ray_origin, ray_direction, t)
        
        # Query brickmap layers to determine step size [S1, unofficial]
        # This involves checking the occupancy bitmask
        brick = get_brick_at(voxel_pos, current_sector)
        
        if brick is None:
            # Sector/Brick empty, skip using bitmask logic
            t = skip_empty_region(ray_origin, ray_direction, brick.occupancy_mask, t)
            continue
            
        # Check occupancy mask for sub-cuboids
        # Use popcnt for random access if needed to find next valid block
        if is_empty(brick.occupancy_mask, voxel_pos):
            t = skip_sub_cuboid(ray_origin, ray_direction, brick.occupancy_mask, t)
            continue
            
        # Traverse voxels within the brick
        voxel = get_voxel_at(voxel_pos, brick)
        
        if voxel.is_transparent():
            # Step to next voxel
            t += compute_step_size(ray_origin, ray_direction, voxel_pos)
        else:
            return hit(ray_origin, ray_direction, t, voxel)
            
    return NO_HIT
```

**Edge Cases and Pitfalls:**
-   **Floating-point precision:** Can cause infinite backtracking. Workaround: Bias intersection distances [S1, unofficial].
-   **Dynamic Edits:** Supports arbitrary edits within a brick. However, insertions and deletions may require reallocations at the sector level to make space for new bricks [S1, unofficial].
-   **Memory Layout:** Left-packing of empty bricks requires careful index mapping using `popcnt` [S1, unofficial].

## Subtopics

### [Ray-guided streaming with atomic bitfield marking](10274-ray-guided-streaming-with-atomic-bitfield-marking.md)
This method tracks which finest-level brick cells have been visited by rays to enable selective streaming and refinement, preventing redundant data transfers when multiple rays share a cell [10274-ray-guided-streaming-with-atomic-bitfield-marking.md]. It differs from XBrickMap by adding a runtime occupancy buffer that complements the static hierarchical skipping with dynamic visitation history [10274-ray-guided-streaming-with-atomic-bitfield-marking.md].

### Two-Level Voxel Grid with Solid Color Optimization
A data structure storing voxels in a two-level hierarchy to optimize ray traversal and memory usage [S2, unofficial]. It consists of a coarse $128^3$ top-level grid pointing to $8^3$ voxel bricks, resulting in a total world resolution of $1024^3$ [S2, unofficial].
-   **Layout:** Top-level grid cell is a 32-bit value. Bit 0 determines type: 0 indicates solid color (bits 1-31 store color), 1 indicates pointer to brick (bits 1-31 store brick index) [S2, unofficial]. Value 0 represents transparent solid color [S2, unofficial].
-   **Voxel Storage:** Each voxel in a brick is 8-bit. 0 is transparent. Non-zero is 3-3-2 RGB triplet [S2, unofficial].
-   **Memory:** Top-level grid is 8MB ($128^3 \times 4$ bytes). Max brick memory is 1GB ($128^3$ unique bricks $\times 8^3$ voxels $\times 1$ byte) [S2, unofficial].
-   **Pitfall:** Initializing the world to empty/transparent is done via `memset(0)` because 0 is a valid transparent solid color [S2, unofficial].

## Operating characteristics
-   **Time Complexity:** Approximately $O(1)$ per step, though dependent on the density of non-empty regions [S1, unofficial].
-   **Memory Complexity:** 8³ bits per brick + 12 bytes per sector [S1, unofficial].
-   **Performance:** On the Sponza 2k scene, XBrickMap achieves 165.6 Mrays/s for primary rays and 125.1 Mrays/s for primary + 1 diffuse bounce [S1, unofficial].
-   **Strengths:** Good performance relative to simplicity; supports arbitrary edits within bricks; efficient space skipping via bitwise operations [S1, unofficial].
-   **Limitations:** Insertions/deletions may require sector-level reallocations; floating-point precision issues require biasing; performance on incoherent rays is ~30-50% worse than coherent rays but still worse than other space-partitioning methods [S1, unofficial].
-   **When to use:** When random access and in-place editing of voxel data are required, and the scene has significant empty space to skip [S1, unofficial].
-   **When not to use:** When the scene is highly dense with no empty space, or when dynamic topology changes are frequent and costly reallocations are unacceptable [S1, unofficial].

## Background
XBrickMap was developed as part of the VoxelRT project by Thijs van Wingerden and Stijn Herfst [S1, unofficial]. It is described in the thesis "VoxelRT: Visualization and Editing of Large Voxel Scenes" [S1, unofficial]. The method builds on earlier brick map concepts and aims to provide a high-performance realtime CUDA voxel path tracer [S1, unofficial]. It is designed to handle large voxel scenes with efficient space skipping and dynamic editing capabilities.

## Sources
[S1] dubiousconst282/VoxelRT, https://github.com/dubiousconst282/VoxelRT, fetched 2026-10-24 (unofficial)
[S2] A Voxel Renderer for Learning C/C++ – Jacco's Blog, https://jacco.ompf2.com/2021/02/01/a-voxel-renderer-for-learning-c-c/, fetched 2026-10-24 (unofficial)
