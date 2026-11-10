# Per-Frame Multi-Kernel Isosurface Ray-Tracing Pipeline

Also called: MK pipeline, Multi-Kernel Isosurface Ray-casting Pipeline, multi-kernel rendering pipeline

## Overview
This method is a GPU-based image-order rendering pipeline for isosurface extraction that divides the ray-casting workload into four distinct, lightweight kernels: ray creation, BVH traversal, leaf intersection, and shading [S1, unofficial]. It solves the problem of thread divergence and inefficient memory access patterns in single-kernel ray tracers by using in-kernel stream compaction to filter out non-intersecting rays between stages [S1, unofficial]. Unlike sibling methods that may use static acceleration structures or single-pass execution, this pipeline dynamically rebuilds an explicit BVH for every frame based on the current isovalue, optimizing traversal for the specific geometry present [S1, unofficial]. It builds on [Isosurface and first-hit ray casting](20-isosurface-and-first-hit-ray-casting.md).

## How it works
The pipeline operates on a per-frame basis, assuming the isosurface may change every frame. It relies on a balanced, median-split BVH where leaves are sized to 163 voxels, a value chosen to optimize texture caching on the target GPU hardware [S1, unofficial].

### Data Structures
*   **Min/Max Sub-volume:** A pre-processed volume containing the minimum and maximum scalar values for each leaf node (sub-volume). This is uploaded to GPU memory before rendering [S1, unofficial].
*   **Implicit BVH:** A balanced binary tree structure created once, where node positions are determined implicitly by array indices (e.g., children of node $x$ are at $2x$ and $2x+1$ for an x-split) [S1, unofficial].
*   **Explicit BVH:** A per-frame derived structure where nodes are flagged as valid or invalid based on the current isovalue. Invalid nodes are compacted out [S1, unofficial].
*   **Ray Stack:** A data structure passed between kernels to maintain traversal state (current node, $t_{near}$, $t_{far}$) for rays that have not yet hit a leaf [S1, unofficial].
*   **Work-lists:** Lists of ray indices. Compaction is used to remove rays that have finished (hit surface or left volume) to maintain thread coherency for the next kernel [S1, unofficial].

### Parameters
*   **Leaf Size:** 163 voxels [S1, unofficial].
*   **Tree Memory:** < 2.3 MB for a $512^3$ 8-bit volume [S1, unofficial].
*   **Hardware:** NVIDIA GeForce GTX 480 (used for testing) [S1, unofficial].
*   **Resolution:** $1024^2$ (used for testing) [S1, unofficial].

### Steps

1.  **Pre-processing (Offline/Initial):**
    *   Compute the min/max scalar value for every leaf in the volume.
    *   Create an implicit, balanced BVH structure large enough to hold all leaves.

2.  **Per-Frame Initialization:**
    *   **Ray Creation Kernel:** Generate one ray per pixel. Test against the global volume bounding box. Use stream compaction to discard rays that do not intersect the volume. Output a compacted list of valid ray indices [S1, unofficial].
    *   **BVH Update Kernel:** Convert the implicit BVH into an explicit BVH specific to the current isovalue $\tau$:
        *   For each leaf, compare $\tau$ against the pre-computed min/max values. Set a validity flag on the leaf node if $\min \le \tau \le \max$ [S1, unofficial].
        *   Propagate validity flags up the tree. For each parent node:
            *   If both children are invalid, mark parent invalid.
            *   If one child is valid, copy the valid child's details to the parent (tree compaction) [S1, unofficial].
            *   If both children are valid, mark parent valid and compute bounds encompassing both children.
        *   Determine split dimensions based on the largest volume dimension (e.g., x-split offsets $2x + [0, 1]$) [S1, unofficial].

3.  **Traversal (BVH Traversal Kernel):**
    *   Input: Compacted list of ray indices and their initial states.
    *   For each active ray:
        *   Load the current node.
        *   If internal node: Test ray against both child AABBs.
        *   Traverse the child with the closest $t_{near}$ first.
        *   Push the other child to the ray's stack.
        *   If leaf node: Push leaf to stack, stop traversal for this ray, and call `InK-Compact` to filter out rays that did not reach a leaf (or finished). Output the list of rays reaching leaves [S1, unofficial].

4.  **Intersection (Iso-Intersection Kernel):**
    *   Input: List of ray indices reaching leaves.
    *   For each ray:
        *   Pop the ray stack to retrieve state.
        *   Recompute $t_{near}$ and $t_{far}$ relative to the leaf's bounding box.
        *   Step along the ray incrementally within the leaf.
        *   Detect isovalue crossing ($f(p_1) \cdot f(p_2) < 0$ relative to $\tau$).
        *   Perform linear interpolation to find the precise intersection location [S1, unofficial].
        *   Compute the surface normal $\nabla f$ at the hit point.
        *   Set an output-to-shader predicate.
        *   If no intersection is found, the ray is discarded via compaction [S1, unofficial].

5.  **Shading (Shading Kernel):**
    *   Input: Rays that hit the surface.
    *   Compute color using local lighting models (ambient, diffuse, specular) based on the computed normal and position. Rays that did not hit the surface are excluded [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-processing (done once)
def pre_process_volume(volume_data, leaf_size=163):
    """
    Computes min/max for each leaf and builds implicit BVH structure.
    """
    num_leaves = ceil(volume_dims / leaf_size)
    leaf_mins = [0.0] * num_leaves
    leaf_maxs = [0.0] * num_leaves
    
    # Compute min/max for each leaf
    for i in range(num_leaves):
        leaf_mins[i], leaf_maxs[i] = compute_leaf_bounds(volume_data, i, leaf_size)
        
    # Implicit BVH is just an array of nodes, size determined by num_leaves
    # Structure is balanced binary tree
    return leaf_mins, leaf_maxs

def update_bvh_per_frame(leaf_mins, leaf_maxs, isovalue, num_leaves):
    """
    Updates BVH validity flags for the current isovalue.
    Returns explicit BVH node list with validity flags.
    """
    # Initialize leaf validity
    leaf_valid = [False] * num_leaves
    for i in range(num_leaves):
        if leaf_mins[i] <= isovalue <= leaf_maxs[i]:
            leaf_valid[i] = True
            
    # Propagate up to root
    # Assuming implicit indexing: parent of i is (i-1)//2
    # Process from last leaf up to root
    for i in range(num_leaves - 1, -1, -1):
        if i == 0: # Root
            break
        parent = (i - 1) // 2
        left_child = 2 * parent + 0
        right_child = 2 * parent + 1
        
        left_valid = leaf_valid[left_child] if left_child < num_leaves else False
        right_valid = leaf_valid[right_child] if right_child < num_leaves else False
        
        if left_valid and right_valid:
            leaf_valid[parent] = True
            # Compute bounds for parent if needed
        elif left_valid or right_valid:
            # Tree compaction: copy valid child to parent
            leaf_valid[parent] = True
            # In implementation, this might involve swapping data or flags
        else:
            leaf_valid[parent] = False
            
    return leaf_valid

def ray_creation_kernel(pixels, volume_bounds):
    """
    Generates rays and compacts those intersecting the volume.
    """
    rays = []
    for pixel in pixels:
        ray = generate_ray_from_pixel(pixel)
        if intersects_aabb(ray, volume_bounds):
            rays.append(ray)
    return rays # Compacted list

def bvh_traversal_kernel(rays, bvh_nodes, leaf_valid):
    """
    Traverses BVH, stops at leaves. Compacts rays that finished.
    """
    active_rays = []
    for ray in rays:
        stack = [0] # Start at root
        hit_leaf = False
        
        while stack:
            node_idx = stack.pop()
            
            if not leaf_valid[node_idx]:
                continue # Skip invalid nodes
                
            if is_leaf(node_idx):
                # Push to output, stop traversal for this ray
                ray.current_node = node_idx
                active_rays.append(ray)
                hit_leaf = True
                break
            else:
                left = 2 * node_idx + 0
                right = 2 * node_idx + 1
                
                # Test children
                t_near_left, t_far_left = intersect_aabb(ray, bvh_nodes[left].aabb)
                t_near_right, t_far_right = intersect_aabb(ray, bvh_nodes[right].aabb)
                
                # Determine traversal order
                if t_near_left < t_near_right:
                    # Traverse left first
                    if t_far_left > ray.t_far:
                        continue
                    stack.append(right) # Push right
                    node_idx = left
                else:
                    if t_far_right > ray.t_far:
                        continue
                    stack.append(left) # Push left
                    node_idx = right
                    
                # Update ray state for new node
                ray.t_near = min(t_near_left, t_near_right) # Simplified
                # In real implementation, update ray state properly
                
        if not hit_leaf:
            pass # Ray discarded by compaction
            
    return active_rays

def iso_intersection_kernel(rays, volume_data, isovalue, leaf_size):
    """
    Finds intersection within leaf, computes normal.
    """
    hit_rays = []
    for ray in rays:
        node_idx = ray.current_node
        # Recompute bounds for leaf
        leaf_bounds = get_leaf_bounds(node_idx)
        
        # Step along ray within leaf
        t = ray.t_near
        while t < ray.t_far:
            p = ray.origin + t * ray.direction
            val = sample_volume(volume_data, p)
            
            # Check crossing
            if isovalue_crossing(val, isovalue):
                # Linear interpolation
                t_hit = linear_interpolate(t, p, val, isovalue)
                hit_point = ray.origin + t_hit * ray.direction
                normal = compute_gradient(volume_data, hit_point)
                
                ray.hit_point = hit_point
                ray.normal = normal
                ray.t_hit = t_hit
                hit_rays.append(ray)
                break
            t += step_size
            
    return hit_rays

def shading_kernel(hit_rays):
    """
    Computes final color.
    """
    colors = []
    for ray in hit_rays:
        color = phong_shading(ray.hit_point, ray.normal, ray.direction)
        colors.append(color)
    return colors
```

## Subtopics

### Single-Kernel Ray-Tracing Pipeline
A variant that uses the same ray-tracing code but splits it into three kernels with additional work to initialize/from and output-to compacted work-lists [S1, unofficial]. It outperforms the multi-kernel pipeline only when isosurfaces are simple (little work) because the CPU-side overhead of the multi-kernel becomes a burden. Otherwise, the multi-kernel is faster [S1, unofficial].

## Operating characteristics
*   **Performance:** On an NVIDIA GTX 480 at $1024^2$ resolution, the multi-kernel pipeline outperforms the single-kernel pipeline by an average of 15% [S1, unofficial].
*   **Complexity Dependence:** The multi-kernel approach is significantly faster for complex isosurfaces. For simple isosurfaces (little to be rendered), the single-kernel pipeline can be faster due to lower overhead, though the multi-kernel remains competitive (>100 fps) [S1, unofficial].
*   **Memory:** The BVH tree memory usage is less than 2.3 MB for a $512^3$ 8-bit volume with 163-voxel leaves [S1, unofficial].
*   **Strengths:** High thread coherency due to compaction; optimized for GPU texture caching via leaf size; dynamic BVH update handles changing isosurfaces efficiently.
*   **Limitations:** CPU-side overhead for kernel launches and compaction can negate benefits for very simple scenes; assumes isosurface changes every frame (cost included in timing) [S1, unofficial].

## Background
The method was introduced by D. M. Hughes et al. in 2013 in the paper "In-Kernel Stream Compaction and Its Application to Isosurface Ray-Tracing" [S1, unofficial]. It was developed to address inefficiencies in single-kernel ray tracers on GPUs, specifically thread divergence and memory access patterns, by leveraging in-kernel stream compaction to maintain coherency between processing stages. The work was presented at Computer Graphics Forum, published by The Eurographics Association and John Wiley & Sons Ltd. [S1, unofficial].

## Sources
[S1] InKernel Stream Compaction and Its Application to ..., https://www.sci.utah.edu/~knolla/hughes13.pdf, fetched 2026-10-29 (unofficial)
