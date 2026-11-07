# Octree Traversal with Min/Max Pruning

Also called: Min/Max Tree Traversal, Knoll et al. Min/Max Tree, Min/Max Tree Space Skipping, Min-Max Octree for Transparent Region Skipping

## Overview
Octree Traversal with Min/Max Pruning is an acceleration technique for ray tracing isosurfaces in scalar volumes stored in an octree [S3]. It solves the problem of efficiently skipping transparent or empty regions during rendering by using hierarchical min/max scalar value bounds to prune nodes that cannot contain the target isovalue [S2][S3]. Unlike sibling methods that may rely on brick-level classifications or view-dependent working sets, this method embeds the acceleration structure directly within the volume data container, allowing for interactive transfer function changes by recomputing classification based on stored scalar ranges [S1][S4, unofficial]. It builds on [Empty Space Skipping with Octree](1610-empty-space-skipping-with-octree.md) [S4, unofficial].

## How it works
The method traces rays through an octree where interior nodes store the minimum and maximum scalar values of their children, and leaf nodes store the min/max values of the 8 vertices of the corresponding cell [S1][S2]. The same structure serves as both the data container and the acceleration structure [S2].

### Inputs and Outputs
*   **Input:** Ray origin $\mathbf{o}$ and direction $\mathbf{d}$; target isovalue $I$; octree root node; maximum octree depth $D_{max}$; transfer function (RGB$\alpha$ map) [S1][S4, unofficial].
*   **Output:** Boolean indicating if the isosurface was hit [S4, unofficial].

### Data Structures
The node structure contains the following fields [S4, unofficial]:
*   `child_mins`: Array of minimum scalar values for the 8 children.
*   `child_maxs`: Array of maximum scalar values for the 8 children.
*   `child_offset`: Integer offset. If `child_offset == -1`, the node is a leaf (scalar leaf) [S4, unofficial].
*   Implicit state during traversal: `child` (current child index, 0–7), `child_cell` (current cell coordinates), `tenter` (ray entry time), `texit` (ray exit time) [S4, unofficial].

### Algorithm Steps
1.  **Initialization:** Start traversal at the root node with the full ray segment $[t_{enter}, t_{exit}]$ defined by the volume bounding box [S4, unofficial].
2.  **Traversal Loop:** For each interior node, iterate through valid axis intersections to determine which child octants the ray passes through in front-to-back visibility order [S1][S4, unofficial].
3.  **Child Selection:** Determine the child index based on the ray's position relative to the octree mid-planes. Update `child` and `child_cell` coordinates using bit manipulation based on the axis of intersection [S4, unofficial].
4.  **Pruning Check:** For the current child, check if the target isovalue $I$ lies within the child's scalar range [S4, unofficial]:
    $$ I \ge \text{node.child\_mins[child]} \quad \text{OR} \quad I \le \text{node.child\_maxs[child]} $$
    If this condition is false, the child cannot contain the isosurface; skip this child and continue to the next intersection [S4, unofficial].
5.  **Recursion/Leaf Handling:** If the condition is true, determine the action based on the node type and depth [S4, unofficial]:
    *   If `node.child_offset == -1` (Leaf): Traverse the scalar leaf. If a hit is found, return true [S4, unofficial].
    *   If depth equals `octdata.max_depth`: Traverse the cap geometry. If a hit is found, return true [S4, unofficial].
    *   Otherwise (Internal Node): Recursively traverse the child node with updated $t_{enter}$ and $t_{exit}$ bounds [S4, unofficial].
6.  **Termination:** If the calculated child exit time equals the global exit time, the ray has exited the volume or the current bounding region without hitting anything relevant; return false [S4, unofficial].
7.  **Update Bounds:** Update $t_{enter}$ to the current child's exit time for the next iteration [S4, unofficial].

### Pseudocode
The following pseudocode implements the traversal logic described in [S4, unofficial]. It defines all required helpers to be self-contained and valid Python.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Vec3i:
    def __init__(self, x=0, y=0, z=0):
        self.x, self.y, self.z = x, y, z
    def __getitem__(self, i):
        return (self.x, self.y, self.z)[i]
    def __setitem__(self, i, v):
        vals = (self.x, self.y, self.z)
        vals = vals[:i] + (v,) + vals[i+1:]
        self.x, self.y, self.z = vals

def calculate_axis_intersections(ray_origin, ray_direction, t_enter, t_exit):
    """Standard slab intersection: returns sorted list of (time, axis_index) for mid-planes."""
    # Placeholder for standard axis-aligned slab intersection logic [G]
    # Returns list of tuples sorted by time, e.g., [(t1, 0), (t2, 1), (t3, 2)]
    pass

def traverse_scalar_leaf(ray, node, t_enter, t_exit, isovalue):
    """Handles ray-casting for the scalar leaf cell primitive composed of 8 voxels [S2]."""
    # Placeholder for actual scalar leaf traversal [G]
    pass

def traverse_cap(ray, node, t_enter, t_exit, isovalue):
    """Handles ray-casting for cap geometry at maximum octree depth [S4, unofficial]."""
    # Placeholder for cap geometry traversal [G]
    pass

def traverse_octree_minmax(ray, node, depth, t_enter, t_exit, isovalue, max_depth):
    """
    Traverses an octree node using min/max pruning.
    
    Args:
        ray: Ray object with origin and direction.
        node: Current octree node.
        depth: Current depth in the octree.
        t_enter: Ray entry time for the current node's bounding box.
        t_exit: Ray exit time for the current node's bounding box.
        isovalue: The target isovalue for the isosurface.
        max_depth: The maximum depth of the octree.
        
    Returns:
        True if the isosurface is hit, False otherwise.
    """
    child = 0
    child_cell = Vec3i(0, 0, 0)
    
    # axis_table maps axis index to bit position (e.g., {4, 2, 1} for X, Y, Z) [S4, unofficial]
    axis_table = [4, 2, 1]
    
    # Calculate sorted intersections with X, Y, Z octant mid-planes [S4, unofficial]
    axis_isects = calculate_axis_intersections(ray.origin, ray.direction, t_enter, t_exit)
    
    child_t_enter = t_enter
    
    for i in range(len(axis_isects)):
        t_center, axis_idx = axis_isects[i]
        
        if t_center >= t_exit:
            break
            
        child_t_exit = min(t_center, t_exit)
        
        # Pruning check: Does the isovalue lie within the child's scalar range? [S4, unofficial]
        if isovalue >= node.child_mins[child] or isovalue <= node.child_maxs[child]:
            if node.child_offset == -1:
                if traverse_scalar_leaf(ray, node, child_t_enter, child_t_exit, isovalue):
                    return True
            elif depth == max_depth:
                if traverse_cap(ray, node, child_t_enter, child_t_exit, isovalue):
                    return True
            else:
                if traverse_octree_minmax(ray, node.children[child], depth + 1, child_t_enter, child_t_exit, isovalue, max_depth):
                    return True
        
        if child_t_exit == t_exit:
            return False
            
        child_t_enter = child_t_exit
        
        # Determine the axis bit and update child index/cell coordinates [S4, unofficial]
        axis_bit = axis_table[axis_idx]
        child_bit = 1 << axis_bit
        
        if child & axis_bit:
            child &= ~axis_bit
            child_cell[axis_idx] &= ~child_bit
        else:
            child |= axis_bit
            child_cell[axis_idx] |= child_bit
            
    return False
```

### Parameters
*   `node.child_mins[child]`: The minimum scalar value present in the child node [S4, unofficial].
*   `node.child_maxs[child]`: The maximum scalar value present in the child node [S4, unofficial].
*   `octdata.max_depth`: The maximum depth of the octree structure [S4, unofficial].
*   `Transfer Function (RGB$\alpha$ map)`: The mapping from scalar values to color and opacity [S4, unofficial].

### Edge Cases and Pitfalls
*   **Max Depth Handling:** At the maximum octree depth, the algorithm does not traverse further into children but instead calls `traverse_cap`, implying leaf nodes at max depth represent cap geometry rather than volumetric data [S4, unofficial].
*   **Transfer Function Changes:** When the transfer function changes, the min/max ranges stored in the accelerator leaves are used as indices into the RGB$\alpha$ map to classify regions. This allows for interactive changes without rebuilding the entire structure, though local majorants must be recomputed by iterating and computing the maximum extinction/α value of the transfer function inside the scalar range [S4, unofficial].

## Subtopics

### [Min-Max Octree Rendering](3709-min-max-octree-rendering.md)
Min-Max Octree Rendering is an acceleration technique for ray-casting unclassified scalar volumes, designed for scenarios where classification parameters (transfer functions) change frequently. It differs from pre-classified rendering by storing raw voxel data and using a hierarchical min-max structure to prune empty space and skip nodes that do not contain the target isovalue range, avoiding the costly recomputation of classification tables. It builds on [Octree Traversal with Min/Max Pruning](1739-octree-traversal-with-min-max-pruning.md).

### Min-Max Hierarchy with Branch-on-Need Octree
A technique to reduce the visited set of cells in marching cubes by embedding a min-max hierarchy in a branch-on-need octree to trivially ignore empty regions of a volume [S5, unofficial]. It uses the min-max hierarchy to identify empty regions of the volume [S5, unofficial]. This variant enables interactive rendering of data up to $512^3$ resolution [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** The traversal time depends on the sparsity of the isosurface and the effectiveness of the pruning. In the worst case, it is proportional to the number of nodes visited, which is bounded by the total number of nodes in the octree [S4, unofficial].
*   **Memory Complexity:** The method uses a single structure for both data and acceleration, reducing memory overhead compared to maintaining separate acceleration structures [S2]. The storage requirement is proportional to the number of octree nodes, with each internal node storing 16 scalar values (8 mins, 8 maxs) [S1][S4, unofficial].
*   **Performance:** Competitive frame rates are achieved on common multicore architectures, especially for large time-variant data [S2]. The method allows for interactive transfer function changes, which is a significant advantage over methods that require pre-computed classification tables [S1][S4, unofficial].
*   **Strengths:** Interactive transfer function changes; efficient memory usage; effective pruning of transparent regions [S1][S2][S4, unofficial].
*   **Limitations:** Requires the volume to be stored in an octree structure [S2].
*   **When to use:** Suitable for large-scale volume rendering where interactive transfer function changes are required and the data can be represented as an octree [S1][S2].
*   **When not to use:** Not given in the sources.

## Background
The method traces rays through a min/max hierarchy inside interior octree nodes to accelerate traversal and avoid regions not containing the isovalue [S3]. It embeds the acceleration tree and scalar data in a single structure, employing optimized octree hash schemes to achieve competitive frame rates on common multicore architectures [S2]. This approach allows interactive transfer function changes by recomputing majorants from stored ranges, whereas direct computation of local majorants is not possible due to the indirection of the transfer function [S4, unofficial]. It has been applied to Adaptive Mesh Refinement (AMR) data rendering [S4, unofficial].

## Sources
[S1] A new object-order ray-casting algorithm, https://doi.org/10.5555/602099.602130, fetched 2026-10-26
[S2] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~wald/Publications/2006/OctIso/octiso.pdf, fetched 2026-10-27
[S3] PDF Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~knolla/octiso-rt06.pdf, fetched 2026-10-28
[S4] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.researchgate.net/publication/261082346_Interactive_Isosurface_Ray_Tracing_of_Large_Octree_Volumes, fetched 2026-10-29 (unofficial)
[S5] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
