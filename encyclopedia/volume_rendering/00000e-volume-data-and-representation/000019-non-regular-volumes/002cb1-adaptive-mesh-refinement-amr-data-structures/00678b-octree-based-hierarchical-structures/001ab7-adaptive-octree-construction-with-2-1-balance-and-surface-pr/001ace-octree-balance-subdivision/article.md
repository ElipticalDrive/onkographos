# Octree Balance Subdivision


## Overview
Octree Balance Subdivision splits a single octree node into eight child nodes by halving the bounding box along each axis, preparing the structure for topological analysis or data redistribution [S1, unofficial]. Unlike higher-level strategies that manage global balance or parallel load distribution, this method focuses strictly on the local geometric decomposition of a cell and the initialization of its child data structures [S1, unofficial].

## How it works
**Inputs:**
- `parent_cell`: The current octree node to be subdivided [S1, unofficial].
- `iso_value`: The scalar threshold for isosurface extraction [S1, unofficial].
- `eval_func`: A function evaluating the implicit field $F(x,y,z)$ and optionally its gradient $\nabla F$ at a given point [S1, unofficial].
- `cache`: An evaluation cache storing previously computed values and gradients [S1, unofficial].

**Outputs:**
- A new `Branch` node containing an array of 8 child nodes (leaves or further branches) [S1, unofficial].
- The `parent_cell` is replaced in the tree structure by this new `Branch` [S1, unofficial].

**Data Structures:**
- **OctreeNode**: Can be a `Leaf` or `Branch` [S1, unofficial].
  - `Leaf`: Stores `CornerData` (8 corners, each with value $v$ and optional gradient $\vec{g}$) and `cell_type` (Empty, Full, Leaf) [S1, unofficial].
  - `Branch`: Stores `children`, a vector/array of 8 pointers to `OctreeNode`s [S1, unofficial].
- **CornerData**:
  - `value`: float, $F(x,y,z)$ [S1, unofficial].
  - `gradient`: tuple of 3 floats, $\nabla F(x,y,z)$, or `None` [S1, unofficial].

**Steps:**
1. **Compute Child Coordinates**: For each of the 8 octants $i \in \{0, \dots, 7\}$, compute the child grid coordinates $(cx, cy, cz)$ by adding `child_scale` to the parent grid coordinates if the corresponding octant bit is set [S1, unofficial]. `child_scale` = $s/2$ [S1, unofficial]. For each of the 8 corners of the child cell, compute the grid position $(gx, gy, gz)$ by adding `child_scale` to the child cell's base coordinates if the corresponding corner bit is set [S1, unofficial].
2. **Evaluate Field**: Retrieve the scalar value and gradient for each corner from the evaluation cache [S1, unofficial]. If a gradient is missing, set a flag `has_grads` to false [S1, unofficial].
3. **Construct Leaf Data**: Construct a `LeafData` object with the corner values and `iso_value` [S1, unofficial]. If gradients are available for all corners, attach them to the leaf [S1, unofficial].
4. **Determine Cell Type**: Determine the cell type for the child octant: if the leaf has a sign change, it is a `Leaf` [S1, unofficial]. If the corner mask is 0, it is `Empty` [S1, unofficial]. Otherwise, it is `Full` [S1, unofficial].
5. **Update Tree Structure**: Compute the path to the current cell in the octree structure [S1, unofficial]. Navigate to the cell [S1, unofficial]. If it is already a `Branch`, return immediately to avoid orphaning existing children [S1, unofficial]. Push the new child cells array to the octree's children vector [S1, unofficial]. Update the current cell to be a `Branch` pointing to this new index [S1, unofficial]. This order prevents use-after-free bugs caused by vector reallocation invalidating pointers [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class CellType:
    EMPTY = 0
    FULL = 1
    LEAF = 2

class CornerData:
    def __init__(self):
        self.value = 0.0
        self.gradient = None

class LeafData:
    def __init__(self):
        self.corners = [CornerData() for _ in range(8)]
        self.iso_value = 0.0
        self.gradients_available = False
        self.cell_type = CellType.FULL

class Branch:
    def __init__(self, children):
        self.children = children

def compute_corner_position(bbox, octant_idx, corner_idx, child_scale):
    cx = bbox.x + (octant_idx & 1) * child_scale
    cy = bbox.y + (octant_idx & 2) * child_scale
    cz = bbox.z + (octant_idx & 4) * child_scale
    gx = cx + (corner_idx & 1) * child_scale
    gy = cy + (corner_idx & 2) * child_scale
    gz = cz + (corner_idx & 4) * child_scale
    return (gx, gy, gz)

def has_sign_change(corners, iso_value):
    min_val = min(c.value for c in corners)
    max_val = max(c.value for c in corners)
    return min_val <= iso_value <= max_val

def is_empty_mask(corners, iso_value):
    return all(c.value < iso_value for c in corners) or all(c.value > iso_value for c in corners)

def subdivide_octree_node(parent_node, iso_value, eval_func, cache):
    if isinstance(parent_node, Branch):
        return parent_node

    new_children = []
    parent_bbox = parent_node.bbox
    child_scale = parent_bbox.size / 2.0

    for octant_idx in range(8):
        child_leaf_data = LeafData()
        has_grads = True

        for corner_idx in range(8):
            pos = compute_corner_position(parent_bbox, octant_idx, corner_idx, child_scale)
            val, grad = cache.get_or_compute(pos, eval_func)

            child_leaf_data.corners[corner_idx].value = val
            if grad is None:
                has_grads = False
            else:
                child_leaf_data.corners[corner_idx].gradient = grad

        child_leaf_data.iso_value = iso_value
        if has_grads:
            child_leaf_data.gradients_available = True
        else:
            child_leaf_data.gradients_available = False

        if has_sign_change(child_leaf_data.corners, iso_value):
            child_leaf_data.cell_type = CellType.LEAF
        elif is_empty_mask(child_leaf_data.corners, iso_value):
            child_leaf_data.cell_type = CellType.EMPTY
        else:
            child_leaf_data.cell_type = CellType.FULL

        new_children.append(child_leaf_data)

    octree.children_vector.append(new_children)
    new_branch = Branch(children=new_children)
    parent_node.replace_with(new_branch)
    return new_branch
```

**Parameters:**
- `iso_value`: Typical value 0.0 for signed distance fields [S1, unofficial].
- `min_depth`, `max_depth`: Depth limits for recursion (not part of single subdivision step, but context for when to stop) [S1, unofficial].
- `child_scale`: Derived as $s/2$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Vector Reallocation**: Pushing children to a dynamic array before updating parent pointers is critical to avoid dangling pointers if the vector reallocates memory [S1, unofficial].
- **Missing Gradients**: If gradients are not available for all corners, the `has_grads` flag is set to false and gradients are not attached to the leaf [S1, unofficial].
- **Non-Cubic Bounding Boxes**: If the initial bounding box is not cubic, it is expanded to the smallest enclosing cube centered on the original bbox center [S1, unofficial].

## Subtopics

### [Octree Subdivision](4029-octree-subdivision.md)
Octree Subdivision splits an octree node into eight children by halving its bounding box along each axis and redistributing associated data elements like surfels. It provides the local geometric refinement step required for adaptive sparse voxel representations. This is the core geometric operation described in the main method.

### [Graph-based Load Balancing for Parallel Subdivision](8380-graph-based-load-balancing-for-parallel-subdivision.md)
This method balances parallel workloads for isosurface extraction by distributing volumetric blocks such that each process holds an equal number of active cells for any isovalue. It differs from spatial-only partitioning by analyzing the topological connectivity of active cells via graph partitioning, ensuring uniform computational load across parallel processes regardless of the specific isosurface threshold. It operates on the results of subdivision to manage parallelism.

### Local Refinement and Manifold Verification
This algorithm solves non-manifold configurations caused by the local application of subdivision rules [S2]. It extracts information about the shape of the isosurface $\partial V$ in face-adjacent cells over the faces of compact cells and recursively refines them if complex faces or edges are detected [S2].
**Steps:**
1. For every compact cell $c$, get adjacent cells at the same octree level [S2].
2. For every adjacent cell through a face $f$, extract all leaf cells $S$ that are face-adjacent with $c$ [S2].
3. Build a quadtree with faces of cells in $S$ adjacent to $c$ [S2]. This quadtree contains $F(x,y,z)$ values at vertices and represents the intersection between $V$ and $f$ [S2].
4. Apply complex face and complex edge tests [S2].
5. If a face is complex, divide the adjacent compact cell and add its sub-cells to the compact cells list [S2].
6. Recursively repeat the test until all compact cells having complex faces or edges have been eliminated by subdivision [S2].
Unlike Westermann et al. [19], who limit fixing to neighboring cells with at most one level of difference, this approach has no such limitation and is guided by topological complexity [S2].

## Operating characteristics
- **Time Complexity**: $O(1)$ per node subdivided, assuming constant-time evaluation and cache lookup [S1, unofficial]. However, the total time depends on the number of nodes visited, which is determined by the implicit function's complexity and depth limits [S1, unofficial].
- **Memory Complexity**: Each subdivision increases the number of nodes by a factor of 8 [S1, unofficial]. Memory usage scales with the number of leaf nodes in the octree [S1, unofficial].
- **Performance**: Performance is heavily dependent on the efficiency of the evaluation cache and the sparsity of the isosurface [S1, unofficial]. Dense regions cause rapid memory growth [S1, unofficial].
- **Strengths**: Simple, deterministic, and provides the necessary resolution for accurate isosurface extraction [S1, unofficial].
- **Limitations**: Does not inherently ensure manifold connectivity or balance; requires additional steps (like Local Refinement) to fix topological artifacts [S2].
- **When to use**: When adaptive resolution is needed to capture fine geometric details of an implicit surface [S1, unofficial].
- **When not to use**: When global load balancing or strict topological manifold guarantees are required without post-processing [S2].

## Background
The method is part of the adaptive octree construction pipeline for isosurface extraction, specifically within the context of Manifold Dual Contouring (DMC) [S1, unofficial]. The specific implementation details regarding vector reallocation safety and cache integration are derived from the `isomesh` library [S1, unofficial]. The topological refinement aspect (Local Refinement and Manifold Verification) is described in the work on adaptive surface extraction from anisotropic volumetric data, which addresses non-manifold configurations that can arise from local subdivision [S2].

## Sources
[S1] narnia-ai-mason/isomesh, https://github.com/narnia-ai-mason/isomesh, fetched 2026-10-27 (unofficial)
[S2] Adaptive surface extraction from anisotropic volumetric data: contouring on generalized octrees, https://doi.org/10.1007/s12243-013-0369-4, fetched 2026-10-29
