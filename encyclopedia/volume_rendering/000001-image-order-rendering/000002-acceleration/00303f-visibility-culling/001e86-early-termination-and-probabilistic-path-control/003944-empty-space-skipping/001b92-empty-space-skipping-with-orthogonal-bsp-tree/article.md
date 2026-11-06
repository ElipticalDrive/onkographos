# Empty Space Skipping with Orthogonal BSP Tree

Also called: Growing boxes with orthogonal BSP, Orthogonal BSP tree empty space skipping

## Overview
This method accelerates texture-based volume rendering by partitioning the volume into sub-volumes (boxes) based on voxel properties, reorganizing them into an orthogonal BSP tree for visibility sorting, and skipping sub-volumes that are fully transparent according to the transfer function [S1]. It extends previous 2D texture approaches to 3D textures, which produce better image quality but require new sorting strategies due to arbitrary slicing plane orientations [S1]. Unlike siblings that use uniform grids or octrees, this method uses a one-stage goal-oriented growth process to separate surface voxels from interior voxels, optimizing lit rendering by minimizing unnecessary lighting computations [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset represented as one or more 3D textures [S1]. A transfer function (typically a 1D lookup table mapping density to color and opacity) [S1]. The current viewpoint (for perspective projection) or view direction (for parallel projection) [S1].
*   **Output:** A rendered image identical to the unaccelerated result, but computed faster [S1].

**Data Structures:**
1.  **Sub-volumes (Boxes):** Arbitrary axis-aligned bounding boxes enclosing clusters of voxels [S1]. Each box is defined by its minimum and maximum coordinates in the volume domain [G].
2.  **Orthogonal BSP Tree:** A binary space partitioning tree where all partitioning planes are orthogonal to one of the major axes (X, Y, or Z) [S1]. Each node represents a sub-volume or a partition [S1].
3.  **Opacity Map:** A low-resolution map on a plane orthogonal to one of the major axes of the dataset, used for occlusion culling [S1].

**Steps:**

1.  **Partition the Volume into Sub-volumes:**
    Group voxels with similar properties in both the volume domain (position) and the transfer function domain (densities and gradient magnitudes) into the same sub-volume [S1].
    *   **Uniform Boxes:** For unilluminated rendering, partition neighboring texels whose densities vary in a small range [S1]. A typical parameter value is a density range of 32 [S1].
    *   **Gradient Boxes:** For lit rendering, separate voxels with non-zero gradient magnitude from the rest to avoid lighting computation on interior voxels with zero gradient [S1]. This is done via a direct one-stage goal-oriented growth process [S1].

2.  **Classify and Skip Invisible Sub-volumes:**
    Classify a sub-volume as invisible if it is comprised of only voxels mapped to empty (fully transparent) by the transfer function [S1]. These sub-volumes are skipped for rendering [S1].

3.  **Reorganize into Orthogonal BSP Tree:**
    Reorganize the arbitrary set of sub-volume bounding boxes into an orthogonal BSP tree [S1]. All partitioning planes in this tree are orthogonal to a major axis [S1]. This simplifies visibility sorting compared to arbitrary oriented planes required for 3D textures [S1].

4.  **Sort and Render:**
    Render the sub-volumes in visibility order determined by the orthogonal BSP tree [S1]. The tree must be re-sorted whenever the viewpoint (for perspective projection) or view direction (for parallel projection) changes [S1].

5.  **Incremental Intersection Computation:**
    Compute incrementally the intersection of the axis-aligned sub-volumes with the slicing planes [S1]. This avoids the overhead of intersection and texture coordinate computation introduced by the partitioning [S1].

6.  **Occlusion Culling:**
    Create an opacity map on a plane orthogonal to one of the major axes [S1]. Update the opacity map from the contents of the frame buffer during rendering using GPU functionalities such as projective texture, interpolation, and blending functions [S1]. Project sub-volumes onto the opacity map and cull occluded ones [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_with_orthogonal_bsp(volume_data, transfer_function, viewpoint):
    # Step 1: Partition volume into sub-volumes using growing boxes
    sub_volumes = grow_boxes(volume_data, transfer_function)

    # Step 2: Classify invisible sub-volumes
    visible_sub_volumes = []
    for box in sub_volumes:
        if not is_fully_transparent(box, transfer_function):
            visible_sub_volumes.append(box)

    # Step 3: Reorganize into Orthogonal BSP Tree
    bsp_tree = build_orthogonal_bsp_tree(visible_sub_volumes)

    # Step 4: Sort and Render
    sorted_boxes = sort_boxes_by_visibility(bsp_tree, viewpoint)

    # Initialize frame buffer for occlusion culling
    frame_buffer = []

    for box in sorted_boxes:
        # Step 5: Incremental intersection computation
        intersections = compute_incremental_intersection(box, viewpoint)

        # Render the sub-volume using 3D texture
        render_sub_volume(box, intersections, transfer_function, frame_buffer)

    # Step 6: Occlusion Culling
    opacity_map = update_opacity_map(frame_buffer)
    cull_occluded_sub_volumes(visible_sub_volumes, opacity_map)

def grow_boxes(volume_data, transfer_function):
    # Groups voxels with similar properties in volume and TF domains
    # Returns list of boxes with min/max coordinates
    return []

def is_fully_transparent(box, transfer_function):
    # Checks if all voxels in box map to empty/transparent
    return False

def build_orthogonal_bsp_tree(boxes):
    # Recursively partitions boxes using planes orthogonal to major axes
    # Sources do not specify the exact partitioning algorithm [G]
    return None

def sort_boxes_by_visibility(bsp_tree, viewpoint):
    # Traverses BSP tree to return boxes in back-to-front order
    return []

def compute_incremental_intersection(box, viewpoint):
    # Computes intersection of axis-aligned box with slicing planes incrementally
    # Sources do not specify the exact incremental algorithm [G]
    return []

def render_sub_volume(box, intersections, transfer_function, frame_buffer):
    # Rasterizes box onto frame buffer using 3D texture and transfer function
    pass

def update_opacity_map(frame_buffer):
    # Projects frame buffer contents onto a low-resolution plane orthogonal to a major axis
    return []

def cull_occluded_sub_volumes(visible_sub_volumes, opacity_map):
    # Projects boxes onto opacity map and removes occluded ones
    pass
```

**Parameters:**
*   **Density Range for Uniform Boxes:** Neighboring texels whose densities vary in a small range are partitioned into a sub-texture. A typical value is 32 [S1].

**Edge Cases and Pitfalls:**
*   **Viewpoint Changes:** The BSP tree must be re-sorted whenever the viewpoint or view direction changes [S1]. Failure to do so results in incorrect rendering order [G].
*   **3D Texture Slicing:** Unlike 2D textures, 3D textures require handling arbitrary slicing plane orientations, which necessitates the orthogonal BSP tree for efficient sorting [S1].
*   **Lit vs. Unlit Rendering:** For lit rendering, it is crucial to separate surface voxels (non-zero gradient) from interior voxels (zero gradient) to avoid unnecessary lighting computations [S1].

## Subtopics

### [Growing Boxes Partitioning](1065-growing-boxes-partitioning.md)
Growing Boxes Partitioning is an adaptive volume partitioning method that divides a volume into sub-volumes (boxes) by clustering voxels based on their properties in both the volume domain and the transfer function domain. It aims to ensure that the visible box subset encloses few invisible voxels, thereby accelerating texture-based volume rendering by minimizing unnecessary lighting computations and texture switches. Unlike uniform partitioning or octrees, it separates surface voxels (non-zero gradient) from interior voxels (zero gradient) to optimize lit rendering, using a direct one-stage goal-oriented growth process rather than region growing followed by division.

## Operating characteristics
*   **Performance:** Rendering with empty space skipping is 2 to 5 times faster than without it, with identical image quality [S1].
*   **Hardware:** Tested on commodity graphics hardware with 3D textures [S1].
*   **Strengths:** Accelerates texture-based volume rendering by skipping invisible voxels; handles 3D textures with better image quality; incremental intersection computation minimizes overhead [S1].
*   **Limitations:** Requires re-sorting the BSP tree when the viewpoint or view direction changes; partitioning adds complexity compared to uniform grids [S1].
*   **When to Use:** When rendering 3D textures with lighting where many voxels are transparent or have zero gradient, and performance is critical [S1].
*   **When Not to Use:** When the volume has few transparent regions or when the overhead of partitioning and sorting outweighs the benefits of skipping [S1].

## Background
This method was proposed by Li, Mueller, and Kaufman [S1]. It extends their previous work on empty space skipping using stacks of 2D textures [S1]. The key innovation is the use of an orthogonal BSP tree to handle the arbitrary slicing plane orientations required by 3D textures, which produce better image quality than 2D textures [S1]. The method is part of the broader field of texture-based volume rendering acceleration [S1].

## Sources
[S1] Empty Space Skipping and Occlusion Clipping for Texture-based Volume ..., https://www3.cs.stonybrook.edu/~mueller/papers/OcclusionClipping.pdf, fetched 2026-10-26
