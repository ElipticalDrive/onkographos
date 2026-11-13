# Bounding-Sphere Tree Traversal with Frame-Rate Control

Also called: Adaptive Tree Traversal with Frame-Rate Control, Visibility Culling Tests

## Overview
This method renders hierarchical shell and solid volumetric structures by traversing a bounding-sphere tree and adaptively controlling the frame rate through dynamic adjustment of minimum splat sizes [S1, unofficial]. It distinguishes itself from standard octree traversal by replacing fixed-resolution voxel sampling with screen-space splat rendering, where the granularity of the visual output is directly tied to performance feedback [S1, unofficial]. Unlike sibling methods that rely on geometric bounding boxes for culling, this approach uses bounding spheres and hierarchical normal cones to optimize visibility tests for point-based primitives [S1, unofficial].

## How it works
The algorithm traverses a hierarchical tree (Shell or Solid) using a recursive descent. It combines visibility culling tests with a feedback loop that adjusts rendering parameters to maintain a target frame rate.

**Inputs and Outputs:**
*   **Input:** Root node of the bounding-sphere tree; current camera parameters (view frustum); current frame rate measurement.
*   **Output:** Rendered image composed of splats.
*   **Global Parameters:** $\delta_r$ (minimum regular splat size in pixels), $\delta_b$ (minimum boundary splat size in pixels) [S1, unofficial].

**Data Structures:**
The tree nodes store bounding sphere attributes. These attributes are compressed and decompressed on-the-fly using table lookups during traversal to save memory and bandwidth [S1, unofficial].
*   **Shell Nodes:** Store hierarchical normal vectors and bounding normal cones [S1, unofficial].
*   **Solid Nodes:** Do not store normal information during traversal; normals are derived from a CSG probe at render time [S1, unofficial].

**Numbered Steps:**

1.  **Frame-Rate Feedback:** Measure the current frame rate. If it is too low, smoothly increase $\delta_r$ and $\delta_b$. If the frame rate is too high (after a completed traversal), allow $\delta_r$ and $\delta_b$ to decrease. Note: In the original implementation, these parameters are only allowed to decrease after a fully completed tree traversal [S1, unofficial].
2.  **Traverse Tree:** Call `TraverseTree` on the root node.
3.  **Culling Tests:** For each child of the current node, perform the following tests in order. If a test fails, the child and its subtree are ignored.
    *   **View-Frustum Culling:**
        *   If the bounding sphere is completely outside the viewing frustum, ignore the node and all children [S1, unofficial].
        *   If the bounding sphere intersects any plane of the view frustum, traverse to the children [S1, unofficial].
        *   If the bounding sphere is completely inside the view frustum, disable this culling test for all children [S1, unofficial].
    *   **Back-Face Culling (Shell):** Use the hierarchically encoded normal vectors and bounding normal cones to eliminate branches where all nodes point entirely away from the camera [S1, unofficial].
    *   **Back-Face Culling (Solid):** Hierarchical culling is not performed during traversal because normals depend on the CSG probe. Instead, back-face culling is performed before rendering individual Solid splats using normals derived from the CSG probe [S1, unofficial].
    *   **Occlusion Culling:** Hierarchical occlusion culling is considered too hard for point-based primitives. Instead, enable the z-buffer algorithm in the graphics hardware for occlusion culling [S1, unofficial].
4.  **Splat Sizing:** Calculate the screen size $S$ of the child's bounding sphere in pixels.
5.  **Leaf/Node Handling:**
    *   If the child is a leaf node, render it as a splat of size $S$ pixels [S1, unofficial].
    *   If the child is a regular node and $S < \delta_r$, render it as a splat of size $\delta_r$ pixels [S1, unofficial].
    *   If the child is a boundary node and $S < \delta_b$, render it as a splat of size $\delta_b$ pixels [S1, unofficial].
    *   Otherwise, recursively call `TraverseTree(child)` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global parameters
delta_r = 1.0  # Minimum regular splat size in pixels
delta_b = 1.0  # Minimum boundary splat size in pixels

def update_frame_rate_control(current_frame_rate, target_frame_rate):
    """
    Adjusts delta_r and delta_b based on frame rate feedback.
    Note: Implementation constraint - deltas only decrease after 
    a fully completed traversal leads to too high frame rates.
    """
    if current_frame_rate < target_frame_rate:
        # Increase splat sizes to reduce rendering load
        delta_r = delta_r * 1.1
        delta_b = delta_b * 1.1
    elif current_frame_rate > target_frame_rate:
        # Only allow decrease after full traversal completion
        # This check is typically done outside the recursive loop
        delta_r = max(1.0, delta_r * 0.9)
        delta_b = max(1.0, delta_b * 0.9)

def traverse_tree(node):
    """
    Recursively traverses the bounding-sphere tree.
    """
    # Decompress attributes on-the-fly for culling tests
    attributes = decompress_node_attributes(node)
    
    for child in node.children:
        # 1. View-Frustum Culling
        frustum_status = check_frustum_intersection(child.bounding_sphere)
        if frustum_status == OUTSIDE:
            continue # Ignore node and children
        elif frustum_status == INTERSECTS:
            # Traverse to children, keep culling active
            pass
        elif frustum_status == INSIDE:
            # Disable frustum culling for children
            # (Logic handled in recursive call or child check)
            pass
        
        # 2. Back-Face Culling (Shell specific)
        if node.type == SHELL:
            if is_back_face(child.normal_cone, camera_position):
                continue # Eliminate branch
        
        # 3. Calculate screen size
        S = calculate_screen_size(child.bounding_sphere)
        
        # 4. Leaf or Node Handling
        if node.is_leaf:
            render_splat(child, size=S)
        elif node.type == REGULAR and S < delta_r:
            render_splat(child, size=delta_r)
        elif node.type == BOUNDARY and S < delta_b:
            render_splat(child, size=delta_b)
        else:
            traverse_tree(child)

def render_splat(splat, size):
    """
    Renders a splat with the given screen size.
    For Solid nodes, back-face culling is performed here using 
    normals from the CSG probe.
    """
    if splat.type == SOLID:
        normal = get_normal_from_csg_probe(splat)
        if is_back_face_single(normal, camera_position):
            return
    
    # Draw splat
    draw_point(size)
```

**Edge Cases and Pitfalls:**
*   **Parameter Adjustment:** $\delta_r$ and $\delta_b$ are only allowed to decrease after a fully completed tree traversal has led to too high frame rates. This prevents oscillation during partial traversals [S1, unofficial].
*   **Solid Node Normals:** Solid nodes lack hierarchical normal information. Back-face culling must be deferred to the splat rendering stage using the CSG probe, which adds overhead compared to Shell nodes [S1, unofficial].
*   **Occlusion:** Hierarchical occlusion culling is not implemented due to the point-based nature of the primitives. Relying on the hardware z-buffer is the only option, which may be less efficient for highly overlapping structures [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the tree depth and the selectivity of the culling tests. The adaptive frame-rate control ensures that the number of rendered splats is bounded by the performance target, effectively capping the rendering cost [S1, unofficial].
*   **Memory:** The tree is stored breath-first, allowing progressive loading of only the parts of the data structure actually rendered during traversal [S1, unofficial]. Attributes are compressed and decompressed on-the-fly [S1, unofficial].
*   **Strengths:** Real-time performance is guaranteed through adaptive refinement. The use of bounding spheres and normal cones allows for efficient visibility culling [S1, unofficial].
*   **Limitations:** Occlusion culling is limited to hardware z-buffering. Solid nodes require additional computation for normal derivation [S1, unofficial].

## Background
The method was introduced by Museth and Lombeyna in 2004 as part of the TetSplat system for real-time rendering and volume clipping of large volumetric datasets [S1, unofficial]. It was designed to handle both shell (surface) and solid (volume) representations efficiently by leveraging hierarchical tree structures and adaptive rendering techniques.

## Sources
[S1] TetSplat: Real-time Rendering and Volume Clipping of Large ... - Museth, https://ken.museth.org/Publications_files/Museth-Lombeyda_VIS04.pdf, fetched 2026-10-28 (unofficial)
