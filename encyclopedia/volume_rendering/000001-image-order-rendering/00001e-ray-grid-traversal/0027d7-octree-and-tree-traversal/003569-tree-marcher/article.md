# Tree Marcher

Also called: Tree Marcher for Generic Grids

## Overview
The Tree Marcher is a hierarchical ray traversal algorithm for generic grid structures [S1]. It solves the problem of efficiently sampling sparse volumetric data by combining hierarchical empty-space skipping with Digital Differential Analyzer (DDA) stepping within occupied regions [S1]. It explicitly manages the transition between tree-level traversal and voxel-level stepping using a fixed-step DDA, handling arbitrary grid topologies defined by template parameters [S1].

## How it works
The method operates on a generic grid structure where the topology is defined by a `NodeT` type, while the ray, coordinate system, and data access are handled by `RayT`, `CoordT`, and `AccT` respectively [S1].

**Inputs and Outputs:**
*   **Input:** A ray defined by `RayT`, an accessor interface `AccT` providing grid data, and the root of the grid tree.
*   **Output:** Intersection status, voxel index `ijk`, grid value `v`, and ray parameter `t` at the intersection [S1].

**Data Structures:**
The implementation relies on template specialization rather than fixed memory layouts, allowing it to adapt to various node types (`NodeT`) [S1]. The core state tracks the ray parameter $t$, the voxel index $\mathbf{ijk}$, and the grid value $v$ [S1].

**Algorithm Steps:**
1.  **Hierarchical Descent:** The traversal begins at the root and uses a Hierarchical DDA (HDDA) to step through tree levels [S1].
2.  **Empty Space Skipping:** The HDDA advances the ray parameter $t$ to the exit of empty nodes, effectively skipping entire subtrees [S1].
3.  **Voxel-Level Stepping:** Upon reaching occupied regions, the method switches to a fixed-step DDA with a step size defined by the template parameter `Dim` [S1].
4.  **Zero-Crossing Detection:** The `ZeroCrossing` function checks for intersections at the voxel level [S1]. If detected, it returns `true`, updates $\mathbf{ijk}$ to the closest voxel after the intersection, sets $v$ to the grid value at $\mathbf{ijk}$, and sets $t$ to the intersection time [S1].
5.  **Continuation:** If no zero-crossing is found, the DDA continues stepping through leaf voxels until the leaf is exhausted or an intersection is found [S1]. Backtracking to the parent node occurs when a ray exits a subtree [G].

**Pseudocode:**
The following pseudocode illustrates the core logic derived from the `ZeroCrossing` and `TreeMarcher` descriptions [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def TreeMarcher(ray, accessor, grid_root):
    t = ray.origin_t
    node = grid_root
    while node is not None:
        if node.is_empty():
            t = max(t, node.exit_t)
            node = node.next_sibling_or_parent()
            continue
        # Fixed step size defined by template parameter Dim [S1]
        step = get_fixed_step_size(Dim)  # Not given in the sources.
        current_ijk = ray_to_voxel_index(ray, t)  # Not given in the sources.
        while is_within_bounds(current_ijk, node.bounds):
            found, v, t = ZeroCrossing(ray, accessor, current_ijk)
            if found:
                return current_ijk, v, t
            current_ijk = dda_step(current_ijk, ray.direction, step)  # Not given in the sources.
        node = node.next_sibling_or_parent()
        if node is None:
            break

def ZeroCrossing(ray, accessor, ijk):
    v_current = accessor.get_value(ijk)
    v_next = accessor.get_value(ijk + ray.direction_sign)
    if v_current * v_next < 0:
        t_intersect = interpolate_zero_crossing(v_current, v_next, ray)  # Not given in the sources.
        return True, v_next, t_intersect
    return False, v_current, ray.t
```

**Parameters:**
*   `NodeT`: Template parameter for the node type [S1].
*   `RayT`: Template parameter for the ray type [S1].
*   `AccT`: Template parameter for the accessor type [S1].
*   `CoordT`: Template parameter for the coordinate type [S1].
*   `Dim`: Template parameter defining the fixed step size for the DDA [S1].

**Edge Cases:**
Not given in the sources.

## Subtopics

### PointTreeMarcher
A specialized variant of the Tree Marcher designed for point grids [S1]. It differs from the generic `TreeMarcher` by omitting the `NodeT` template parameter, as point grids do not have the same hierarchical node structure as generic grids [S1]. It relies on `AccT`, `RayT`, and `CoordT` for operation [S1].

## Operating characteristics
*   **Complexity:** Time complexity depends on the sparsity of the grid and the depth of the tree. Empty space is skipped efficiently via HDDA, reducing the number of voxel samples [S1].
*   **Memory:** Memory usage is dominated by the grid structure itself. The algorithm itself uses minimal stack space for traversal [S1].
*   **Limitations:** The fixed step size in the DDA may not be optimal for all grid resolutions, potentially requiring tuning of the `Dim` parameter [S1].

## Background
The Tree Marcher is part of the OpenVDB library, specifically defined in `nanovdb/util/HDDA.h` [S1]. The method builds on the principles of Digital Differential Analyzers (DDA) and Hierarchical DDA (HDDA) [S1].

## Sources
[S1] nanovdb/util/HDDA.h File Reference - OpenVDB, https://academysoftwarefoundation.github.io/openvdb/HDDA_8h.html, fetched 2026-10-25
