# View-Dependent Visibility Culling

Also called: Camera-ordered visibility, Livnat and Hansen culling

## Overview
View-dependent visibility culling is an acceleration technique for volume rendering that discards voxels or nodes occluded by previously traversed geometry closer to the camera [S1, S2, unofficial]. It extends empty-space skipping by ordering traversal based on camera position, allowing the renderer to skip nodes that are hidden behind already-rendered surfaces [S1, unofficial]. Unlike static culling, it leverages the multiresolution nature of hierarchical structures (like octrees or quadtrees) to adaptively decide extraction or rendering resolution based on visibility [S1, unofficial].

## How it works
**Inputs:** Camera position and orientation; hierarchical data structure (octree or quadtree) representing the volume or heightmap; traversal state (list of visible/occluded nodes).
**Outputs:** Set of visible nodes/voxels to be rendered or extracted.

**Data Structures:**
*   **Octree/Quadtree:** Nodes store metadata indicating emptiness and potentially maximum values for culling decisions [S1, S2, unofficial].
*   **Traversal Stack:** Maintains the current path in the hierarchy.

**Steps:**
1.  **Initialize Traversal:** Start at the root of the hierarchical structure.
2.  **Camera-Ordered Traversal:** Traverse the tree in an order determined by the camera's view direction (e.g., front-to-back) [S1, unofficial].
3.  **Empty Subtree Pruning:** If a node is marked as empty (containing no surface/data), prune its entire subtree and do not traverse its children [S1, unofficial].
4.  **Visibility Culling:** For non-empty nodes, check if the node is occluded by any previously traversed node that is closer to the camera [S1, unofficial].
    *   If occluded, discard the node and its children [S1, unofficial].
    *   If visible, process the node (e.g., extract mesh or render voxels) and mark its spatial extent as occluded for subsequent nodes [S1, unofficial].
5.  **Recursive Descent:** If the node is not a leaf and not culled, recursively traverse its children [S1, unofficial].
6.  **Termination:** Stop when all reachable nodes have been processed or culled [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def view_dependent_culling(node, camera, occlusion_map):
    # Step 1: Check if node is empty
    if node.is_empty:
        return # Prune empty subtree [S1, unofficial]

    # Step 2: Check if node is occluded by previously traversed nodes
    if is_occluded(node, occlusion_map):
        return # Cull occluded nodes [S1, unofficial]

    # Step 3: Process visible node
    if node.is_leaf:
        render_or_extract(node)
        update_occlusion_map(node, occlusion_map)
    else:
        # Step 4: Traverse children in camera-ordered manner
        children = get_children_in_camera_order(node, camera)
        for child in children:
            view_dependent_culling(child, camera, occlusion_map)
```

**Parameters:**
*   **Occlusion Test:** Not given in the sources. [G] Typically involves bounding box intersection with previously rendered surfaces or depth buffer checks.
*   **Traversal Order:** Front-to-back relative to camera [S1, unofficial].

**Edge Cases:**
*   **Precision:** Occlusion tests must handle floating-point inaccuracies near node boundaries.
*   **Degenerate Input:** Nodes with zero volume or coincident geometry may cause incorrect occlusion if not handled.

## Subtopics

### Livnat and Hansen Culling
Uses an octree to order visibility based on camera position and cull occluded nodes [S1, unofficial]. Choose when using octrees for real-time adaptive surface reconstruction [S1, unofficial]. [Livnat and Hansen, 1998]

### Quadtree Heightmap Culling
Uses a quadtree stored like mipmaps with maximum values to discard invisible voxels in O(log n) time [S2, unofficial]. Choose for heightmap-based terrain rendering [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** O(log n) in the best cases for discarding invisible voxels using a quadtree [S2, unofficial]. Octree traversal is generally efficient but depends on the density of visible data [S1, unofficial].
*   **Memory Complexity:** Requires storing the hierarchical structure (octree/quadtree) and potentially an occlusion map [S1, S2, unofficial].
*   **Strengths:** Reduces computation by skipping occluded regions; adapts to camera movement [S1, S2, unofficial].
*   **Limitations:** Requires hierarchical data structure; occlusion testing can be costly if not optimized [S1, S2, unofficial].
*   **When to Use:** Real-time rendering with dynamic camera movement; scenes with significant occlusion [S1, S2, unofficial].
*   **When Not to Use:** Static scenes with no occlusion; non-hierarchical data structures [S1, S2, unofficial].

## Background
Originated with Livnat and Hansen [LH98] for real-time adaptive surface reconstruction using octrees [S1, unofficial]. Westermann et al. [WKE99] recognized the benefits of exploiting multiresolution for speedups [S1, unofficial]. Also discussed in the context of voxel landscape engines using quadtrees [S2, unofficial].

## Sources
[S1] A Survey of Octree Volume Rendering Methods, https://www.sci.utah.edu/~knolla/octsurvey.pdf, fetched 2026-10-29 (unofficial)
[S2] Realtime Voxel Landscape Engines, https://www.flipcode.com/archives/Realtime_Voxel_Landscape_Engines-Part_4_Level_of_Detail_and_Optimisation.shtml, fetched 2026-10-30 (unofficial)
