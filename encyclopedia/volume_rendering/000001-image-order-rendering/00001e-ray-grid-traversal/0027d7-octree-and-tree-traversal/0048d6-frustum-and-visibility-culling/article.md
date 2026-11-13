# Frustum and Visibility Culling

Also called: Implicit Frustum Culling, Ray-Determined Visibility

## Overview
Frustum and Visibility Culling is a visibility optimization technique used in ray-guided volume rendering where the working set of data (bricks or voxels) is determined implicitly during the ray traversal process itself [S1, unofficial]. Unlike traditional methods that require explicit pre-passes to cull objects against the view frustum or occlusion buffers, this approach leverages the fact that rays originating from the camera only intersect visible geometry [S1, unofficial]. It sets itself apart from its siblings by eliminating the overhead of separate culling stages, allowing rays to directly write out the list of required bricks in front-to-back order [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Camera parameters (position, orientation, frustum planes), ray origins and directions (one per pixel or sample), and the hierarchical volume data structure (e.g., octree with brick metadata).
- **Output:** A list of intersected bricks/voxels for each ray, or the final rendered image if compositing is performed immediately.

**Data Structures:**
- **Ray Buffer:** A buffer storing ray origins $\mathbf{o}$ and directions $\mathbf{d}$ for each pixel/sample.
- **Brick List/Buffer:** A structure (e.g., using `GL_ARB_shader_storage_buffer_object`) to store the indices of bricks intersected by each ray [S1, unofficial].
- **Octree/Brick Hierarchy:** Nodes containing bounding boxes and occupancy flags.

**Steps:**
1. **Ray Generation:** Generate rays from the camera through each pixel.
2. **Traversal:** For each ray, traverse the octree hierarchy starting from the root.
3. **Intersection Test:** At each node, test if the ray intersects the node's bounding box. If not, skip the subtree.
4. **Frustum Check (Implicit):** Since rays are generated from the camera, any node intersected by a ray is, by definition, within the view frustum. No explicit plane tests are needed [S1, unofficial].
5. **Occlusion Check (Implicit):** As rays traverse front-to-back, they accumulate opacity. If a ray is fully opaque before reaching a node, the node is skipped. This provides implicit occlusion culling without extra buffers [S1, unofficial].
6. **Leaf Sampling:** Upon reaching a leaf node (brick), sample the volume data.
7. **Working Set Recording:** Record the intersected brick indices in the output buffer [S1, unofficial].

**Formulas:**
- Ray intersection with AABB (Axis-Aligned Bounding Box):
  $$
  t_{min} = \max \left( \frac{b_{min,x} - o_x}{d_x}, \frac{b_{min,y} - o_y}{d_y}, \frac{b_{min,z} - o_z}{d_z} \right)
  $$
  $$
  t_{max} = \min \left( \frac{b_{max,x} - o_x}{d_x}, \frac{b_{max,y} - o_y}{d_y}, \frac{b_{max,z} - o_z}{d_z} \right)
  $$
  Intersection occurs if $t_{min} \le t_{max}$ and $t_{max} > 0$.

**Parameters:**
- Ray step size: Determined by leaf node size or brick resolution.
- Opacity threshold: For early ray termination.

**Edge Cases:**
- **Degenerate Rays:** Rays with zero length or direction should be handled or skipped.
- **Precision Issues:** Near-zero $t$ values may cause self-intersection artifacts; use a small epsilon.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_volume(rays, octree, brick_data):
    # rays: list of (origin, direction) tuples
    # octree: root node of the octree
    # brick_data: dictionary mapping brick_id to voxel data
    
    for ray in rays:
        o, d = ray
        t = 0.0
        current_node = octree
        accumulated_opacity = 0.0
        
        while current_node is not None:
            # Check if ray intersects current node's bounding box
            if not ray_aabb_intersect(o, d, current_node.bbox):
                current_node = None
                continue
            
            # Implicit Frustum Culling: Ray is already within frustum
            # Implicit Occlusion Culling: Check accumulated opacity
            if accumulated_opacity >= 1.0:
                current_node = None
                continue
            
            # If leaf node, sample and record
            if current_node.is_leaf:
                brick_id = current_node.brick_id
                # Sample volume data
                color, opacity = sample_brick(brick_id, o, d, t, brick_data)
                # Composite
                # ...
                # Record brick in working set
                working_set.append(brick_id)
                current_node = None
            else:
                # Traverse children
                # Determine which child to visit first (front-to-back)
                child = get_frontmost_child(current_node, o, d)
                current_node = child
                # Update t if needed (e.g., if child is further away)
        
        # End of ray
```

## Subtopics

### [Bounding-Sphere Tree Traversal with Frame-Rate Control](7227-bounding-sphere-tree-traversal-with-frame-rate-control.md)
Uses adaptive bounding-sphere culling and frame-rate feedback for point-based splatting.

### [Frustum Culling Integration](8798-frustum-culling-integration.md)
Filters 3D Gaussian Splatting elements based on user pose for progressive loading.

### [Frustum-Occlusion Culling for Octree Traversal](12324-frustum-occlusion-culling-for-octree-traversal.md)
Screen-space culling of octants based on projected area for voxel rasterization.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of rays, but with a significantly reduced constant factor due to implicit culling. The traversal depth is logarithmic with respect to the volume size.
- **Memory Complexity:** $O(M)$ where $M$ is the size of the octree and brick data. The working set buffer size depends on the visible portion of the volume.
- **Performance:** Provides significant speedups by avoiding explicit culling passes and occlusion buffer updates [S1, unofficial]. Performance scales well with hardware acceleration (e.g., GPU) [S1, unofficial].
- **Strengths:** Simple implementation, no extra passes, efficient use of hardware.
- **Limitations:** Relies on the assumption that rays are generated from the camera; may not be suitable for non-camera-based ray tracing (e.g., global illumination) without modification.

## Background
Introduced by Crassin et al. in 2009, this method is part of the evolution of ray-guided volume rendering [S1, unofficial]. It builds on earlier bricked single-pass and multi-resolution techniques [S1, unofficial].

## Sources
[S1] GPU-Based Large-Scale Scientific Visualization, https://johanna-b.github.io/LargeSciVis2018/slides/LargeScaleVisualization_Architectures.pdf, fetched 2026-10-29 (unofficial)
