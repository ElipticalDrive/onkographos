# Octree Ray Marching with Brick Interpolation and Early Termination

Also called: Brick-based SVO Ray Marching, GigaVoxels Ray Marching

## Overview
This method performs volume rendering by traversing a Sparse Voxel Octree (SVO) using a hybrid approach that combines coarse space-skipping via precomputed "bricks" (sub-cubes of voxels) with fine-grained interpolation within those bricks. It solves the performance bottleneck of standard SVO traversal by reducing the number of node fetches and memory accesses required to traverse empty or uniform regions. It sets itself apart from [SVO Ray Marching with Stack-Based Navigation](2454-svo-ray-marching-with-stack-based-navigation.md) by utilizing brick interpolation to handle view-dependent sampling rates more efficiently, avoiding the overhead of maintaining a full traversal stack for every pixel.

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin ($O$) and direction ($D$) in world space; Sparse Voxel Octree structure; Brick textures (scalar values and optionally gradients); Transfer function; Camera projection parameters.
*   **Output:** Color ($C$) and accumulated opacity ($\alpha$) for the pixel.

**Data Structures:**
*   **Octree Node:** Contains a pointer to child nodes (or null for leaves) and metadata including brick dimensions and offsets.
*   **Brick:** A fixed-size sub-cube of voxels (e.g., $8^3$ or $16^3$) stored in a texture. Bricks are stored as unmodified scalar values to allow dynamic transfer function application in the fragment shader without texture regeneration [S1, unofficial].
*   **Node Usage List:** A list tracking recently traversed nodes to optimize memory management and brick loading [S1, unofficial].

**Parameters:**
*   `projVoxelSize`: Projected voxel size in world coordinates, used to determine interpolation ratios [S1, unofficial].
*   `parentVoxelSize`: Voxel size at the parent octree level, used in the interpolation denominator [S1, unofficial].
*   `k_ChildNodeSizeScalar`: Scalar multiplier applied to `nodeSize` and `voxelHalfSize` on each descent step [S1, unofficial].
*   `MaxNodesToPushThisFrame`: Maximum number of nodes allowed to be pushed to the usage list per frame [S1, unofficial].
*   `maxNodesPerPixelBlock`: Threshold for `nodeListTraversalCount` before it resets to 0 [S1, unofficial].
*   `startPushingNodes`, `stopPushingNodes`: Range for pushing nodes to the usage list [S1, unofficial].

**Steps:**

1.  **Initialization:**
    *   Reset traversal state: Set `octTreeNode` to `rootOctTreeNode`, `extents` to `VolExtentMin`/`VolExtentMax`, `nodeSize` to $1.0f$, and `voxelHalfSize` to `RootVoxelHalfSize` [S1, unofficial].
    *   Compute ray containment test:
        $$ test1 = \text{sign}(O - \text{VolExtentMin}) $$
        $$ test2 = \text{sign}(\text{VolExtentMax} - O) $$
        $$ inside = \text{dot}(test1, test2) $$
        If $inside < 3$, the ray does not intersect the volume extent [S1, unofficial].
    *   Initialize counters: `traversalCount = 0`, `nodeListTraversalCount = 0`, `pushedNodeCount = 0` [S1, unofficial].

2.  **Traversal Loop:**
    *   Increment counters: `++traversalCount`, `++nodeListTraversalCount`. If `nodeListTraversalCount >= maxNodesPerPixelBlock`, reset `nodeListTraversalCount` to 0 [S1, unofficial].
    *   **Node Management:** If `nodeListTraversalCount` is within `[startPushingNodes, stopPushingNodes)` and `pushedNodeCount < MaxNodesToPushThisFrame`, increment `pushedNodeCount` and call `AddToNodeUsageList(octTreeNode)` [S1, unofficial].
    *   **Interpolation Decision:**
        *   Compute interpolation factor based on `projVoxelSize` and `parentVoxelSize`.
        *   Check conditions: If `projVoxelSize > parentVoxelSize`, `curDepth != 0`, and `interpolation factor > 0.001f`, proceed to **Brick Interpolation** step. Otherwise, proceed to **Brick Traversal** step.
        *   *Pitfall:* Interpolation is disabled at the root node (`curDepth == 0`) to prevent invalid ratios when `parentVoxelSize` equals `projVoxelSize` [S1, unofficial].
        *   *Pitfall:* Interpolation factor is clamped to $[0.0f, 1.0f]$ and thresholded at $0.001f$ to avoid division by zero or negligible interpolation steps [S1, unofficial].

    *   **Brick Interpolation Step:**
        *   Compute `prevBrickRayPos` based on previous step position.
        *   Call `TraverseBrickInterp(rayBrickPosition, prevBrickRayPos, ...)` to sample the brick texture with interpolation.
        *   Update accumulated color and opacity using front-to-back compositing.

    *   **Brick Traversal Step:**
        *   Call `TraverseBrick(rayBrickPosition, ...)` to sample the brick texture without interpolation (nearest neighbor or simple fetch).
        *   Update accumulated color and opacity.

    *   **Descent Logic (if node is not a leaf or brick is not fully traversed):**
        *   Compute `brickOffset = rayBrickPosition * k_TreeN` (where $k\_TreeN$ is the number of bricks per dimension) [S1, unofficial].
        *   Clamp to `childOffset = clamp(uvec3(brickOffset), 0u, 1u)` [S1, unofficial].
        *   Update `rayBrickPosition = brickOffset - vec3(childOffset)` [S1, unofficial].
        *   Update extents:
            $$ nodeExtentMin += (vec3(childOffset) * nodeSize) $$
            $$ nodeExtentMax -= ((vec3(1.0f) - vec3(childOffset)) * nodeSize) $$
        *   Fetch next octree node: `octTreeNode = texelFetch(OctTreeSampler, octTreeNodePtr, 0)` [S1, unofficial].
        *   Update `nodeSize *= k_ChildNodeSizeScalar` and `voxelHalfSize *= k_ChildNodeSizeScalar` [S1, unofficial].
        *   Increment `curDepth` [S1, unofficial].

    *   **Ascent/Backtracking:**
        *   If the current node is a leaf or the ray has exited the brick, backtrack to the parent node.
        *   Update `rayBrickPosition` and `nodeSize` accordingly.
        *   Decrement `curDepth` [S1, unofficial].

    *   **Early Termination:**
        *   If accumulated opacity $> 0.995$ AND the current step actually traversed a brick or constant value, terminate the ray march [S1, unofficial].
        *   *Pitfall:* Early termination only triggers if the current step traversed a brick/constant value, preventing premature cutoff during initialization or empty space [S1, unofficial].

3.  **Compositing:**
    *   For each step, compute the sample value $V$ from the brick texture.
    *   Apply transfer function to get color $C_s$ and opacity $\alpha_s$.
    *   Update accumulated color $C$ and opacity $\alpha$:
        $$ \alpha = \alpha + (1 - \alpha) \alpha_s $$
        $$ C = C + (1 - \alpha) C_s \alpha_s $$
        (Note: Standard front-to-back compositing formula, adjusted for the specific implementation details if necessary [G]).

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def octree_ray_marching_brick_interpolation(ray_origin, ray_direction, octree, brick_textures, transfer_function):
    # Initialization
    oct_tree_node = octree.root
    extents_min = vol_extent_min
    extents_max = vol_extent_max
    node_size = 1.0
    voxel_half_size = root_voxel_half_size
    cur_depth = 0
    traversal_count = 0
    node_list_traversal_count = 0
    pushed_node_count = 0
    
    # Ray containment test
    test1 = sign(ray_origin - extents_min)
    test2 = sign(extents_max - ray_origin)
    inside = dot(test1, test2)
    if inside < 3:
        return clear_color, 0.0

    accumulated_color = vec3(0.0)
    accumulated_opacity = 0.0
    ray_brick_position = compute_brick_position(ray_origin)
    prev_brick_ray_pos = ray_brick_position

    while accumulated_opacity < 0.995:
        traversal_count += 1
        node_list_traversal_count += 1
        
        # Node management
        if start_pushing_nodes <= node_list_traversal_count < stop_pushing_nodes:
            if pushed_node_count < max_nodes_to_push_this_frame:
                pushed_node_count += 1
                add_to_node_usage_list(oct_tree_node)
        
        if node_list_traversal_count >= max_nodes_per_pixel_block:
            node_list_traversal_count = 0

        # Interpolation decision
        proj_voxel_size = compute_projected_voxel_size(ray_origin, ray_direction, cur_depth)
        parent_voxel_size = compute_parent_voxel_size(cur_depth)
        interpolation_factor = compute_interpolation_factor(proj_voxel_size, parent_voxel_size)
        
        # Clamp and threshold interpolation factor
        interpolation_factor = clamp(interpolation_factor, 0.0, 1.0)
        
        # Pitfall: Disable interpolation at root
        if cur_depth == 0:
            interpolation_factor = 0.0
            
        if proj_voxel_size > parent_voxel_size and cur_depth != 0 and interpolation_factor > 0.001:
            # Brick Interpolation
            accumulated_color, accumulated_opacity = traverse_brick_interpolation(
                ray_brick_position, prev_brick_ray_pos, oct_tree_node, brick_textures, 
                transfer_function, node_size, accumulated_color, accumulated_opacity
            )
        else:
            # Brick Traversal
            accumulated_color, accumulated_opacity = traverse_brick(
                ray_brick_position, oct_tree_node, brick_textures, 
                transfer_function, node_size, accumulated_color, accumulated_opacity
            )
            
        # Early termination check
        if accumulated_opacity > 0.995:
            # Pitfall: Ensure we actually traversed a brick/constant value
            if step_traversed_brick_or_constant:
                break

        # Descent logic
        if not is_leaf(oct_tree_node):
            brick_offset = ray_brick_position * k_tree_n
            child_offset = clamp(uvec3(brick_offset), 0, 1)
            ray_brick_position = brick_offset - vec3(child_offset)
            
            extents_min += vec3(child_offset) * node_size
            extents_max -= (vec3(1.0) - vec3(child_offset)) * node_size
            
            oct_tree_node = texel_fetch(oct_tree_sampler, oct_tree_node_ptr, 0)
            node_size *= k_child_node_size_scalar
            voxel_half_size *= k_child_node_size_scalar
            cur_depth += 1
            prev_brick_ray_pos = ray_brick_position
        else:
            # Ascent/Backtracking
            # Update ray_brick_position and node_size for parent
            # Decrement cur_depth
            # Fetch parent node
            pass # Implementation details for backtracking depend on specific SVO structure

    return accumulated_color, accumulated_opacity
```

## Subtopics

### [Two-staged Space-Skipping with Bricking and Octrees](5210-two-staged-space-skipping-with-bricking-and-octrees.md)
This method balances GPU bottlenecks by dividing space-skipping into two stages: coarse division using bricking to address bus and texture cache limitations, and fine division using octrees within bricks to address the rasterization bottleneck. It differs from Tong et al.'s bricking technique by storing unmodified scalar values in textures and using a fragment shader for post-interpolative RGB$\alpha$ lookup, avoiding time-consuming texture regeneration when the transfer function changes.

### [Stackless GPU Raycasting with Octree Traversal](5222-stackless-gpu-raycasting-with-octree-traversal.md)
This method performs volume rendering by traversing a Sparse Voxel Octree (SVO) on the GPU without using a stack, computing the volume rendering integral in front-to-back order. It solves the problem of efficient memory access and traversal overhead in sparse volumetric data by combining space-skipping (via empty node detection) with adaptive sampling density. It sets itself apart from sibling methods like [Two-staged Space-Skipping with Bricking and Octrees](5210-two-staged-space-skipping-with-bricking-and-octrees.md) by performing neighbor lookups implicitly on-the-fly using the octree's regular structure, rather than relying on precomputed neighbor pointers or a two-stage bricking pipeline. It builds on the concepts of [Octree Ray Marching with Brick Interpolation and Early Termination](4147-octree-ray-marching-with-brick-interpolation-and-early-termi.md) but replaces explicit brick interpolation with adaptive sampling based on leaf resolution.

## Operating characteristics
*   **Time Complexity:** Depends on the sparsity of the volume and the depth of the octree. Brick interpolation reduces the number of texture fetches compared to per-voxel traversal. Early termination further reduces cost for opaque regions.
*   **Memory Complexity:** Requires storage for the octree nodes and brick textures. The brick size (e.g., $8^3$ or $16^3$) impacts memory usage and cache efficiency.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales well with resolution due to the adaptive nature of the octree and brick interpolation. Performance is sensitive to the brick size and the efficiency of the texture sampling.
*   **Strengths:** Reduces memory bandwidth usage by fetching multiple voxels per texture access; handles view-dependent sampling rates efficiently via interpolation; early termination speeds up rendering of opaque objects.
*   **Limitations:** Interpolation artifacts may occur if the brick size is too large or the transfer function is highly non-linear; requires careful tuning of parameters like `projVoxelSize` and interpolation thresholds.
*   **When to use:** Suitable for real-time volume rendering of large, sparse datasets where memory bandwidth is a bottleneck.
*   **When not to use:** Not suitable for volumes with very fine, high-frequency details that require per-voxel accuracy without interpolation.

## Background
This method is based on the GigaVoxels framework, which was developed to enable real-time exploration of large volumetric datasets. The specific implementation details, including brick interpolation and early termination, are derived from the VoxViz project and associated thesis [S1, unofficial]. The approach addresses the limitations of traditional ray casting on sparse voxel grids by leveraging the hierarchical structure of the octree and the spatial coherence of volumetric data.

## Sources
[S1] apecoraro/VoxVizThesis, https://github.com/apecoraro/VoxVizThesis, fetched 2026-10-27 (unofficial)
