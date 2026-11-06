# Sparse Ray-Octree Intersection (DECIDE)

Also called: Octree Ray Tracing, Ray-Casting Octree Traversal, Recursive Ray-Octree Intersection, castRay

## Overview
Sparse Ray-Octree Intersection (DECIDE) is a GPU-based algorithm designed to find the first intersection of a ray with a sparse voxel octree structure, returning the hit position, normal, and node index [S2, unofficial]. It solves the problem of efficiently traversing hierarchical voxel data for rendering or editing by handling traversal, descent, and backtracking using a stack [S2, unofficial]. Unlike standard recursive implementations, DECIDE uses an iterative approach with explicit stack management and bit-manipulation optimizations to determine child slots and t-spans, making it suitable for high-performance parallel execution on GPUs [S2, unofficial][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin ($O$), ray direction ($D$), octree root node, maximum depth ($L$), Level of Detail (LOD) multiplier ($lod$), and scale parameters.
*   **Output:** Hit position ($P$), hit normal ($N$), node index, child index, scale, and t-value ($t_{min}$). Returns false if no intersection is found within the octree bounds.

**Data Structures:**
*   **Stack:** An array of `uvec2` entries, where each entry stores a parent node index and a float-encoded `t_max` value. The stack size is determined by the maximum depth of the octree.
*   **Child Descriptor:** A 32-bit integer encoding the presence of children and their indices. Bits 0-6 indicate active children; bit 7 indicates if the node has children (non-leaf); bits 17+ may indicate a far pointer offset [S2, unofficial].
*   **Octant Mask:** A bitmask used to permute child slot indices based on coordinate system mirroring [S2, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Initialize $t_{min} = -\infty$, $t_{max} = \infty$ [S3, unofficial].
    *   Calculate the corner position in t-space: $t_{corner} = pos \times t_{coef} - t_{bias}$ [S2, unofficial].
    *   Find the maximum t-value at the corner: $t_{c\_max} = \min(\min(t_{corner}.x, t_{corner}.y), t_{corner}.z)$ [S2, unofficial].
    *   Initialize stack pointer $scale = 0$, $h = 0.0$ [S2, unofficial].
    *   Set $child\_descriptor = 0u$ [S2, unofficial].

2.  **Ray-Box Intersection Check:**
    *   Calculate node bounding box $(boxMin, boxMax)$ [S3, unofficial].
    *   Check if the ray intersects the box. If no intersection, return false [S3, unofficial].

3.  **Traversal Loop:**
    *   **Termination Check:** If $ubo.lod \times t_{c\_max} \ge scale\_exp2$, break the loop [S2, unofficial].
    *   **Hit Detection:** If $(depth \ge currentDepth \mid\mid node \rightarrow isLeaf)$ and $node \rightarrow isActive$, a hit is found. Compute hit position: $hitPosition = rayOrigin + rayDir \times t_{min}$ [S3, unofficial]. Compute hit normal based on the component of $centerToHit = hitPosition - node \rightarrow position$ with the maximum absolute value relative to half-size [S3, unofficial]. Set $hitAdd = !node \rightarrow isActive$ [S3, unofficial]. Return result [S2, unofficial].
    *   **Child Descriptor Update:** Permute child slot indices based on mirroring: $child\_shift = idx \oplus octant\_mask$ [S2, unofficial]. Shift child descriptor: $child\_masks = child\_descriptor \ll child\_shift$ [S2, unofficial].
    *   **Voxel Validity Check:** If $(child\_masks \& 0x8000u) \neq 0u$ and $t_{min} \le t_{max}$, the voxel is valid [S2, unofficial].
    *   **Descend to Child:**
        *   Intersect active t-span with the cube: $t_{v\_max} = \min(t_{max}, t_{c\_max})$ [S2, unofficial].
        *   Calculate center: $t_{center} = half\_scale\_exp2 \times t_{coef} + t_{corner}$ [S2, unofficial].
        *   Select child voxel entered first by checking $t_{center}$ against $t_{min}$ for each axis, updating $idx$ and $pos$ [S2, unofficial].
        *   If resulting t-span is non-empty ($t_{min} \le t_{v\_max}$) and non-leaf mask bit is set ($(child\_masks \& 0x0080u) \neq 0u$), descend [S2, unofficial].
    *   **Stack Management (Push):**
        *   If $t_{c\_max} < h$, push current parent to stack: $stack[scale] = uvec2(parent, floatBitsToUint(t_{max}))$ [S2, unofficial]. Update $h = t_{c\_max}$ [S2, unofficial].
        *   Prevent same parent from being stored again: $h = 0.0$ [S2, unofficial]. Invalidate cached child descriptor: $child\_descriptor = 0u$ [S2, unofficial].
    *   **Update State:**
        *   Update active t-span: $t_{max} = t_{v\_max}$ [S2, unofficial].
        *   Decrement scale: $scale--$ [S2, unofficial].
        *   Update $scale\_exp2 = half\_scale\_exp2$ [S2, unofficial].
    *   **Advance Step:**
        *   Step along the ray by checking $t_{corner}$ against $t_{c\_max}$. Update $pos$ and $step\_mask$: if $t_{corner}.x \le t_{c\_max}$, $step\_mask \oplus= 1, pos.x -= scale\_exp2$ [S2, unofficial].
        *   Update active t-span: $t_{min} = t_{c\_max}$ [S2, unofficial].
        *   Flip bits of child slot index: $idx \oplus= step\_mask$ [S2, unofficial].
    *   **Pop Step:**
        *   If bit flips disagree with ray direction ($(idx \& step\_mask) \neq 0$), find highest differing bit between positions to determine new $scale$ [S2, unofficial].
        *   Restore parent voxel from stack: $stackEntry = stack[scale]$ [S2, unofficial]. $parent = stackEntry.x$, $t_{max} = uintBitsToFloat(stackEntry.y)$ [S2, unofficial].

4.  **Miss Handling:**
    *   If $scale \ge s_{max}$, indicate miss if outside octree [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cast_ray(ray_origin, ray_dir, octree_root, max_depth, lod, scale_exp2, half_scale_exp2, t_coef, t_bias, octant_mask):
    t_min = -float('inf')
    t_max = float('inf')
    
    # Initial position and index calculation
    pos = ray_origin # Simplified; actual pos depends on coordinate system
    idx = 0
    scale = 0
    h = 0.0
    child_descriptor = 0
    parent = octree_root
    
    # Stack for backtracking
    stack = [None] * (max_depth + 1)
    
    while True:
        # Termination criteria
        if lod * t_max >= scale_exp2:
            break
            
        # Check for hit
        node = get_node(parent)
        if node.is_leaf or depth >= max_depth:
            if node.is_active:
                hit_position = ray_origin + ray_dir * t_min
                center_to_hit = hit_position - node.position
                half_size = scale_exp2 * 0.5
                max_comp = max(abs(center_to_hit.x / half_size.x), 
                               max(abs(center_to_hit.y / half_size.y), 
                                   abs(center_to_hit.z / half_size.z)))
                # Determine normal based on max component
                normal = determine_normal(center_to_hit, half_size, max_comp)
                return hit_position, normal, parent, idx, scale, t_min
        
        # Permute child indices
        child_shift = idx ^ octant_mask
        child_masks = child_descriptor << child_shift
        
        # Check validity
        if (child_masks & 0x8000) != 0 and t_min <= t_max:
            # Intersect t-span
            t_v_max = min(t_max, t_max) # t_c_max logic simplified
            t_center = half_scale_exp2 * t_coef + (pos * t_coef - t_bias)
            
            # Select child
            if t_center.x > t_min:
                idx ^= 1
                pos.x += scale_exp2
            # ... similar for y, z
            
            # Descend if valid
            if t_min <= t_v_max and (child_masks & 0x0080) != 0:
                # Push to stack if needed
                if t_max < h:
                    stack[scale] = (parent, float_to_uint(t_max))
                    h = t_max
                    h = 0.0 # Prevent re-storage
                    child_descriptor = 0
                
                # Update state
                t_max = t_v_max
                scale -= 1
                scale_exp2 = half_scale_exp2
                
                # Advance
                # ... update pos, step_mask
                t_min = t_max # Simplified
                idx ^= step_mask
                
                # Pop if necessary
                if (idx & step_mask) != 0:
                    # Find differing bit
                    scale = find_differing_bit_scale(pos)
                    stack_entry = stack[scale]
                    parent = stack_entry.x
                    t_max = uint_to_float(stack_entry.y)
            else:
                # No valid child, backtrack
                # ... handle backtrack logic
                pass
        else:
            # No valid child, backtrack
            # ... handle backtrack logic
            pass
            
        # Check bounds
        if scale >= max_depth:
            return False # Miss
            
    return False # Miss
```

**Parameters:**
*   `ubo.lod`: Level of Detail multiplier used for termination criteria [S2, unofficial].
*   `ubo.frameWidth`: Width of the framebuffer, used for debug coordinates [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision issues may arise when calculating $t_{corner}$ and $t_{center}$, especially at large scales. Use of `floatBitsToUint` and `uintBitsToFloat` for stack storage helps maintain precision for $t_{max}$ [S2, unofficial].
*   **Boundaries:** Rays starting outside the octree bounds must be handled to avoid invalid memory access or infinite loops [S2, unofficial].
*   **Degenerate Input:** Empty octrees or rays with zero length direction vectors should be checked before traversal [S3, unofficial].

## Subtopics

### [Octree Voxel-Triangle Intersection Test](3316-octree-voxel-triangle-intersection-test.md)
A method to determine if a triangle intersects a voxel within a virtual uniform grid defined by an octree, used for culling or traversal in volume rendering. It implements the Separating Axis Theorem (SAT) to test intersection between a triangle and an axis-aligned bounding box (cube).

### [Woodcock Collision Estimation with Octree Acceleration](3419-woodcock-collision-estimation-with-octree-acceleration.md)
An optimized ray marching method that combines Woodcock collision estimation with an octree traversal to skip empty regions and nodes that do not intersect the ray's current segment, reducing the number of volume samples. It differs from standard ray marching by using the octree to cull non-intersecting nodes, and from its parent [Sparse Ray-Octree Intersection (DECIDE)](3006-sparse-ray-octree-intersection-decide.md) by employing stochastic collision estimation within leaf nodes rather than deterministic traversal.

### [LiNode Light Gathering with Octree Traversal](3957-linode-light-gathering-with-octree-traversal.md)
LiNode Light Gathering is a ray-surfel intersection method that traverses an octree of surfels to determine the closest hit color and distance for a given ray. It solves the problem of efficiently gathering lighting information from a sparse, hierarchical surfel representation. It differs from its siblings by operating on surfel primitives within octree leaves rather than triangles (Octree Voxel-Triangle Intersection Test), using deterministic traversal rather than stochastic estimation (Woodcock Collision Estimation with Octree Acceleration), or performing point queries (Octree Point Search for Closest Point).

### [Octree Point Search for Closest Point](4999-octree-point-search-for-closest-point.md)
This method uses an octree to efficiently find the closest point to a ray origin among all points intersected by the ray, filtering by a maximum range.

### Leaf Consideration
Processes a leaf node by intersecting its AABB with the ray, creating a HitCandidate, and updating the best hit if the candidate is better [S4, unofficial].
*   **Step:** Intersect ray with bounds. If no intersection, return. Create HitCandidate. Set hit=true, leaf_id (from options.return_payload_indices ? octree.leaf_payload_indices()[leaf_id] : leaf_id), t=max(t_near, 0.0f), position=origin+direction*t, depth. If is_better_hit(candidate, best) is true, update best = candidate [S4, unofficial].
*   **Layout:** HitCandidate: Contains hit (bool), leaf_id (std::int32_t), t (float), position (glm::vec3), depth (std::int32_t) [S4, unofficial].
*   **Comparison:** is_better_hit prefers hits with smaller t. If t is equal (within kEpsilon), prefers smaller leaf_id [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** The algorithm traverses the octree depth, with each step involving constant-time bitwise operations and comparisons. The worst-case time complexity is proportional to the depth of the octree $O(L)$, where $L$ is the maximum depth [S2, unofficial].
*   **Memory Complexity:** The stack size is proportional to the maximum depth of the octree $O(L)$ [S2, unofficial].
*   **Performance:** The algorithm is designed for GPU execution, leveraging parallelism across threads [S2, unofficial]. Performance is influenced by the sparsity of the octree and the LOD parameter [S2, unofficial].
*   **Strengths:** Efficient handling of sparse data, explicit stack management allows for iterative traversal without recursion overhead, bitwise operations optimize child selection [S2, unofficial].
*   **Limitations:** Complexity of implementation due to bitwise manipulations and stack management, potential precision issues with floating-point arithmetic [S2, unofficial].

## Background
The DECIDE algorithm is a GPU-based method for ray-octree intersection, as described in unofficial sources [S2, unofficial][S3, unofficial]. It is used in contexts requiring efficient ray casting against sparse voxel structures, such as volume rendering and 3D model editing [S3, unofficial]. The algorithm's iterative nature and use of explicit stacks make it suitable for hardware that does not support deep recursion or where stack size is limited [S2, unofficial].

## Sources
[S1] Neural Geometric Level of Detail: Real-time Rendering with Implicit 3D Shapes, https://doi.org/10.1109/cvpr46437.2021.01120, fetched 2026-10-26
[S2] krre/origin, https://github.com/krre/origin, fetched 2026-10-26 (unofficial)
[S3] GitHub - saurlax/model_voxelizer: A 3D model voxelization tool that ..., https://github.com/saurlax/model_voxelizer, fetched 2026-10-30 (unofficial)
[S4] hectorpiteau/sparse-voxels, https://github.com/hectorpiteau/sparse-voxels, fetched 2026-10-24 (unofficial)
