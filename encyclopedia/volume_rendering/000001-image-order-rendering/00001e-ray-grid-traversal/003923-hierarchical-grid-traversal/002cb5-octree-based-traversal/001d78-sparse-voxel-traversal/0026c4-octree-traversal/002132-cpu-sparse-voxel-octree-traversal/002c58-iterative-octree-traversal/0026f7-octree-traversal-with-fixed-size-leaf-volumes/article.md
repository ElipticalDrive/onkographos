# Octree Traversal with Fixed-Size Leaf Volumes


## Overview
This method optimizes hierarchical ray marching by fixing the leaf node size to a constant volume (32×32×32 voxels) instead of using the smallest available octree nodes [S1, unofficial]. It switches between octree levels and voxel levels based on hit/miss status to reduce traversal overhead for sparse data, at the cost of potential false positives along the ray direction [S1, unofficial]. It sets itself apart from standard iterative octree traversal by decoupling the inner marching loop from the octree depth, simplifying the inner loop and reducing memory overhead for sparse data [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin ($rayPos$), ray direction ($rayDir$), total volume side length ($s$), and a 3D voxel grid containing density/material data [S1, unofficial].
*   **Output:** Hit position, traversal status, and total iteration count [S1, unofficial].

**Data Structures:**
*   **Skip List:** An array `skipListPerDepth` where each entry is a bitmask tracking visited child nodes at each octree level to prevent re-examination [S1, unofficial].
*   **Traversal State:** Variables for `currentDepth`, `parentIndex`, and `parentPos` to track the traversal path and enable level adjustments [S1, unofficial].

**Parameters:**
*   Leaf node size: Fixed at 32×32×32 voxels [S1, unofficial].
*   Total volume side length: $s$ [S1, unofficial].
*   Maximum traversal depth: Implementation-dependent, typically bounded by $\log_2(s/32)$ [G].

**Steps:**
1.  Initialize `currentDepth` to 0, `pos` to $rayPos$, and `skipListPerDepth` to all zeros [S1, unofficial].
2.  Enter a loop bounded by a maximum iteration count to prevent infinite traversal [G].
3.  Calculate the step size for the current level as $32 \ll currentDepth$ [G].
4.  March along the ray using a DDA algorithm up to the calculated step size [G].
5.  If a voxel with non-zero data is encountered, record the hit position and return [S1, unofficial].
6.  If the ray exits the current block without a hit, update the skip list for the current depth: `skipListPerDepth[currentDepth] |= (1 << usedIndex)` [S1, unofficial].
7.  Save the index and position of the current parent node, then increment `currentDepth` to traverse one level deeper into the tree [S1, unofficial].
8.  If the ray misses at the current level and `currentDepth` is below the maximum, increase the level (move to a coarser octant) and continue [S1, unofficial].
9.  If the ray hits a node but not a leaf, decrease the level (move to a finer octant) and continue [S1, unofficial].
10. If the ray exits the volume bounds or exceeds the maximum depth, return a miss or out-of-bounds status [S1, unofficial].

**Formulas:**
*   Number of leaf nodes to examine along the diagonal: $tcountNodes = \frac{s}{64} \cdot 3 \cdot \left(\frac{s}{32}\right)^2$ (based on the Pythagorean theorem) [S1, unofficial].
*   Total steps for worst-case traversal: $tTotalCount = tcountNodes \cdot \log s + 3 \cdot 32^2$ [S1, unofficial].
*   Asymptotic worst-case complexity: $O(s \cdot \log s)$ [S1, unofficial].

**Parameters with Typical Values:**
*   Leaf size: 32 voxels per dimension [S1, unofficial].
*   Volume side $s$: Typically a power of two (e.g., 256, 512, 1024) [G].

**Edge Cases and Pitfalls:**
*   **Noise:** Noise in volume data significantly reduces efficiency by causing false positives along the ray direction, forcing the examination of unnecessary leaf nodes [S1, unofficial].
*   **Precision:** Unpredictable precision errors can cause rays to miss or hit incorrectly, requiring robust boundary checks [S1, unofficial].
*   **Zero Direction Components:** DDA step calculations must handle $dir.x = 0$ (or $y, z$) by setting the corresponding $t_{step}$ to infinity to avoid division by zero [G].
*   **Boundary Conditions:** Explicit bounds checking is required at each level to prevent out-of-bounds memory access [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def traverse_fixed_leaf_octree(ray_origin, ray_dir, volume_side, voxel_grid):
    LEAF_SIZE = 32
    MAX_DEPTH = 10
    MAX_ITERATIONS = 10000
    
    pos = list(ray_origin)
    dir = list(ray_dir)
    level = 0
    depth = 0
    skip_list = [0] * MAX_DEPTH
    total_iters = 0
    
    for _ in range(MAX_ITERATIONS):
        step_size = LEAF_SIZE << level
        
        # DDA initialization for the current block [G]
        step_x = 1 if dir[0] > 0 else -1
        step_y = 1 if dir[1] > 0 else -1
        step_z = 1 if dir[2] > 0 else -1
        
        t_max_x = (step_x * (math.floor(pos[0]) + (1 if step_x > 0 else 0) - pos[0])) / dir[0] if dir[0] != 0 else float('inf')
        t_max_y = (step_y * (math.floor(pos[1]) + (1 if step_y > 0 else 0) - pos[1])) / dir[1] if dir[1] != 0 else float('inf')
        t_max_z = (step_z * (math.floor(pos[2]) + (1 if step_z > 0 else 0) - pos[2])) / dir[2] if dir[2] != 0 else float('inf')
        
        t_step_x = step_x / dir[0] if dir[0] != 0 else float('inf')
        t_step_y = step_y / dir[1] if dir[1] != 0 else float('inf')
        t_step_z = step_z / dir[2] if dir[2] != 0 else float('inf')
        
        iters = 0
        hit = False
        hit_pos = None
        hit_val = None
        
        # March through the fixed-size block
        while iters < step_size:
            # Find minimum t_max
            if t_max_x <= t_max_y and t_max_x <= t_max_z:
                pos[0] += t_max_x * dir[0]
                t_max_x += t_step_x
                iters += 1
            elif t_max_y <= t_max_x and t_max_y <= t_max_z:
                pos[1] += t_max_y * dir[1]
                t_max_y += t_step_y
                iters += 1
            else:
                pos[2] += t_max_z * dir[2]
                t_max_z += t_step_z
                iters += 1
                
            # Boundary and voxel check
            idx = (math.floor(pos[0]), math.floor(pos[1]), math.floor(pos[2]))
            if not (0 <= idx[0] < volume_side and 0 <= idx[1] < volume_side and 0 <= idx[2] < volume_side):
                return {'status': 'OUT_OF_BOUNDS', 'pos': pos, 'iters': total_iters}
                
            val = voxel_grid[idx[0]][idx[1]][idx[2]]
            if val > 0:
                hit = True
                hit_pos = pos
                hit_val = val
                break
                
        total_iters += iters
        
        if hit:
            return {'status': 'HIT', 'pos': hit_pos, 'val': hit_val, 'iters': total_iters}
            
        # Level adjustment logic from source [S1, unofficial]
        # Source does not specify exact octree node traversal logic; standard iterative DDA used [G].
        # Update skip list and parent tracking as described
        used_index = 0 # Placeholder for child index calculation based on ray entry point [G]
        skip_list[depth] |= (1 << used_index)
        # parent_index and parent_pos would be saved here per source [S1, unofficial]
        depth += 1
        
        if level < MAX_DEPTH - 1:
            level += 1 # Move to coarser level on miss
        else:
            return {'status': 'MISS', 'pos': pos, 'iters': total_iters}
            
    return {'status': 'MISS', 'pos': pos, 'iters': total_iters}
```

## Operating characteristics
**Time and Memory Complexity:**
*   **Best-case time complexity per voxel:** $O(\log s)$, where $s$ is the initial size of the Octree (side length of the volume) [S1, unofficial].
*   **Worst-case time complexity:** $O(s \cdot \log s)$ [S1, unofficial]. This occurs when every 32×32×32 leaf node along the diagonal (through the volume center) must be examined [S1, unofficial].
*   The number of leaf nodes to examine along the diagonal is calculated as: $tcountNodes = \frac{s}{64} \cdot 3 \cdot \left(\frac{s}{32}\right)^2$ (based on the Pythagorean theorem) [S1, unofficial].
*   Total steps for worst-case traversal: $tTotalCount = tcountNodes \cdot \log s + 3 \cdot 32^2$ [S1, unofficial].
*   **Memory:** Memory usage is lower than using the smallest octree nodes because the leaf size is fixed, but larger leaf sizes increase the probability of false positives [S1, unofficial].

**Measured Performance:**
*   Warp divergence is minimal because similar rays produce similar results within a warp [S1, unofficial].
*   Divergence is further mitigated because the number of iterations is relatively low for noise-free voxel data [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Efficient for sparse data due to fixed leaf size; reduced memory overhead; minimal warp divergence [S1, unofficial].
*   **Limitations:** Sensitive to noise; false positives can degrade performance; trade-off between leaf size and false positive rate [S1, unofficial].

**When to Use:**
*   Use when data is sparse and noise levels are low [S1, unofficial].
*   Use when memory efficiency is a priority [S1, unofficial].

**When Not to Use:**
*   Avoid when data is dense and noisy, as false positives will dominate traversal cost [S1, unofficial].
*   Avoid when exact leaf-level precision is required without post-processing [S1, unofficial].

## Background
This method is described in the context of raycasting-based voxel rendering, specifically in a thesis on recursive-free octree traversal [S1, unofficial]. The approach leverages the Pythagorean theorem to estimate the number of leaf nodes along the diagonal for complexity analysis [S1, unofficial]. It was developed to optimize traversal performance by parameterizing leaf size, balancing memory usage against false positive rates [S1, unofficial].

## Sources
[S1] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
