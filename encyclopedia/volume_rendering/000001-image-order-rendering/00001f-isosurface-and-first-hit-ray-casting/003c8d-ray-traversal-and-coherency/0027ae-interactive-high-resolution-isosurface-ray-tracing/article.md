# Interactive High Resolution Isosurface Ray Tracing

Also called: BONO-based ray tracing, Two-phase isosurface ray tracing

## Overview
This method renders isosurfaces in volumetric data on multi-core processors using a two-phase algorithm that decouples visibility culling from ray intersection [S1, unofficial]. Phase I traverses a coarse BONO (Branch-On-Need Octree) to build a short, front-to-back ordered list of visible, isosurface-intersecting data blocks for each pixel (or packet) [S1, unofficial]. Phase II shoots rays through these lists to find the first isosurface intersection, performing trilinear interpolation and shading [S1, unofficial]. It differs from standard packet ray casting by relying on high-level algorithmic list merging rather than SIMD instructions, avoiding performance drops on incoherent scenes [S1, unofficial].

## How it works
**Inputs:** Volumetric scalar field data, camera parameters (viewpoint, projection), isovalue $\tau$, and maximum list size $k$.
**Outputs:** Image with isosurface colors.

**Data Structures:**
1.  **Block Grid:** Data is organized into a coarse grid of equal-sized blocks. Scalar field values within each block are stored contiguously. Each block is identified by the coordinates of a pre-specified corner.
2.  **Block Octree:** An octree indexes the data within a block. Leaves correspond to $2 \times 2 \times 2$ cells, containing a pointer to such a cell. Each node stores the minimum and maximum scalar values ($v_{min}, v_{max}$) of voxels in its region.
3.  **BONO Tree:** A Branch-On-Need Octree indexes the coarse grid of blocks. It is augmented with value ranges ($v_{min}, v_{max}$) at each node. Unlike a standard octree, BONO avoids allocating nodes for empty subtrees when data resolution is not a power of two [S1, unofficial].
4.  **Pixel Lists:** For each pixel, a list of pointers to data blocks. The list is maintained in front-to-back order. The list size is capped at $k$. If the list is full (size $k$), a pointer to the next BONO node to traverse (for potential resumption) is stored at the end of the list [S1, unofficial].

**Parameters:**
*   $k$: Maximum size of the block list per pixel. Typical value: $k \le 20$ [S1, unofficial].
*   Block Size: Adaptive based on view distance. Far views use large blocks (e.g., $8 \times 8 \times 8$). Close views use smaller blocks (e.g., $4 \times 4 \times 4$) [S1, unofficial].

**Algorithm:**

**Phase I: List Generation**
1.  Traverse the BONO tree starting from the root.
2.  For a node $v$:
    a.  If $v_{min} \le \tau \le v_{max}$:
        i.  Compute the 3D Axis-Aligned Bounding Box (AABB) of $v$. This is derived from the coordinates of a pre-specified corner and the level of $v$.
        ii. Project the AABB onto the screen.
        iii. Identify all pixels falling within the projected area.
        iv. For each such pixel, if the current list size is less than $k$:
            - Traverse $v$'s children in front-to-back order relative to the viewpoint.
        v.  Else (list size is $k$):
            - Skip the subtree rooted at $v$. Store the resume point (the next node to visit in a depth-first traversal) at the end of the pixel's list.
    b.  Else ($\tau$ not in range):
        - Skip the subtree rooted at $v$.
3.  When a leaf is reached (representing a data block):
    a.  Compute the minimum bounding box of the block.
    b.  Project it onto the screen.
    c.  For each pixel falling within the projection, append a pointer to this block to the pixel's list, unless the list already has $k$ blocks.
4.  **Adaptive Block Size (Close Views):** If the view is close, after the initial BONO traversal, continue traversing the octree of the BONO leaves until the desired block size is reached. Then compute the minimum bounding box and construct lists for the resulting blocks [S1, unofficial].

**Phase II: Ray Casting**
1.  For each pixel with a non-empty list:
    a.  Shoot a ray from the pixel through the list of blocks in order.
    b.  For each block in the list:
        i.  Check if the ray intersects a voxel on the isosurface within the block.
        ii. If an intersection is found:
            - Perform trilinear interpolation using the unit cell containing the voxel to refine the position.
            - Compute the surface normal using forward differences.
            - Compute diffuse shading.
            - Stop processing for this ray.
        iii. If no intersection is found in the block, continue to the next block in the list.
    c.  If the ray reaches the end of the list:
        i.  If the list size is $k$: Resume the BONO tree traversal from the stored resume point. Continue Phase I traversal logic for the remaining nodes, appending new blocks to the list as they are found, and immediately check for intersections in newly added blocks.
        ii. If the list size is less than $k$: The ray does not intersect the isosurface.
2.  If the list is empty, the ray does not intersect the isosurface.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_isosurface(data, camera, isovalue, k, block_size_threshold):
    # Initialize pixel lists
    pixel_lists = {p: [] for p in camera.pixels}
    
    # Phase I: Initial BONO Traversal
    resume_points = {} # Map pixel -> next BONO node to resume
    
    def traverse_bono(node, pixel_lists, k, resume_points):
        if node is None:
            return
            
        # Check if isovalue is in range
        if not (node.v_min <= isovalue <= node.v_max):
            return
            
        # Project node AABB to screen
        projected_pixels = project_aabb_to_pixels(node.aabb, camera)
        
        for p in projected_pixels:
            if len(pixel_lists[p]) < k:
                # Traverse children front-to-back
                for child in node.children_front_to_back(camera):
                    traverse_bono(child, pixel_lists, k, resume_points)
            else:
                # List is full, store resume point
                resume_points[p] = node.next_node_in_traversal()
                
        # If leaf (data block)
        if node.is_leaf:
            block = node.block
            block_aabb = block.compute_min_bounding_box()
            block_pixels = project_aabb_to_pixels(block_aabb, camera)
            
            for p in block_pixels:
                if len(pixel_lists[p]) < k:
                    pixel_lists[p].append(block)
                else:
                    resume_points[p] = node.next_node_in_traversal()

    traverse_bono(bono_root, pixel_lists, k, resume_points)
    
    # Adaptive Block Size for Close Views
    # If close view, traverse octree of BONO leaves to reach desired block size
    if is_close_view(camera):
        for leaf in bono_root.leaves():
            sub_octree = leaf.sub_octree
            # Traverse until block size is small enough
            small_blocks = get_blocks_below_size_threshold(sub_octree, block_size_threshold)
            for block in small_blocks:
                block_aabb = block.compute_min_bounding_box()
                block_pixels = project_aabb_to_pixels(block_aabb, camera)
                for p in block_pixels:
                    if len(pixel_lists[p]) < k:
                        pixel_lists[p].append(block)
                    else:
                        resume_points[p] = leaf.next_node_in_traversal()

    # Phase II: Ray Casting
    for p in camera.pixels:
        ray = camera.get_ray(p)
        blocks = pixel_lists[p]
        intersection_found = False
        
        for block in blocks:
            hit = ray_intersect_isosurface_block(ray, block, isovalue)
            if hit:
                # Trilinear interpolation and shading
                color = compute_shading(hit.position, hit.normal, ray)
                image[p] = color
                intersection_found = True
                break
        
        if not intersection_found and len(blocks) == k:
            # Resume BONO traversal
            resume_node = resume_points[p]
            while resume_node is not None:
                # Re-traverse from resume_node, adding blocks to list
                # and checking for intersections immediately
                new_blocks = traverse_and_check(resume_node, ray, isovalue, pixel_lists[p], k)
                if new_blocks:
                    # Check intersections in new blocks
                    for block in new_blocks:
                        hit = ray_intersect_isosurface_block(ray, block, isovalue)
                        if hit:
                            color = compute_shading(hit.position, hit.normal, ray)
                            image[p] = color
                            intersection_found = True
                            break
                    if intersection_found:
                        break
                # Update resume point if list is full
                if len(pixel_lists[p]) == k:
                    resume_node = resume_node.next_node_in_traversal()
                else:
                    resume_node = None # List not full, no need to resume further
        
        if not intersection_found:
            image[p] = background_color

def ray_intersect_isosurface_block(ray, block, isovalue):
    # Use method from [10] for ray-voxel intersection
    # Check if ray intersects any voxel in block where isosurface exists
    for voxel in block.voxels:
        if isovalue_in_range(voxel):
            hit = ray_voxel_intersection(ray, voxel)
            if hit:
                # Trilinear interpolation in unit cell
                position = trilinear_interpolate(hit.position, voxel.cell)
                # Normal via forward difference
                normal = compute_gradient_forward_difference(voxel.cell)
                return Hit(position, normal)
    return None
```

**Edge Cases and Pitfalls:**
*   **Precision:** Trilinear interpolation is used for refinement. Forward differences are used for normals, which may be less accurate than central differences but faster [S1, unofficial].
*   **Boundaries:** The BONO structure handles non-power-of-two resolutions by avoiding empty node allocation [S1, unofficial].
*   **Degenerate Input:** If the isovalue is outside the range of all voxels, no intersections are found. If $k$ is too small, the algorithm may require extensive resumption, impacting performance, though correctness is maintained [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Phase I is efficient due to the compact BONO structure and projection-based culling. Phase II is fast if intersections are found in the initial lists. Resumption adds overhead but occurs rarely if $k$ is chosen appropriately [S1, unofficial].
*   **Memory Complexity:** Depends on the BONO tree size and the number of pixel lists. The BONO is more space-efficient than a standard octree for non-power-of-two resolutions [S1, unofficial].
*   **Measured Performance:** On a Clovertown platform (2x Quad-Core Intel Xeon 5320, 1.86 GHz, 8GB RAM) with a 1024x1024 screen:
    *   Far view, $k=3$ (single pixel lists): 273 msec.
    *   Far view, $k=6$ (2x2 pixel packets): 110 msec (2.48x speedup).
    *   Close view, $k=3$ (single pixel lists): 228 msec.
    *   Close view, $k=6$ (2x2 pixel packets): 68 msec (3.35x speedup) [S1, unofficial].
*   **Strengths:** High performance on multi-core processors, effective memory hierarchy usage, robust for incoherent scenes due to algorithmic parallelism over SIMD.
*   **Limitations:** Performance depends on appropriate choice of $k$ and block size. Resumption can be costly if $k$ is too small.
*   **When to use:** Interactive rendering of high-resolution volumetric data on multi-core CPUs, especially for incoherent scenes.
*   **When not to use:** Scenes with extreme coherence where SIMD packet ray casting might be more efficient, or on hardware with limited multi-core capabilities.

## Background
The method was introduced by J. Deng, C. Liu, and M. C. Yuen in "Interactive High Resolution Isosurface Ray Tracing on Multi-Core Processors" [S1, unofficial]. It addresses the need for efficient isosurface rendering on modern multi-core architectures by leveraging a compact BONO data structure and a two-phase approach to minimize redundant ray casting. The BONO (Branch-On-Need Octree) structure was previously introduced in [21] (not cited in the provided sources, but referenced in [S1, unofficial]). The method was tested on LLNL datasets.

## Sources
[S1] PDF Interactive High Resolution Isosurface Ray Tracing on Multi-Core ... - UMD, https://www.umiacs.umd.edu/~joseph/isosurface-paper.pdf, fetched 2026-10-26 (unofficial)
