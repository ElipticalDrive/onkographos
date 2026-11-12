# Mesh Extraction from K-D Tree

Also called: K-D Tree Contouring, Kdtree::extractMesh

## Overview
Extracts a triangle mesh from a K-D tree by traversing the tree structure, generating vertex indices, and performing contouring on cells, faces, and edges [S1, unofficial]. Unlike octrees which group cells by spatial position, this method groups cells by value intervals, allowing for faster determination of active cells at the cost of higher memory usage [S2, unofficial]. It builds on [Dual Contouring on Octree](4469-dual-contouring-on-octree.md) by adapting the recursive traversal and contouring logic to a K-D tree hierarchy.

## How it works
**Inputs:** A K-D tree representing a scalar field, a threshold value for isosurface extraction, and a grid of vertices.
**Outputs:** A mesh object containing vertices and triangle indices.

**Data Structures:**
*   **Mesh:** Stores vertex positions and triangle indices.
*   **GridHolder:** Represents a node in the K-D tree. Contains `grid.minCode` and `grid.maxCode` (PositionCode arrays defining the bounding box of the node in grid coordinates), `children` (array of two GridHolder pointers), `planeDir` (direction of the splitting plane), and `axis` (axis along which the split occurs).
*   **PositionCode:** An array or structure representing coordinates in the grid space.
*   **ScalarField:** Provides access to scalar values at grid positions via `index(pos)`.

**Algorithm Steps:**

1.  **Initialization:** Create a new `Mesh` object.
2.  **Vertex Generation:** Traverse the K-D tree to collect all unique vertices from `grid.vertices` and add them to the mesh. [S1, unofficial]
3.  **Recursive Contouring:** Call `contourCell` starting from the root node.

    **`contourCell(node, threshold)`:**
    1.  If `node` is null, return.
    2.  If `isContouringLeaf(node, threshold)` is true, return. [G] (This function likely checks if the node's value range crosses the threshold or if it is a leaf node suitable for direct contouring).
    3.  Create a `FaceKd` object from `node->children[0]` and `node->children[1]`. [G] (This represents the interface between the two children).
    4.  Call `contourFace(node->planeDir, node->axis(), FaceKd)`.
    5.  Recursively call `contourCell(node->children[0], threshold)`.
    6.  Recursively call `contourCell(node->children[1], threshold)`.

    **`contourFace(planeDir, axis, faceNodes)`:**
    1.  Let `nodes[0]` and `nodes[1]` be the two nodes defining the face.
    2.  If `nodes[0]` or `nodes[1]` is null, return.
    3.  If both `nodes[0]` and `nodes[1]` are `isContouringLeaf`, return.
    4.  Check if the face is minimal using `checkMinimalFace(nodes[0], nodes[1])`. [G] (If not minimal, return).
    5.  Traverse down the children of `nodes[0]` and `nodes[1]` until `isContouringLeaf` is true or the node's `axis` differs from `planeDir`.
    6.  For each node in the traversal:
        a.  If the node is not a leaf, recursively call `contourFace` on its children.
        b.  If the node's `axis` is within the face bounds, construct a line segment along the intersection and call `contourEdge`.

    **`checkSign(nodes, quadDir1, quadDir2, scalarField, side, minEnd, maxEnd)`:**
    This function determines if a quad defined by `quadDir1` and `quadDir2` intersects the isosurface.
    1.  Calculate `dir = 3 - quadDir1 - quadDir2`.
    2.  Determine `minEnd` and `maxEnd` coordinates for the edge along `dir`:
        *   If `nodes[0] != nodes[1]`:
            `minEnd = maxEnd = nodes[0]->grid.maxCode`
        *   Else:
            `minEnd = maxEnd = nodes[3]->grid.minCode`
        *   Update `maxEnd[dir]`:
            $$maxEnd[dir] = \min(\min(nodes[0]->grid.maxCode[dir], nodes[1]->grid.maxCode[dir]), \min(nodes[2]->grid.maxCode[dir], nodes[3]->grid.maxCode[dir]))$$
        *   Update `minEnd[dir]`:
            $$minEnd[dir] = \max(\max(nodes[0]->grid.minCode[dir], nodes[1]->grid.minCode[dir]), \max(nodes[2]->grid.minCode[dir], nodes[3]->grid.minCode[dir]))$$
    3.  If `minEnd[dir] >= maxEnd[dir]`, return `false`.
    4.  Get scalar values: $v_1 = s->index(minEnd)$, $v_2 = s->index(maxEnd)$.
    5.  If $(v_1 \ge 0 \land v_2 \ge 0) \lor (v_1 < 0 \land v_2 < 0)$, return `false`.
    6.  Set `side`: If $v_2 \ge 0$ and $v_1 \le 0$, `side = 0`; else `side = 1`.
    7.  Return `true`.

    **`generateQuad(nodes, quadDir1, quadDir2, mesh, scalarField, threshold)`:**
    1.  Call `checkSign` to determine `side`, `minEnd`, and `maxEnd`.
    2.  If `checkSign` returns `false`, return.
    3.  Generate the quad geometry using the determined endpoints and add to `mesh`. [G] (Specific vertex generation logic is not fully detailed in the provided snippets, but it uses the scalar field and threshold to interpolate or position vertices).

**Parameters:**
*   `threshold`: The scalar value defining the isosurface. Typical values depend on the scalar field range.
*   `unitSize`: A static float member in `RectilinearGrid`, likely representing the physical size of a grid cell. [S1, unofficial]

**Edge Cases and Pitfalls:**
*   **Null Nodes:** The algorithm explicitly checks for null nodes in `contourFace` and `contourCell` to prevent dereferencing errors.
*   **Non-Minimal Faces:** The `checkMinimalFace` function (implementation not provided) is crucial for avoiding redundant or invalid contouring on large faces. [G]
*   **Precision:** The comparison `minEnd[dir] >= maxEnd[dir]` relies on integer grid coordinates, avoiding floating-point precision issues in boundary checks. However, scalar value comparisons ($v_1, v_2$) are subject to floating-point precision.

## Subtopics

### [Discrete k-d tree isosurface extraction](4964-discrete-k-d-tree-isosurface-extraction.md)
Extracts an isosurface by traversing a discrete k-d tree hierarchy along contour edges, refining nodes until contouring leaves are found, and generating quads. Groups cells by value intervals for faster active cell determination at the cost of higher memory usage.

## Operating characteristics
*   **Time Complexity:** Depends on the tree depth and the number of active cells. Traversal is $O(N)$ where $N$ is the number of nodes, but contouring is only performed on active cells.
*   **Memory Complexity:** Higher than octree-based methods due to the grouping of cells by value intervals, which may require storing more metadata or larger node structures. [S2, unofficial]
*   **Strengths:** Faster determination of active cells compared to spatial grouping (octrees).
*   **Limitations:** Higher memory usage.
*   **When to use:** When the scalar field has distinct value intervals that can be efficiently grouped, and memory is not the primary constraint.
*   **When not to use:** When memory usage is critical or the scalar field does not exhibit clear value-based clustering.

## Background
This method is implemented in the open-source project `KdtreeISO` by Danielmelody [S1, unofficial]. It is an adaptation of dual contouring techniques to K-D trees, leveraging the tree's structure for efficient isosurface extraction.

## Sources
[S1] Danielmelody/KdtreeISO, https://github.com/Danielmelody/KdtreeISO, fetched 2026-10-27 (unofficial)
[S2] Discrete k-d tree isosurface extraction, https://github.com/Danielmelody/KdtreeISO/blob/master/README.md (inferred from context), fetched 2026-10-27 [S2, unofficial]
