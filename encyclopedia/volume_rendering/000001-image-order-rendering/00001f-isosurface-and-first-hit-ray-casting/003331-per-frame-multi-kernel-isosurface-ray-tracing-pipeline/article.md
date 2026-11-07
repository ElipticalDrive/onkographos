# Per-Frame Multi-Kernel Isosurface Ray-Tracing Pipeline

Also called: MK pipeline, Multi-Kernel Isosurface Ray-casting Pipeline, multi-kernel rendering pipeline

## Overview
This method accelerates isosurface ray-tracing by dividing the rendering workload into four distinct GPU kernels—ray creation, BVH traversal, leaf intersection, and shading—to maximize thread coherency and concurrency [S1, unofficial]. It differs from single-kernel approaches by using in-kernel stream compaction to manage variable workloads between stages, making it particularly effective for complex isosurfaces where many rays may miss the volume or require extensive traversal [S1, unofficial].

## How it works
The pipeline operates on a per-frame basis, assuming the isosurface (defined by an isovalue $\tau$) may change every frame. It builds on [Isosurface and first-hit ray casting](20-isosurface-and-first-hit-ray-casting.md).

### Data Structures
*   **Min/Max Sub-volume:** A pre-computed structure storing the minimum and maximum scalar values for each leaf node.
*   **Implicit BVH:** A balanced Binary Volume Hierarchy (BVH) tree structure allocated in memory but not yet filtered by the current isovalue. It uses median-split nodes.
*   **Explicit BVH:** The implicit BVH updated with validity flags specific to the current frame's isovalue.
*   **Work-lists:** Lists of ray indices passed between kernels. These are compacted (holes removed) to ensure contiguous memory access and thread coherency.
*   **Ray State:** Stored on a stack within the intersection kernel to preserve traversal state (e.g., `tnear`, `tfar`, node pointers) across kernel boundaries.

### Parameters
*   **Leaf Size:** 163 voxels. This specific size was chosen empirically to optimize texture caching on the target hardware (NVIDIA 480 GTX) [S1, unofficial].
*   **Tree Memory:** For a $512^3$ 8-bit volume with 163-voxel leaves, the BVH tree memory usage is less than 2.3 MB [S1, unofficial].
*   **Resolution:** Tested at $1024^2$ pixels [S1, unofficial].

### Steps

1.  **Pre-processing (One-time or infrequent):**
    *   Partition the volume into leaf nodes of size 163 voxels.
    *   Compute the min/max scalar values for each leaf and store them in the Min/Max Sub-volume.
    *   Allocate memory for a balanced implicit BVH with enough nodes to cover the volume.

2.  **Per-Frame Initialization:**
    *   **BVH Update Kernel:** Update the implicit BVH to reflect the current isovalue $\tau$.
        *   For each leaf, compare $\tau$ against the leaf's min/max values.
        *   Set a validity flag on the leaf node: `true` if $\min \le \tau \le \max$, `false` otherwise.
        *   Propagate validity up the tree:
            *   Determine the split dimension (largest dimension of the bounding box).
            *   Compute child offsets (e.g., for x-split: $2x + [0, 1]$).
            *   If a parent has no valid children, mark it invalid.
            *   If a parent has one valid child, copy the child's details to the parent (tree compaction/condensation).
            *   If a parent has two valid children, mark it valid.
            *   Compute bounding boxes encompassing valid children.
    *   **Ray Creation Kernel:**
        *   Generate one ray per pixel.
        *   Test each ray against the global volume bounding box.
        *   Use in-kernel compaction to remove rays that do not intersect the volume.
        *   Output a compacted list of ray indices.

3.  **Rendering Loop (Traversal and Intersection):**
    *   **Traverse BVH Kernel:**
        *   Input: Compacted list of ray indices.
        *   For each ray, load the current BVH node.
        *   Test the ray against the AABBs of both children.
        *   Traverse the child with the closest intersection (`tnear`) first; push the other child to the traversal stack.
        *   If a leaf node is reached:
            *   Push the leaf to the stack.
            *   Stop traversal for this ray.
            *   Call `InK-Compact` to compact the list of rays that hit leaves.
        *   Output: Compacted list of ray indices pointing to leaf nodes.
    *   **Iso-Intersection Kernel:**
        *   Input: Compacted list of ray indices.
        *   For each ray:
            *   Access ray information and pop the ray stack to restore state.
            *   Recompute `tnear` and `tfar` for the leaf bounding box.
            *   Step along the ray incrementally within the leaf.
            *   Detect isovalue crossing (sign change or threshold crossing).
            *   Perform linear interpolation to find the precise intersection location.
            *   Compute the surface normal $\mathbf{n}$ using finite differences or tri-cubic filtering of the gradient $\nabla f$.
            *   Store the intersection point and normal.
            *   Set a predicate flag indicating a hit.
            *   If no hit is found, peek the top of the ray stack to determine if traversal should continue or if the ray is done.
        *   Output: Compacted list of rays that hit the isosurface.

4.  **Shading Kernel:**
    *   Input: Compacted list of hit rays.
    *   Compute lighting (ambient, diffuse, specular) using the stored normal and position.
    *   Write final color to the image buffer.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-processing
def setup_volume(volume_data, leaf_size=163):
    # Compute min/max for each leaf
    min_max_subvolume = compute_leaf_min_max(volume_data, leaf_size)
    # Create balanced implicit BVH structure
    implicit_bvh = create_balanced_bvh(volume_data.shape, leaf_size)
    return min_max_subvolume, implicit_bvh

# Per-frame update
def update_bvh(implicit_bvh, min_max_subvolume, isovalue):
    # Update leaves
    for leaf in implicit_bvh.leaves:
        leaf_min, leaf_max = min_max_subvolume[leaf.id]
        leaf.valid = leaf_min <= isovalue <= leaf_max
    
    # Propagate up
    for level in implicit_bvh.levels_from_bottom_up:
        for node in level:
            children = node.children
            valid_children = [c for c in children if c.valid]
            
            if len(valid_children) == 0:
                node.valid = False
            elif len(valid_children) == 1:
                # Tree compaction: copy child to parent
                node.copy_from(valid_children[0])
                node.valid = True
            else:
                node.valid = True
                node.bounds = union_bounds([c.bounds for c in valid_children])

# Kernel 1: Ray Creation
def ray_creation_kernel(pixels, volume_bounds):
    rays = []
    for pixel in pixels:
        ray = create_ray_from_pixel(pixel, volume_bounds)
        if intersects_volume(ray, volume_bounds):
            rays.append(ray)
    # InK-Compact: remove gaps in rays list
    return compact(rays)

# Kernel 2: BVH Traversal
def traverse_bvh_kernel(rays, bvh):
    work_list = []
    for ray in rays:
        node = bvh.root
        stack = []
        while node is not None:
            if node.is_leaf:
                stack.append(node)
                break
            # Test children
            left_hit, left_t = ray.intersect_aabb(node.left.bounds)
            right_hit, right_t = ray.intersect_aabb(node.right.bounds)
            
            if left_hit and right_hit:
                if left_t < right_t:
                    stack.append(node.right)
                    node = node.left
                else:
                    stack.append(node.left)
                    node = node.right
            elif left_hit:
                node = node.left
            elif right_hit:
                node = node.right
            else:
                break # No hit
        
        if stack:
            work_list.append((ray, stack))
    
    return compact(work_list)

# Kernel 3: Iso-Intersection
def iso_intersection_kernel(work_list, min_max_subvolume, isovalue):
    hits = []
    for ray, stack in work_list:
        # Pop stack to get leaf
        leaf = stack.pop()
        
        # Recompute bounds for leaf
        leaf_tnear, leaf_tfar = ray.intersect_aabb(leaf.bounds)
        
        # Step along ray
        hit_found = False
        for t in range(leaf_tnear, leaf_tfar, step_size):
            p = ray.origin + t * ray.direction
            val = sample_volume(p)
            
            if val >= isovalue: # Or sign change detection
                # Linear interpolation for precision
                p_prev = ray.origin + (t - step_size) * ray.direction
                val_prev = sample_volume(p_prev)
                alpha = (isovalue - val_prev) / (val - val_prev)
                intersection = p_prev + alpha * (p - p_prev)
                
                # Compute normal
                normal = compute_gradient(p)
                normal = normalize(normal)
                
                hits.append((ray, intersection, normal))
                hit_found = True
                break
        
        if not hit_found and stack:
            # Continue traversal if stack not empty
            # Note: In practice, this might require re-traversal or a loop
            # For simplicity, we assume leaf traversal is the main step
            pass
            
    return compact(hits)

# Kernel 4: Shading
def shading_kernel(hits):
    for ray, point, normal in hits:
        color = compute_phong_shading(point, normal, ray.direction)
        write_to_image(ray.pixel_id, color)
```

## Subtopics

### Single-Kernel Ray-Tracing Pipeline
A variant that uses the same ray-tracing code but splits it into only three kernels, with additional overhead to initialize and output compacted work-lists. It outperforms the multi-kernel pipeline only when isosurfaces are simple (little work) because the CPU-side overhead of the multi-kernel becomes a burden. Otherwise, the multi-kernel is faster [S1, unofficial].

## Operating characteristics
*   **Performance:** On a standard NVIDIA 480 GTX at $1024^2$ resolution, the multi-kernel pipeline outperforms the single-kernel pipeline by an average of 15% [S1, unofficial].
*   **Complexity Dependence:** The multi-kernel approach is significantly faster for complex isosurfaces (e.g., Bonsai, Aneurysm datasets with thin features and large gaps) where many rays miss the surface or require extensive traversal. For simple isosurfaces (e.g., Head, LLNL datasets), the single-kernel may be faster, but the multi-kernel remains competitive (>100 fps) [S1, unofficial].
*   **Memory:** The BVH tree memory is low (<2.3 MB for $512^3$ 8-bit volume) [S1, unofficial].
*   **Strengths:** High thread coherency due to compaction; efficient handling of sparse or complex isosurfaces; reduced code branches.
*   **Limitations:** CPU-side overhead for kernel launches and data transfer between compaction steps; less efficient for very simple isosurfaces compared to a single-kernel approach.

## Background
The method was introduced by D. M. Hughes et al. in 2013 as part of the "In-Kernel Stream Compaction" paper [S1, unofficial]. It was designed to address the inefficiencies of single-kernel ray-tracing pipelines on GPUs, where variable workloads lead to thread divergence and underutilization. The approach leverages the NVIDIA 480 GTX hardware characteristics, specifically texture caching, to optimize leaf size and traversal.

## Sources
[S1] InKernel Stream Compaction and Its Application to ..., https://www.sci.utah.edu/~knolla/hughes13.pdf, fetched 2026-10-29 (unofficial)
