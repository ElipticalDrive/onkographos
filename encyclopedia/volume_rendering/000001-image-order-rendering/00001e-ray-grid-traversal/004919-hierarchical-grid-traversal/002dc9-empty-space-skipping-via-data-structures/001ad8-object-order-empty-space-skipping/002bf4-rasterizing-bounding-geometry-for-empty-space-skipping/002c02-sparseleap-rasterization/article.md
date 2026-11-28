# SparseLeap Rasterization

Also called: SparseLeap rasterization details

## Overview
SparseLeap is a method for efficient empty space skipping in very large, sparse volume data, such as terabyte-sized microscopy volumes [S2, unofficial]. It solves the problem of fragmented space subdivisions making empty regions hard to skip and the infeasibility of pre-computing well-adapted space subdivisions due to frequent changes in active objects [S2, unofficial]. It sets itself apart from its siblings by exploiting hardware-accelerated rasterization to create per-pixel ray segment lists, thereby avoiding hierarchy traversal on the GPU during the ray-casting stage [S1, unofficial]. Unlike standard octree empty space skipping, SparseLeap scales better to large, sparse data [S2, unofficial]. It is agnostic of the underlying data structure used for managing volume data (e.g., page-table hierarchies or octrees) [S1, unofficial].

## How it works
SparseLeap operates in two main stages: a preprocessing/rasterization stage and a ray-casting stage [S2, unofficial].

**Inputs and Outputs:**
*   **Input:** An occupancy-histogram tree where each node has an occupancy class (empty, non-empty, or unknown) [S1, unofficial]. The tree is lazily constructed [S1, unofficial].
*   **Output:** A set of ray segment lists, one per pixel, identifying non-empty regions along the viewing ray [S2, unofficial].

**Data Structures:**
1.  **Occupancy-Histogram Tree:** A hierarchical structure where each node stores an occupancy class. Leaf nodes store the class directly; non-leaf nodes determine their class via a majority vote over their occupancy histogram (the class occurring most often in all leaf nodes of its sub-tree) [S3, unofficial].
2.  **Occupancy Geometry:** An array of bounding boxes extracted from the tree. Each emitted box stores:
    *   Center coordinates.
    *   Size (dimensions).
    *   Node's occupancy class.
    *   Parent's occupancy class [S3, unofficial].
3.  **Ray Segment List:** A per-pixel list of ray events (entry/exit) defining intervals (segments) with associated occupancy classes [S1, unofficial].

**Steps:**

1.  **Occupancy Geometry Generation:**
    *   Traverse the occupancy-histogram tree in breadth-first order, visiting child nodes in a fixed order (not visibility order) [S3, unofficial].
    *   For the root node, set the parent occupancy class to invalid (not equal to any other class). The root always emits its bounding box [S3, unofficial].
    *   For each visited non-leaf node, determine its occupancy class by taking the majority vote over its occupancy histogram [S3, unofficial].
    *   For leaf nodes, use the occupancy class already stored in the node [S3, unofficial].
    *   Compare the node's occupancy class with its parent's occupancy class.
        *   If they are the same, skip emission.
        *   If they are different, emit a bounding box containing the node's center, size, its own occupancy class, and its parent's occupancy class [S3, unofficial].
    *   If the number of emitted bounding boxes exceeds a prespecified optional resource limit, stop traversal [S3, unofficial].

2.  **Rasterization (Object-Order Stage):**
    *   Rasterize the set of fine-grained, view-independent bounding boxes generated in Step 1 using hardware-accelerated rasterization [S2, unofficial].
    *   For each pixel, compute an ordered list of stabbing boundary faces [S2, unofficial].
    *   Convert this into a ray segment list where consecutive ray segments of the same occupancy class are merged [S3, unofficial]. This creates singly-linked lists of ray events per pixel [S1, unofficial].
    *   Exploit frame coherence by re-using the same bounding boxes unless the set of active objects changes [S2, unofficial].

3.  **Ray-Casting (Image-Order Stage):**
    *   Perform ray casting independently for each pixel [S2, unofficial].
    *   Use the pre-computed ray segment lists to identify non-empty regions along the ray [S2, unofficial].
    *   Leap over empty space without hierarchy traversal by skipping segments marked as empty [S2, unofficial].
    *   Sample the non-empty volume data only within the non-empty segments [S2, unofficial].

4.  **Color Calculation:**
    *   Apply a simple analytical lighting model to each intersection of the ray and the volume cell [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sparse_leap_render(occupancy_tree, camera, volume_data):
    # Step 1: Generate Occupancy Geometry
    bounding_boxes = []
    stack = [(occupancy_tree.root, None)] # (node, parent_class)
    
    while stack:
        node, parent_class = stack.pop(0) # Breadth-first traversal
        
        # Determine current node's class
        if node.is_leaf:
            current_class = node.occupancy_class
        else:
            current_class = get_majority_vote(node.occupancy_histogram)
            
        # Geometry Emission Rule
        if current_class != parent_class:
            box = {
                'center': node.center,
                'size': node.size,
                'my_class': current_class,
                'parent_class': parent_class
            }
            bounding_boxes.append(box)
            
        # Add children to stack
        for child in node.children:
            stack.append((child, current_class))
            
    # Step 2: Rasterization
    # Hardware rasterization of bounding_boxes produces per-pixel ray segment lists
    # This step is typically handled by the GPU rasterizer, not explicit CPU code
    ray_segment_lists = hardware_rasterize(bounding_boxes, camera)
    
    # Step 3 & 4: Ray Casting and Color Calculation
    for pixel in pixels:
        segments = ray_segment_lists[pixel]
        color = vec3(0.0)
        
        for segment in segments:
            if segment.occupancy_class == 'empty':
                continue # Leap over empty space
            
            # Sample volume data in non-empty segment
            sample = sample_volume(segment.ray, volume_data)
            
            # Apply analytical lighting model
            light_color = calculate_lighting(sample, segment.ray)
            color += light_color
            
        pixel.color = color
```

**Parameters:**
*   **Resource Limit:** An optional prespecified maximum number of bounding boxes. If exceeded, traversal stops [S3, unofficial]. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Fragmentation:** Some fragmentation of space still remains in the occupancy geometry, which is resolved during the subsequent rasterization of ray segment lists where consecutive ray segments of the same occupancy class are merged [S3, unofficial].
*   **Active Objects:** The method relies on frame coherence. If the set of active objects changes frequently, the bounding boxes must be regenerated, incurring preprocessing cost [S2, unofficial].

## Subtopics

### Occupancy Geometry Generation
A view-independent process that extracts nested bounding boxes from an occupancy histogram tree to represent space occupancy, reducing spatial fragmentation compared to standard octrees. It traverses the tree in breadth-first order, emitting boxes only when the occupancy class differs from the parent's class. It allows for an optional resource limit to cap the number of emitted boxes. See [Occupancy Geometry Generation](9798-ray-segment-list-generation.md) for details on the subsequent conversion to ray segment lists.

## Operating characteristics
*   **Time Complexity:** The preprocessing stage (occupancy geometry generation) is linear with respect to the number of nodes in the occupancy-histogram tree [S3, unofficial]. The rasterization stage is hardware-accelerated and efficient [S2, unofficial]. The ray-casting stage is proportional to the number of non-empty samples, avoiding traversal of empty space [S2, unofficial].
*   **Memory Complexity:** Depends on the size of the occupancy-histogram tree and the number of emitted bounding boxes [S3, unofficial]. The ray segment lists are stored per pixel [S2, unofficial].
*   **Measured Performance:** SparseLeap scales better to large, sparse data than standard octree empty space skipping [S2, unofficial]. It optimizes both the number of look-ups into the data structure used for empty-space skipping and the number of samples along each viewing ray [S1, unofficial].
*   **Strengths:** Agnostic of the underlying data structure (e.g., page-table hierarchies, octrees) [S1, unofficial]. Efficient for very large, sparse volumes with fine structures [S2, unofficial].
*   **Limitations:** Requires hardware rasterization capabilities [S2, unofficial]. Performance may degrade if the set of active objects changes frequently, requiring regeneration of bounding boxes [S2, unofficial].
*   **When to use:** Large, sparse volume data (e.g., terabyte-sized microscopy volumes) where empty space skipping is critical and the active regions change frequently [S2, unofficial].
*   **When not to use:** Small, dense volumes where standard traversal is sufficient, or hardware rasterization is not available [S2, unofficial].

## Background
SparseLeap was introduced by Markus Hadwiger, Ali K Al-Awami, Johanna Beyer, and Hanspeter Pfister in 2017 [S2, unofficial]. It addresses the challenges of volume rendering for very high-resolution, large-size data, such as terabyte-sized microscopy volumes, which contain many fine and intricate structures [S2, unofficial]. The method moves the major cost of empty space skipping out of the ray-casting stage and into a hybrid strategy that balances computational load between determining empty ray segments in a rasterization (object-order) stage and sampling non-empty volume data in the ray-casting (image-order) stage [S2, unofficial].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S2] Faster RTX-Accelerated Empty Space Skipping using ... - ResearchGate, https://www.researchgate.net/publication/351428822_Faster_RTX-Accelerated_Empty_Space_Skipping_using_Triangulated_Active_Region_Boundary_Geometry, fetched 2026-11-03 (unofficial)
[S3] SparseLeap: Efficient Empty Space Skipping, https://vcg.seas.harvard.edu/publications/20180101-sparseleap-efficient-empty-space-skipping-for-large-scale-volume-rendering/paper, fetched 2026-10-24 (unofficial)
