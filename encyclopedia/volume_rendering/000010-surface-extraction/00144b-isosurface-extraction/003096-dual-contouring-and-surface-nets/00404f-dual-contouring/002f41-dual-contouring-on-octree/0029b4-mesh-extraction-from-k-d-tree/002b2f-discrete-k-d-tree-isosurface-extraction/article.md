# Discrete k-d tree isosurface extraction

Also called: KD-tree contour edge traversal

## Overview
Extracts an isosurface from a scalar field by traversing a discrete k-d tree hierarchy along contour edges, refining nodes until contouring leaves are found, then generating quads [S1, unofficial]. It differs from standard marching cubes by operating directly on the hierarchical k-d tree structure, allowing for adaptive resolution and handling of complex topologies through recursive contour edge traversal [S1, unofficial].

## How it works
**Inputs:**
- A discrete k-d tree representing a scalar field [S1, unofficial].
- A threshold value defining the isosurface level [S1, unofficial].
- An error threshold for precision checks [S1, unofficial].

**Outputs:**
- A triangle mesh representing the isosurface [S1, unofficial].

**Data Structures:**
- `Kdtree::EdgeKd`: An array of 4 `Kdtree::Node` pointers, indexed 0..3. Accessed as `nodes[i*2]` and `nodes[i*2+1]` for pairs [S1, unofficial].
- `AALine`: A struct containing a `point` (of type `PositionCode`) and a `dir` (integer) [S1, unofficial].
- `PositionCode`: A vector-like type with integer components accessible via `[dir]`, representing grid coordinates [S1, unofficial].
- `Vertex`: Represents a point in space, often with Hermite interpolation data (`hermiteP`) [S1, unofficial].

**Parameters:**
- `threshold` (float): Scalar value defining the isosurface level, used in `isContouringLeaf` checks [S1, unofficial].
- `error` (float): Command-line argument for error threshold, defaults to $10^{-7}$ [S1, unofficial].

**Steps:**
1. Initialize an `EdgeKd` array of 4 nodes [S1, unofficial].
2. Call `detectQuad` to traverse down the tree for each pair until nodes are contouring leaves or hit boundaries [S1, unofficial].
3. Compute `quadDir2 = 3 - quadDir1 - line.dir` [S1, unofficial].
4. Check minimal edge validity by intersecting min/max codes of the 4 nodes along `line.dir`; if `minEnd[dir] >= maxEnd[dir]`, the edge is degenerate and the quad is skipped [S1, unofficial].
5. Traverse each node while not a contouring leaf and `planeDir != line.dir`, updating via `nextQuadIndex`; if a node becomes null, return immediately to prevent segfaults [S1, unofficial].
6. If all 4 nodes are contouring leaves, call `generateQuad` [S1, unofficial].
7. Otherwise, for each node that is not a contouring leaf and `planeDir == line.dir`, split into two recursive `contourEdge` calls with `children[0]` and `children[1]` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class PositionCode:
    def __init__(self, x, y, z):
        self.x, self.y, self.z = x, y, z
    def __getitem__(self, dir):
        return (self.x, self.y, self.z)[dir]

class AALine:
    def __init__(self, point, dir):
        self.point = point
        self.dir = dir

class Node:
    def __init__(self, is_leaf, value, children=None, min_code=None, max_code=None):
        self.is_leaf = is_leaf
        self.value = value
        self.children = children if children else [None, None]
        self.min_code = min_code
        self.max_code = max_code

class Vertex:
    def __init__(self, pos):
        self.pos = pos
        self.hermiteP = pos

class Mesh:
    def __init__(self):
        self.vertices = []
        self.triangles = []
    def add_vertex(self, v):
        self.vertices.append(v)
    def add_triangle(self, v0, v1, v2):
        self.triangles.append((v0, v1, v2))

def is_contouring_leaf(node, threshold):
    return node.is_leaf and abs(node.value - threshold) < 1e-7

def code_to_pos(code, unit_size):
    return Vertex(PositionCode(code[0] * unit_size, code[1] * unit_size, code[2] * unit_size))

def solve(p1, p2, threshold):
    t = (threshold - p1.pos[0]) / (p2.pos[0] - p1.pos[0])
    return Vertex(PositionCode(
        p1.pos[0] + t * (p2.pos[0] - p1.pos[0]),
        p1.pos[1] + t * (p2.pos[1] - p1.pos[1]),
        p1.pos[2] + t * (p2.pos[2] - p1.pos[2])
    ))

def next_quad_index(node, line_dir):
    # Advances node pointer along the contour edge direction [G]
    return node.children[0] if line_dir == 0 else node.children[1]

def detect_quad(node, line, threshold, edge_kd):
    if node is None:
        return
    for i in range(4):
        current = node
        while current is not None and not is_contouring_leaf(current, threshold):
            current = next_quad_index(current, line.dir)
        edge_kd[i] = current

def check_minimal_edge(edge_kd, line_dir):
    min_end = PositionCode(float('inf'), float('inf'), float('inf'))
    max_end = PositionCode(float('-inf'), float('-inf'), float('-inf'))
    for node in edge_kd:
        if node is not None:
            min_end[line_dir] = max(min_end[line_dir], node.min_code[line_dir])
            max_end[line_dir] = min(max_end[line_dir], node.max_code[line_dir])
    return min_end[line_dir] < max_end[line_dir]

def generate_quad(edge_kd, line, mesh, threshold, unit_size):
    quad_dir1 = line.dir
    quad_dir2 = 3 - quad_dir1 - line.dir
    if not check_minimal_edge(edge_kd, line.dir):
        return

    p1 = code_to_pos(edge_kd[0].min_code, unit_size)
    p2 = code_to_pos(edge_kd[0].max_code, unit_size)
    edge_vertex = solve(p1, p2, threshold)
    mesh.add_vertex(edge_vertex)

    polygons = [n for n in edge_kd if n is not None]
    for i in range(1, len(polygons) - 1):
        mesh.add_triangle(edge_vertex, polygons[i-1], polygons[i])

def contour_edge(node, line, threshold, error, mesh, unit_size):
    if node is None:
        return
    edge_kd = [None] * 4
    detect_quad(node, line, threshold, edge_kd)

    if all(is_contouring_leaf(n, threshold) for n in edge_kd if n is not None):
        generate_quad(edge_kd, line, mesh, threshold, unit_size)
    else:
        for i in range(4):
            if edge_kd[i] is not None and not is_contouring_leaf(edge_kd[i], threshold):
                plane_dir = i % 3
                if plane_dir == line.dir:
                    contour_edge(edge_kd[i].children[0], line, threshold, error, mesh, unit_size)
                    contour_edge(edge_kd[i].children[1], line, threshold, error, mesh, unit_size)
```

**Edge Cases and Pitfalls:**
- **Null Pointers:** During traversal, if a node pointer becomes null (e.g., boundary of tree), the function returns immediately to prevent segfaults [S1, unofficial].
- **Degenerate Edges:** The algorithm computes minEnd and maxEnd by intersecting min/max codes of the 4 nodes along `line.dir`; if `minEnd[dir] >= maxEnd[dir]`, the edge is degenerate and the quad is skipped [S1, unofficial].
- **Precision:** The `error` parameter (default $10^{-7}$) is used for precision checks, though its exact application in comparisons is not fully detailed in the notes [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not explicitly given [S1, unofficial]. Depends on the depth of the k-d tree and the number of contouring leaves [S1, unofficial].
- **Memory Complexity:** Not explicitly given [S1, unofficial]. Depends on the size of the k-d tree and the number of vertices/triangles generated [S1, unofficial].
- **Performance:** Implemented in Danielhu/KdtreeISO example viewer for extracting iso-surfaces from volume data (tiff files) using oct/kd trees [S1, unofficial].
- **Strengths:** Adaptive resolution, handles complex topologies [S1, unofficial].
- **Limitations:** Complexity of implementation, potential for degenerate edges [S1, unofficial].

## Background
- **Origin:** Implemented in Danielhu/KdtreeISO example viewer [S1, unofficial].
- **History:** Not explicitly given [S1, unofficial].
- **Usage:** Extracting iso-surfaces from volume data (tiff files) using oct/kd trees [S1, unofficial].

## Sources
[S1] Danielmelody/KdtreeISO, https://github.com/Danielmelody/KdtreeISO, fetched 2026-10-27 (unofficial)
