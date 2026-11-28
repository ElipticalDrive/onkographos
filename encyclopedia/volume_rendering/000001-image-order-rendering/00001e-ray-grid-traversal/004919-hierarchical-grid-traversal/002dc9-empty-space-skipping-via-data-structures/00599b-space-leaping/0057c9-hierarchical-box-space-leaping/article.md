# Hierarchical Box Space Leaping

Also called: Box enclosure, Simplest form of space leaping, Traversing empty space

## Overview
Hierarchical Box Space Leaping is a ray casting acceleration technique that skips empty space to speed up rendering without affecting image quality [S1, unofficial]. It facilitates fast traversal of regions with homogeneous structure by enclosing the object into a set of boxes, possibly hierarchical, allowing the ray to leap over empty regions before engaging in volumetric traversal [S2, book]. Unlike methods that sample every voxel, this method skips empty space, providing significant speedup [S1, unofficial].

## How it works
The method operates in two distinct phases: an empty space search phase and a volumetric integration phase.

**Inputs and Outputs**
*   **Input:** A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$; a hierarchical data structure of bounding boxes enclosing the volume data.
*   **Output:** The final color and opacity for the pixel associated with the ray.

**Data Structures**
The core data structure is a hierarchy of axis-aligned bounding boxes (AABBs) [S2, book].
*   **Box Layout:** Each box is defined by its minimum corner $\mathbf{min} = (x_{min}, y_{min}, z_{min})$ and maximum corner $\mathbf{max} = (x_{max}, y_{max}, z_{max})$.
*   **Hierarchy:** Boxes are organized in a tree structure (e.g., a BVH or simple octree-like hierarchy) where parent boxes enclose child boxes [S2, book]. The root box typically encloses the entire volume.

**Algorithm Steps**
1.  **Ray-Box Intersection Test:** For the current ray, traverse the hierarchy from the root. At each node, test if the ray intersects the bounding box.
    *   If the ray does not intersect the box, skip the entire subtree rooted at that box.
    *   If the ray intersects, proceed to the children.
    *   This continues until a leaf box is reached or the ray exits the volume.
2.  **Determine Entry and Exit Points:** If the ray intersects the bounding box, calculate the entry point $t_{enter}$ and exit point $t_{exit}$ along the ray parameter $t$.
    *   The ray segment within the box is $[\mathbf{o} + t_{enter}\mathbf{d}, \mathbf{o} + t_{exit}\mathbf{d}]$.
    *   If $t_{enter} > t_{exit}$ or the interval is invalid, the ray misses the box.
3.  **Volumetric Traversal (Phase 2):** Only if the ray intersects the box, perform the time-consuming volumetric traversal of the material within the box boundaries [S2, book].
    *   Sample the volume data along the ray segment between $t_{enter}$ and $t_{exit}$.
    *   Integrate colors and opacities using standard volume rendering integration (e.g., front-to-back compositing) [S1, unofficial].
4.  **Empty Space Skipping (Phase 1):** The ray advances through empty space searching for an object. This phase corresponds to the traversal of the hierarchy where no volumetric sampling occurs [S1, unofficial].

**Formulas**
Ray-box intersection for an AABB defined by $\mathbf{min}$ and $\mathbf{max}$:
$$ t_{min} = \max \left( \frac{x_{min} - o_x}{d_x}, \frac{y_{min} - o_y}{d_y}, \frac{z_{min} - o_z}{d_z} \right) $$
$$ t_{max} = \min \left( \frac{x_{max} - o_x}{d_x}, \frac{y_{max} - o_y}{d_y}, \frac{z_{max} - o_z}{d_z} \right) $$
The ray intersects the box if $t_{max} \ge t_{min}$ and $t_{max} \ge 0$.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_ray(ray_origin, ray_direction, box_hierarchy):
    """
    Renders a single ray using Hierarchical Box Space Leaping.
    """
    # Phase 1: Skip empty space
    t_enter, t_exit = traverse_hierarchy(ray_origin, ray_direction, box_hierarchy)
    
    # If no intersection with any box, the ray passes through empty space
    if t_enter is None or t_exit is None:
        return background_color

    # Phase 2: Integrate colors and opacities within the intersected box
    start_point = ray_origin + t_enter * ray_direction
    end_point = ray_origin + t_exit * ray_direction
    
    color = 0.0
    opacity = 0.0
    
    # Sample along the segment within the box
    # Note: Actual implementation requires sampling the underlying volume data
    # at points between start_point and end_point.
    # This step is omitted as it depends on the specific volume data structure
    # and integration scheme not detailed in the source notes.
    
    return composite(color, opacity)

def traverse_hierarchy(ray_origin, ray_direction, node):
    """
    Recursively traverse the box hierarchy to find the entry and exit points.
    Returns (t_enter, t_exit) if intersection found, else (None, None).
    """
    # Test intersection with current node's bounding box
    t_min, t_max = ray_box_intersection(ray_origin, ray_direction, node.bbox_min, node.bbox_max)
    
    if t_max < t_min or t_max < 0:
        return None, None
    
    # If leaf node, return the intersection interval
    if node.is_leaf:
        return t_min, t_max
    
    # If internal node, check children
    # To optimize, sort children by distance or use a stack
    for child in node.children:
        t_enter, t_exit = traverse_hierarchy(ray_origin, ray_direction, child)
        if t_enter is not None:
            return t_enter, t_exit
            
    return None, None

def ray_box_intersection(origin, direction, bbox_min, bbox_max):
    """
    Computes the intersection interval [t_min, t_max] for a ray and an AABB.
    """
    t_min = -float('inf')
    t_max = float('inf')
    
    for i in range(3):
        if direction[i] == 0:
            if origin[i] < bbox_min[i] or origin[i] > bbox_max[i]:
                return None, None
        else:
            t1 = (bbox_min[i] - origin[i]) / direction[i]
            t2 = (bbox_max[i] - origin[i]) / direction[i]
            t_min = max(t_min, min(t1, t2))
            t_max = min(t_max, max(t1, t2))
            
    if t_max < t_min:
        return None, None
        
    return t_min, t_max
```

**Edge Cases and Pitfalls**
*   **Degenerate Boxes:** Boxes with zero volume should be handled or avoided in the hierarchy construction.
*   **Ray Parallel to Box Face:** Division by zero in ray-box intersection if the ray direction component is zero. The pseudocode handles this by checking `direction[i] == 0`.
*   **Precision:** Floating-point precision errors in intersection tests may cause gaps or overlaps in the hierarchy traversal.

## Subtopics

## Operating characteristics
*   **Time Complexity:** The traversal time depends on the depth of the hierarchy and the sparsity of the volume. In the best case (empty space), it is $O(\log N)$ where $N$ is the number of boxes. In the worst case (ray passes through all boxes), it is $O(N)$.
*   **Memory Complexity:** $O(N)$ to store the bounding boxes in the hierarchy.
*   **Strengths:** Significant speedup for sparse volumes by skipping empty space [S1, unofficial]. Simple to implement [S2, book].
*   **Limitations:** The bounding boxes may not tightly fit the object, leading to false positives (rays intersecting the box but not the volume data). This results in unnecessary volumetric traversal.
*   **When to Use:** Suitable for sparse volumes where empty space constitutes a large portion of the volume.
*   **When Not to Use:** For dense volumes where the overhead of hierarchy traversal outweighs the benefit of skipping empty space.

## Background
The concept of space leaping was introduced as a prolific and effective branch of volume rendering acceleration techniques [S1, unofficial]. The simplest form, using bounding boxes, was described as facilitating fast leaping across empty space [S2, book]. The technique is based on the principle that the passage of a ray through empty space does not contribute to the final image, so skipping it provides a significant speedup without affecting image quality [S1, unofficial].

## Sources
[S1] Volume Rendering, https://www.cs.unh.edu/~cs880/visualization/Celebi-Volume%20Rendering.pdf, fetched 2026-10-26 (unofficial)
[S2] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
