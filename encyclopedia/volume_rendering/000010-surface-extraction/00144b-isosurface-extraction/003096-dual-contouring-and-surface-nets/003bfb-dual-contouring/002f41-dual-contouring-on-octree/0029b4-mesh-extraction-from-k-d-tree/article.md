# Mesh Extraction from K-D Tree
Also called: K-D Tree Contouring, Kdtree::extractMesh

## Overview
Extracts a triangle mesh from a scalar field by traversing a hierarchical K-D tree structure, generating vertex indices, and performing contouring on cells, faces, and edges [S1, unofficial]. It differs from standard marching cubes by operating directly on the hierarchical K-D tree structure, allowing for adaptive resolution and handling of complex topologies through recursive contour edge traversal [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A K-D tree representing a scalar field, a threshold value for isosurface extraction [S1, unofficial].
- Output: A mesh object containing vertices and triangle indices [S1, unofficial].

**Data Structures:**
- `Mesh`: Contains vertex positions and triangle indices [S1, unofficial].
- `GridHolder`: Represents a node in the K-D tree, containing `grid.minCode`, `grid.maxCode`, and references to children [S1, unofficial].
- `PositionCode`: A structure representing coordinates in the grid, with fields `minCode` and `maxCode` for each dimension [S1, unofficial].
- `ScalarField`: Provides access to scalar values at specific grid positions via `index()` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_mesh(kd_tree, threshold):
    mesh = Mesh()
    # Generate vertex indices by traversing the tree and adding vertices from grid.vertices to mesh [S1, unofficial]
    for node in traverse(kd_tree):
        mesh.add_vertices(node.grid.vertices)
    contour_cell(kd_tree, threshold, mesh)
    return mesh

def contour_cell(node, threshold, mesh):
    if node is None or is_contouring_leaf(node, threshold):
        return [S1, unofficial]
    face_kd = FaceKd(node.children[0], node.children[1]) [S1, unofficial]
    contour_face(node.plane_dir, node.axis(), face_kd, mesh) [S1, unofficial]
    contour_cell(node.children[0], threshold, mesh) [S1, unofficial]
    contour_cell(node.children[1], threshold, mesh) [S1, unofficial]

def contour_face(plane_dir, axis, face_kd, mesh):
    if face_kd.nodes[0] is None or face_kd.nodes[1] is None:
        return [S1, unofficial]
    if is_contouring_leaf(face_kd.nodes[0]) and is_contouring_leaf(face_kd.nodes[1]):
        return [S1, unofficial]
    if not check_minimal_face(face_kd):
        return [S1, unofficial]
    # Traverse down children until is_contouring_leaf or plane_dir != dir [S1, unofficial]
    for node in traverse_down(face_kd):
        if not is_leaf(node):
            contour_face(plane_dir, node.axis(), FaceKd(node.children[0], node.children[1]), mesh) [S1, unofficial]
        if node.axis_within_face_bounds():
            line = construct_line(node)
            contour_edge(line, mesh) [S1, unofficial]

def check_sign(nodes, quad_dir1, quad_dir2, scalar_field, side, min_end, max_end):
    dir = 3 - quad_dir1 - quad_dir2 [S1, unofficial]
    if nodes[0] != nodes[1]:
        max_end = min_end = nodes[0].grid.max_code [S1, unofficial]
    else:
        max_end = min_end = nodes[3].grid.min_code [S1, unofficial]
    max_end[dir] = min(nodes[0].grid.max_code[dir], nodes[1].grid.max_code[dir],
                       nodes[2].grid.max_code[dir], nodes[3].grid.max_code[dir]) [S1, unofficial]
    min_end[dir] = max(nodes[0].grid.min_code[dir], nodes[1].grid.min_code[dir],
                       nodes[2].grid.min_code[dir], nodes[3].grid.min_code[dir]) [S1, unofficial]
    if min_end[dir] >= max_end[dir]:
        return False [S1, unofficial]
    v1 = scalar_field.index(min_end)
    v2 = scalar_field.index(max_end) [S1, unofficial]
    if (v1 >= 0 and v2 >= 0) or (v1 < 0 and v2 < 0):
        return False [S1, unofficial]
    side[0] = 0 if v2 >= 0 and v1 <= 0 else 1 [S1, unofficial]
    return True [S1, unofficial]

def generate_quad(nodes, quad_dir1, quad_dir2, mesh, scalar_field, threshold):
    side = [0]
    min_end = PositionCode()
    max_end = PositionCode()
    if check_sign(nodes, quad_dir1, quad_dir2, scalar_field, side, min_end, max_end): [S1, unofficial]
        mesh.add_quad(min_end, max_end) [S1, unofficial]
```

**Parameters:**
- `threshold`: The scalar value at which the isosurface is extracted. Typical values depend on the scalar field's range [S1, unofficial].

**Edge Cases and Pitfalls:**
- Null nodes in the K-D tree traversal must be handled to avoid segmentation faults [S1, unofficial].
- Precision issues may arise when comparing scalar values for sign changes; ensure robust comparison logic [S1, unofficial].
- Degenerate cases where `minEnd[dir] >= maxEnd[dir]` indicate no intersection and should be skipped [S1, unofficial].

## Subtopics

### [Discrete k-d tree isosurface extraction](4964-discrete-k-d-tree-isosurface-extraction.md)
Extracts an isosurface from a scalar field by traversing a discrete k-d tree hierarchy along contour edges, refining nodes until contouring leaves are found, then generating quads. It differs from standard marching cubes by operating directly on the hierarchical k-d tree structure, allowing for adaptive resolution and handling of complex topologies through recursive contour edge traversal.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Adaptive resolution, efficient handling of complex topologies, direct operation on hierarchical structures [S1, unofficial].
- **Limitations:** Not given in the sources.
- **When to Use:** Not given in the sources.
- **When Not to Use:** Not given in the sources.

## Background
The method is implemented in the `KdtreeISO` repository by Danielmelody [S1, unofficial]. It leverages the hierarchical nature of K-D trees to efficiently traverse and contour scalar fields, offering an alternative to traditional grid-based surface extraction techniques [S1, unofficial].

## Sources
[S1] Danielmelody/KdtreeISO, https://github.com/Danielmelody/KdtreeISO, fetched 2026-10-27 (unofficial)
