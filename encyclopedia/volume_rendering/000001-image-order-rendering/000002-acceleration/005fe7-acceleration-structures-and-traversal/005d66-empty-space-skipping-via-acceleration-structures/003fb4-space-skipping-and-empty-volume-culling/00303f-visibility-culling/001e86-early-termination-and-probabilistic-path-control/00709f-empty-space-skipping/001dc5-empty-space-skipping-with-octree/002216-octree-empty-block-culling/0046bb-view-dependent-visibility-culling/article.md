# View-Dependent Visibility Culling

Also called: Camera-ordered visibility, Livnat and Hansen culling

## Overview
View-dependent visibility culling is an acceleration technique for volume rendering that orders the traversal of an octree based on the camera position to cull nodes occluded by previously traversed, closer nodes [S1, unofficial]. It extends standard empty-space skipping by ensuring that only visible surface nodes are processed, thereby accelerating both extraction and rasterization [S1, unofficial]. Unlike parent methods that only skip empty space, this method explicitly handles occlusion between non-empty nodes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An octree structure representing the volume data, and a camera position/orientation.
*   **Output:** A set of visible nodes (or a traversal order) that excludes occluded subtrees.

**Data Structures:**
*   **Octree:** The standard octree layout where each node indicates if it contains surface data (non-empty) or is empty [S1, unofficial].

**Steps:**
1.  **Camera-Ordered Traversal:** Traverse the octree starting from the root, prioritizing nodes closer to the camera. This establishes a depth order for visibility [S1, unofficial].
2.  **Empty Subtree Pruning:** During traversal, if a node is identified as empty (containing no surface), prune its entire subtree immediately [S1, unofficial].
3.  **Occlusion Culling:** For each non-empty node reached, check if it is occluded by any previously traversed node that is closer to the camera [S1, unofficial].
    *   If the current node is occluded by a closer, already-processed node, cull it (skip its processing) [S1, unofficial].
    *   If it is visible, process it (e.g., extract mesh or render) and mark its spatial extent as "occupied" for subsequent occlusion checks [S1, unofficial].

**Parameters:**
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Occlusion checks require robust geometric intersection tests or depth buffer comparisons to determine if a node is truly behind a closer node [G].
*   **Dynamic Cameras:** If the camera moves significantly, the traversal order and occlusion status must be recomputed [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def view_dependent_visibility_culling(octree, camera):
    # List to keep track of visible bounding boxes for occlusion testing
    visible_boxes = []
    visible_nodes = []

    # Helper to check if a box is occluded by any previously visible box
    def is_occluded(box):
        for vb in visible_boxes:
            if box_is_behind_camera_and_occluded(box, vb, camera):
                return True
        return False

    # Recursive traversal function
    def traverse(node):
        # Base case: leaf node
        if node.is_leaf():
            if not node.is_empty():
                if not is_occluded(node.bounding_box):
                    visible_nodes.append(node)
                    visible_boxes.append(node.bounding_box)
            return

        # Recursive case: internal node
        # Sort children by distance to camera to process closer nodes first
        children = sorted(node.children, key=lambda c: distance_to_camera(c.center, camera))

        for child in children:
            # Prune empty subtrees
            if child.is_empty():
                continue
            
            # Cull occluded nodes
            if not is_occluded(child.bounding_box):
                traverse(child)
                # If visible, add to occlusion list for children
                visible_boxes.append(child.bounding_box)

    traverse(octree.root)
    return visible_nodes
```

## Subtopics

### Quadtree-based Visibility Culling
A proposed visibility culling method using a quadtree structure to discard invisible voxels in $O(\log n)$ time in the best cases [S2, unofficial]. The quadtree is stored like mipmaps, containing the maximum value of all the subpixels [S2, unofficial]. It is suitable for heightmap-based terrain rendering where hierarchical visibility can be determined by value thresholds [S2, unofficial].

### Isometric Culling
A culling method for grid-aligned triles (tiles) that finds which triles are visible in the screen or not, and selectively renders them, including handling depth ranges and see-through flags [S3, unofficial]. It traverses to the first visible trile for each screen-space tile and renders this one only [S3, unofficial]. If a trile is flagged as see-through, it continues traversing until a non-seethrough trile or the level’s boundaries are hit [S3, unofficial]. For tilted isometric views, it finds triles with no neighbours on the faces that the camera can see [S3, unofficial]. It is used in games like *Fez* for efficient 3D tile rendering from an isometric perspective [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the occlusion test cost. In the best case for quadtree variants, it is $O(\log n)$ [S2, unofficial]. For octree variants, it is proportional to the number of visible nodes plus the cost of occlusion checks [S1, unofficial].
*   **Memory Complexity:** Requires storing the octree/quadtree and potentially a list of visible bounding boxes for occlusion testing [S1, unofficial][S2, unofficial].
*   **Strengths:** Reduces rendering load by skipping occluded geometry; leverages existing octree structures [S1, unofficial].
*   **Limitations:** Requires camera-dependent traversal order; occlusion checks can be expensive if not optimized [S1, unofficial]. Not suitable for perspective views without occlusion queries as noted in *Fez* development [S3, unofficial].
*   **When to use:** Real-time adaptive surface reconstruction where camera position is known and dynamic [S1, unofficial].
*   **When not to use:** When occlusion queries are unavailable and perspective views are dominant [S3, unofficial].

## Background
The technique was exploited by Livnat and Hansen [LH98] in the context of real-time adaptive surface reconstruction, using the octree structure to order visibility based on the user-specified camera position [S1, unofficial]. It was recognized that the octree could be used not only to speed extraction but also rasterization by culling occluded nodes [S1, unofficial]. Westermann et al. [WKE99] later recognized that speedups could be gained by exploiting the multiresolution nature of the octree structure, adaptively deciding to extract the mesh from fewer, coarser-resolution nodes [S1, unofficial].

## Sources
[S1] A Survey of Octree Volume Rendering Methods, https://www.sci.utah.edu/~knolla/octsurvey.pdf, fetched 2026-10-29 (unofficial)
[S2] Realtime Voxel Landscape Engines, https://www.flipcode.com/archives/Realtime_Voxel_Landscape_Engines-Part_4_Level_of_Detail_and_Optimisation.shtml, fetched 2026-10-30 (unofficial)
[S3] Behind Fez : Trixels (part two) – The Instruction Limit, https://theinstructionlimit.com/behind-fez-trixels-part-two, fetched 2026-10-28 (unofficial)
