# Mesh Extraction from K-D Tree

Also called: K-D Tree Contouring, Kdtree::extractMesh

## Overview
This method extracts a triangle mesh from a K-D tree by traversing the tree structure, generating vertex indices, and performing contouring operations on cells, faces, and edges [S1, unofficial]. It sets itself apart from its siblings by operating directly on the hierarchical K-D tree decomposition rather than requiring a structured octree or uniform grid, allowing for efficient spatial partitioning of the scalar field [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A K-D tree representing the spatial partitioning of the volume, a scalar field $S$ defined on the grid vertices, and a threshold value $T$.
*   **Output:** A `Mesh` object containing vertices and triangle indices.

**Data Structures:**
*   **GridHolder:** Represents a node in the K-D tree. Contains `grid` (bounding box information), `children` (array of two pointers), `planeDir` (normal of the splitting plane), and `axis` (dimension of the split).
*   **PositionCode:** A structure representing coordinates in the grid, used to index into the scalar field.
*   **Mesh:** Standard triangle mesh structure with vertex and index buffers.

**Algorithm Steps:**

1.  **Initialization:** Create a new `Mesh` object.
2.  **Vertex Generation:** Traverse the K-D tree. For each leaf node, add the vertices from `grid.vertices` to the mesh. Assign unique indices to these vertices [S1, unofficial].
3.  **Cell Contouring:** Call `contourCell` starting from the root node of the K-D tree.
    *   **`contourCell(node)`:**
        1.  If `node` is null or `isContouringLeaf(node, threshold)` is true, return.
        2.  Create a `FaceKd` object from `node->children[0]` and `node->children[1]`. This represents the interface between the two child subtrees.
        3.  Call `contourFace` with `node->planeDir` and `node->axis()`.
        4.  Recursively call `contourCell` on `node->children[0]` and `node->children[1]` [S1, unofficial].

4.  **Face Contouring:**
    *   **`contourFace(nodes, planeDir, dir)`:**
        1.  If `nodes[0]` or `nodes[1]` is null, return.
        2.  If both `nodes[0]` and `nodes[1]` are `isContouringLeaf`, return.
        3.  Check if the face is minimal using `checkMinialFace`. If not minimal, return.
        4.  Traverse down the children of `nodes[0]` and `nodes[1]` until a node is a leaf or its axis does not match `dir`.
        5.  For each node encountered during traversal:
            *   If the node is not a leaf, recursively call `contourFace` on its children.
            *   If the node's axis is within the face bounds, construct a line segment and call `contourEdge` [S1, unofficial].

5.  **Edge Contouring and Quad Generation:**
    *   **`checkSign(nodes, quadDir1, quadDir2, s, side, minEnd, maxEnd)`:**
        1.  Determine the edge direction: $dir = 3 - quadDir1 - quadDir2$.
        2.  If `nodes[0] != nodes[1]`:
            *   Set `minEnd` and `maxEnd` to `nodes[0]->grid.maxCode`.
        3.  Else:
            *   Set `minEnd` and `maxEnd` to `nodes[3]->grid.minCode`.
        4.  Update `maxEnd[dir]` to the minimum of the max codes of all four nodes in the direction $dir$:
            $$maxEnd[dir] = \min(\min(nodes[0]->grid.maxCode[dir], nodes[1]->grid.maxCode[dir]), \min(nodes[2]->grid.maxCode[dir], nodes[3]->grid.maxCode[dir]))$$
        5.  Update `minEnd[dir]` to the maximum of the min codes of all four nodes in the direction $dir$:
            $$minEnd[dir] = \max(\max(nodes[0]->grid.minCode[dir], nodes[1]->grid.minCode[dir]), \max(nodes[2]->grid.minCode[dir], nodes[3]->grid.minCode[dir]))$$
        6.  If $minEnd[dir] \ge maxEnd[dir]$, return false (no intersection).
        7.  Sample the scalar field at the endpoints:
            $$v_1 = S(minEnd)$$
            $$v_2 = S(maxEnd)$$
        8.  If $v_1$ and $v_2$ have the same sign (both $\ge 0$ or both $< 0$), return false.
        9.  Set `side`: if $v_2 \ge 0$ and $v_1 \le 0$, `side = 0`; else `side = 1`.
        10. Return true [S1, unofficial].

    *   **`generateQuad(nodes, quadDir1, quadDir2, mesh, t, threshold)`:**
        1.  Call `checkSign` to determine if the quad intersects the isosurface.
        2.  If intersection exists, generate the quad geometry and add it to `mesh` [S1, unofficial].

**Parameters:**
*   `threshold`: The scalar value defining the isosurface. Typical values depend on the scalar field normalization [S1, unofficial].
*   `unitSize`: A static float variable in `RectilinearGrid`, likely representing the voxel size or scaling factor [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Null Nodes:** The algorithm explicitly checks for null nodes in `contourCell` and `contourFace` to prevent dereferencing errors [S1, unofficial].
*   **Degenerate Faces:** The `checkMinialFace` check ensures that only valid, non-degenerate faces are processed [S1, unofficial].
*   **Precision:** The sign check relies on floating-point comparison. If $v_1$ or $v_2$ is exactly zero, the behavior depends on the implementation of the sign check (e.g., $v_1 \le 0$ vs $v_1 < 0$) [S1, unofficial].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Efficiently handles hierarchical data structures without requiring a full octree build.
*   **Limitations:** The algorithm's performance depends heavily on the balance of the K-D tree and the distribution of the scalar field [S1, unofficial].

## Background
This method is implemented in the `KdtreeISO` repository by Danielmelody [S1, unofficial]. It is an unofficial implementation of isosurface extraction using K-D trees.

## Sources
[S1] Danielmelody/KdtreeISO, https://github.com/Danielmelody/KdtreeISO, fetched 2026-10-27 (unofficial)
