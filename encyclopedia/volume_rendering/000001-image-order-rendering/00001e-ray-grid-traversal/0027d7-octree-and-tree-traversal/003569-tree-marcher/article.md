# Tree Marcher

Also called: Tree Marcher for Generic Grids

## Overview
The Tree Marcher is a hierarchical ray traversal algorithm designed for generic grid structures, specifically optimized for OpenVDB grids [S1]. It solves the problem of efficiently sampling sparse volumetric data by combining hierarchical empty-space skipping with Digital Differential Analyzer (DDA) stepping within occupied regions [S1]. Unlike sibling methods that may rely on fixed-step traversal or specific path-computation heuristics, the Tree Marcher explicitly manages the transition between tree-level traversal and voxel-level stepping using a specialized DDA, allowing it to handle arbitrary grid topologies defined by template parameters [S1].

## How it works
The method operates on a generic grid structure where the topology is defined by a `NodeT` type, while the ray, coordinate system, and data access are handled by `RayT`, `CoordT`, and `AccT` respectively [S1].

**Inputs and Outputs:**
*   **Input:** A ray defined by `RayT`, an accessor interface `AccT` providing grid data, and the root of the grid tree.
*   **Output:** Intersection points, voxel indices (`ijk`), and grid values (`v`) at those intersections.

**Data Structures:**
The implementation relies on template specialization rather than fixed memory layouts, allowing it to adapt to various node types (`NodeT`) [S1]. The core state involves the current ray parameter $t$, the current voxel index $\mathbf{ijk}$, and the grid value $v$ [S1].

**Algorithm Steps:**
1.  **Hierarchical Descent:** The traversal begins at the root. It uses a Hierarchical DDA (HDDA) to step through the tree levels [S1]. This involves determining which child node the ray intersects, similar to the principles in [Octree and Tree Traversal](4134-octree-and-tree-traversal.md).
2.  **Empty Space Skipping:** If a node is marked as empty, the HDDA advances the ray parameter $t$ to the exit of the current node's bounding box, effectively skipping the entire subtree [S1].
3.  **Voxel-Level Stepping:** Upon reaching a leaf node or an occupied region, the method switches to a fixed-step DDA. Unlike the HDDA, this DDA uses a fixed step size defined by the template parameter `Dim` (dimension) [S1].
4.  **Zero-Crossing Detection:** At each voxel step, the algorithm checks for zero-crossings in the scalar field. This is performed by the `ZeroCrossing` function [S1].
    *   If a zero-crossing is detected, the function returns `true`.
    *   The voxel index $\mathbf{ijk}$ is updated to the index of the closest voxel *after* the intersection point.
    *   The grid value $v$ at $\mathbf{ijk}$ is returned.
    *   The ray parameter $t$ is set to the time of intersection [S1].
5.  **Continuation:** If no zero-crossing is found, the DDA continues stepping through the leaf voxels until the leaf is exhausted or a zero-crossing is found. If the leaf is exhausted, the traversal backtracks to the parent node [S1, G].

**Pseudocode:**
The following pseudocode illustrates the core logic derived from the `ZeroCrossing` and `TreeMarcher` descriptions [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def TreeMarcher(ray, accessor, grid_root):
    # Initialize ray parameter t to start of ray
    t = ray.origin_t
    
    # Traverse the tree using HDDA logic (simplified)
    node = grid_root
    while node is not None:
        if node.is_empty():
            # Skip empty subtree
            t = max(t, node.exit_t)
            node = node.next_sibling_or_parent()
            continue
        
        # Reached occupied region, switch to voxel-level DDA
        # DDA uses fixed step size defined by Dim
        voxel_step = get_fixed_step_size(Dim)
        
        current_ijk = ray_to_voxel_index(ray, t)
        
        while current_ijk is within node.bounds:
            # Check for zero-crossing at current voxel
            found_intersection, v, intersection_t = ZeroCrossing(ray, accessor, current_ijk)
            
            if found_intersection:
                # Update state with intersection details
                t = intersection_t
                # Return or process intersection
                return current_ijk, v, t
            
            # Step to next voxel
            current_ijk = dda_step(current_ijk, ray.direction, voxel_step)
        
        # Leaf exhausted, backtrack
        node = node.next_sibling_or_parent()
        if node is None:
            break

def ZeroCrossing(ray, accessor, ijk):
    # Sample grid values at current and next voxel along ray
    v_current = accessor.get_value(ijk)
    v_next = accessor.get_value(ijk + ray.direction_sign)
    
    # Check for sign change (zero-crossing)
    if v_current * v_next < 0:
        # Interpolate to find precise intersection t
        # Note: Exact interpolation formula not provided in sources [G]
        t_intersect = interpolate_zero_crossing(v_current, v_next, ray)
        return True, v_next, t_intersect
    else:
        return False, v_current, ray.t
```

**Parameters:**
*   `NodeT`: Template parameter for the node type [S1].
*   `RayT`: Template parameter for the ray type [S1].
*   `AccT`: Template parameter for the accessor type [S1].
*   `CoordT`: Template parameter for the coordinate type [S1].
*   `Dim`: Template parameter defining the fixed step size for the DDA [S1].

**Edge Cases:**
*   **Precision:** The fixed step size in the DDA may lead to missed intersections if the step is too large relative to the feature size [S1].
*   **Boundaries:** The algorithm must handle ray exits from the grid volume gracefully [S1].

## Subtopics

### PointTreeMarcher
A specialized variant of the Tree Marcher designed for point grids [S1]. It differs from the generic `TreeMarcher` by omitting the `NodeT` template parameter, as point grids do not have the same hierarchical node structure as generic grids [S1]. It relies on `AccT`, `RayT`, and `CoordT` for operation [S1].

## Operating characteristics
*   **Complexity:** Time complexity depends on the sparsity of the grid and the depth of the tree. Empty space is skipped efficiently via HDDA, reducing the number of voxel samples [S1].
*   **Memory:** Memory usage is dominated by the grid structure itself. The algorithm itself uses minimal stack space for traversal [S1].
*   **Performance:** Optimized for OpenVDB grids, which are commonly used in visual effects for sparse data [S1].
*   **Limitations:** The fixed step size in the DDA may not be optimal for all grid resolutions, potentially requiring tuning of the `Dim` parameter [S1].

## Background
The Tree Marcher is part of the OpenVDB library, specifically defined in `nanovdb/util/HDDA.h` [S1]. It is designed to work with the NVIDIA OpenVDB (nanovdb) implementation, which is optimized for GPU rendering [S1]. The method builds on the principles of Digital Differential Analyzers (DDA) and Hierarchical DDA (HDDA) [S1].

## Sources
[S1] nanovdb/util/HDDA.h File Reference - OpenVDB, https://academysoftwarefoundation.github.io/openvdb/HDDA_8h.html, fetched 2026-10-25
