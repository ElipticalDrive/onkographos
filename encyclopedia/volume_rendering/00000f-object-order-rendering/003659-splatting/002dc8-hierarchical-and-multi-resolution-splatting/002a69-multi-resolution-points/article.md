# Multi-Resolution Points

Also called: Error-controlled Point Splatting, Reduced Point Cloud Rendering

## Overview
Multi-Resolution Points is a technique to render one large point in place of many small points to reduce rasterization cost and storage [S1, unofficial]. It solves the problem of high computational load when rendering dense point clouds by adaptively selecting point sizes based on a local error metric [S1, unofficial]. Unlike siblings that may rely solely on screen-space footprint, this method explicitly controls point size by volume content using an error threshold [S1, unofficial].

## How it works
**Inputs:** A set of points $P$ with attributes (position, color, opacity, size).
**Outputs:** A rasterized image.
**Data Structures:** An octree (or similar hierarchical tree) where internal nodes store aggregate properties (mean position, color, size) of their children [S1, unofficial].

1.  **Tree Construction:** Organize points into a tree. Internal nodes store aggregate properties of their children [S1, unofficial].
2.  **Error Calculation:** Compute a local error metric, typically Root Mean Square (RMS) error, between a node's aggregate and its children [S1, unofficial]. The RMS error $E$ for a node $N$ with children $C_i$ and aggregate value $A_N$ is:
    $$ E = \sqrt{\frac{1}{N} \sum_{i=1}^{N} (A_{C_i} - A_N)^2} $$
    where $N$ is the number of children [S1, unofficial].
3.  **Traversal and Selection:** Traverse the tree. If the error $E$ is below a user-defined threshold, render the node as a single large point. Otherwise, subdivide and process children [S1, unofficial].
4.  **Occlusion Culling (Optional):** Use a visibility map quadtree to cull occluded octree nodes [S1, unofficial].
5.  **Rasterization:** Rasterize points that fail the occlusion test [S1, unofficial].
6.  **Update:** Update the visibility map [S1, unofficial].

**Parameters:**
*   Error threshold: User-set value to control tree traversal [S1, unofficial].

**Edge Cases:**
*   Precision: RMS error calculation must handle floating-point precision for large point sets [S1, unofficial].
*   Boundaries: Nodes at the boundary of the point cloud may have fewer children, affecting error calculation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_multi_resolution_points(points, threshold):
    tree = build_octree(points)
    render_node(tree.root, threshold)

def render_node(node, threshold):
    if node.is_leaf():
        rasterize_point(node.point)
        return
    
    # Calculate RMS error between node aggregate and children
    error = calculate_rms_error(node)
    
    if error < threshold:
        # Render as single large point
        rasterize_point(node.aggregate)
    else:
        # Subdivide
        for child in node.children:
            render_node(child, threshold)
```

## Subtopics

### [Cloud Impostor Rendering](5125-cloud-impostor-rendering.md)
Cloud Impostor Rendering is a technique to accelerate cloud rendering by replacing geometric particle models with 2D texture impostors, exploiting the lack of high-frequency edges in clouds to tolerate lower resolution textures and frequent updates. Unlike geometric models which require high-resolution impostors only at a distance, this method allows impostors to be used for clouds even when the viewer is inside or very close to them, significantly reducing overdraw and fill-rate requirements. It builds on [Multi-Resolution Points](4784-multi-resolution-points.md) by using pre-shaded textures to represent volumetric density.

## Operating characteristics
*   **Time Complexity:** Depends on tree depth and error threshold. Lower threshold increases traversal cost but improves accuracy [S1, unofficial].
*   **Memory Complexity:** Requires storage for the octree structure [S1, unofficial].
*   **Performance:** Welsh '02 reduced 240k points to 90k points using a frequency-space metric [S1, unofficial].
*   **Strengths:** Reduces rasterization cost and storage [S1, unofficial].
*   **Limitations:** Requires careful tuning of error threshold [S1, unofficial].

## Background
Originated from work on volumetric points rendering [S1, unofficial]. Laur and Hanrahan used RMS error for error-controlled point splatting [S1, unofficial]. Welsh '02 applied frequency-space metrics to reduce point counts [S1, unofficial].

## Sources
[S1] Volumetric Points - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/volPts.pdf, fetched 2026-10-29 (unofficial)
