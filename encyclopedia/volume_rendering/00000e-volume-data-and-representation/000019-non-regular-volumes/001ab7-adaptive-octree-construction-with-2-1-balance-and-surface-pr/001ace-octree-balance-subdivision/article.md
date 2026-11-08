# Octree Balance Subdivision


## Overview
Octree Balance Subdivision is the recursive refinement step in adaptive octree construction for volume rendering and isosurface extraction [S1, unofficial]. It splits a parent node into eight child octants, evaluates the implicit function at the children's corners, and classifies each child as Empty, Full, or Leaf based on sign changes relative to an isosurface [S1, unofficial]. It ensures tree integrity by preventing orphaning of existing children and maintaining the 2:1 balance constraint required by the parent algorithm [Adaptive Octree Construction with 2:1 Balance and Surface Propagation](830-adaptive-octree-construction-with-2-1-balance-and-surface-pr.md) [S1, unofficial].

## How it works
**Inputs:**
- `parent_grid_coords`: $(gx, gy, gz)$, integer coordinates of the parent cell in the global grid [S1, unofficial].
- `child_scale`: The side length of a child cell in grid units, typically half the parent's side length [S1, unofficial].
- `iso_value`: The scalar threshold for sign change detection [S1, unofficial].
- `eval_cache`: A lookup structure mapping grid corner coordinates to $(value, gradient)$ pairs [S1, unofficial].
- `octree_node`: Reference to the current octree node being subdivided [S1, unofficial].

**Outputs:**
- Updated `octree_node`: Now a Branch with 8 children [S1, unofficial].
- Classification of each child node as Empty, Full, or Leaf [S1, unofficial].

**Data Structures:**
- `LeafData`: Contains `corner_values` (array of 8 floats), `iso_value` (float), and optionally `corner_gradients` (array of 8 vectors) [S1, unofficial].
- `CellType`: Enum with values `Empty`, `Full`, `Leaf` [S1, unofficial].

**Steps:**

1. **Compute Child Grid Coordinates:** For each of the 8 octants $i \in \{0, \dots, 7\}$, extract bits $b_x, b_y, b_z$. Compute the child's base grid coordinates $(cx, cy, cz)$ by adding `child_scale` to the parent coordinates if the corresponding octant bit is set: $cx = gx + (b_x \cdot s)$, $cy = gy + (b_y \cdot s)$, $cz = gz + (b_z \cdot s)$ [S1, unofficial].
2. **Compute Child Corner Positions:** For each of the 8 corners of the child cell, extract corner bits $c_x, c_y, c_z$. Compute the grid position $(gx_c, gy_c, gz_c)$ by adding `child_scale` to the child cell's base coordinates if the corresponding corner bit is set: $gx_c = cx + (c_x \cdot s)$, $gy_c = cy + (c_y \cdot s)$, $gz_c = cz + (c_z \cdot s)$ [S1, unofficial].
3. **Retrieve Scalar Values and Gradients:** For each corner $(gx_c, gy_c, gz_c)$ of child $i$, retrieve $(v_c, g_c)$ from `eval_cache` [S1, unofficial]. If any gradient $g_c$ is missing, set a flag `has_grads = false` for that child [S1, unofficial].
4. **Construct LeafData:** Create a `LeafData` object for child $i$ containing the 8 corner values $v_c$ and `iso_value` [S1, unofficial]. If `has_grads` is true, attach the 8 corner gradients to the `LeafData` [S1, unofficial].
5. **Determine Cell Type:** Compute a `corner_mask` where bit $k$ is 1 if corner $k$ value $> iso\_value$ [S1, unofficial]. If `corner_mask` is 0, type is `Empty`; if 255, type is `Full`; if there is a sign change (mask is neither 0 nor 255), type is `Leaf` [S1, unofficial].
6. **Navigate and Update Octree:** Compute the path to the current cell in the octree structure [S1, unofficial]. Navigate to the cell and return immediately if it is already a Branch to avoid orphaning existing children [S1, unofficial]. Push the new child cells array to the octree's children vector and update the current cell to be a Branch pointing to this new index [S1, unofficial]. This order prevents use-after-free bugs caused by vector reallocation invalidating pointers [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class OctreeNode:
    def __init__(self):
        self.is_branch = False
        self.children = []
        self.leaf_data = None
        self.cell_type = None

class LeafData:
    def __init__(self, corner_values, iso_value, corner_gradients=None):
        self.corner_values = corner_values
        self.iso_value = iso_value
        self.corner_gradients = corner_gradients

def octree_balance_subdivision(parent_grid_coords, child_scale, iso_value, eval_cache, octree_node):
    gx, gy, gz = parent_grid_coords

    # Navigate to the cell. If it is already a Branch, return immediately to avoid orphaning existing children.
    if octree_node.is_branch:
        return

    children = []

    for i in range(8):
        bx = (i >> 2) & 1
        by = (i >> 1) & 1
        bz = i & 1

        # Compute child grid coordinates by adding child_scale to parent coordinates if octant bit is set
        cx = gx + (bx * child_scale)
        cy = gy + (by * child_scale)
        cz = gz + (bz * child_scale)

        corner_values = []
        corner_gradients = []
        has_grads = True

        for corner_idx in range(8):
            cbx = (corner_idx >> 2) & 1
            cby = (corner_idx >> 1) & 1
            cbz = corner_idx & 1

            # Compute corner grid position by adding child_scale to child base coordinates if corner bit is set
            corner_gx = cx + (cbx * child_scale)
            corner_gy = cy + (cby * child_scale)
            corner_gz = cz + (cbz * child_scale)

            key = (corner_gx, corner_gy, corner_gz)
            if key in eval_cache:
                val, grad = eval_cache[key]
                corner_values.append(val)
                if grad is None:
                    has_grads = False
                else:
                    corner_gradients.append(grad)
            else:
                # [G] Fallback: evaluate implicit function if cache miss occurs
                pass

        # Construct LeafData with corner values and iso_value
        leaf_data = LeafData(corner_values, iso_value)

        # If gradients are available for all corners, attach them to the leaf
        if has_grads:
            leaf_data.corner_gradients = corner_gradients

        # Determine cell type
        corner_mask = 0
        for idx, val in enumerate(corner_values):
            if val > iso_value:
                corner_mask |= (1 << idx)

        if corner_mask == 0:
            cell_type = "Empty"
        elif corner_mask == 255:
            cell_type = "Full"
        else:
            cell_type = "Leaf"

        child_node = OctreeNode()
        child_node.is_branch = False
        child_node.leaf_data = leaf_data
        child_node.cell_type = cell_type
        children.append(child_node)

    # Push the new child cells array to the octree's children vector and update the current cell to be a Branch
    # This order prevents use-after-free bugs caused by vector reallocation invalidating pointers
    octree_node.children.extend(children)
    octree_node.is_branch = True
```

**Parameters:**
- `min_depth`, `max_depth`: Define the range of subdivision [S1, unofficial]. Typical values are `min_depth=3` and `max_depth=7` [S1, unofficial].
- `iso_value`: The isosurface level threshold [S1, unofficial]. Typical value is `0.0` [S1, unofficial].
- `child_scale`: Determined by the current depth level relative to the root [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Vector Reallocation:** Pushing children to a vector before updating the branch pointer is critical to avoid invalidating pointers due to reallocation [S1, unofficial].
- **Missing Gradients:** If gradients are missing for any corner, `has_grads` is set to false and gradients are not attached to the leaf [S1, unofficial].
- **Non-Cubic Bounding Box:** If the bounding box is non-cubic, it is expanded to the smallest enclosing cube centered on the original bounding box center [S1, unofficial].

## Subtopics

### Local Refinement and Manifold Verification
Algorithm to resolve non-manifold configurations caused by local application of subdivision rules [S2]. It extracts information about $\partial V$ shape in face-adjacent cells over faces of compact cells and recursively refines them if complex faces or edges are detected [S2]. Steps: (1) For every compact cell $c$, get adjacent cells at the same octree level [S2]. (2) For every adjacent cell through a face $f$, extract all leaf cells $S$ face-adjacent with $c$ [S2]. (3) Build a quadtree with faces of cells in $S$ adjacent to $c$, containing $F(x,y,z)$ values at vertices to represent the intersection between $V$ and $f$ [S2]. (4) Apply complex face and complex edge tests [S2]. (5) If a face is complex, divide the adjacent compact cell and add its sub-cells to the compact cells list [S2]. (6) Recursively repeat the test until all compact cells having complex faces or edges have been eliminated by subdivision [S2]. Unlike Westermann et al. [19], which limits fixing to neighboring cells with at most one level of difference, this approach has no such limitation and is guided purely by topological complexity [S2].

## Operating characteristics
- **Time Complexity:** $O(1)$ per node subdivision, with total time proportional to the number of visited nodes [G].
- **Memory Complexity:** $O(N)$ where $N$ is the number of leaf nodes, each storing 8 values and optionally 8 gradients [G].
- **Strengths:** Efficiently adapts resolution to geometric features, reducing memory usage compared to uniform grids [G].
- **Limitations:** Requires careful memory management to avoid use-after-free bugs, and performance degrades if the isosurface is highly complex or if the depth range is too large [G].

## Background
The method is part of the `isomesh` library, an implementation of fast isosurface extraction via adaptive octree and Manifold Dual Contouring [S1, unofficial]. The algorithm builds on the principles of adaptive octree construction with 2:1 balance and surface propagation [Adaptive Octree Construction with 2:1 Balance and Surface Propagation](830-adaptive-octree-construction-with-2-1-balance-and-surface-pr.md) [S1, unofficial].

## Sources
[S1] narnia-ai-mason/isomesh, https://github.com/narnia-ai-mason/isomesh, fetched 2026-10-27 (unofficial)
[S2] Adaptive surface extraction from anisotropic volumetric data: contouring on generalized octrees, https://doi.org/10.1007/s12243-013-0369-4, fetched 2026-10-29
