# Mesh Extraction from K-D Tree
Also called: K-D Tree Contouring, Kdtree::extractMesh

## Overview
Extracts a triangle mesh from a K-D tree representation of a scalar field by traversing the tree hierarchy, generating vertex indices, and performing contouring on cells, faces, and edges [S1, unofficial]. It differs from its parent [Dual Contouring on Octree](4469-dual-contouring-on-octree.md) by using a K-D tree structure instead of an octree, allowing for adaptive spatial partitioning based on data distribution rather than fixed recursive subdivision. It is distinct from [Isomesh Octree Build and Clustering](4984-isomesh-octree-build-and-clustering.md) and [Octree Construction for Dual Contouring](5913-octree-construction-for-dual-contouring.md) by its specific traversal logic for contouring along tree nodes and its reliance on `checkSign` and `generateQuad` for mesh generation.

## How it works
**Inputs:** A K-D tree representing a scalar field, a threshold value for isosurface extraction, and a mesh object to store results.
**Outputs:** A triangle mesh containing vertices and indices.

**Data Structures:**
*   `Mesh`: Stores vertices and triangle indices.
*   `GridHolder`: Represents a node in the K-D tree, containing `grid.minCode`, `grid.maxCode`, and references to children.
*   `PositionCode`: A structure representing coordinates, with components accessed via indices (e.g., `minEnd[dir]`).
*   `ScalarField`: Provides `index(pos)` to retrieve scalar values at specific positions.

**Algorithm Steps:**

1.  **Initialize Mesh:** Create a new `Mesh` object.
2.  **Generate Vertices:** Traverse the K-D tree and add unique vertices from `grid.vertices` to the mesh.
3.  **Contour Cells:** Call `contourCell` starting from the root node.
    *   **`contourCell(node, threshold)`:**
        1.  If `node` is null or `isContouringLeaf(node, threshold)`, return.
        2.  Create a `FaceKd` object from `node->children[0]` and `node->children[1]`.
        3.  Call `contourFace` with `node->planeDir` and `node->axis()`.
        4.  Recursively call `contourCell` on `node->children[0]` and `node->children[1]`.
    *   **`contourFace(nodes, planeDir, axis)`:**
        1.  If `nodes[0]` or `nodes[1]` is null, return.
        2.  If both `nodes[0]` and `nodes[1]` are `isContouringLeaf`, return.
        3.  Check if the face is minimal using `checkMinialFace`. If not minimal, return.
        4.  Traverse down children until `isContouringLeaf` or `planeDir != dir`.
        5.  For each node, if not a leaf, recursively call `contourFace` on its children.
        6.  If the node's axis is within face bounds, construct a line and call `contourEdge`.
    *   **`checkSign(nodes, quadDir1, quadDir2, s, side, minEnd, maxEnd)`:**
        1.  Determine `dir = 3 - quadDir1 - quadDir2`.
        2.  If `nodes[0] != nodes[1]`:
            *   `maxEnd = minEnd = nodes[0]->grid.maxCode`.
        3.  Else:
            *   `maxEnd = minEnd = nodes[3]->grid.minCode`.
        4.  Update `maxEnd[dir]`:
            $$ \text{maxEnd}[dir] = \min(\min(\text{nodes}[0]->\text{grid}.\text{maxCode}[dir], \text{nodes}[1]->\text{grid}.\text{maxCode}[dir]), \min(\text{nodes}[2]->\text{grid}.\text{maxCode}[dir], \text{nodes}[3]->\text{grid}.\text{maxCode}[dir])) $$
        5.  Update `minEnd[dir]`:
            $$ \text{minEnd}[dir] = \max(\max(\text{nodes}[0]->\text{grid}.\text{minCode}[dir], \text{nodes}[1]->\text{grid}.\text{minCode}[dir]), \max(\text{nodes}[2]->\text{grid}.\text{minCode}[dir], \text{nodes}[3]->\text{grid}.\text{minCode}[dir])) $$
        6.  If `minEnd[dir] >= maxEnd[dir]`, return `false`.
        7.  Retrieve scalar values: $v_1 = s->\text{index}(\text{minEnd})$, $v_2 = s->\text{index}(\text{maxEnd})$.
        8.  If $(v_1 \ge 0 \land v_2 \ge 0) \lor (v_1 < 0 \land v_2 < 0)$, return `false`.
        9.  If $v_2 \ge 0$ and $v_1 \le 0$, set `side = 0`; else `side = 1`.
        10. Return `true`.
    *   **`generateQuad(nodes, quadDir1, quadDir2, mesh, t, threshold)`:**
        1.  Call `checkSign` to determine `side`, `minEnd`, and `maxEnd`.
        2.  If `checkSign` returns `false`, return.
        3.  Generate quad geometry using `minEnd`, `maxEnd`, `side`, and `threshold` (implementation details for quad generation are not fully specified in the provided notes, but it uses the determined endpoints and side to create mesh primitives) [S1, unofficial].

**Parameters:**
*   `threshold`: The scalar value for isosurface extraction. Typical values depend on the scalar field range.
*   `quadDir1`, `quadDir2`: Indices defining the orientation of the quad face.

**Edge Cases and Pitfalls:**
*   **Null Nodes:** The algorithm checks for null nodes in `contourFace` and `contourCell` to prevent dereferencing errors.
*   **Non-Minimal Faces:** `checkMinialFace` ensures that only relevant faces are processed, avoiding redundant or invalid geometry.
*   **Precision:** Scalar value comparisons ($v_1 \ge 0$, etc.) are sensitive to floating-point precision. [G] Consider using a small epsilon for comparisons if the scalar field values are close to zero.

## Subtopics

## Operating characteristics
*   **Time Complexity:** Depends on the depth and balance of the K-D tree. Traversal is recursive, and contouring operations are performed on nodes that intersect the isosurface.
*   **Memory Complexity:** Proportional to the number of vertices and triangles in the resulting mesh, plus the memory for the K-D tree structure.
*   **Strengths:** Adaptive resolution based on data distribution; efficient for sparse or unevenly distributed scalar fields.
*   **Limitations:** Performance may degrade if the K-D tree is unbalanced or if the isosurface intersects many nodes.
*   **When to Use:** Suitable for extracting isosurfaces from scalar fields represented by K-D trees, especially when the data distribution is not uniform.
*   **When Not to Use:** Not ideal for regular grid data where octrees or simpler marching cubes algorithms might be more efficient.

## Background
Originated from the implementation in the `KdtreeISO` repository by Danielmelody [S1, unofficial]. The method leverages the hierarchical structure of K-D trees to efficiently locate and extract isosurfaces, adapting the contouring process to the tree's nodes.

## Sources
[S1] Danielmelody/KdtreeISO, https://github.com/Danielmelody/KdtreeISO, fetched 2026-10-27 (unofficial)
