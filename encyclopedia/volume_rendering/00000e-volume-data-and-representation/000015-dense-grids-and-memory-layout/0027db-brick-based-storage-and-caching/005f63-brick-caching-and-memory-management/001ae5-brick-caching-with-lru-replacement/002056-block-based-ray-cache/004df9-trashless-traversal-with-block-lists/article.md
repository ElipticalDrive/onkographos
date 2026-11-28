# Trashless Traversal with Block Lists

Also called: Trashless Distribution Scheme

## Overview
Trashless Traversal with Block Lists is a cache-oblivious traversal strategy for volume rendering that eliminates cache trashing by ensuring that any data block evicted from the cache is never accessed again during the rendering of a frame [S1, unofficial]. It achieves this by subdividing the volume into blocks, sorting them front-to-back, and organizing them into disjoint block lists such that no single ray intersects more than one block within the same list [S1, unofficial]. This approach distinguishes itself from standard spatial traversal by decoupling the processing order of blocks from the ray order within a frame, allowing for parallel processing of blocks within a list while maintaining strict cache locality [S1, unofficial].

## How it works
**Inputs:** Volume dataset subdivided into bricks/blocks; viewing direction (camera position and orientation); ray generation parameters (image resolution, ray origins, directions).
**Outputs:** Rendered image; updated ray states (completed rays removed, active rays moved to next blocks).

**Data Structures:**
1.  **Block List:** A sequence of blocks sorted front-to-back.
2.  **Ray List:** Associated with each block, containing indices or pointers to rays currently intersecting that block.
3.  **Global Address Lookup Table:** Shared by all blocks to map block IDs to memory addresses [S1, unofficial].

**Algorithm Steps:**

1.  **Precomputation (Optional for Parallel Projection):**
    For parallel projection, the relative order of blocks remains constant for eight distinct viewing direction octants. The block lists can be precomputed for these eight cases to avoid per-frame sorting overhead [S1, unofficial].

2.  **Initialization:**
    a. Subdivide the volume into blocks (bricks) of fixed size [S1, unofficial].
    b. Determine the viewing direction.
    c. Sort all blocks in front-to-back order based on the viewing direction [S1, unofficial].
    d. Distribute blocks into block lists. The distribution rule is: assign blocks to lists such that no ray originating from the camera intersects more than one block in the same list. This ensures that when a ray moves from one block to the next, it moves to a block in a *different* list [S1, unofficial].
    e. For each ray, identify the first block it intersects. Add the ray to the ray list of that initial block [S1, unofficial].

3.  **Traversal Loop:**
    Process block lists sequentially from front to back. Let $L$ be the current block list index.
    
    While $L$ is valid:
    a. **Process Blocks in List $L$:**
       Iterate through all blocks $B$ in list $L$. The order of blocks within a list does not matter for correctness and can be processed in parallel [S1, unofficial].
       
       For each block $B$ in $L$:
       i. Retrieve the ray list $R_B$ associated with block $B$.
       ii. If $R_B$ is empty, skip to the next block.
       iii. For each ray $r$ in $R_B$:
           1. Perform volume sampling/raycasting for ray $r$ within the bounds of block $B$.
           2. Determine if ray $r$ exits block $B$.
           3. If ray $r$ exits block $B$:
               - Remove $r$ from $R_B$.
               - Identify the next block $B_{next}$ that ray $r$ intersects.
               - Add $r$ to the ray list $R_{B_{next}}$ of block $B_{next}$.
           4. If ray $r$ is fully absorbed or terminated within $B$, remove $r$ from $R_B$ and discard.
       
       iv. After processing all rays in $R_B$, if $R_B$ is empty, the block $B$ is done.
    
    b. **Advance to Next List:**
       Once all blocks in list $L$ have been processed (and their ray lists are empty or rays have been moved to subsequent lists), increment $L$ to process the next block list [S1, unofficial].

**Parameters:**
-   **Brick Size:** Must be chosen appropriately to ensure that a block entirely remains in the fast cache (e.g., L1/L2) while its associated rays are being processed [S1, unofficial]. The optimal size depends on the cache capacity and the number of rays per block.

**Edge Cases and Pitfalls:**
-   **Cache Trashing:** The primary pitfall is violating the disjoint list property. If a ray intersects two blocks in the same list, the second block might be evicted before the ray returns to it, causing trashing. The construction of block lists must strictly enforce that no ray intersects multiple blocks in the same list [S1, unofficial].
-   **Ray List Management:** Efficiently adding and removing rays from ray lists is critical. Using linked lists or dynamic arrays with swap-and-pop techniques can minimize overhead [G].
-   **Parallel Processing:** While blocks within a list can be processed in parallel, rays moving between blocks must be handled carefully to avoid race conditions if multiple threads process different blocks in the same list simultaneously. Since rays move to blocks in *different* lists, and lists are processed sequentially, race conditions are avoided if ray lists are thread-local or protected [G].

## Operating characteristics
-   **Time Complexity:** Precomputation is $O(N \log N)$ for sorting blocks, where $N$ is the number of blocks. Traversal is $O(R \cdot D)$, where $R$ is the number of rays and $D$ is the average depth (number of blocks intersected per ray). The overhead of managing ray lists is proportional to the number of block intersections.
-   **Memory Complexity:** $O(N + R)$, where $N$ is the number of blocks and $R$ is the number of rays. The global address lookup table adds minimal overhead [S1, unofficial].
-   **Performance:** Significantly reduces cache misses compared to standard traversal by ensuring cache lines are not evicted before use. Performance scales well with cache size; larger bricks improve locality but may reduce parallelism if bricks become too large [S1, unofficial].
-   **Strengths:** Eliminates cache trashing; allows parallel processing of blocks within a list; low memory overhead [S1, unofficial].
-   **Limitations:** Requires precomputation or per-frame sorting; complex to implement for perspective projection due to varying block orders; ray list management can become a bottleneck if rays frequently cross block boundaries [S1, unofficial].

## Background
The method is based on the "trashless distribution scheme" presented by Law and Yagel [S1, unofficial]. Originally developed in the context of parallelization to avoid redundant distribution of data blocks over a network, it was adapted for single-processor cache optimization [S1, unofficial]. The approach is described in the context of efficient volume visualization of large medical datasets [S1, unofficial].

## Sources
[S1] Efficient Volume Visualization of Large Medical Datasets, https://old.cescg.org/CESCG-2004/web/Bruckner-Stefan/html/, fetched 2026-10-30 (unofficial)
