# Octree Traversal with Min/Max Pruning

Also called: Knoll et al. Min/Max Tree, Min/Max Hierarchy Ray Traversal, Min/Max Tree, Min/Max Tree Space Skipping, Min/Max Tree Traversal, Octree Ray Tracing with Min/Max Acceleration, Scalar Octree Traversal

## Overview
Octree Traversal with Min/Max Pruning is an acceleration technique for ray tracing isosurfaces in scalar volumes stored in an octree [S1]. It solves the problem of efficiently skipping transparent or irrelevant regions during rendering by storing the minimum and maximum scalar values of each node's children [S2, unofficial]. Unlike standard empty-space skipping which may rely on coarse bricks, this method uses the hierarchical min/max ranges to prune nodes that do not contain the target isovalue, allowing for interactive transfer function changes without rebuilding the entire geometry [S2, unofficial]. It sets itself apart by embedding the acceleration structure and scalar data in a single octree, leveraging the duality between voxels and cells to define cell primitives for ray intersection [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ [S1].
*   Ray entry time $t_{\text{enter}}$ and exit time $t_{\text{exit}}$ for the volume bounding box [S2, unofficial].
*   Target isovalue $I$ [S1].
*   Octree structure containing scalar data and min/max metadata [S2, unofficial].
*   Maximum octree depth $D_{\text{max}}$ [S2, unofficial].

**Outputs:**
*   Boolean indicating if the ray intersects the isosurface within the volume [S2, unofficial].

### Data Structures
The octree is stored as a single structure serving as both data container and acceleration tree [S1].

**Node Layout:**
Each node contains:
*   `child_mins`: Array of size 8, storing the minimum scalar value present in each of the 8 children [S2, unofficial].
*   `child_maxs`: Array of size 8, storing the maximum scalar value present in each of the 8 children [S2, unofficial].
*   `child_offset`: Integer. If `child_offset == -1`, the node is a leaf (scalar data) [S2, unofficial]. Otherwise, it is an internal node.
*   Implicit state during traversal:
    *   `child`: Current child index (0-7) [S2, unofficial].
    *   `child_cell`: 3D integer coordinate of the current cell within the octree level [S2, unofficial].

**Cell Primitive:**
The algorithm traces rays through cells composed of 8 voxels [S1]. A duality exists between voxels and cells: each voxel is mapped to the lower-left corner of a cell by shifting the scalar field backward by half a unit of voxel width [S1]. Values outside the octree data are defined as zero [S1]. This allows the ray to intersect well-defined cell primitives at each step [S1].

### Algorithm Steps

1.  **Initialization**: Start traversal at the root of the octree with the global ray parameters $(\mathbf{o}, \mathbf{d}, t_{\text{enter}}, t_{\text{exit}})$ [S2, unofficial].
2.  **Axis Intersection Calculation**: For the current node, calculate the intersection times with the X, Y, and Z octant mid-planes. Store the corresponding axis indices (0, 1, 2) sorted by increasing intersection time in `axis_isects` [S2, unofficial].
3.  **Child Traversal Loop**: Initialize `child_tenter = t_{\text{enter}}` [S2, unofficial]. Iterate through the valid axis intersections in `axis_isects` while the center time $t_{\text{center}} < t_{\text{exit}}$ [S2, unofficial]:
    a.  **Calculate Exit Time**: Compute `child_texit` as the minimum of the intersection time on the current axis (`t_center`) and the global `t_exit` [S2, unofficial].
    b.  **Min/Max Pruning Check**: Check if the isovalue $I$ lies within the child node's scalar range [S2, unofficial]:
        $$ I \geq \text{node.child\_mins[child]} \quad \text{AND} \quad I \leq \text{node.child\_maxs[child]} $$
        If this condition is false, the node does not contain the isosurface; skip to the next iteration (update `child_tenter` and `child` index) [S2, unofficial].
    c.  **Node Type Handling**:
        *   **Leaf**: If `node.child_offset == -1`, traverse the scalar leaf [S2, unofficial]. If it returns true, the ray hit and the function returns true [S2, unofficial].
        *   **Max Depth Cap**: If the current `depth == octdata.max_depth` and the node is not a leaf, traverse the cap geometry [S2, unofficial]. If it returns true, return true [S2, unofficial].
        *   **Internal Node**: Recursively traverse it with updated `tenter`/`texit` bounds [S2, unofficial]. If it returns true, return true [S2, unofficial].
    d.  **Early Exit**: If `child_texit == t_{\text{exit}}`, the ray has exited the volume or the current bounding region without hitting anything relevant; return false [S2, unofficial].
    e.  **Update Entry Time**: Set `child_tenter = child_texit` for the next iteration [S2, unofficial].
    f.  **Update Child Index**: Determine the axis bit from `axis_table` (e.g., `{4, 2, 1}` for X, Y, Z) using the current axis index [S2, unofficial].
        *   If the current `child` index has the axis bit set, clear the bit in `child` and the corresponding bit in `child_cell` [S2, unofficial].
        *   Otherwise, set the bit in `child` and the corresponding bit in `child_cell` [S2, unofficial].
4.  **Return**: If the loop completes without a hit, return false [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_minmax(ray, node, depth, t_enter, t_exit, isovalue, max_depth):
    """
    Traverses an octree using min/max pruning for isosurface rendering.
    """
    # Compute intersection times with X, Y, Z mid-planes of the current cell.
    # Returns axis indices (0, 1, 2) sorted by increasing intersection time.
    axis_order = compute_axis_order(ray, node.cell_coord, t_enter, t_exit)

    child_tenter = t_enter
    child = 0  # Initial child index
    child_cell = node.cell_coord
    axis_table = [4, 2, 1]  # Maps axis index to bit mask

    for axis_idx in axis_order:
        # Intersection time with the current axis mid-plane
        t_center = compute_intersection_time(ray, node.cell_coord, axis_idx, t_enter, t_exit)

        if t_center >= t_exit:
            break

        child_texit = min(t_center, t_exit)

        # Min/Max Pruning Check
        if isovalue >= node.child_mins[child] and isovalue <= node.child_maxs[child]:
            if node.child_offset == -1:
                # Traverse scalar leaf (standard ray-march/isosurface check on 8 voxels)
                if traverse_scalar_leaf(ray, node, child_tenter, child_texit, isovalue):
                    return True
            elif depth == max_depth:
                # Traverse cap geometry at max depth
                if traverse_cap(ray, node, child_tenter, child_texit, isovalue):
                    return True
            else:
                # Recursive internal node traversal
                if traverse_minmax(ray, node.children[child], depth + 1,
                                   child_tenter, child_texit, isovalue, max_depth):
                    return True

        if child_texit == t_exit:
            return False

        child_tenter = child_texit

        axis_bit = axis_table[axis_idx]
        child_bit = 1 << axis_idx

        if child & axis_bit:
            child &= ~axis_bit
            child_cell[axis_idx] &= ~child_bit
        else:
            child |= axis_bit
            child_cell[axis_idx] |= child_bit

    return False

def compute_axis_order(ray, cell_coord, t_enter, t_exit):
    """Returns axis indices [0, 1, 2] sorted by intersection time with cell mid-planes."""
    times = []
    for axis in range(3):
        t = compute_intersection_time(ray, cell_coord, axis, t_enter, t_exit)
        times.append((t, axis))
    times.sort(key=lambda x: x[0])
    return [axis for _, axis in times]

def compute_intersection_time(ray, cell_coord, axis, t_enter, t_exit):
    """Standard ray-plane intersection with the mid-plane of the current cell along the given axis."""
    mid = cell_coord[axis] + 0.5
    denom = ray.d[axis]
    if abs(denom) < 1e-9:
        return float('inf')
    t = (mid - ray.o[axis]) / denom
    if t_enter <= t <= t_exit:
        return t
    return float('inf')

def traverse_scalar_leaf(ray, node, t_enter, t_exit, isovalue):
    # Sources do not specify the exact scalar leaf traversal routine.
    # Standard implementation: ray-march through the 8 voxels of the cell primitive
    # and check for isosurface intersection within [t_enter, t_exit].
    pass

def traverse_cap(ray, node, t_enter, t_exit, isovalue):
    # Sources do not specify the exact cap traversal routine.
    # Standard implementation: intersect ray with the cap geometry bounding the cell.
    pass
```

### Parameters
*   `node.child_mins[child]`: Minimum scalar value in the child node [S2, unofficial].
*   `node.child_maxs[child]`: Maximum scalar value in the child node [S2, unofficial].
*   `octdata.max_depth`: Maximum depth of the octree [S2, unofficial].
*   Transfer Function (RGB$\alpha$ map): Used to classify regions interactively [S2, unofficial].

### Edge Cases and Pitfalls
*   **Max Depth Cap**: At the maximum octree depth, the algorithm does not traverse further into children but instead calls `traverse_cap`, implying leaf nodes at max depth represent cap geometry rather than volumetric data [S2, unofficial].
*   **Transfer Function Changes**: When the transfer function changes, the min/max ranges are used to reclassify regions. Empty Adaptive Block Refinement (ABR) regions can be deactivated, requiring an interactive rebuild of the BVH if used in conjunction [S2, unofficial].
*   **Precision**: The min/max pruning relies on the scalar range. Boundary cases where the isovalue exactly matches a stored min or max value are handled by the inclusive range check, ensuring safe traversal at the cost of potentially reduced pruning efficiency [S2, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Memory**: The octree stores min/max values for each node, adding overhead to the scalar data. The structure is lossless compressed, occupying a fraction of the original memory footprint [S1].
*   **Time**: Traversal is front-to-back, allowing early termination. Pruning efficiency depends on the variance of scalar values within nodes [S1].

### Measured Performance
*   Competitive frame rates on common multicore architectures for large time-variant data [S1].
*   Superior to using bricks' min/max values for space-skipping classification, which is reported to be of much lower quality [S2, unofficial].

### Strengths and Limitations
*   **Strengths**:
    *   Allows interactive transfer function changes by recomputing majorants from stored ranges [S2, unofficial].
    *   Efficient memory usage due to compression and single structure for data and acceleration [S1].
*   **Limitations**:
    *   Indirection of the transfer function prevents direct computation of local majorants, requiring subdivision and min/max range storage [S2, unofficial].
    *   Requires pre-processing to store min/max ranges [S2, unofficial].

### When to Use
*   Use for interactive isosurface rendering of large scalar volumes where transfer function changes are frequent [S1].
*   Use when memory bandwidth is a bottleneck and compression is beneficial [S1].

### When Not to Use
*   Not suitable for direct volume rendering with complex transfer functions that cannot be approximated by min/max ranges [S2, unofficial].
*   Not suitable if the volume data is not structured or cannot be converted to an octree [S1].

## Background

### Origin
The method was introduced by Knoll et al. in 2006 [S1]. It is described in "Interactive Isosurface Ray Tracing of Large Octree Volumes" [S1].

### History and Development
The technique builds on the concept of min/max trees for empty space skipping [S2, unofficial]. It embeds the acceleration tree and scalar data in a single structure, employing optimized octree hash schemes [S1].

### Applications
*   Large-scale volume visualization [S1].
*   Interactive isosurface rendering [S1].
*   Adaptive Mesh Refinement (AMR) data rendering [S2, unofficial].

## Sources

[S1] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~knolla/octiso-rt06.pdf, fetched 2026-10-28
[S2] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.researchgate.net/publication/261082346_Interactive_Isosurface_Ray_Tracing_of_Large_Octree_Volumes, fetched 2026-10-29 (unofficial)
