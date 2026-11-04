# Ray-Occlusion Traversal in Octree

Also called: Octree Ray Traversal, Octree Shadow Ray Traversal, Ray-Octree Traversal, enterOctree, genShadowRay, traverseOctree, traverseOctreeShadowRay

## Overview
Ray-Occlusion Traversal in Octree is a method for determining shadow ray intersections within a hierarchical voxel (octree) structure by traversing the tree to find the first non-empty leaf or surface [S1, S3, unofficial]. It handles rays starting inside or outside the root voxel by computing entry points via bounding box intersection and stepping through empty space or descending into child nodes [S1, unofficial]. It sets itself apart from its siblings by explicitly managing shadow ray occlusion logic, including specific attenuation factors and boundary precision handling, whereas [Octree Path Computation](897-octree-path-computation.md) focuses on path generation and [Compositing Tree Rendering](5860-compositing-tree-rendering.md) handles alpha blending.

## How it works
**Inputs:** Ray origin ($O$), ray direction ($D$), octree structure (nodes, depths, bounds), maximum traversal depth ($MAX\_DEPTH$), maximum traversal steps ($MAX\_TRAVERSALS$).
**Outputs:** Intersection point, shadow factor, or miss status.

**Data Structures:**
*   `OctreeNode`: Struct containing 8 integer child pointers (`p0` to `p7`). Indices $\ge 0$ point to child nodes in `octreeArray`; indices $< 0$ (specifically $-1$) indicate no child or leaf status [S1, unofficial].
*   `octreeArray`: Flat vector of `OctreeNode` structs, stored in reverse depth order (deepest layers first) [S1, unofficial].
*   `OctreeSSBO`: Read-only storage buffer containing `octreeArray` [S1, unofficial].
*   `parentVoxels`: Array tracking the voxel coordinates of the parent node at each depth [S1, unofficial].
*   `nodeIndices`: Array tracking the index in `octreeArray` for the current node at each depth [S1, unofficial].

**Parameters:**
*   $MAX\_TRAVERSALS = 300$ [S1, unofficial].
*   $MAX\_DEPTH = 100$ [S1, unofficial].
*   $INF$: Large constant for invalid intersections [S1, unofficial].
*   $\epsilon_{entry} = 0.001$: Offset for collision point validation [S1, unofficial].
*   $\epsilon_{step} = 0.01$: Offset for position update after voxel transition [S1, unofficial].
*   Shadow attenuation multiplier: $0.3$ [S1, unofficial].

**Algorithm Steps:**

1.  **Ray-Root Intersection (Entry Point Calculation):**
    If the ray origin is outside the root voxel bounding box, calculate intersection distances ($xDist, yDist, zDist$) from $O$ to the six planes of the root voxel defined by `rootNodePos` and `voxelSize` [S1, unofficial].
    *   For each axis, use `intersectRayPlane` to find near/far plane intersections based on ray direction signs [S1, unofficial].
    *   Clamp negative distances to $INF$ [S1, unofficial].
    *   Compute candidate collision points: $xCollision = O + xDist \cdot D$, etc. [S1, unofficial].
    *   Validate points using `voxelInParent`. Add $\epsilon_{entry} \cdot D$ to the collision point along the intersecting axis to avoid self-intersection [S1, unofficial].
    *   Select minimum valid distance ($minDist$). If $minDist = INF$, return miss [S1, unofficial].
    *   Determine entry voxel index ($0-5$) based on axis and direction sign. Call `traverseOctree` with the collision point, $D$, initial voxel size, rounded voxel coordinates, root position, depth $0$, and collision type [S1, unofficial].

2.  **Initialization:**
    Set $curDepth = 0$, $nodeIndices[0] = initialIndex$, $parentVoxels[0] = parentVoxel$, $voxelSize = parentVoxelSize / 2$ [S1, unofficial].

3.  **Outer Loop (Traversal Limit):**
    Iterate $i$ from $0$ to $MAX\_TRAVERSALS - 1$ [S1, unofficial].

4.  **Descent (Node Selection):**
    While $subNodeValue \ge 0$ and $curDepth < MAX\_DEPTH$:
    *   Calculate `parentVoxels[curDepth]` by scaling `curVoxel` with `parentVoxelSize` [S1, unofficial].
    *   Calculate `curVoxelScaled` by scaling `curVoxel` with `voxelSize` [S1, unofficial].
    *   Determine child index ($p0-p7$) by comparing `curVoxelScaled` components to `parentVoxels[curDepth]` components (equality checks) [S1, unofficial].
    *   Update $subNodeValue = octree.octreeArray[nodeIndices[curDepth]].pN$ [S1, unofficial].
    *   **Leaf Handling:**
        *   If $curDepth + 1 == MAX\_DEPTH$:
            *   If $subNodeValue < -1$: Compute shadow factor ($shadowFactor *= 0.3$ if hit, else use normals). Return color [S1, unofficial].
            *   If $subNodeValue \ge -1$: Increment $curDepth$, halve sizes, break descent [S1, unofficial].
        *   If $subNodeValue == -1$: Break descent (empty leaf) [S1, unofficial].
        *   If $subNodeValue < -1$ (and not max depth): Compute shadow factor. Return color [S1, unofficial].
    *   **Advance:**
        *   If $subNodeValue > -1$: Set $nodeIndices[curDepth + 1] = subNodeValue$, increment $curDepth$, halve `parentVoxelSize` and `voxelSize` [S1, unofficial].

5.  **Voxel Advancement (Stepping):**
    If the descent loop breaks due to an empty leaf or max depth edge case:
    *   Update current position: $curPosi = curPos$. Add $\epsilon_{step} \cdot D$ to the axis of $minDelta$ in $curPosi$ [S1, unofficial].
    *   Compute new $curVoxel = roundVoxel(curPosi, 1)$ [S1, unofficial].
    *   Determine `collType` ($0-5$) based on direction sign [S1, unofficial].
    *   **Parent Boundary Check:** While `!voxelInParent(curPosi, parentVoxels[curDepth], curParentEnds)` and $curDepth \ge 0$:
        *   Decrement $curDepth$, double `parentVoxelSize` and `voxelSize`, update `curParentEnds` [S1, unofficial].
        *   If $curDepth < 0$, return miss [S1, unofficial].
    *   Advance ray position: $O += minDelta \cdot D$ [S1, unofficial].
    *   Update $curVoxel$ by rounding new position with $\epsilon_{step}$ [S1, unofficial].

6.  **Recursion/Iteration:**
    The process repeats from Step 3 with the updated state.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_octree_shadow_ray(ray_origin, ray_dir, octree, max_traversals=300, max_depth=100):
    # 1. Entry Point Calculation
    if not voxel_in_parent(ray_origin, octree.root_bounds):
        x_dist, y_dist, z_dist = intersect_ray_planes(ray_origin, ray_dir, octree.root_bounds)
        candidates = [
            ray_origin + x_dist * ray_dir,
            ray_origin + y_dist * ray_dir,
            ray_origin + z_dist * ray_dir
        ]
        valid_candidates = []
        for c in candidates:
            # Epsilon offset to avoid self-intersection
            offset_axis = get_intersecting_axis(c, ray_dir)
            c += 0.001 * ray_dir[offset_axis]
            if voxel_in_parent(c, octree.root_bounds):
                valid_candidates.append(c)
        
        if not valid_candidates:
            return MISS
        
        min_dist = min(valid_candidates, key=lambda c: length(c - ray_origin))
        entry_point = min_dist
        coll_type = get_collision_type(min_dist, ray_dir)
        cur_pos = entry_point
    else:
        cur_pos = ray_origin
        coll_type = 0 # Inside root

    cur_voxel = round_voxel(cur_pos, 1)
    cur_depth = 0
    node_indices = [octree.root_index]
    parent_voxels = [get_parent_voxel(cur_voxel)]
    voxel_size = octree.root_size / 2
    parent_voxel_size = octree.root_size
    shadow_factor = 1.0

    for i in range(max_traversals):
        sub_node_value = -1
        
        # 2. Descent
        while sub_node_value >= 0 and cur_depth < max_depth:
            parent_voxels[cur_depth] = scale_voxel(cur_voxel, parent_voxel_size)
            cur_voxel_scaled = scale_voxel(cur_voxel, voxel_size)
            
            # Determine child index based on comparison
            child_idx = 0
            if cur_voxel_scaled.x >= parent_voxels[cur_depth].x: child_idx |= 4
            if cur_voxel_scaled.y >= parent_voxels[cur_depth].y: child_idx |= 2
            if cur_voxel_scaled.z >= parent_voxels[cur_depth].z: child_idx |= 1
            
            node = octree.octree_array[node_indices[cur_depth]]
            sub_node_value = node.p[child_idx]
            
            # Leaf Handling
            if cur_depth + 1 == max_depth:
                if sub_node_value < -1:
                    shadow_factor *= 0.3 # Hit
                    return shadow_factor * determine_color(sub_node_value, coll_type)
                else:
                    cur_depth += 1
                    parent_voxel_size /= 2
                    voxel_size /= 2
                    break
            
            if sub_node_value == -1:
                break # Empty leaf
            
            if sub_node_value < -1:
                shadow_factor *= 0.3
                return shadow_factor * determine_color(sub_node_value, coll_type)
            
            # Advance
            if sub_node_value > -1:
                node_indices[cur_depth + 1] = sub_node_value
                cur_depth += 1
                parent_voxel_size /= 2
                voxel_size /= 1 # Halve size each level

        # 3. Voxel Advancement
        if sub_node_value == -1 or cur_depth + 1 == max_depth:
            # Calculate delta to next boundary
            deltas = get_deltas(cur_pos, ray_dir, voxel_size)
            min_delta = min(deltas)
            
            # Update position with epsilon
            cur_pos += min_delta * ray_dir
            axis = get_axis_of_min_delta(deltas)
            cur_pos[axis] += 0.01 * ray_dir[axis]
            
            cur_voxel = round_voxel(cur_pos, 1)
            coll_type = get_collision_type(cur_pos, ray_dir)
            
            # Parent Boundary Check
            while not voxel_in_parent(cur_pos, parent_voxels[cur_depth], get_parent_ends(cur_depth)) and cur_depth >= 0:
                cur_depth -= 1
                parent_voxel_size *= 2
                voxel_size *= 2
                if cur_depth < 0:
                    return MISS

    return MISS
```

## Subtopics

### Octree Traversal via Voxel Grid
A method to traverse an octree by mapping ray origin/direction to chunk coordinates and checking if the ray stays within the current chunk bounds, using a loop condition based on distance from center [S1, unofficial]. It defines `voxelCenter = vec3(octreeSize / 2, octreeSize / 2, octreeSize / 2)` and computes `curVoxel = octreeSize * floor(rayOrigin / octreeSize)` [S1, unofficial]. It loops while `dot(rayOrigin - voxelCenter, rayOrigin - voxelCenter) > dot(rayOrigin + rayDir - voxelCenter, voxelCenter)` [S1, unofficial]. If `curVoxel` is within bounds `[0, octreeSize)`, it returns success; otherwise, it advances `rayOrigin += rayDir` [S1, unofficial].

### Hero-like Octree Traversal
A depth-first search approach for ray-tracing octrees that uses a 2x2x2 voxel test to determine the order of cell hits and descend into occupied nodes [S4, unofficial]. It starts from the root, performs a 2x2 test to find which cells the ray hits and their order, and uses intersection values for middle planes ($t_{mx}, t_{my}$) and exit planes ($t_{ex}, t_{ey}$) to determine the ray's position and path [S4, unofficial]. If any top cells are occupied, it descends by calculating new plane positions based on cell position [S4, unofficial].

### Octree-based Raycasting
A raycasting method that uses an octree structure to skip empty space regions and accelerate traversal [S5, unofficial]. It requires updating the octree with every voxel modification: if a voxel is deleted within a node, the node's voxel count is decremented; if the count reaches 0, the node is deactivated [S5, unofficial]. Each octree node must store the exact number of contained voxels to correctly calculate deactivation status [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the depth of the octree and the number of empty nodes skipped. Hierarchical traversal reduces steps compared to non-hierarchical methods (e.g., 5 steps vs 11 in SVO examples) [S4, unofficial].
*   **Memory Complexity:** Linear with respect to the number of octree nodes, stored in a flat array [S1, unofficial].
*   **Strengths:** Efficiently skips empty space; handles rays starting inside or outside the root [S1, unofficial].
*   **Limitations:** Requires careful epsilon handling to avoid precision errors at boundaries [S1, unofficial]. Infinite loop prevention is necessary (e.g., counter check) [S1, unofficial].
*   **When to use:** Shadow ray computation in voxel-based scenes [S1, unofficial].
*   **When not to use:** When the scene is fully dense, as the overhead of tree traversal may outweigh benefits.

## Background
Originating from implementations in Vulkan-based raymarchers, this method is detailed in unofficial sources [S1, S3, unofficial]. It builds on general octree traversal techniques [S4, unofficial] and is used in real-time rendering applications like the VkEngine project [S1, unofficial]. The approach is also related to techniques used in games like Teardown [S4, unofficial].

## Sources
[S1] sebbooth/vkEngine, https://github.com/sebbooth/vkEngine, fetched 2026-10-24 (unofficial)
[S3] itsyouyi/Voxelizer, https://github.com/itsyouyi/Voxelizer, fetched 2026-10-27 (unofficial)
[S4] versal algorithm for ray tracing Hur man gör en hierarkisk ..., https://kth.diva-portal.org/smash/get/diva2:2069409/FULLTEXT01.pdf, fetched 2026-10-25 (unofficial)
[S5] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
