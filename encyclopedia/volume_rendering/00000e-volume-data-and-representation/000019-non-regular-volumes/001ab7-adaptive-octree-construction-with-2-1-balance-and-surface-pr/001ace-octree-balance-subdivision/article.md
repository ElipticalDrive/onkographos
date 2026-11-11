# Octree Balance Subdivision

Also called: Octree Subdivision, Cell Splitting, Node Refinement

## Overview
Octree Balance Subdivision is the fundamental geometric refinement operation in adaptive octree construction. It splits a single octree node into eight child nodes by halving the bounding box along each axis, preparing the structure for further topological analysis or data redistribution [S1, unofficial]. It serves as the atomic step for increasing spatial resolution in sparse voxel representations [S1, unofficial]. Unlike higher-level strategies that manage global balance or parallel load distribution, this method focuses strictly on the local geometric decomposition of a cell and the initialization of its child data structures [S1, unofficial].

## How it works
**Inputs:**
- `parent_cell`: The current octree node to be subdivided. Contains bounding box coordinates $(x, y, z)$ and size $s$.
- `iso_value`: The scalar threshold for isosurface extraction.
- `eval_func`: A function evaluating the implicit field $F(x,y,z)$ and optionally its gradient $\nabla F$ at a given point.
- `cache`: An evaluation cache storing previously computed values and gradients.

**Outputs:**
- A new `Branch` node containing an array of 8 child nodes (leaves or further branches).
- The `parent_cell` is replaced in the tree structure by this new `Branch`.

**Data Structures:**
- **OctreeNode**: Can be a `Leaf` or `Branch`.
  - `Leaf`: Stores `CornerData` (8 corners, each with value $v$ and optional gradient $\vec{g}$) and `cell_type` (Empty, Full, Leaf).
  - `Branch`: Stores `children`, a vector/array of 8 pointers to `OctreeNode`s.
- **CornerData**:
  - `value`: float, $F(x,y,z)$.
  - `gradient`: tuple of 3 floats, $\nabla F(x,y,z)$, or `None`.

**Steps:**
1. **Compute Child Coordinates**:
   For each of the 8 octants $i \in \{0, \dots, 7\}$:
   - Determine the child's base grid coordinates $(cx, cy, cz)$ by adding `child_scale` to the parent's base coordinates if the corresponding bit in $i$ is set.
   - `child_scale` = $s / 2$.
   - For each of the 8 corners of the child cell, compute the world grid position $(gx, gy, gz)$ by adding `child_scale` to the child cell's base coordinates if the corresponding corner bit is set [S1, unofficial].

2. **Evaluate Field**:
   - For each of the 8 corners of the child cell:
     - Retrieve $F(gx, gy, gz)$ and $\nabla F(gx, gy, gz)$ from `cache`.
     - If the gradient is missing, set a flag `has_grads = false` [S1, unofficial].
     - If not in cache, call `eval_func` and store in `cache`.

3. **Construct Leaf Data**:
   - Create a `LeafData` object containing the 8 corner values and `iso_value`.
   - If `has_grads` is true for all 8 corners, attach the gradients to the leaf. Otherwise, gradients are null [S1, unofficial].

4. **Determine Cell Type**:
   - Check for sign changes among the 8 corner values relative to `iso_value`.
   - If a sign change exists, `cell_type = Leaf` (potential surface intersection).
   - If the corner mask is 0 (all values on one side of iso-value), `cell_type = Empty`.
   - Otherwise, `cell_type = Full` (all values on one side, but potentially complex internal structure requiring further subdivision or specific handling) [S1, unofficial].

5. **Update Tree Structure**:
   - Compute the path to the `parent_cell` in the octree.
   - Navigate to the cell. If it is already a `Branch`, return immediately to avoid orphaning existing children [S1, unofficial].
   - Create a new `Branch` node.
   - Push the new array of 8 child cells to the octree's children vector.
   - Update the `parent_cell` pointer to point to this new `Branch` index.
   - **Critical Implementation Detail**: The new children array must be pushed to the vector *before* updating the parent pointer to prevent use-after-free bugs caused by vector reallocation invalidating pointers [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def subdivide_octree_node(parent_node, iso_value, eval_func, cache):
    """
    Subdivides a single octree node into 8 children.
    """
    # 1. Check if already a branch
    if isinstance(parent_node, Branch):
        return parent_node  # Avoid orphaning existing children

    # 2. Prepare new branch
    new_children = []
    parent_bbox = parent_node.bbox
    child_size = parent_bbox.size / 2.0
    
    # 3. Iterate over 8 octants
    for octant_idx in range(8):
        child_leaf_data = LeafData()
        has_grads = True
        
        # 4. Evaluate 8 corners for this octant
        for corner_idx in range(8):
            # Compute world position of the corner
            # Base coordinates of the octant + offset for specific corner
            pos = compute_corner_position(parent_bbox, octant_idx, corner_idx, child_size)
            
            # Retrieve or compute value and gradient
            val, grad = cache.get_or_compute(pos, eval_func)
            
            child_leaf_data.corners[corner_idx].value = val
            if grad is None:
                has_grads = False
            else:
                child_leaf_data.corners[corner_idx].gradient = grad
        
        # 5. Determine cell type
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

    # 6. Update tree structure safely
    # Push new children to the vector first to prevent reallocation issues
    octree.children_vector.append(new_children)
    
    # Create new branch pointing to these children
    new_branch = Branch(children=new_children)
    
    # Update parent node to be this branch
    parent_node.replace_with(new_branch)
    
    return new_branch
```

**Parameters:**
- `iso_value`: Typical value 0.0 for signed distance fields.
- `min_depth`, `max_depth`: Depth limits for recursion (not part of single subdivision step, but context for when to stop).
- `child_scale`: Derived as $s/2$.

**Edge Cases and Pitfalls:**
- **Vector Reallocation**: Pushing children to a dynamic array before updating parent pointers is critical to avoid dangling pointers if the vector reallocates memory [S1, unofficial].
- **Missing Gradients**: If gradients are not available for all corners, they must be explicitly marked as null to prevent downstream manifold reconstruction errors [S1, unofficial].
- **Non-Cubic Bounding Boxes**: If the initial bounding box is not cubic, it is expanded to the smallest enclosing cube centered on the original bbox center. This affects the initial coordinate scaling [S1, unofficial].

## Subtopics

### [Octree Subdivision](4029-octree-subdivision.md)
Octree Subdivision splits an octree node into eight children by halving its bounding box along each axis and redistributing associated data elements like surfels. It provides the local geometric refinement step required for adaptive sparse voxel representations. This is the core geometric operation described in the main method.

### [Graph-based Load Balancing for Parallel Subdivision](8380-graph-based-load-balancing-for-parallel-subdivision.md)
This method balances parallel workloads for isosurface extraction by distributing volumetric blocks such that each process holds an equal number of active cells for any isovalue. It differs from spatial-only partitioning by analyzing the topological connectivity of active cells via graph partitioning, ensuring uniform computational load across parallel processes regardless of the specific isosurface threshold. It operates on the results of subdivision to manage parallelism.

### Local Refinement and Manifold Verification
This algorithm solves non-manifold configurations caused by the local application of subdivision rules. It extracts information about the shape of the isosurface $\partial V$ in face-adjacent cells over the faces of compact cells and recursively refines them if complex faces or edges are detected [S2].
**Steps:**
1. For every compact cell $c$, get adjacent cells at the same octree level.
2. For every adjacent cell through a face $f$, extract all leaf cells $S$ that are face-adjacent with $c$.
3. Build a quadtree with faces of cells in $S$ adjacent to $c$. This quadtree contains $F(x,y,z)$ values at vertices and represents the intersection between $V$ and $f$ [S2].
4. Apply complex face and complex edge tests.
5. If a face is complex, divide the adjacent compact cell and add its sub-cells to the compact cells list [S2].
6. Recursively repeat the test until all compact cells having complex faces or edges have been eliminated by subdivision [S2].
Unlike Westermann et al. [19], who limit fixing to neighboring cells with at most one level of difference, this approach has no such limitation and is guided by topological complexity [S2].

## Operating characteristics
- **Time Complexity**: $O(1)$ per node subdivided, assuming constant-time evaluation and cache lookup. However, the total time depends on the number of nodes visited, which is determined by the implicit function's complexity and depth limits.
- **Memory Complexity**: Each subdivision increases the number of nodes by a factor of 8. Memory usage scales with the number of leaf nodes in the octree.
- **Performance**: Performance is heavily dependent on the efficiency of the evaluation cache and the sparsity of the isosurface. Dense regions cause rapid memory growth.
- **Strengths**: Simple, deterministic, and provides the necessary resolution for accurate isosurface extraction.
- **Limitations**: Does not inherently ensure manifold connectivity or balance; requires additional steps (like Local Refinement) to fix topological artifacts.
- **When to use**: When adaptive resolution is needed to capture fine geometric details of an implicit surface.
- **When not to use**: When global load balancing or strict topological manifold guarantees are required without post-processing.

## Background
The method is part of the adaptive octree construction pipeline for isosurface extraction, specifically within the context of Manifold Dual Contouring (DMC). The specific implementation details regarding vector reallocation safety and cache integration are derived from the `isomesh` library [S1, unofficial]. The topological refinement aspect (Local Refinement and Manifold Verification) is described in the work on adaptive surface extraction from anisotropic volumetric data, which addresses non-manifold configurations that can arise from local subdivision [S2].

## Sources
[S1] narnia-ai-mason/isomesh, https://github.com/narnia-ai-mason/isomesh, fetched 2026-10-27 (unofficial)
[S2] Adaptive surface extraction from anisotropic volumetric data: contouring on generalized octrees, https://doi.org/10.1007/s12243-013-0369-4, fetched 2026-10-29
