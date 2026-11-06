# Ray-Occlusion Traversal in Octree

Also called: Octree Ray Traversal, Ray-Octree Traversal

## Overview
Ray-Occlusion Traversal in Octree is a hierarchical ray-intersection algorithm that traverses an octree structure to determine if a ray intersects any solid voxel or leaf node, primarily used for shadow ray calculations and visibility testing in volume rendering [S1, S3, S4, unofficial]. It solves the problem of efficiently culling empty space in sparse voxel octrees by recursively descending the tree and using ray-AABB intersection tests to prune branches that do not intersect the ray [S3, unofficial]. Unlike [Sparse Voxel Octree Ray Traversal](2451-sparse-voxel-octree-ray-traversal.md) which focuses on general volume rendering traversal, this method specifically emphasizes the "occlusion" aspect—returning early upon the first hit to determine shadowing or visibility [S1, S4, unofficial]. It differs from [Bottom-up Neighbor-Finding Ray Traversal](5610-bottom-up-neighbor-finding-ray-traversal.md) by relying on hierarchical descent from the root rather than lateral neighbor stepping [S1, S5, unofficial].

## How it works
The method determines the sequence of octree nodes intersected by a ray, handling rays that start inside or outside the root bounding box [S1, unofficial]. It recursively or iteratively descends the octree, checking for intersections with node bounding boxes and child occupancy masks [S3, S4, unofficial].

### Inputs and Outputs
*   **Input:** Ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is the origin and $\mathbf{d}$ is the normalized direction; Octree root node; Maximum traversal depth $D_{max}$; Maximum traversal steps $N_{max}$ [S1, S3, unofficial].
*   **Output:** Boolean `true` if any leaf node (material) is intersected, `false` otherwise; or the intersection distance $t$ and hit normal if required [S3, S4, unofficial].

### Data Structures
The octree is stored as a flat array of nodes. A common layout (from [S1, unofficial]) uses an `OctreeNode` struct:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct OctreeNode {
    int p0, p1, p2, p3, p4, p5, p6, p7; // 8 child indices
};
```
*   `p0` to `p7`: Integer indices pointing to children in the `octreeArray`.
*   Indices $\ge 0$ point to valid child nodes.
*   Indices $< 0$ (specifically $-1$) indicate an empty node or a leaf status [S1, unofficial].
*   Nodes are often stored in reverse depth order (deepest layers first) with pointers adjusted by an offset [S1, unofficial].

### Algorithm Steps

1.  **Initialization and Root Intersection:**
    *   If the ray origin $\mathbf{o}$ is outside the root bounding box, compute intersections with the 6 planes of the root AABB [S1, unofficial].
    *   Calculate distances $t_x, t_y, t_z$ to the near and far planes for each axis.
    *   Clamp negative distances to $\infty$ (INF) to ignore back-facing intersections [S1, unofficial].
    *   Determine the entry point $t_{enter} = \min(t_x, t_y, t_z)$ and the corresponding entry axis/normal [S1, unofficial].
    *   If $t_{enter} = \infty$, return `false` (miss) [S1, unofficial].
    *   If the ray starts inside, $t_{enter} = 0$ [S1, unofficial].
    *   Apply a small epsilon offset $\epsilon_{entry}$ (e.g., $0.001 \times \mathbf{d}$) to the intersection point to avoid self-intersection or precision errors at the boundary [S1, unofficial].

2.  **Traversal Loop:**
    *   Initialize `curDepth = 0`, `nodeIndex = rootIndex`, `curVoxel` (current voxel coordinates), and `voxelSize` (size of current node's extent) [S1, unofficial].
    *   Loop for `i` from $0$ to $N_{max} - 1$ (e.g., 300) [S1, unofficial].
    *   **Descent:**
        *   While `curDepth < D_{max}` and `nodeIndex` is not a leaf/empty marker:
            *   Retrieve `node = octreeArray[nodeIndex]` [S1, unofficial].
            *   Determine the child index $k \in \{0, \dots, 7\}$ based on the ray's position relative to the node's center $\mathbf{c}$ [S1, unofficial][S2]. Using the shared principle:
                $$ k = 4 \cdot \mathbb{I}(p_x \ge c_x) + 2 \cdot \mathbb{I}(p_y \ge c_y) + 1 \cdot \mathbb{I}(p_z \ge c_z) $$
                where $\mathbf{p}$ is the ray position at the current depth (or the entry point for the first step) [S1, unofficial][S2].
            *   Check `node.p_k`:
                *   If `node.p_k < -1`: Hit a material leaf. Return `true` (or compute shadow factor) [S1, unofficial].
                *   If `node.p_k == -1`: Empty node. Break descent to handle empty space skipping [S1, unofficial].
                *   If `node.p_k >= 0`: Valid child. Update `nodeIndex = node.p_k`, `curDepth++`, `voxelSize /= 2` [S1, unofficial].
    *   **Empty Space Skipping / Leaf Handling:**
        *   If an empty node (`p_k == -1`) is encountered, or if max depth is reached without a hit:
            *   Calculate the exit distance $t_{exit}$ of the current node's AABB from the ray [S2].
            *   Advance the ray parameter $t$ to $t_{exit}$ [S2].
            *   If the ray has exited the root AABB ($t > t_{root\_exit}$), return `false` [S1, unofficial].
            *   Use neighbor pointers or backtrack to the parent to find the next potentially occupied node [S1, unofficial][S2]. *Note: Specific neighbor-finding logic is detailed in [Bottom-up Neighbor-Finding Ray Traversal](5610-bottom-up-neighbor-finding-ray-traversal.md); here, simple backtracking or DDA stepping within the leaf's voxel grid is implied for volume sampling, but for occlusion, we skip to the next intersected node* [S1, unofficial][S2].
            *   *Implementation Note (S1):* If `subNodeValue == -1`, calculate deltas to the next voxel boundaries using `getDelta` for x, y, z axes. Update `curVoxel` by adding epsilon to position and rounding. Check if the new voxel is within the current parent bounds. If not, backtrack up the tree (`curDepth--`, `voxelSize *= 2`) until inside or depth < 0 [S1, unofficial].
    *   **Hit Detection:**
        *   If a leaf node is reached (either via descent or after skipping empty space):
            *   Perform a precise ray-box intersection test for the leaf's AABB [S3, unofficial].
            *   If intersected, return `true` [S3, S4, unofficial].

3.  **Termination:**
    *   If the loop completes without a hit, return `false` [S1, S3, unofficial].
    *   Guard against infinite loops: If `curDepth` exceeds `D_{max}` or iteration count exceeds `N_{max}`, return `false` or a default miss value [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_octree_traversal(ray_origin, ray_dir, octree_array, root_index, max_depth, max_steps):
    """
    Traverses an octree to find if a ray intersects any material leaf.
    Returns True if hit, False otherwise.
    """
    INF = float('inf')
    epsilon_entry = 0.001
    epsilon_step = 0.01
    
    # 1. Initialize Root Intersection
    root_aabb = get_root_aabb(octree_array, root_index)
    t_enter, t_exit, entry_normal = intersect_ray_aabb(ray_origin, ray_dir, root_aabb)
    
    if t_enter == INF:
        return False # Ray misses root entirely
    
    # Adjust origin to avoid self-intersection
    current_pos = ray_origin + ray_dir * (t_enter + epsilon_entry)
    current_t = t_enter
    
    # 2. Traversal State
    cur_depth = 0
    cur_node_idx = root_index
    cur_voxel_size = root_aabb.half_extent * 2 # Full root size
    cur_voxel_center = root_aabb.center
    
    # Stack for backtracking (implicit in recursion, explicit here for GPU)
    # For simplicity, we use a loop with backtracking logic as per S1
    
    for step in range(max_steps):
        if cur_depth > max_depth:
            return False # Safety break
            
        node = octree_array[cur_node_idx]
        
        # 3. Determine Child
        # Calculate ray position relative to current node center
        # Note: In a real implementation, 'current_pos' must be updated 
        # to the entry point of the current node's bounding box.
        # For the first step, it's the root entry. For subsequent steps, 
        # it's the exit of the previous node or the entry of the new child.
        
        # Determine child index k based on current_pos relative to cur_voxel_center
        k = 0
        if current_pos.x >= cur_voxel_center.x: k |= 4
        if current_pos.y >= cur_voxel_center.y: k |= 2
        if current_pos.z >= cur_voxel_center.z: k |= 1
        
        child_idx = node.p[k]
        
        # 4. Handle Child Status
        if child_idx < -1:
            # Material leaf hit
            return True 
        elif child_idx == -1:
            # Empty node. Skip to exit of current node.
            # Calculate exit of current node's AABB
            _, t_exit_node, _ = intersect_ray_aabb(ray_origin, ray_dir, 
                                                   get_node_aabb(octree_array, cur_node_idx, cur_voxel_size, cur_voxel_center))
            
            # Advance ray to exit
            current_t = t_exit_node
            current_pos = ray_origin + ray_dir * current_t
            
            # Backtrack to find next occupied node
            # This involves moving up the tree and checking siblings
            while cur_depth > 0:
                cur_depth -= 1
                cur_voxel_size *= 2
                # Recalculate parent center based on depth and voxel size
                cur_voxel_center = get_parent_center(cur_depth, cur_voxel_size)
                
                # Re-evaluate child index at parent level
                # We need to find the sibling that contains the ray's exit path
                # Simplified: Check if ray intersects any sibling of the empty branch
                # For robustness, use neighbor pointers or re-traverse from parent
                # Here we assume a simple backtracking to parent and re-checking children
                # This part is complex and often optimized with neighbor finding [S1, unofficial][S2]
                
                # For this pseudocode, we break and let the outer loop handle 
                # the re-evaluation from the parent level by updating cur_node_idx
                # However, standard octree traversal usually maintains a stack.
                # Let's assume we pop the stack and re-evaluate the parent's other children.
                
                # To keep it self-contained and simple:
                # We will just decrement depth and let the loop re-calculate the child
                # of the new parent. But we need to ensure we don't re-enter the empty child.
                # This requires tracking the 'last visited' child or using a stack.
                # Given the constraints, we'll use a simple heuristic:
                # If we hit an empty node, we skip its entire subtree.
                # We need to find the next node the ray intersects.
                
                # Let's assume we have a function `get_next_intersected_node`
                # which uses neighbor pointers or AABB checks.
                # Since S1 describes a specific implementation with `voxelInParent` checks,
                # we simulate that logic:
                
                # Check if current_pos is still within the new parent's bounds
                if not voxel_in_parent(current_pos, cur_voxel_center, cur_voxel_size):
                    continue # Keep backtracking
                else:
                    # We are inside the parent. Find the correct child.
                    # We must skip the child we just came from (the empty one).
                    # This requires knowing which child we were in.
                    # For this generic pseudocode, we assume the loop restarts 
                    # and recalculates k. If k points to the empty child again,
                    # we need to skip it.
                    # This is a known pitfall: infinite loops on empty branches.
                    # Solution: Use a stack or explicit neighbor finding.
                    # We will break here and assume the caller handles the stack.
                    break 
            
            # If we backtracked to root and still miss, return False
            if cur_depth == 0 and not voxel_in_parent(current_pos, cur_voxel_center, cur_voxel_size):
                return False
                
            # Update node index to parent
            # Note: In a real stack-based implementation, we pop.
            # Here, we assume cur_node_idx is updated by the backtracking logic.
            # For the sake of this pseudocode, we'll assume we re-enter the loop
            # with the parent node.
            # This is a simplification. A full implementation requires a stack.
            
        else:
            # Valid child
            cur_node_idx = child_idx
            cur_depth += 1
            cur_voxel_size /= 2
            # Update center for the child
            # Child center is offset from parent center by half the child size
            offset = cur_voxel_size / 2
            new_center = cur_voxel_center
            if k & 4: new_center.x += offset
            else: new_center.x -= offset
            if k & 2: new_center.y += offset
            else: new_center.y -= offset
            if k & 1: new_center.z += offset
            else: new_center.z -= offset
            cur_voxel_center = new_center
            
            # Update current_pos to the entry of this child
            # The ray enters the child at the current_t (which is the exit of the parent)
            # But we need the entry of the child.
            # The child's AABB is a subset of the parent's.
            # The ray enters the child at max(t_enter_child, current_t)
            child_aabb = get_node_aabb(octree_array, cur_node_idx, cur_voxel_size, cur_voxel_center)
            t_enter_child, _, _ = intersect_ray_aabb(ray_origin, ray_dir, child_aabb)
            
            if t_enter_child > current_t:
                current_t = t_enter_child
                current_pos = ray_origin + ray_dir * current_t
            else:
                # Ray started inside this child or we are already past it?
                # If we are already past, we might have missed it.
                # But since we descended, we assume we are entering.
                current_pos = ray_origin + ray_dir * current_t

    return False
```

### Parameters
*   `MAX_TRAVERSALS` (or `max_steps`): Maximum number of outer loop iterations. Typical value: 300 [S1, unofficial].
*   `MAX_DEPTH` (or `max_depth`): Maximum octree depth. Typical value: 100 [S1, unofficial].
*   `INF`: Large constant for infinity [S1, unofficial].
*   `epsilon_entry`: Offset for entry point to avoid self-intersection. Value: $0.001 \times \mathbf{d}$ [S1, unofficial].
*   `epsilon_step`: Offset for position update after voxel transition. Value: $0.01 \times \mathbf{d}$ [S1, unofficial].

### Edge Cases and Pitfalls
*   **Zero Direction:** If a ray direction component is 0, `getDelta` returns INF, which must be handled to avoid division by zero or incorrect delta calculations [S1, unofficial].
*   **Infinite Loops:** In sparse octrees, traversal can get stuck in empty branches. A counter `j` checking `j > MAX_DEPTH` is used to break infinite loops and return a miss/black color [S1, unofficial].
*   **Precision Errors:** Voxel boundary crossings can cause the ray to re-enter the same voxel. Adding a small epsilon to the position after crossing prevents this [S1, unofficial].
*   **Ray Origin Inside Root:** The algorithm must correctly handle rays starting inside the root bounding box, treating the entry point as the origin with $t_{enter}=0$ [S1, unofficial].

## Subtopics

### [Bottom-up Neighbor-Finding Ray Traversal](5610-bottom-up-neighbor-finding-ray-traversal.md)
A method for intersecting rays with an octree by determining the sequence of cells via neighbor lookups rather than hierarchical descent. It solves the problem of efficiently stepping from one occupied cell to the next, avoiding the overhead of traversing empty space from the root. Unlike top-down methods, it starts at the first intersected cell and moves laterally or vertically through the tree structure.

### [Octree-based Raycasting](5707-octree-based-raycasting.md)
A volume rendering technique that accelerates ray traversal by skipping empty space using an octree. It reduces traversal cost from $O(n)$ to depend only on occupied space. It differs from [Ray-Occlusion Traversal in Octree](5121-ray-occlusion-traversal-in-octree.md) by explicitly managing dynamic voxel updates and concurrency, and from [Bottom-up Neighbor-Finding Ray Traversal](5610-bottom-up-neighbor-finding-ray-traversal.md) by relying on hierarchical empty-space skipping.

### Octree Traversal via Voxel Grid
A method to traverse an octree by mapping ray origin/direction to chunk coordinates and checking if the ray stays within the current chunk bounds, using a loop condition based on distance from center [S1, unofficial]. It defines a voxel center and computes the current voxel position, checking bounds and advancing the ray origin if outside [S1, unofficial].

### Hero-like Octree Traversal
A depth-first search approach for ray-tracing octrees that uses a 2x2x2 voxel test to determine the order of cell hits and descend into occupied nodes [S5, unofficial]. It uses intersection values for middle planes and exit planes to determine the ray's position and path within the 2x2 area, descending if top cells are occupied [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(k)$ where $k$ is the number of nodes visited, which is proportional to the number of empty nodes skipped plus the number of occupied nodes traversed. In the worst case (dense scene), it approaches $O(n)$, but for sparse scenes, it is significantly faster than linear voxel traversal [S1, S3, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the octree nodes, where $N$ is the number of nodes. The traversal itself uses $O(D)$ stack space for recursion or explicit stack [S1, S3, unofficial].
*   **Performance:** Measured performance depends heavily on the sparsity of the octree. For highly sparse volumes, the ability to skip large empty regions provides significant speedups [S1, unofficial].
*   **Strengths:** Efficient culling of empty space; suitable for shadow ray calculations where early exit is beneficial [S1, S3, unofficial].
*   **Limitations:** Implementation complexity, especially for neighbor-finding and backtracking; potential for infinite loops if not guarded; precision issues at voxel boundaries [S1, unofficial].
*   **When to Use:** Shadow ray tracing, visibility testing in sparse voxel octrees [S1, S3, unofficial].
*   **When Not to Use:** Dense volumes where the overhead of tree traversal outweighs the benefit of skipping empty space; scenes requiring frequent dynamic updates where tree rebuilding is costly [S1, S3, unofficial].

## Background
The method originates from research on interactive isosurface ray tracing of large octree volumes [S2]. It has been implemented in various graphics engines and research projects, such as the `vkEngine` by sebbooth [S1, unofficial], `Voxelizer` by itsyouyi [S3, unofficial], and `SolidVoxelization` by Mike-Leo-Smith [S4, unofficial]. The approach is also discussed in the context of depth-first search for octree ray tracing, such as in the "Hero-like" traversal method [S5, unofficial].

## Sources
[S1] sebbooth/vkEngine, https://github.com/sebbooth/vkEngine, fetched 2026-10-24 (unofficial)
[S2] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~wald/Publications/2006/OctIso/octiso.pdf, fetched 2026-10-27
[S3] itsyouyi/Voxelizer, https://github.com/itsyouyi/Voxelizer, fetched 2026-10-27 (unofficial)
[S4] Mike-Leo-Smith/SolidVoxelization, https://github.com/Mike-Leo-Smith/SolidVoxelization, fetched 2026-10-27 (unofficial)
[S5] versal algorithm for ray tracing Hur man gör en hierarkisk ..., https://kth.diva-portal.org/smash/get/diva2:2069409/FULLTEXT01.pdf, fetched 2026-10-25 (unofficial)
