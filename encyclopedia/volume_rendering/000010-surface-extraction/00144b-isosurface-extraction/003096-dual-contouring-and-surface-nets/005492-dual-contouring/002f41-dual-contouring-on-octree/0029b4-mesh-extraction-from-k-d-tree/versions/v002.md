# Mesh Extraction from K-D Tree
Also called: K-D Tree Contouring, Kdtree::extractMesh

## Overview
Extracts a triangle mesh from a scalar field by traversing a hierarchical K-D tree structure, generating vertex indices, and performing contouring on cells, faces, and edges [S1, unofficial]. It differs from standard marching cubes by operating directly on the hierarchical K-D tree structure, allowing for adaptive resolution and handling of complex topologies through recursive contour edge traversal [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A K-D tree representing a scalar field, a threshold value for isosurface extraction.
- Output: A mesh object containing vertices and triangle indices.

**Data Structures:**
- `Mesh`: Contains vertex positions and triangle indices.
- `GridHolder`: Represents a node in the K-D tree, containing `grid.minCode`, `grid.maxCode`, and references to children.
- `PositionCode`: A structure representing coordinates in the grid, with fields `minCode` and `maxCode` for each dimension.
- `ScalarField`: Provides access to scalar values at specific grid positions via `index()`.

**Algorithm Steps:**
1. Create a new `Mesh` object.
2. Generate vertex indices by traversing the K-D tree and adding vertices from `grid.vertices` to the mesh.
3. Perform contouring on cells starting from the root of the K-D tree.
   - **contourCell(node, threshold):**
     - If `node` is null or `isContouringLeaf(node, threshold)`, return.
     - Create a `FaceKd` object from `node->children[0]` and `node->children[1]`.
     - Call `contourFace(node->planeDir, node->axis(), FaceKd)`.
     - Recursively call `contourCell` on `node->children[0]` and `node->children[1]`.
   - **contourFace(planeDir, axis, FaceKd):**
     - If `FaceKd.nodes[0]` or `FaceKd.nodes[1]` is null, return.
     - If both `FaceKd.nodes[0]` and `FaceKd.nodes[1]` are `isContouringLeaf`, return.
     - Check if the face is minimal using `checkMinialFace`. If not minimal, return.
     - Traverse down children until `isContouringLeaf` or `planeDir != dir`.
     - For each node, if not a leaf, recursively call `contourFace` on its children.
     - If the node's axis is within face bounds, construct a line and call `contourEdge`.
   - **checkSign(nodes, quadDir1, quadDir2, s, side, minEnd, maxEnd):**
     - Determine the direction `dir = 3 - quadDir1 - quadDir2`.
     - If `nodes[0] != nodes[1]`, set `maxEnd = minEnd = nodes[0]->grid.maxCode`.
     - Else, set `maxEnd = minEnd = nodes[3]->grid.minCode`.
     - Update `maxEnd[dir]` to the minimum of the maximum codes of all four nodes in the `dir` dimension.
     - Update `minEnd[dir]` to the maximum of the minimum codes of all four nodes in the `dir` dimension.
     - If `minEnd[dir] >= maxEnd[dir]`, return false.
     - Evaluate scalar values `v1 = s->index(minEnd)` and `v2 = s->index(maxEnd)`.
     - If `v1` and `v2` have the same sign (both >= 0 or both < 0), return false.
     - Set `side = 0` if `v2 >= 0 && v1 <= 0`, else `side = 1`.
     - Return true.
   - **generateQuad(nodes, quadDir1, quadDir2, mesh, t, threshold):**
     - Call `checkSign` to determine `edgeSide`, `minEnd`, and `maxEnd`.
     - If `checkSign` returns true, generate a quad using the determined endpoints and add it to the mesh.

**Parameters:**
- `threshold`: The scalar value at which the isosurface is extracted. Typical values depend on the scalar field's range.

**Edge Cases and Pitfalls:**
- Null nodes in the K-D tree traversal must be handled to avoid segmentation faults.
- Precision issues may arise when comparing scalar values for sign changes; ensure robust comparison logic.
- Degenerate cases where `minEnd[dir] >= maxEnd[dir]` indicate no intersection and should be skipped.

## Subtopics

### [Discrete k-d tree isosurface extraction](4964-discrete-k-d-tree-isosurface-extraction.md)
Extracts an isosurface from a scalar field by traversing a discrete k-d tree hierarchy along contour edges, refining nodes until contouring leaves are found, then generating quads. It differs from standard marching cubes by operating directly on the hierarchical k-d tree structure, allowing for adaptive resolution and handling of complex topologies through recursive contour edge traversal.

## Operating characteristics
- **Time Complexity:** Depends on the depth and structure of the K-D tree. In the worst case, it may traverse all nodes, leading to $O(N)$ complexity where $N$ is the number of nodes. However, adaptive traversal can reduce this significantly for sparse fields.
- **Memory Complexity:** Proportional to the size of the K-D tree and the resulting mesh. The tree structure requires $O(N)$ memory, and the mesh size depends on the surface complexity.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales well with adaptive resolution, as it only processes relevant parts of the K-D tree.
- **Strengths:** Adaptive resolution, efficient handling of complex topologies, direct operation on hierarchical structures.
- **Limitations:** Potential for high memory usage with deep trees, sensitivity to threshold selection.
- **When to Use:** Suitable for extracting isosurfaces from scalar fields with varying density or complexity, where adaptive resolution is beneficial.
- **When Not to Use:** Not ideal for uniform scalar fields where simpler methods like marching cubes might be more efficient.

## Background
Originated from the implementation in the `KdtreeISO` repository by Danielmelody [S1, unofficial]. The method leverages the hierarchical nature of K-D trees to efficiently traverse and contour scalar fields, offering an alternative to traditional grid-based surface extraction techniques.

## Sources
[S1] Danielmelody/KdtreeISO, https://github.com/Danielmelody/KdtreeISO, fetched 2026-10-27 (unofficial)
