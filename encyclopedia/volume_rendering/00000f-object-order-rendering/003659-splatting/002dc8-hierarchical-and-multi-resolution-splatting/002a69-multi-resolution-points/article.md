# Multi-Resolution Points

Also called: Error-controlled Point Splatting, Reduced Point Cloud Rendering

## Overview
Multi-Resolution Points is a technique to render one large point in place of many small points to reduce rasterization cost and storage [S1, unofficial]. It addresses the high computational cost of rendering large point clouds by adaptively merging nearby points into larger splats based on a local error metric [S1, unofficial]. Unlike [Hierarchical Splatting](1168-hierarchical-splatting.md) which often relies on fixed hierarchical levels, this method explicitly controls point size via a user-defined error threshold to balance visual fidelity and performance [S1, unofficial].

## How it works
**Inputs:** A set of points $\{P_i\}$ where each point $i$ has position $\mathbf{p}_i$, density $\rho_i$, and a reconstruction kernel $h$ (typically Gaussian). A user-defined error threshold $\epsilon$.
**Outputs:** A reduced set of splatted points mapped to the screen.

**Data Structures:**
1.  **Octree:** Points are organized into an octree structure. Each node represents a spatial region containing a subset of points.
2.  **Visibility Map:** A quadtree used to track occlusion status of octree nodes [S1, unofficial].

**Steps:**
1.  **Tree Construction:** Organize the input points into an octree. Each leaf node contains a cluster of points.
2.  **Error Calculation:** For each octree node, calculate a local error metric. Laur and Hanrahan use Root Mean Square (RMS) error [S1, unofficial]. Let $E_{node}$ be the error of a node. If $E_{node} \le \epsilon$, the node can be represented by a single larger point.
3.  **Tree Traversal & Culling:**
    *   Traverse the octree.
    *   Check the visibility map. If a node is occluded, cull it.
    *   If not occluded, check if the node meets the error threshold.
    *   If the node is too large (error $> \epsilon$), subdivide it and process children.
    *   If the node meets the error threshold, mark it as a leaf for rendering.
4.  **Rasterization:** Rasterize the points corresponding to the leaf nodes that failed the occlusion test. Each leaf node is rendered as a single large splat.
5.  **Update Visibility:** Update the visibility map based on the rendered splats to assist in culling subsequent frames or nodes [S1, unofficial].

**Parameters:**
*   $\epsilon$: Error threshold. Controls the trade-off between detail and performance. Typical values are not specified in the sources.

**Edge Cases:**
*   **Precision:** RMS error calculation must be robust to floating-point inaccuracies in small clusters.
*   **Boundaries:** Nodes at the boundary of the volume must be handled correctly during subdivision.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_multi_resolution_points(points, error_threshold):
    # Build octree from points
    octree = build_octree(points)
    
    # Initialize visibility map (quadtree)
    visibility_map = initialize_visibility_map()
    
    # List of splats to render
    splats = []
    
    def traverse_and_render(node):
        # Check occlusion using visibility map
        if is_occluded(node, visibility_map):
            return
        
        # Calculate local error for this node
        error = calculate_rms_error(node.points)
        
        if error > error_threshold:
            # Node is too detailed, subdivide
            if node.has_children:
                for child in node.children:
                    traverse_and_render(child)
            else:
                # Leaf node but error too high? 
                # Render as is or handle as degenerate case [G]
                splats.append(node.to_splat())
        else:
            # Node meets error threshold, render as single large point
            splats.append(node.to_splat())
            # Update visibility map for this region
            update_visibility_map(node, visibility_map)
            
    traverse_and_render(octree.root)
    
    # Rasterize all collected splats
    for splat in splats:
        rasterize(splat)
```

## Subtopics

### [Cloud Impostor Rendering](5125-cloud-impostor-rendering.md)
Cloud Impostor Rendering is a technique to accelerate cloud rendering by replacing geometric particle models with 2D texture impostors, exploiting the lack of high-frequency edges in clouds to tolerate lower resolution textures and frequent updates. Unlike geometric models which require high-resolution impostors only at a distance, this method allows impostors to be used for clouds even when the viewer is inside or very close to them, significantly reducing overdraw and fill-rate requirements. It builds on [Multi-Resolution Points](4784-multi-resolution-points.md) by using pre-shaded textures to represent volumetric density.

## Operating characteristics
*   **Time Complexity:** Depends on the tree traversal and occlusion culling. In the worst case, all points are rendered. With effective culling, it scales sub-linearly with the number of input points.
*   **Memory Complexity:** Requires storage for the octree and visibility map. The reduced point set requires less storage than the original point cloud.
*   **Performance:** Welsh '02 reduced 240k points to 90k points using a frequency-space metric [S1, unofficial]. This results in less rasterization cost due to overlap areas and less storage required [S1, unofficial].
*   **Strengths:** Reduces rasterization cost and storage. Provides lossless data compression by retaining only relevant points [S1, unofficial].
*   **Limitations:** Requires building and maintaining a tree structure. Error metric selection is critical.

## Background
The method was introduced in the context of volumetric point rendering. Laur and Hanrahan used RMS error to control point size [S1, unofficial]. Welsh '02 applied a frequency-space metric to reduce point counts [S1, unofficial]. The technique is part of the broader field of hierarchical and multi-resolution splatting.

## Sources
[S1] Volumetric Points - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/volPts.pdf, fetched 2026-10-29 (unofficial)
