# 3DDDA Ray Traversal for Hierarchical Grids
Also called: 3DDDA for Hierarchical Grids
## Overview
This method performs ray traversal on hierarchical grid structures by combining a top-level grid traversal with 3DDDA (3D Digital Differential Analyzer) steps at each level. It solves the problem of efficiently skipping empty space in sparse volumetric data or hierarchical scene representations. It differs from its parent [Hierarchical DDA](12757-hierarchical-dda.md) by explicitly utilizing the 3DDDA algorithm for stepping through voxels within a grid instance, and from its sibling [Hierarchical DDA with Chunk Stepping](8591-hierarchical-dda-with-chunk-stepping.md) by relying on standard 3DDDA logic rather than chunk-based optimization.
## How it works
**Inputs:** A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$; a hierarchical grid structure where each node contains a list of objects or children.
**Outputs:** Intersection status and distance, or null if no intersection is found within the grid's bounds.

**Data Structures:**
*   **Grid Node:** Contains a bounding box, a list of child nodes (or primitives), and a flag indicating if the node has been tested.
*   **3DDDA State:** For each grid instance, precomputed constants including step sizes $\Delta t_x, \Delta t_y, \Delta t_z$ and initial t-values $t_{max_x}, t_{max_y}, t_{max_z}$ based on the ray and the grid's bounding box.

**Algorithm Steps:**
1.  **Cache Check:** If the current grid node has already been tested for this ray, return the known result [S1, unofficial].
2.  **Initialization:** Prepare the 3DDDA constants for the current grid instance based on the ray parameters and the grid's bounding box [S1, unofficial].
3.  **Child Intersection Test:** Within the current voxel (defined by the 3DDDA position), test all objects or child nodes contained in the list for intersection. If an intersection exists and lies within the current voxel's bounds, return the intersection [S1, unofficial].
4.  **Step Forward:** If no intersection is found in the current voxel, proceed to the next voxel along the ray path using the 3DDDA algorithm [S1, unofficial].
5.  **Termination:** If there is no next voxel in the current grid (ray has exited the grid bounds), return no intersection. Otherwise, return to Step 3 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_hierarchical_grid(ray, grid_node):
    # Step 1: Cache check
    if grid_node.tested:
        return grid_node.result

    # Step 2: Prepare 3DDDA constants
    # Calculate t_max and step values for x, y, z axes based on ray and grid bounds
    t_max_x, t_max_y, t_max_z = compute_3ddda_constants(ray, grid_node.bbox)
    step_x, step_y, step_z = compute_3ddda_steps(ray)
    
    # Initialize current voxel coordinates
    curr_x, curr_y, curr_z = grid_node.bbox.min
    t_current = 0.0

    while True:
        # Step 3: Test children/objects in current voxel
        for child in grid_node.children:
            if intersect(ray, child, t_current, t_max_x, t_max_y, t_max_z):
                grid_node.tested = True
                grid_node.result = intersection
                return intersection
        
        # Step 4: Proceed to next voxel
        # Determine which axis to step based on smallest t_max
        if t_max_x < t_max_y and t_max_x < t_max_z:
            curr_x += step_x
            t_max_x += step_x
        elif t_max_y < t_max_x and t_max_y < t_max_z:
            curr_y += step_y
            t_max_y += step_y
        else:
            curr_z += step_z
            t_max_z += step_z
            
        # Check if ray has exited the grid
        if not grid_node.bbox.contains(curr_x, curr_y, curr_z):
            grid_node.tested = True
            grid_node.result = None
            return None
            
        # Step 5: Loop back to test children in new voxel
```

**Edge Cases:**
*   **Superfluous Tests:** Comparing only object bounding boxes to grid cells can result in objects being added to cells they do not actually overlap, causing unnecessary intersection tests [S1, unofficial]. Pruning by testing primitives against cells can reduce this but may not significantly improve performance for SPD scenes [S1, unofficial].
*   **Precision:** Standard 3DDDA precision issues may arise near voxel boundaries; careful handling of floating-point comparisons is required [G].

## Subtopics
## Operating characteristics
*   **Complexity:** Depends on the sparsity of the grid and the depth of the hierarchy. In the worst case (fully occupied), it degrades to linear traversal.
*   **Performance:** Pruning bounding box tests against cells can improve performance by at most 10% for SPD scenes [S1, unofficial].
*   **Strengths:** Efficiently skips empty space using hierarchical structure and 3DDDA stepping.
*   **Limitations:** Bounding box overlap can lead to redundant intersection tests if not pruned [S1, unofficial].

## Background
This method is described in the context of ray tracing for hierarchical grids, where the traversal logic is similar across approaches but differs in how the hierarchy is structured (e.g., BVH vs. grid) [S1, unofficial]. The specific 3DDDA steps for hierarchical grids are outlined in early real-time rendering news discussions [S1, unofficial].

## Sources
[S1] RTNv12n1.txt, https://realtimerendering.com/resources/RTNews/text/RTNv12n1.txt, fetched 2026-10-24 (unofficial)
