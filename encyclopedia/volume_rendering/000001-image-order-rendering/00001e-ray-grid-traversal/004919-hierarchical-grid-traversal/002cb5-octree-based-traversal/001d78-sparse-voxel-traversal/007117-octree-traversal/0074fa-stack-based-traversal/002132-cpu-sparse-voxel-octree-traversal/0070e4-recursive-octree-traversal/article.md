# Recursive Octree Traversal
Also called: Octree Node Traversal (Standard), Raycast Entry Point

## Overview
Recursive octree traversal is a standard algorithm for ray casting against a 3D sparse voxel octree, where the traversal logic is implemented via a recursive function that visits nodes in depth-first order [S1, unofficial]. It solves the problem of efficiently locating and sampling leaf nodes (or specific depth levels) along a ray by recursively descending into child nodes only when the ray intersects their axis-aligned bounding boxes (AABBs) [S1, unofficial]. It differs from its siblings by relying on the call stack for state management rather than an explicit stack or parametric surface equations [S1, unofficial].

## How it works
**Inputs and outputs:**
- **Inputs:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, octree root node, maximum traversal depth, and a branching mode (e.g., Wide4) [S1, unofficial].
- **Outputs:** A hit record (closest intersection, leaf ID, depth) or an empty result if no intersection occurs [S1, unofficial].

**Data structures:**
- **NodeDescriptor:** A compact structure stored in a linear array. It contains:
  - `child_mask`: `std::uint8_t`, bitmask of existing children [S1, unofficial].
  - `leaf_mask`: `std::uint8_t`, bitmask of children that are leaves [S1, unofficial].
  - `internal_child_mask`: `std::uint8_t`, bitmask of children that are internal nodes [S1, unofficial].
  - `payload_base`: Implied offset to leaf data [S1, unofficial].
- **Morton Code:** Used to map spatial position to linear index. Calculated as `parent_morton | (i << (3 * (depth - 1)))` for the $i$-th child [S1, unofficial].

**Algorithm Steps:**
1. **Validation:** Validate that the ray origin is finite and the direction is non-zero. Normalize the direction [S1, unofficial].
2. **Root Check:** If the octree is empty, return an empty hit. Get the root AABB and intersect the ray with it. If no intersection, return empty [S1, unofficial].
3. **Depth Check:** If `max_depth` is 0, treat the root as a leaf with `leaf_id=0` and `depth=0`, then return [S1, unofficial].
4. **Dispatch:** If the branching mode is `Wide4`, call `traverse_wide_node`. Otherwise, call `traverse_node` with the root, origin, normalized direction, and initial $t$-near/far values [S1, unofficial].
5. **Recursive Traversal (`traverse_node`):**
   - **Base Case:** If `node_index` is out of bounds or `depth_remaining <= 0`, return [S1, unofficial].
   - **Descriptor Load:** Retrieve `NodeDescriptor` for `node_index`. Extract `child_mask`, `leaf_mask`, and `internal_child_mask` [S1, unofficial].
   - **Child Iteration:** Loop through 8 children ($i = 0$ to $7$).
     - **Mask Check:** If the bit `(1u << i)` is not set in `child_mask`, skip [S1, unofficial].
     - **AABB Intersection:** Compute the child's AABB using `child_bounds(bounds, i)`. Intersect the ray with this AABB. If no intersection, continue [S1, unofficial].
     - **Pruning:** If a previous hit exists (`best.hit`) and the new $t_{near} > best.t + \epsilon$, skip this child [S1, unofficial].
     - **Leaf Handling:** If the bit `(1u << i)` is set in `leaf_mask`:
       - Compute `leaf_id` using `descriptor.payload_base()` and `prefix_rank(leaf_mask, i)` [S1, unofficial].
       - Compute `depth` as `octree.max_depth() - depth_remaining + 1` [S1, unofficial].
       - Call `consider_leaf(child_aabb, leaf_id, depth)` [S1, unofficial].
     - **Internal Node Recursion:** If the bit `(1u << i)` is set in `internal_child_mask`:
       - Calculate the child's Morton code: `parent_morton | (i << (3 * (depth - 1)))` [S1, unofficial].
       - Recursively call `traverse_node` with the child's address/offset and `depth + 1` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_recursive(node_index, depth_remaining, ray_origin, ray_direction, t_near, t_far, octree, best_hit):
    # Base case: out of bounds or depth limit reached
    if node_index >= octree.num_nodes or depth_remaining <= 0:
        return

    # Load node descriptor
    descriptor = octree.nodes[node_index]
    child_mask = descriptor.child_mask
    leaf_mask = descriptor.leaf_mask
    internal_child_mask = descriptor.internal_child_mask

    # Iterate over 8 children
    for i in range(8):
        child_bit = 1 << i
        
        # Skip if child does not exist
        if not (child_bit & child_mask):
            continue

        # Compute child AABB
        child_aabb = compute_child_aabb(octree.root_bounds, node_index, i, depth_remaining)
        
        # Intersect ray with child AABB
        t_child_near, t_child_far = intersect_ray_aabb(ray_origin, ray_direction, child_aabb)
        
        # If no intersection, skip
        if t_child_far < t_near or t_child_near > t_far:
            continue

        # Pruning: if we already have a hit closer than the current node's entry
        if best_hit.hit and t_child_near > best_hit.t + EPSILON:
            continue

        # Check if leaf
        if child_bit & leaf_mask:
            # Calculate leaf ID using prefix rank
            leaf_id = descriptor.payload_base + prefix_rank(leaf_mask, i)
            depth = octree.max_depth - depth_remaining + 1
            # Process leaf (e.g., update best_hit if closer)
            if t_child_near < best_hit.t:
                best_hit.t = t_child_near
                best_hit.leaf_id = leaf_id
                best_hit.depth = depth
                best_hit.hit = True
        else:
            # Recurse into internal node
            # Calculate child node index or offset
            child_index = get_child_index(octree, node_index, i)
            traverse_recursive(child_index, depth_remaining - 1, ray_origin, ray_direction, t_child_near, t_child_far, octree, best_hit)

def raycast_one(octree, ray_origin, ray_direction):
    # Validate inputs
    if not is_finite(ray_origin) or not is_finite(ray_direction) or length(ray_direction) == 0:
        raise ValidationError("Invalid ray")
    
    normalized_direction = normalize(ray_direction)
    
    # Check if octree is empty
    if octree.num_nodes == 0:
        return EmptyHit()
    
    # Check root intersection
    root_aabb = octree.root_bounds
    t_near, t_far = intersect_ray_aabb(ray_origin, normalized_direction, root_aabb)
    if t_far < 0 or t_near > t_far:
        return EmptyHit()
    
    # Handle max_depth == 0 case
    if octree.max_depth == 0:
        return Hit(leaf_id=0, depth=0, t=t_near)
    
    # Dispatch based on branching mode
    best_hit = Hit()
    best_hit.t = infinity
    
    if octree.branching_mode == Wide4:
        traverse_wide_node(octree.root, ray_origin, normalized_direction, t_near, t_far, best_hit)
    else:
        traverse_recursive(octree.root_index, octree.max_depth, ray_origin, normalized_direction, t_near, t_far, octree, best_hit)
        
    return best_hit
```

**Parameters:**
- `BranchingMode::Wide4`: Indicates the octree uses wide nodes (4x4x4 cells) for traversal [S1, unofficial].
- `max_depth`: Maximum depth of the octree. Typically in the range [0, 30] [S1, unofficial].

**Edge cases and pitfalls:**
- **Precision:** Use `prefix_rank` to correctly map child indices to payload indices based on sparse masks [S1, unofficial].
- **Boundaries:** Ensure $t_{near}$ is clamped to 0.0 if the ray origin is inside the volume [S1, unofficial].
- **Degenerate Input:** Throw `ValidationError` if the ray direction is zero or origin is non-finite [S1, unofficial].

## Subtopics

### [Top Level Octree Rendering](4990-top-level-octree-rendering.md)
Top Level Octree Rendering is a rendering technique that stops octree traversal at a specific depth level determined by the pixel's y-coordinate, then applies lighting. It differs from standard recursive traversal by using the vertical screen position to limit the traversal depth, effectively culling deeper nodes based on their apparent height in the viewport.

## Operating characteristics
- **Time Complexity:** $O(N)$ in the worst case where $N$ is the number of nodes visited, but typically much faster for sparse volumes due to early culling of empty space.
- **Memory Complexity:** $O(D)$ where $D$ is the maximum depth of the octree, due to the recursion stack.
- **Performance:** Efficient for CPU implementation due to simple control flow. Performance degrades if the ray passes through many small, non-empty nodes.
- **Strengths:** Simple to implement, handles sparse data well, easy to integrate with other recursive algorithms (e.g., path tracing).
- **Limitations:** Stack depth limits maximum octree depth. Not suitable for GPU where explicit stacks or wide traversal are preferred.

## Background
Recursive octree traversal is a fundamental technique in volume rendering and geometric processing. It is widely used in sparse voxel octree renderers, dual contouring implementations, and procedural generation pipelines. The method is described in various open-source implementations for sparse voxel visualization and mesh generation [S1, unofficial][S2, unofficial][S3, unofficial][S4, unofficial].

## Sources
[S1] hectorpiteau/sparse-voxels, https://github.com/hectorpiteau/sparse-voxels, fetched 2026-10-24 (unofficial)
[S2] nickgildea/DualContouringSample, https://github.com/nickgildea/DualContouringSample, fetched 2026-10-26 (unofficial)
[S3] Tsarpf/procgen, https://github.com/Tsarpf/procgen, fetched 2026-10-26 (unofficial)
[S4] Svenny/isomesh, https://github.com/Svenny/isomesh, fetched 2026-10-27 (unofficial)
[S5] loryruta/voxelizer, https://github.com/loryruta/voxelizer, fetched 2026-10-27 (unofficial)
[S6] GitHub - saurlax/model_voxelizer: A 3D model voxelization tool that ..., https://github.com/saurlax/model_voxelizer, fetched 2026-10-30 (unofficial)
