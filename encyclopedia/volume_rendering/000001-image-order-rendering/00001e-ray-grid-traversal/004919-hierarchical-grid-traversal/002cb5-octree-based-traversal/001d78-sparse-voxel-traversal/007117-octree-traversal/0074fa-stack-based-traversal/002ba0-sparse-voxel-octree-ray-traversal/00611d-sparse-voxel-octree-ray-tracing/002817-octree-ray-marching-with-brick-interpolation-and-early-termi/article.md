# Octree Ray Marching with Brick Interpolation and Early Termination

Also called: GigaVoxels Ray Marching, Brick-Based Octree Traversal

## Overview
This method performs volume rendering by traversing a Sparse Voxel Octree (SVO) organized into fixed-size cubic "bricks" stored in 3D textures. It solves the problem of inefficient memory access and cache thrashing during ray traversal by combining coarse space-skipping (via brick-level occupancy) with fine-grained interpolation at brick boundaries [S1, unofficial]. Unlike standard SVO traversal which steps voxel-by-voxel or node-by-node, this approach interpolates values across brick borders to maintain visual continuity while minimizing texture fetches [S1, unofficial]. It differs from [Two-staged Space-Skipping with Bricking and Octrees](5210-two-staged-space-skipping-with-bricking-and-octrees.md) by using a fragment shader for post-interpolative RGB$\alpha$ lookup rather than pre-computed textures, and from [Stackless GPU Raycasting with Octree Traversal](5222-stackless-gpu-raycasting-with-octree-traversal.md) by maintaining an explicit node stack for out-of-core management [S1, S2, unofficial].

## How it works
**Inputs:** Ray origin ($O$), ray direction ($D$), volume bounding box ($VolExtentMin, VolExtentMax$), root octree node pointer, brick dimensions ($brickDimX, brickDimY, brickDimZ$), brick border sizes ($brickBorderX, brickBorderY, brickBorderZ$), projected voxel size ($projVoxelSize$), parent voxel size ($parentVoxelSize$), and texture samplers for the octree structure and brick data (colors/gradients) [S1, unofficial].

**Outputs:** Final color and opacity for the pixel [S1, unofficial].

**Data Structures:**
*   **Octree Node:** Contains a pointer to child nodes or a brick. Stores occupancy status.
*   **Brick:** A fixed-size cubic block of voxels (e.g., $8^3$ or $16^3$). Stored in a 3D texture. Includes a border region for interpolation [S1, unofficial].
*   **Node Usage List:** A circular buffer tracking recently traversed nodes to optimize memory management (pushing nodes to GPU) [S1, unofficial].

**Parameters:**
*   $k\_ChildNodeSizeScalar$: Scalar multiplier for node size on descent (typically 0.5 for octrees) [S1, unofficial].
*   $MaxNodesToPushThisFrame$: Max nodes added to usage list per frame [S1, unofficial].
*   $maxNodesPerPixelBlock$: Threshold for resetting traversal counters [S1, unofficial].
*   Interpolation threshold: $0.001f$ [S1, unofficial].
*   Early termination opacity threshold: $0.995$ [S1, unofficial].

**Steps:**

1.  **Initialization:**
    *   Set $octTreeNode$ to root.
    *   Set $extents$ to $VolExtentMin/Max$.
    *   Set $nodeSize$ to $1.0f$ (or root voxel size).
    *   Set $voxelHalfSize$ to $RootVoxelHalfSize$.
    *   Reset traversal counters: $traversalCount = 0$, $nodeListTraversalCount = 0$, $pushedNodeCount = 0$ [S1, unofficial].

2.  **Ray Containment Test:**
    *   Compute $test1 = sign(O - VolExtentMin)$.
    *   Compute $test2 = sign(VolExtentMax - O)$.
    *   Compute $inside = dot(test1, test2)$.
    *   If $inside < 0$, the ray origin is outside; handle accordingly (e.g., skip or compute entry) [S1, unofficial].

3.  **Traversal Loop:**
    *   Increment $traversalCount$ and $nodeListTraversalCount$.
    *   If $nodeListTraversalCount \ge maxNodesPerPixelBlock$, reset $nodeListTraversalCount$ to $0$ [S1, unofficial].
    *   **Node Management:** If $nodeListTraversalCount$ is within $[startPushingNodes, stopPushingNodes)$ and $pushedNodeCount < MaxNodesToPushThisFrame$, increment $pushedNodeCount$ and call $AddToNodeUsageList(octTreeNode)$ [S1, unofficial].

4.  **Descent and Brick Handling:**
    *   **Check Interpolation Condition:**
        *   Calculate $interpolationFactor$. If $projVoxelSize > parentVoxelSize$ and $curDepth \neq 0$ and $interpolationFactor > 0.001f$, enable interpolation [S1, unofficial].
        *   *Pitfall:* Interpolation is disabled at $curDepth == 0$ (root) to prevent invalid ratios when $parentVoxelSize == projVoxelSize$ [S1, unofficial].
    *   **Traverse:**
        *   If interpolation is enabled, compute $prevBrickRayPos$ and call $TraverseBrickInterp$.
        *   Otherwise, call $TraverseBrick$ [S1, unofficial].
    *   **Inside TraverseBrick/TraverseBrickInterp:**
        *   Compute brick offset: $brickOffset = rayBrickPosition \times k\_TreeN$ (where $k\_TreeN$ is likely the number of children or a stride factor, context implies mapping to child index) [S1, unofficial].
        *   Clamp to child offset: $childOffset = clamp(uvec3(brickOffset), 0u, 1u)$ [S1, unofficial].
        *   Update ray position within brick: $rayBrickPosition = brickOffset - vec3(childOffset)$ [S1, unofficial].
        *   Update extents:
            *   $nodeExtentMin += (vec3(childOffset) \times nodeSize)$
            *   $nodeExtentMax -= ((vec3(1.0f) - vec3(childOffset)) \times nodeSize)$ [S1, unofficial].
        *   Fetch next node: $octTreeNode = texelFetch(OctTreeSampler, octTreeNodePtr, 0)$ [S1, unofficial].
        *   Update sizes: $nodeSize *= k\_ChildNodeSizeScalar$, $voxelHalfSize *= k\_ChildNodeSizeScalar$ [S1, unofficial].

5.  **Ray Marching within Brick:**
    *   Sample the brick texture at the current ray position.
    *   If interpolation is active, blend values from adjacent bricks or border regions based on $interpolationFactor$ [S1, unofficial].
    *   Accumulate color and opacity using volume rendering integral approximation (e.g., standard front-to-back compositing) [G].

6.  **Early Termination:**
    *   Check if accumulated opacity $> 0.995$ [S1, unofficial].
    *   *Pitfall:* Termination only triggers if the current step actually traversed a brick or constant value, preventing premature cutoff during initialization or empty space traversal [S1, unofficial].
    *   If terminated, return accumulated color [S1, unofficial].

7.  **Loop Continuation:**
    *   If not terminated and ray is within volume, continue to next node/brick.
    *   If ray exits volume or node is leaf/empty, handle backtracking or termination [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_pixel(ray_origin, ray_direction, vol_extent_min, vol_extent_max, root_node_ptr, 
                 brick_dim, brick_border, proj_voxel_size, parent_voxel_size, 
                 octree_sampler, brick_color_sampler, brick_grad_sampler):
    
    # Initialization
    oct_tree_node = root_node_ptr
    node_size = 1.0
    voxel_half_size = root_voxel_half_size
    extents_min = vol_extent_min
    extents_max = vol_extent_max
    cur_depth = 0
    traversal_count = 0
    node_list_traversal_count = 0
    pushed_node_count = 0
    
    accumulated_color = vec3(0.0)
    accumulated_opacity = 0.0
    transmittance = 1.0
    
    # Ray bounding box intersection (simplified)
    # Compute t_min, t_max for ray vs vol_extent
    
    while transmittance > 0.01: # Standard alpha threshold, adjusted by early termination logic
        traversal_count += 1
        node_list_traversal_count += 1
        
        # Node management for memory optimization
        if (start_pushing_nodes <= node_list_traversal_count < stop_pushing_nodes) and \
           (pushed_node_count < max_nodes_to_push_this_frame):
            pushed_node_count += 1
            add_to_node_usage_list(oct_tree_node)
            
        if node_list_traversal_count >= max_nodes_per_pixel_block:
            node_list_traversal_count = 0
            
        # Check if we are at a brick boundary or need interpolation
        # Note: Logic depends on specific implementation of rayBrickPosition
        # Assuming rayBrickPosition is maintained relative to current brick
        
        interpolation_factor = compute_interpolation_factor(proj_voxel_size, parent_voxel_size)
        
        # Pitfall: Disable interpolation at root
        if cur_depth == 0:
            interpolation_factor = 0.0
            
        if interpolation_factor > 0.001 and cur_depth != 0:
            # Traverse with interpolation
            traverse_brick_interp(oct_tree_node, ray_origin, ray_direction, 
                                  brick_dim, brick_border, brick_color_sampler, 
                                  brick_grad_sampler, accumulated_color, accumulated_opacity, 
                                  transmittance, interpolation_factor)
        else:
            # Standard brick traversal
            traverse_brick(oct_tree_node, ray_origin, ray_direction, 
                           brick_dim, brick_border, brick_color_sampler, 
                           brick_grad_sampler, accumulated_color, accumulated_opacity, 
                           transmittance)
                           
        # Early Termination Check
        # Pitfall: Only trigger if we actually traversed a brick/value
        if accumulated_opacity > 0.995:
            break
            
        # Descent logic would be handled inside traverse_brick functions
        # which update oct_tree_node, extents, node_size, etc.
        # If node is empty or ray exited, break or backtrack
        
        # Update parent voxel size for next level interpolation calc
        parent_voxel_size = proj_voxel_size
        proj_voxel_size *= 0.5 # Assuming uniform subdivision
        
    return accumulated_color, accumulated_opacity

def traverse_brick(node, origin, direction, brick_dim, brick_border, color_sampler, grad_sampler, 
                   acc_color, acc_opacity, transmittance):
    # Sample brick at current ray position
    # Update acc_color, acc_opacity, transmittance
    # Advance ray
    pass

def traverse_brick_interp(node, origin, direction, brick_dim, brick_border, color_sampler, grad_sampler,
                          acc_color, acc_opacity, transmittance, interp_factor):
    # Sample current brick and neighbor/bricks
    # Blend based on interp_factor
    # Update acc_color, acc_opacity, transmittance
    # Advance ray
    pass
```

## Subtopics

### [Two-staged Space-Skipping with Bricking and Octrees](5210-two-staged-space-skipping-with-bricking-and-octrees.md)
This method balances GPU bottlenecks by dividing space-skipping into two stages: coarse division using bricking to address bus and texture cache limitations, and fine division using octrees within bricks to address the rasterization bottleneck. It differs from Tong et al.'s bricking technique by storing unmodified scalar values in textures and using a fragment shader for post-interpolative RGB$\alpha$ lookup, avoiding time-consuming texture regeneration when the transfer function changes. Choose this when transfer functions change frequently and texture cache performance is critical.

### [Stackless GPU Raycasting with Octree Traversal](5222-stackless-gpu-raycasting-with-octree-traversal.md)
This method performs scalable, out-of-core volume rendering on the GPU by decomposing volumetric data into small cubical bricks organized in an octree structure. It differs from its parent by explicitly managing an out-of-core working set and using a 3D texture spatial index to enable stackless traversal without maintaining a node stack in shader memory. It differs from its sibling by integrating the spatial index construction and neighbor pointer encoding directly into the fragment shader's traversal loop, allowing for on-the-fly bounding box computation. The method solves the problem of rendering very large datasets that exceed GPU texture memory by asynchronously fetching a view-dependent working set of bricks. Choose this for datasets larger than GPU memory.

#### Sparse Octree Raytracing Acceleration
Accelerates rendering by using sparse octrees to store caches, enabling raytracing to skip empty space and terminate rays missing the object [S2, unofficial]. It converts density cache to a signed distance function (SDF) by thresholding at 0.0, optionally downsamples the SDF, generates a collision mesh using marching cubes, and traces rays against the BVH of the mesh to find occupied voxels and skip empty space [S2, unofficial]. For rays hitting occupied voxels, it integrates volume using cached components [S2, unofficial]. It uses BVH/ray-triangle intersection instead of hierarchical DDA to leverage hardware acceleration, allow easy mesh deformation/downsampling, and enable compositing of fake shadows/bouncelight [S2, unofficial]. Cache resolution for NeRF Synthetic 360 is $1024^3$ (except ship: $768^3$) with 8 components; for LLFF it is $768^3$ with 6 components [S2, unofficial].

#### Multi-resolution Ray-casting Rendering
Volume rendering via ray-casting that traverses the octree hierarchy to skip empty space and march rays near the surface [S3]. It uses multi-resolution interpolation to handle variable resolution blocks at boundaries [S3]. Steps include: traversing hierarchy to skip empty space and advance ray close to surface, marching ray in steps proportional to distance from zero-crossing in proximity of surface, sampling SDF field via tri-linear interpolation at each step, and computing accurate 3D surface position when ray transitions from positive to negative space [S3].

#### Sparse Voxel Octree Ray Traversal with Early Termination
Uses octree to avoid querying empty/occluded voxels [S4]. Stops ray when accumulated transmittance is below threshold $\tau_t$ (set to 0.01) [S4]. If alpha value of interval after decoding density is below $\tau_t$, it does not evaluate color decoder [S4]. Storage includes indices and values for features of occupied voxels, binary occupancy mask, and weights for density and color decoders, with parameters stored in half precision [S4].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of visible voxels, but significantly reduced by space-skipping. Brick interpolation reduces the number of texture fetches compared to voxel-by-voxel traversal [S1, unofficial].
*   **Memory Complexity:** Depends on the number of bricks loaded into GPU memory. The node usage list and brick pool manage memory efficiently, but large scenes may require out-of-core techniques (see [Stackless GPU Raycasting with Octree Traversal](5222-stackless-gpu-raycasting-with-octree-traversal.md)) [S1, S2, unofficial].
*   **Measured Performance:** GigaVoxels/GigaVoxelsBrickPool implementation shows efficient memory bandwidth usage due to brick-based texture access [S1, unofficial]. FastNeRF using sparse octrees achieves significant speedups over non-cached methods by reducing grid look-ups, which are expensive on GPU RAM [S2, unofficial].
*   **Strengths:** Efficient memory access patterns due to brick coherency; reduced texture cache misses; smooth interpolation at boundaries; early termination improves performance for opaque objects.
*   **Limitations:** Requires pre-computation of brick structures; interpolation adds complexity; out-of-core management is non-trivial; early termination can cause artifacts if not carefully implemented (e.g., premature cutoff) [S1, unofficial].
*   **When to use:** Real-time volume rendering of large, sparse datasets where memory bandwidth is a bottleneck.
*   **When not to use:** Dense volumes where space-skipping provides little benefit; scenes requiring frequent transfer function updates where pre-computed brick textures would need regeneration (though this method mitigates that with fragment shader lookup) [S1, unofficial].

## Background
The method originates from the GigaVoxels project, which introduced bricking techniques to optimize volume rendering on GPUs [S1, unofficial]. The specific combination of octree traversal, brick interpolation, and early termination is detailed in the VoxViz thesis by Apecoraro [S1, unofficial]. It builds on earlier SVO traversal methods [S1, unofficial]. Variants like FastNeRF apply similar sparse octree concepts to neural radiance fields, using BVH acceleration for mesh-based skipping [S2, unofficial]. Multi-resolution rendering approaches use octrees for adaptive resolution [S3]. NeRFLight uses sparse voxel octrees with early termination for neural rendering [S4].

## Sources
[S1] apecoraro/VoxVizThesis, https://github.com/apecoraro/VoxVizThesis, fetched 2026-10-27 (unofficial)
[S2] FastNeRF by Garbin et al., https://arxiv.org/abs/2103.10380, fetched 2026-10-26 (unofficial)
[S3] Adaptive-resolution octree-based volumetric SLAM, https://www.doc.ic.ac.uk/~sleutene/publications/Vespa_3DV19.pdf, fetched 2026-10-29
[S4] NeRFLight: Fast and Light Neural Radiance Fields using a Shared Feature Grid, https://openaccess.thecvf.com/content/CVPR2023/html/Rivas-Manzaneque_NeRFLight_Fast_and_Light_Neural_Radiance_Fields_Using_a_Shared_CVPR_2023_paper.html, fetched 2026-10-27
