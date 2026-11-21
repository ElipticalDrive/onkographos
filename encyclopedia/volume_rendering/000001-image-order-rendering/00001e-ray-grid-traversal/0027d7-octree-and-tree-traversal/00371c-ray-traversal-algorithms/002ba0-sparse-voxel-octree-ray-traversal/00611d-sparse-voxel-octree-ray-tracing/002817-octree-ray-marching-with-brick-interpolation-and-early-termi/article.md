# Octree Ray Marching with Brick Interpolation and Early Termination

Also called: Brick-based SVO Ray Marching, GigaVoxels-style Ray Traversal

## Overview
This method performs volume rendering on a Sparse Voxel Octree (SVO) by traversing the tree structure and sampling data stored in "bricks" (small, fixed-size blocks of voxels) rather than individual leaf voxels. It solves the problem of inefficient memory access and cache thrashing in large-volume rendering by grouping data into bricks, which improves texture cache utilization and reduces traversal overhead compared to single-voxel stepping [S1, unofficial]. It sets itself apart from sibling methods like [SVO Ray Marching with Stack-Based Navigation](2454-svo-ray-marching-with-stack-based-navigation.md) by explicitly handling interpolation between bricks of different resolutions and using early termination based on accumulated opacity to skip empty space, whereas [Octree Voxel Raycasting for Hover (Closest Active Voxel)](3219-octree-voxel-raycasting-for-hover-closest-active-voxel.md) focuses on finding the nearest surface rather than full volume integration.

## How it works
**Inputs:**
- Ray origin $O$ and direction $D$ in world coordinates.
- SVO root node pointer.
- Volume bounding box extents $V_{min}$ and $V_{max}$.
- Current camera projection parameters (to compute projected voxel size).
- Brick dimensions ($B_x, B_y, B_z$) and border sizes.
- Maximum nodes to push per frame ($MaxNodesToPushThisFrame$) and reset threshold ($maxNodesPerPixelBlock$).

**Outputs:**
- Final color $C$ and opacity $\alpha$ for the pixel.
- Updated node usage statistics for asynchronous brick loading.

**Data Structures:**
- **Octree Node:** Contains a pointer to the next node in the hierarchy, a pointer to the brick data (if leaf), and metadata (depth, size).
- **Brick Data:** A 3D texture or buffer containing color and optionally gradient values for a block of voxels.
- **Node Usage List:** A queue of nodes recently traversed, used to trigger asynchronous GPU upload of their brick data.

**Parameters:**
- $projVoxelSize$: Projected voxel size in world units.
- $parentVoxelSize$: Voxel size at the parent octree level.
- $k_{ChildNodeSizeScalar}$: Scalar multiplier for node size on descent (typically 0.5 for octrees).
- $MaxNodesToPushThisFrame$: Limit on asynchronous work per frame.
- $maxNodesPerPixelBlock$: Threshold for resetting the per-block traversal counter.

**Algorithm Steps:**

1.  **Initialization:**
    - Set current node $N$ to root.
    - Set current extents $E_{min} = V_{min}$, $E_{max} = V_{max}$.
    - Set current node size $S = 1.0$ (normalized) or root voxel size.
    - Set voxel half-size $h = RootVoxelHalfSize$.
    - Reset traversal counters: $traversalCount = 0$, $nodeListTraversalCount = 0$.
    - Initialize accumulated color $C = (0,0,0,0)$ and transmittance $T = 1.0$.

2.  **Ray-Volume Intersection Test:**
    - Compute containment:
      $$ test1 = \text{sign}(O - V_{min}) $$
      $$ test2 = \text{sign}(V_{max} - O) $$
      $$ inside = \text{dot}(test1, test2) $$
    - If $inside < 3$, the ray origin is outside the volume bounding box. Handle entry point calculation or terminate if the ray does not intersect the bounding box. [S1, unofficial]

3.  **Traversal Loop:**
    While $T > \epsilon$ and $N$ is valid:
    
    a. **Increment Counters:**
       $$ traversalCount \leftarrow traversalCount + 1 $$
       $$ nodeListTraversalCount \leftarrow nodeListTraversalCount + 1 $$
       If $nodeListTraversalCount \ge maxNodesPerPixelBlock$, reset $nodeListTraversalCount = 0$. [S1, unofficial]

    b. **Node Management (Asynchronous Loading):**
       If $nodeListTraversalCount$ is within $[startPushingNodes, stopPushingNodes)$ and $pushedNodeCount < MaxNodesToPushThisFrame$:
       - Increment $pushedNodeCount$.
       - Call $AddToNodeUsageList(N)$. This schedules the brick data associated with $N$ for asynchronous GPU upload if not already present. [S1, unofficial]

    c. **Interpolation Decision:**
       Check if interpolation is enabled:
       - Condition: $projVoxelSize > parentVoxelSize$ (or similar scale check), $curDepth \neq 0$, and interpolation factor $> 0.001$.
       - If true, compute previous brick ray position and call **TraverseBrickInterp**.
       - Otherwise, call **TraverseBrick**.
       - *Note:* Interpolation is disabled at the root node ($curDepth == 0$) to prevent invalid ratios when $parentVoxelSize$ equals $projVoxelSize$. [S1, unofficial]

    d. **Descent (if moving to child):**
       - Compute brick offset: $brickOffset = rayBrickPosition \times k_{TreeN}$ (where $k_{TreeN}$ is the number of children, typically 8).
       - Clamp to child index: $childOffset = \text{clamp}(\text{uvec3}(brickOffset), 0, 1)$.
       - Update ray position within brick: $rayBrickPosition = brickOffset - vec3(childOffset)$.
       - Update extents:
         $$ E_{min} += (vec3(childOffset) \times S) $$
         $$ E_{max} -= ((vec3(1.0) - vec3(childOffset)) \times S) $$
       - Fetch next node: $N = \text{texelFetch}(OctTreeSampler, N.ptr, 0)$. [S1, unofficial]

    e. **Brick Traversal (TraverseBrick):**
       - Sample the brick data at the current ray position.
       - Compute density/color $\rho, c$.
       - Update transmittance: $T \leftarrow T \times (1 - \rho \cdot \Delta s)$.
       - Update color: $C \leftarrow C + T \cdot c \cdot \rho \cdot \Delta s$.
       - Advance ray by step size $\Delta s$.

    f. **Brick Interpolation (TraverseBrickInterp):**
       - Sample surrounding bricks or interpolate within the brick based on the interpolation factor derived from $projVoxelSize$ and $parentVoxelSize$.
       - Apply the same color/opacity accumulation as above.
       - *Pitfall:* Interpolation factor is clamped to $[0.0, 1.0]$ and thresholded at $0.001$ to avoid division by zero. [S1, unofficial]

    g. **Early Termination:**
       - If accumulated opacity $> 0.995$ AND the current step actually traversed a brick or constant value (not empty space):
         - Terminate loop.
       - This prevents premature cutoff during initialization or in empty space. [S1, unofficial]

4.  **Reset State:**
    - After ray completion, reset $N$ to root, extents, and sizes for the next ray. [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_ray(ray_origin, ray_dir, root_node, volume_extents_min, volume_extents_max, 
               proj_voxel_size, parent_voxel_size, max_nodes_per_block, max_nodes_per_frame):
    # Initialization
    current_node = root_node
    node_size = 1.0  # Normalized or root voxel size
    voxel_half_size = root_voxel_half_size
    extents_min = volume_extents_min
    extents_max = volume_extents_max
    accumulated_color = vec4(0.0)
    transmittance = 1.0
    traversal_count = 0
    node_list_traversal_count = 0
    pushed_node_count = 0
    cur_depth = 0
    
    # Ray-Volume Intersection Check
    test1 = sign(ray_origin - volume_extents_min)
    test2 = sign(volume_extents_max - ray_origin)
    inside = dot(test1, test2)
    if inside < 3.0:
        # Handle ray outside volume or compute entry point
        pass 

    while transmittance > 0.001 and current_node is not None:
        traversal_count += 1
        node_list_traversal_count += 1
        
        # Reset counter if threshold reached
        if node_list_traversal_count >= max_nodes_per_block:
            node_list_traversal_count = 0
            
        # Asynchronous Node Loading
        if (start_pushing_nodes <= node_list_traversal_count < stop_pushing_nodes) and \
           (pushed_node_count < max_nodes_per_frame):
            pushed_node_count += 1
            add_to_node_usage_list(current_node)
            
        # Determine Interpolation Mode
        interp_factor = compute_interpolation_factor(proj_voxel_size, parent_voxel_size)
        enable_interp = (proj_voxel_size > parent_voxel_size) and (cur_depth != 0) and (interp_factor > 0.001)
        
        if enable_interp:
            prev_brick_ray_pos = compute_prev_brick_ray_pos()
            color, alpha, new_transmittance = traverse_brick_interp(current_node, ray_origin, ray_dir, 
                                                                    prev_brick_ray_pos, interp_factor, 
                                                                    node_size, extents_min, extents_max)
        else:
            color, alpha, new_transmittance = traverse_brick(current_node, ray_origin, ray_dir, 
                                                             node_size, extents_min, extents_max)
            
        # Update Accumulators
        accumulated_color += transmittance * color
        transmittance = new_transmittance
        
        # Early Termination Check
        if transmittance < 0.005 and alpha > 0.0: # Opacity > 0.995 implies transmittance < 0.005
            # Ensure we actually traversed something
            if alpha > 0.0: 
                break
                
        # Descent Logic (Simplified for single step)
        # In a real implementation, this happens inside traverse_brick/interp or as a separate step
        # if moving to a child node:
        # brick_offset = ray_brick_position * 8
        # child_offset = clamp(uvec3(brick_offset), 0, 1)
        # ray_brick_position = brick_offset - vec3(child_offset)
        # extents_min += vec3(child_offset) * node_size
        # extents_max -= (vec3(1.0) - vec3(child_offset)) * node_size
        # current_node = fetch_next_node(current_node, child_offset)
        # node_size *= 0.5
        # cur_depth += 1
        
    return accumulated_color
```

## Subtopics

### Sparse Octree Raytracing Acceleration
This variant accelerates rendering by using sparse octrees to store caches, enabling raytracing to skip empty space and terminate rays missing the object [S2, unofficial]. It converts the density cache to a signed distance function (SDF) by thresholding at 0.0, optionally downsamples the SDF, and generates a collision mesh using marching cubes [S2, unofficial]. Rays are traced against the BVH of this mesh to find occupied voxels, skipping empty space efficiently. For rays hitting occupied voxels, volume integration uses cached components [S2, unofficial]. It uses BVH/ray-triangle intersection instead of hierarchical DDA to leverage hardware acceleration, allow easy mesh deformation/downsampling, and enable compositing of fake shadows/bouncelight [S2, unofficial]. Cache resolutions are typically $1024^3$ (8 components) for NeRF Synthetic 360 scenes (except ship: $768^3$) and $768^3$ (6 components) for LLFF [S2, unofficial].

### Multi-resolution Ray-casting Rendering
This method performs volume rendering via ray-casting that traverses the octree hierarchy to skip empty space and march rays near the surface [S3]. It uses multi-resolution interpolation to handle variable resolution blocks at boundaries, avoiding complex synchronization issues [S3]. The hierarchy is traversed to skip empty space and advance the ray as close as possible to the surface. In the proximity of the surface, the ray is marched in steps proportional to the distance from the zero-crossing. At each step, the SDF field is sampled via tri-linear interpolation. The accurate 3D position of the surface is computed when the ray transitions from positive to negative space [S3].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of visible voxels/bricks along the ray. Early termination reduces the effective $N$ significantly in sparse volumes.
- **Memory Complexity:** $O(V)$ where $V$ is the total volume of the SVO, but only active bricks are loaded into GPU memory. The node usage list adds $O(M)$ overhead where $M$ is the number of nodes traversed per frame.
- **Performance:** Improves cache coherency by accessing brick-sized blocks rather than scattered leaf voxels. Asynchronous loading hides memory latency. [S1, unofficial]
- **Strengths:** Efficient handling of large, sparse volumes; reduced memory bandwidth usage; supports dynamic transfer functions without texture regeneration (unlike some bricking techniques). [S1, unofficial]
- **Limitations:** Interpolation artifacts at brick boundaries if not handled carefully; complexity in managing asynchronous loading and node usage lists; early termination may cause visual artifacts if not tuned properly. [S1, unofficial]
- **When to use:** Large-scale volume rendering where memory bandwidth is a bottleneck and the volume is sparse.
- **When not to use:** Dense volumes where brick overhead outweighs benefits, or when exact surface reconstruction is required without post-processing.

## Background
The method originates from the GigaVoxels project, which introduced bricking as a technique to optimize volume rendering on GPUs by grouping voxels into bricks to improve cache performance [S1, unofficial]. The specific combination of octree traversal, brick interpolation, and early termination is detailed in the VoxViz thesis, which describes the implementation of GigaVoxels in a real-time rendering context [S1, unofficial]. The technique addresses the limitations of traditional ray marching on dense textures and the overhead of pure octree traversal on leaf nodes.

## Sources
[S1] apecoraro/VoxVizThesis, https://github.com/apecoraro/VoxVizThesis, fetched 2026-10-27 (unofficial)
[S2] FastNeRF by Garbin et al., https://arxiv.org/abs/2103.10380, fetched 2026-10-26 (unofficial)
[S3] Adaptive-resolution octree-based volumetric SLAM, https://www.doc.ic.ac.uk/~sleutene/publications/Vespa_3DV19.pdf, fetched 2026-10-29
