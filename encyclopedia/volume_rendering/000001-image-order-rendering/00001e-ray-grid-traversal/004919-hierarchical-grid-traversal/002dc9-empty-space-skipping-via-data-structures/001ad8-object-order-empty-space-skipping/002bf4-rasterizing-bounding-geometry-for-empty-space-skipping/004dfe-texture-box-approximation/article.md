# Texture Box Approximation

Also called: Li and Kaufman Texture Box Approximation

## Overview
Texture Box Approximation is a method for accelerating volume rendering by approximating the volume data with a set of bounding boxes, called texture boxes, which are rasterized to efficiently clip empty space from the rendering pipeline [S1, unofficial][S2, book]. It differs from level-of-detail methods that rasterize lower-resolution blocks by focusing on spatial occupancy to discard empty regions entirely before fragment shader execution [S1, unofficial][S2, book].

## How it works
**Inputs:** A 3D volume dataset; a hierarchical structure (e.g., octree or grid) defining leaf nodes; camera parameters.
**Outputs:** A 2D image with empty space skipped.

**Data Structures:**
*   **Texture Box:** A 3D bounding box defined by minimum and maximum coordinates $(x_{min}, y_{min}, z_{min})$ and $(x_{max}, y_{max}, z_{max})$.
*   **Hierarchy:** A tree where leaf nodes correspond to texture boxes. Each leaf node contains a reference to the volume data subset it bounds.

**Steps:**
1.  **Construct Hierarchy:** Build a hierarchical data structure (e.g., octree) over the volume data. Each leaf node represents a sub-volume.
2.  **Define Texture Boxes:** For each leaf node, compute its 3D bounding box in world or view space. This box is the "texture box" [S1, unofficial][S2, book].
3.  **Rasterize Bounding Geometry:** Project the 3D texture boxes onto the 2D screen. Rasterize these projected boxes using standard hardware rasterization [S1, unofficial][S2, book].
4.  **Clip Empty Space:** The rasterization process inherently clips pixels outside the projected boxes. Only pixels covered by at least one texture box are processed by the fragment shader [S1, unofficial][S2, book].
5.  **Render Volume:** For each covered pixel, sample the volume data within the corresponding texture box and compute the final color [S1, unofficial][S2, book].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def texture_box_approximation(volume_data, hierarchy, camera):
    # 1. Construct hierarchy (assumed pre-built)
    # hierarchy.leaves contains leaf nodes
    
    # 2. Define texture boxes for each leaf
    texture_boxes = []
    for leaf in hierarchy.leaves:
        # Compute 3D bounding box of the leaf's volume data
        box = compute_3d_bounding_box(leaf.volume_data)
        # Project to screen space
        screen_box = project_to_screen(box, camera)
        texture_boxes.append(screen_box)
        
    # 3. Rasterize bounding geometry
    # Hardware rasterizer clips pixels outside screen_box
    # Only pixels inside screen_box are sent to fragment shader
    
    # 4. Render volume for covered pixels
    for pixel in covered_pixels:
        # Find the texture box covering this pixel
        box = find_covering_box(pixel, texture_boxes)
        # Sample volume data within the box
        color = sample_volume(volume_data, box, pixel, camera)
        return color
```

**Parameters:**
*   **Hierarchy Type:** Not given in the sources.
*   **Box Resolution:** Not given in the sources.

**Edge Cases:**
*   **Precision:** Not given in the sources.
*   **Boundaries:** Not given in the sources.
*   **Degenerate Input:** Not given in the sources.

## Subtopics

### SparseLeap Rasterization
A sibling method that also uses rasterization for empty space skipping but differs in its specific traversal and data structure handling. Choose when sparse data structures are preferred. [SparseLeap Rasterization](5154-sparseleap-rasterization.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Efficiently clips empty space using hardware rasterization, reducing fragment shader load [S1, unofficial][S2, book].
*   **Limitations:** Not given in the sources.
*   **When to Use:** When hardware rasterization acceleration is available and empty space is significant [S1, unofficial][S2, book].
*   **When Not to Use:** Not given in the sources.

## Background
Texture Box Approximation was proposed by Li and Kaufman [S1, unofficial][S2, book]. It is an alternative to level-of-detail methods for reducing fragment shader load in volume rendering [S1, unofficial][S2, book].

## Sources
[S1] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S2] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
