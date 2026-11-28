# Octree-based Volume Rendering

Also called: Chamberlain's algorithm, LaMar's algorithm, Octal tree structure for volume rendering, Octree Approach (Baseline), Octree on a GPU, Octree-based Volume Rendering with Adaptive Sampling, Weiler's smoothing

## Overview
Octree-based volume rendering accelerates the visualization of volumetric datasets by using a hierarchical octree structure to skip invisible voxels and reduce sampling in uniform areas [S2, unofficial][S3]. It solves the problem that only a small percentage of voxels contribute to the final image, while the rest remain invisible [S2, unofficial]. It is distinguished from its siblings, such as [Hierarchical Traversal](896-hierarchical-traversal.md) and [Residency Octree Traversal](3073-residency-octree-traversal.md), by its specific use of a one-to-one mapping of bricks to octree nodes for culling and spatial subdivision, often relying on CPU-side or unified memory traversal logic rather than specialized GPU residency caches [S3].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset (scalar field), a transfer function mapping scalar values to RGBA, and camera parameters (position, orientation, field of view) [S2, unofficial][G].
*   **Output:** A 2D image with accumulated color and opacity [G].

**Data Structures:**
*   **Octree:** A hierarchical tree where each parent node contains 0 or 8 children, achieving a 2x2x2 regular subdivision of the parent node into 8 equal parts or 'leaves' [Parent].
*   **Node Metadata:** Each node stores a `visibility_ratio`, defined as the ratio of visible data to the total amount of data in the cube [S2, unofficial].
    *   At the leaf level, a node represents one cell that is either completely filled or empty [S2, unofficial].
    *   For higher levels, `visibility_ratio` is calculated by averaging the ratios of its 8 children [S2, unofficial].
*   **Empty Cell Condition:** A cell is considered completely empty only if its eight angular (corner) values are completely transparent after applying the transfer function [S2, unofficial].

**Parameters:**
*   **Adaptive Sampling Threshold:** A predefined threshold for projected size or angular difference to trigger additional sampling points [S1, S2, unofficial].
*   **LaMar's FOV Threshold:** Nodes are selected if their projected angle is less than half the field-of-view angle [S1, unofficial].
*   **Early Stopping (PlenOctree variant):** Accumulated transmittance $T_i < \gamma = 0.01$ [S4, unofficial].

**Algorithm Steps:**

1.  **Preprocessing:**
    *   Perform octree subdivision of the volumetric data [S1, unofficial].
    *   Calculate the `visibility_ratio` for each node. This calculation depends on the transfer function and must be recalculated if the transfer function changes [S2, unofficial].
    *   For multi-channel rendering, this preprocessing is performed per channel or requires ensuring all required subtrees remain cache resident [S3].

2.  **Traversal Initialization:**
    *   Start traversal at the third level of the octree (or from the parent node of the last visited sample's node) to optimize performance [S1, unofficial][S3].
    *   For each ray, determine intersections with the octree structure [S4, unofficial].

3.  **Node Selection and Culling:**
    *   **Chamberlain's Criterion:** Compute the projected size of each octree node on the fly. If the size is smaller than a predefined threshold, consider the node sufficiently accurate to approximate all nodes below it [S1, unofficial].
    *   **LaMar's Criterion:** Use two criteria: distance (similar to Chamberlain) and field-of-view. Discard nodes that do not intersect the view frustum. Select nodes whose projected angle is less than half the field-of-view angle [S1, unofficial].
    *   **Empty Space Skipping:** If a node's `visibility_ratio` indicates it is empty (all corners transparent), skip the node entirely [S2, unofficial][S3].

4.  **Adaptive Sampling:**
    *   Perform adaptive sampling along the ray segment within the node [S2, unofficial].
    *   Take a large step between selection points when the beam passes through uniform areas.
    *   If there is a large difference in values between two neighboring selection points, select an additional point between them [S2, unofficial].
    *   This reduces sample size in areas where opacity makes a small contribution or where the volume is uniform [S2, unofficial].

5.  **Rendering and Blending:**
    *   **Trilinear Interpolation:** Define a cell as a cube whose eight corners are adjacent to voxel values. The intensity value for each position inside the cell is defined as a trilinear interpolation of angular values [S2, unofficial].
    *   **Sorting:** During traversal, sort children of a node in front-to-back order [S2, unofficial].
    *   **Rasterization:** While a node is to be drawn, slice the cuboid box corresponding to this node. Rasterize the slices and blend them into the previously drawn slices [S2, unofficial].
    *   **PlenOctree Variant:** For each ray, produce a sequence of segments between voxel boundaries with constant density and color, of lengths $\{\delta_i\}$. Apply NeRF's volume rendering model:
        $$ C(r) = \sum_{i=0}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) c_i $$
        where $T_i = \exp(-\sum_{j=0}^{i-1} \sigma_j \delta_j)$ [S4, unofficial].
    *   **Early Stopping:** Stop rendering when the accumulated transmittance $T_i$ is less than $\gamma = 0.01$ [S4, unofficial].

6.  **Smoothing (Optional):**
    *   Apply an extension (Weiler's smoothing) to smooth discontinuity artifacts at LOD boundaries caused by inconsistencies of the view-plane at different levels of detail [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_octree_volume(ray_origin, ray_direction, octree_root, transfer_function, camera):
    # Traversal initialization: start at third level or cached parent [S1, unofficial][S3]
    current_node = get_cached_parent_node(ray_origin, ray_direction, octree_root) or octree_root
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_transmittance = 1.0
    node_stack = [current_node]

    while node_stack:
        node = node_stack.pop()
        
        # 1. Frustum and Ray Intersection [S1, unofficial]
        if not intersects_frustum(node, camera) or not intersects_ray(node, ray_origin, ray_direction):
            continue
            
        # 2. Empty Space Skipping [S2, unofficial]
        if is_empty(node, transfer_function):
            continue
            
        # 3. Selection Criteria [S1, unofficial]
        projected_size = compute_projected_size(node, camera)
        if projected_size > camera.threshold:
            # Node too large: push children sorted front-to-back [S2, unofficial]
            children = get_children(node)
            children.sort_by_depth(ray_origin, ray_direction)
            node_stack.extend(children)
            continue
            
        # 4. Adaptive Sampling & Rendering [S2, unofficial]
        step = compute_large_step(node)
        t = node.min_t
        prev_value = None
        
        while t <= node.max_t:
            pos = ray_origin + ray_direction * t
            value = sample_voxel(pos, node)
            rgba = transfer_function(value)
            
            if rgba.a > 0:
                accumulated_color[0] += rgba.r * rgba.a * accumulated_transmittance
                accumulated_color[1] += rgba.g * rgba.a * accumulated_transmittance
                accumulated_color[2] += rgba.b * rgba.a * accumulated_transmittance
                accumulated_transmittance *= (1.0 - rgba.a)
                
            # Adaptive refinement: check difference with previous sample [S2, unofficial]
            if prev_value is not None and abs(value - prev_value) > threshold_diff:
                mid_t = t - step / 2.0
                mid_pos = ray_origin + ray_direction * mid_t
                mid_value = sample_voxel(mid_pos, node)
                mid_rgba = transfer_function(mid_value)
                if mid_rgba.a > 0:
                    accumulated_color[0] += mid_rgba.r * mid_rgba.a * accumulated_transmittance
                    accumulated_color[1] += mid_rgba.g * mid_rgba.a * accumulated_transmittance
                    accumulated_color[2] += mid_rgba.b * mid_rgba.a * accumulated_transmittance
                    accumulated_transmittance *= (1.0 - mid_rgba.a)
                    
            prev_value = value
            t += step
            
            # Early stopping [S4, unofficial]
            if accumulated_transmittance < 0.01:
                break
                
    return accumulated_color

def is_empty(node, transfer_function):
    # A cell is completely empty only if its eight angular values are completely transparent [S2, unofficial]
    corners = get_node_corners(node)
    for corner in corners:
        value = sample_voxel(corner, node)
        if transfer_function(value).a > 0:
            return False
    return True

def trilinear_interpolate(cell_corners, pos):
    # cell_corners: list of 8 scalar values at cube corners
    # pos: local position [0,1]^3 inside cell
    x, y, z = pos
    c000, c100, c010, c110, c001, c101, c011, c111 = cell_corners
    i0 = c000 * (1-x) + c100 * x
    i1 = c010 * (1-x) + c110 * x
    i2 = c001 * (1-x) + c101 * x
    i3 = c011 * (1-x) + c111 * x
    j0 = i0 * (1-y) + i1 * y
    j1 = i2 * (1-y) + i3 * y
    return j0 * (1-z) + j1 * z

def compute_large_step(node):
    # Step size proportional to node spatial extent along ray
    return node.extent_along_ray / 4.0

def get_cached_parent_node(ray_origin, ray_direction, octree_root):
    # Returns parent of last visited node or third level root [S1, unofficial][S3]
    return octree_root.children[0] if hasattr(octree_root, 'children') else octree_root

def compute_projected_size(node, camera):
    # Project node bounding box to screen space
    return node.screen_area

def intersects_frustum(node, camera):
    return camera.frustum.intersects(node.bbox)

def intersects_ray(node, ray_origin, ray_direction):
    return node.bbox.intersects_ray(ray_origin, ray_direction)

def get_children(node):
    return node.children if hasattr(node, 'children') else []

def sample_voxel(pos, node):
    # Fetch scalar value at pos, using trilinear interpolation if inside a cell [S2, unofficial]
    if node.is_leaf:
        return node.data[pos]
    return trilinear_interpolate(node.corners, node.local_pos(pos))

def get_node_corners(node):
    return node.corners if hasattr(node, 'corners') else [0.0]*8
```

## Subtopics

### [Octree-based Volume Rendering with Wavelet Leaves](6683-octree-based-volume-rendering-with-wavelet-leaves.md)
Combines an octree for spatial subdivision with 3D non-standard Haar wavelets in leaf nodes. Distinguished by enabling incremental loading and frustum-clipping-based memory management for large-scale datasets on distributed clusters.

### [Hierarchical Volume Rendering for Cloud Surface](9989-hierarchical-volume-rendering-for-cloud-surface.md)
Renders cloud surfaces using opacities $\rho(X)\sigma_t$ and voxel colors $I_0(X)$ with view-dependent octree cubes based on projected area-weighted approximation errors. Distinguished by its use of least-squares approximations and hardware cell-projection compositing.

### PlenOctree Rendering
Converts a trained NeRF-SH model into a sparse octree representation for real-time rendering. Uses NeRF's volume rendering model with early stopping at transmittance < 0.01 [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the depth of the octree and the sparsity of the data. In sparse volumes, it achieves significant speedups by skipping empty space [S3]. In dense volumes, the overhead of tree traversal may not be compensated by empty space skipping [S3].
*   **Memory Complexity:** Requires storing the octree structure in main memory or cache. For multi-channel rendering, entire subtrees must be resident in cache, leading to higher memory overhead (overhead ratio 1/3 for anisotropic volumes vs 1/7 for others) [S3].
*   **Measured Performance:** On an Intel Core 2 CPU E8400 3.0 GHz and GeForce 8800 GTX with $512^3$ scenes, the optimized version achieved 80 frames per second compared to 9 frames per second for the non-optimized version [S2, unofficial].
*   **Strengths:** Eliminates rasterization as a bottleneck by skipping invisible data; effective for sparse datasets [S2, unofficial][S3].
*   **Limitations:** Substantial visual artifacts at LOD boundaries due to view-plane inconsistencies [S1, unofficial]; inability to reduce volumetric data size in main memory limits rendering of datasets larger than available memory [S1, unofficial]; high computational cost and memory footprint in multi-channel scenarios due to per-channel tree traversal [S3].
*   **When to use:** Sparse volumetric data where empty space skipping provides significant gains; interactive applications requiring adaptive sampling [S2, unofficial].
*   **When not to use:** Dense volumes where traversal overhead dominates; scenarios with limited main memory for large datasets [S1, unofficial][S3].

## Background
The octree spatial hierarchy was first proposed by Chamberlain et al. for rendering surface geometry, involving octree subdivision, recursive traversal, and on-the-fly projection size computation [S1, unofficial]. LaMar et al. developed a similar technique with improved selection criteria (distance and field-of-view) [S1, unofficial]. Weiler et al. proposed extensions to smooth discontinuity artifacts at LOD boundaries [S1, unofficial]. The method is widely used in GPU-accelerated volume rendering pipelines to optimize performance by leveraging spatial coherence and transfer function-based culling [S2, unofficial].

## Sources
[S1] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S2] Optimized volume rendering using octree on a GPU, https://ir.lib.vntu.edu.ua/bitstream/123456789/30555/1/%d0%a2%d0%b5%d0%b7%d0%b8_%d0%9c%d0%9a%d0%9c%d0%9c_2020-53-54.pdf, fetched 2026-10-26 (unofficial)
[S3] Residency Octree: A Hybrid Approach for Scalable Web ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC10840607/, fetched 2026-10-29
[S4] PlenOctrees For Real-time Rendering of Neural Radiance Fields (NeRFs), http://alexyu.net/plenoctrees/, fetched 2026-10-28 (unofficial)
