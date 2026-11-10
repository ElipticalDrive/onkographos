# Bounding-Sphere Tree Traversal with Frame-Rate Control

Also called: Adaptive Tree Traversal with Frame-Rate Control, Visibility Culling Tests

## Overview
This method traverses hierarchical bounding-sphere tree structures (specifically Shell and Solid trees) to render point-based volumetric primitives efficiently. It solves the problem of maintaining real-time performance on large datasets by combining visibility culling tests with adaptive refinement. It is distinguished from sibling methods by its specific use of two distinct minimum splat size parameters ($\delta_r$ and $\delta_b$) to control fidelity at Shell-Solid intersections, and its reliance on hardware z-buffering for occlusion rather than hierarchical tree-based occlusion culling [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Root node of the hierarchical tree (Shell or Solid); current camera parameters (viewing frustum); current frame rate measurement.
*   **Output:** Rendered splats on the screen.

**Data Structures:**
The method operates on trees stored in breadth-first order in physical memory to enable fast recursive refinements and progressive loading [S1, unofficial]. Nodes contain attributes for bounding spheres. These attributes are decompressed on-the-fly using fast table lookups during traversal [S1, unofficial].
*   **Shell Nodes:** Store hierarchically encoded normal vectors and bounding normal cones [S1, unofficial].
*   **Solid Nodes:** Do not store normal information during traversal; normals are derived from a CSG probe at render time [S1, unofficial].

**Parameters:**
*   $\delta_r$: Global minimum regular splat size in pixels [S1, unofficial].
*   $\delta_b$: Global minimum boundary splat size in pixels [S1, unofficial].
*   Typical values are not given in the sources.

**Steps:**
1.  **Frame-Rate Feedback:** Check the current frame rate. If it is too low, increase $\delta_r$ and $\delta_b$ smoothly. If the frame rate is too high (detected after a completed traversal), $\delta_r$ and $\delta_b$ are allowed to decrease. Note: In the implementation, these parameters are only allowed to decrease after a fully completed tree traversal has led to too high frame rates [S1, unofficial].
2.  **Traverse Tree:** Call `TraverseTree` on the root node.
3.  **Process Children:** For each child of the current node:
    a.  **Culling Tests:** Perform visibility culling tests (Frustum, Back-face, Occlusion). If the child fails any test, ignore it and its descendants.
    b.  **Screen Size Calculation:** Calculate $S$, the screen size of the child's bounding sphere in pixels.
    c.  **Leaf/Node Decision:**
        i.  If the child is a leaf node, render it as a splat of size $S$ pixels.
        ii. If the child is a regular node and $S < \delta_r$, render it as a splat of size $\delta_r$ pixels.
        iii. If the child is a boundary node and $S < \delta_b$, render it as a splat of size $\delta_b$ pixels.
        iv. Otherwise, recursively call `TraverseTree(child)`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global parameters
delta_r = initial_minimum_regular_splat_size
delta_b = initial_minimum_boundary_splat_size

def update_frame_rate_control():
    """
    Adjusts delta_r and delta_b based on frame rate.
    Note: Decrease only allowed after a fully completed traversal.
    """
    current_fps = measure_current_frame_rate()
    if current_fps is too_low:
        delta_r = smooth_increase(delta_r)
        delta_b = smooth_increase(delta_b)
    # Decrease logic is handled externally after successful traversal completion
    # if current_fps is too_high and traversal_completed:
    #     delta_r = smooth_decrease(delta_r)
    #     delta_b = smooth_decrease(delta_b)

def traverse_tree(node):
    """
    Recursively traverses the bounding-sphere tree.
    """
    # 1. Frame-rate feedback check
    update_frame_rate_control()
    
    # 2. Process children
    for child in node.children:
        # 3a. Culling Tests
        if not passes_culling_tests(child):
            continue
            
        # 3b. Calculate screen size
        S = calculate_screen_size_of_bounding_sphere(child)
        
        # 3c. Leaf/Node Decision
        if child.is_leaf:
            render_splat(child, size=S)
        elif child.is_regular_node and S < delta_r:
            render_splat(child, size=delta_r)
        elif child.is_boundary_node and S < delta_b:
            render_splat(child, size=delta_b)
        else:
            traverse_tree(child)

def passes_culling_tests(child):
    """
    Performs View-Frustum, Back-face, and Occlusion culling.
    Returns True if the child is visible, False otherwise.
    """
    # View-Frustum Culling
    if is_completely_outside_frustum(child.bounding_sphere):
        return False
    if is_completely_inside_frustum(child.bounding_sphere):
        # Disable frustum culling for children of this node
        child.disable_frustum_culling = True
    elif intersects_frustum_planes(child.bounding_sphere):
        pass # Traverse to children
    else:
        return False # Should not happen if logic above is exhaustive
        
    # Back-face Culling
    if child.type == Shell:
        if is_back_face_hierarchically(child):
            return False
    elif child.type == Solid:
        # No hierarchical back-face culling during traversal
        # Culling happens at render time using CSG probe normals
        pass
        
    # Occlusion Culling
    # Hierarchical occlusion is hard for point-based primitives.
    # Relies on hardware z-buffer. No tree-level rejection here.
    return True

def is_back_face_hierarchically(node):
    """
    For Shell nodes: checks if all nodes in the branch point away from camera
    using encoded normal vectors and bounding normal cones.
    """
    # Implementation detail: Use hierarchical normal cone intersection with view vector
    return normal_cone_opposes_view(node.normal_cone, view_vector)
```

**Edge Cases and Pitfalls:**
*   **Parameter Adjustment:** $\delta_r$ and $\delta_b$ are only allowed to decrease after a fully completed tree traversal has led to too high frame rates. This prevents oscillation or premature refinement [S1, unofficial].
*   **Solid Nodes:** Solid nodes lack normal information during traversal, so hierarchical back-face culling cannot be applied. Back-face culling must be performed per-splat using normals from the CSG probe [S1, unofficial].
*   **Occlusion:** Hierarchical occlusion culling is not performed during traversal due to the point-based nature of primitives. It relies entirely on the graphics hardware's z-buffer [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on tree depth and culling efficiency. Breadth-first storage allows fast recursive refinements [S1, unofficial].
*   **Memory Complexity:** Progressive loading of data structures reduces physical memory usage by only loading parts of the data structure actually rendered [S1, unofficial].
*   **Strengths:** Real-time performance guaranteed through adaptive refinement and effective culling. Fast on-the-fly decompression [S1, unofficial].
*   **Limitations:** Occlusion culling is not hierarchical, relying on hardware. Solid nodes cannot use hierarchical back-face culling [S1, unofficial].
*   **When to use:** Real-time rendering of large volumetric datasets where interactive clipping and exploration are required [S1, unofficial].
*   **When not to use:** Scenes requiring hierarchical occlusion culling for point-based primitives, or where Solid node back-face culling during traversal is critical [S1, unofficial].

## Background
The method was introduced by Museth and Lombeyna in 2004 as part of the TetSplat system for real-time rendering and volume clipping of large datasets [S1, unofficial]. It addresses the need for dynamic volumetric clipping (CSG culling) while maintaining real-time performance, using adaptive splat sizes to manage the fidelity of Shell-Solid intersections [S1, unofficial].

## Sources
[S1] TetSplat: Real-time Rendering and Volume Clipping of Large ..., Museth, https://ken.museth.org/Publications_files/Museth-Lombeyda_VIS04.pdf, fetched 2026-10-28 (unofficial)
