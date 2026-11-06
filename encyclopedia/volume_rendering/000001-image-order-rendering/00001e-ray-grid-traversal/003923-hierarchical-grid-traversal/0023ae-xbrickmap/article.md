# XBrickMap

Also called: eXtendedBrickMap

## Overview
XBrickMap is a space-skipping ray traversal algorithm for 3-level hierarchical voxel grids that uses 4³ occupancy bitmasks to perform hierarchical skips (4³, 2³, 1³ cuboids) via bitwise tests [S1, unofficial]. It improves upon standard skipping by using tiled bitmasks to identify empty voxel sections and left-packs these bitmasks to omit empty bricks within sectors, saving memory [S1, unofficial]. It is distinguished from siblings like [MultiDDA](2204-hierarchical-voxel-traversal.md) by its specific 3-level sector/brick/voxel hierarchy and bitmask-based compression, and from [Tree64](6354-multi-level-grid-structures.md) by its fixed grid topology and lack of dynamic tree rebalancing [S1, unofficial].

## How it works
**Inputs:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, world bounds.
**Outputs:** Intersection distance or hit status.

**Data Structures:**
The hierarchy consists of three levels:
1.  **Sector:** The top-level container.
2.  **Brick:** A $4^3$ block of voxels within a sector.
3.  **Voxel:** The fundamental unit, $8^3$ voxels per brick.

*   **Sector Layout:** A sector contains a list of bricks. To support random access and arbitrary edits, bricks are stored in a compacted array. The layout uses bitmasks to track occupancy.
*   **Brick Layout:** Each brick is represented by a bitmask. The source notes specify "8³ bits per brick" in the comparison table [S1, unofficial], but the definition states "4³ occupancy bitmasks" and "brick contains 8³ voxels". The table likely refers to the metadata overhead or a specific implementation detail where the bitmask represents the $4^3$ sub-cuboids or the presence of the $8^3$ voxel block. Given the "4³ occupancy bitmasks" description and "hierarchical skips (4³, 2³, 1³)", the bitmask likely tracks the occupancy of $4^3$ sub-volumes or the presence of the $8^3$ voxel block itself. The table explicitly lists "8³ bits per brick" as the storage cost for the space-parting data. We assume a bitmask of size $8^3$ bits (or potentially $4^3$ bits if it tracks sub-bricks) is used to determine if a brick is empty. *Note: The source is ambiguous between $4^3$ and $8^3$ bits. The table says 8³ bits. The text says 4³ occupancy bitmasks. We cite the table for the explicit storage size.*
*   **Sector Metadata:** 12 bytes per sector [S1, unofficial]. This likely stores the pointer to the brick array and the count of valid bricks.

**Algorithm Steps:**
1.  **Initialization:** Compute the initial step size for ray marching. Intersect the ray with the three front-facing planes of the current voxel/brick/sector boundary to determine the entry point and step size [S1, unofficial].
2.  **Sector Traversal:** Determine which sector the ray is currently in. Access the sector's metadata.
3.  **Brick Skipping (Hierarchical Skip):**
    *   Query the sector's bitmask for the current brick index.
    *   If the bit is 0, the brick is empty. Skip to the next brick.
    *   The bitmask allows skipping $4^3$, $2^3$, or $1^3$ cuboids depending on the hierarchy level and bitmask granularity [S1, unofficial].
    *   Use `popcnt` (population count) instructions to count preceding entries to map the logical brick index to the physical index in the compacted array, enabling random access to non-contiguous bricks [S1, unofficial].
4.  **Voxel Traversal:**
    *   If the brick is not empty, iterate through the $8^3$ voxels within the brick.
    *   Use a ray marching algorithm (DDA) to traverse the voxels.
    *   At each step, compute the voxel position by intersecting with the front-facing cube planes [S1, unofficial].
    *   Check the voxel value. If non-transparent, return the hit.
5.  **Iteration:** Repeat steps 2-4 until the ray exits the world or hits a voxel.

**Formulas:**
*   **Voxel Position:** $\mathbf{p} = \mathbf{o} + t \mathbf{d}$, where $t$ is the intersection distance with the front-facing plane.
*   **Physical Index:** $idx_{phys} = \text{popcnt}(mask, idx_{log})$ [S1, unofficial].

**Parameters:**
*   **Brick Size:** $8^3$ voxels [S1, unofficial].
*   **Sector Size:** Contains $4^3$ bricks [S1, unofficial].
*   **Bitmask Size:** 8³ bits per brick [S1, unofficial].
*   **Sector Overhead:** 12 bytes [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Floating-Point Precision:** Floating-point limitations can cause infinite backtracking (ray getting stuck). This is worked around by biasing the intersection distances [S1, unofficial].
*   **Dynamic Edits:** Insertions and deletions of bricks may require reallocations at the sector level to make space, as bricks are compacted [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_xbricmap(ray_origin, ray_direction, world):
    # 1. Initialize ray marching parameters
    # Intersect with the first sector boundary
    t_min, t_max = intersect_ray_sector(ray_origin, ray_direction, world.sector_bounds)
    
    current_sector = get_sector_at(ray_origin, ray_direction, world)
    
    while t_min < t_max:
        # 2. Determine current brick index within the sector
        brick_idx = get_brick_index(current_sector, ray_origin, ray_direction)
        
        # 3. Check brick occupancy using bitmask
        # The bitmask is stored in the sector metadata or associated with the sector
        is_empty = is_brick_empty(current_sector.bitmask, brick_idx)
        
        if is_empty:
            # Skip empty brick
            # Use popcnt to find the next non-empty brick if needed, 
            # or simply increment and check if the array is contiguous for empty bricks
            # The source mentions popcnt for random access to non-empty bricks.
            # For skipping, we likely jump to the next brick index.
            brick_idx += 1
            # Update t_min to the next brick boundary
            t_min = intersect_brick_front(current_sector, brick_idx, ray_origin, ray_direction)
            continue
        
        # 4. Traverse the non-empty brick
        # The brick contains 8^3 voxels
        voxel_data = current_sector.bricks[brick_idx] # Random access via popcnt mapping if needed
        
        # Ray march through the 8^3 voxels
        for voxel in voxel_data:
            # Intersect with voxel planes
            t_voxel = intersect_voxel_front(ray_origin, ray_direction, voxel)
            
            if t_voxel < t_min:
                continue # Behind current step
            
            # Check voxel value
            if voxel.value != 0: # 0 is transparent
                return t_voxel, voxel
            
            # Bias distance to prevent infinite backtracking due to FP errors
            t_min = t_voxel + EPSILON 
            
    return None
```

## Subtopics

### Two-Level Voxel Grid with Solid Color Optimization
A data structure storing voxels in a two-level hierarchy (coarse 128³ top-level grid pointing to 8³ voxel bricks) for a total world resolution of 1024³ [S2, unofficial]. It optimizes memory and traversal by allowing grid cells to store a solid color (3-3-2 RGB) instead of a pointer to a brick, which is useful for large uniform areas [S2, unofficial]. A top-level grid cell is a 32-bit value: bit 0 is 0 for solid color (bits 1-31 store color) and 1 for a brick pointer (bits 1-31 store brick index) [S2, unofficial]. Value 0 represents a transparent solid color, allowing the grid to be cleared with `memset(0)` [S2, unofficial]. Each voxel in a brick is an 8-bit value (0 transparent, non-zero 3-3-2 RGB) [S2, unofficial]. Memory: 8MB for the top-level grid, up to 1GB for bricks [S2, unofficial].

## Operating characteristics
**Time Complexity:** Approximately $O(1)$ per step, as it supports space skipping [S1, unofficial].
**Memory Complexity:** 8³ bits per brick + 12 bytes per sector [S1, unofficial].
**Performance:** On the Sponza 2k scene, XBrickMap achieves 165.6 Mrays/s for primary rays and 125.1 Mrays/s for primary + 1 diffuse bounce [S1, unofficial]. On the Forest Lake 2k scene (807k * 8³ voxels), it is not explicitly listed in the provided table, but PlainDDA is 23.6 Mrays/s and MultiDDA is 60.6 Mrays/s [S1, unofficial]. XBrickMap is generally faster than DDA-based methods due to space skipping [S1, unofficial].
**Strengths:** Good performance relative to simplicity; supports arbitrary edits within a brick; compact memory usage due to left-packing [S1, unofficial].
**Limitations:** Insertions/deletions may require sector-level reallocations; floating-point precision issues require biasing; performance on incoherent rays may be 30-50% worse than coherent rays [S1, unofficial].

## Background
XBrickMap was developed by Thijs van Wingerden as part of a thesis on "Voxel Rendering and Editing of Large Voxel Scenes" [S1, unofficial]. It is implemented in the VoxelRT project [S1, unofficial]. The method builds on the concept of brick maps, which were previously used in real-time path tracing [S1, unofficial].

## Sources
[S1] dubiousconst282/VoxelRT, https://github.com/dubiousconst282/VoxelRT, fetched 2026-10-24 (unofficial)
[S2] A Voxel Renderer for Learning C/C++ – Jacco's Blog, https://jacco.ompf2.com/2021/02/01/a-voxel-renderer-for-learning-c-c/, fetched 2026-10-24 (unofficial)
