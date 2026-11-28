# Dual Marching Cubes with Adaptive Octree and QEF Feature Isolation

Also called: DMC with Octree

## Overview
This method extracts an isosurface from a scalar field by recursively subdividing space into an adaptive octree based on a Quadratic Error Function (QEF) threshold, then applying a dual marching cubes algorithm on the leaf nodes [S1, unofficial]. It differs from standard Marching Cubes by replacing the fixed grid with an adaptive octree and using QEF-based feature isolation to determine subdivision, extracting the surface from the dual cells formed by these leaves [S1, unofficial].

## How it works
**Inputs:**
- `field_values`: A 3D array of shape `(Dx, Dy, Dz)` representing the scalar field.
- `isolevel`: The scalar value defining the isosurface.
- `QEF_threshold`: Float, maximum allowed QEF error for a node to remain a leaf. Typical value: `0.01` [S1, unofficial].
- `number_of_points`: Integer, number of random points sampled per cell for QEF computation [S1, unofficial].

**Outputs:**
- `vertices`: List of 3D points.
- `triangles`: List of 3 integer indices into `vertices`.

**Data Structures:**
- `OctreeNode`:
  - `base_coords`: 3D vector `[x, y, z]` of the node's minimum corner.
  - `side_length`: Float, side length of the cube.
  - `number_of_points`: Int, points to sample for QEF.
  - `QEF_threshold`: Float, threshold for leaf decision.
  - `is_leaf`: Bool, true if node is a leaf.
  - `cell_params`: 4D vector `[w, x, y, z]` from QEF optimization.
  - `QEF`: Float, minimized error value.
  - `children`: Array of 8 `OctreeNode` references (or `None`).
- `DMCOutput`:
  - `vertices`: List of 3D points.
  - `triangles`: List of 3 integer indices.
  - `vertices_len`: Int, cumulative count of vertices added.
- `EdgeVertexIndices`: List of 12 pairs of vertex indices for cube edges: `[[0,1], [1,3], [3,2], [2,0], [4,5], [5,7], [7,6], [6,4], [0,4], [1,5], [3,7], [2,6]]` [S1, unofficial].
- `tri_table`: Lookup table mapping cube index (0-255) to a flat list of triangle edge indices [S1, unofficial].

**Algorithm Steps:**

1. **Initialization:** Set global `field_values`, `isolevel`, and `dmcoutput` (empty `DMCOutput`) [S1, unofficial]. Calculate `n = (field_values.shape[0]-1)^3` [S1, unofficial]. Create `root` `OctreeNode` at `base_coords=[0,0,0]`, `side_length=1`, `number_of_points=n` [S1, unofficial].
2. **Recursive Subdivision (`faceProc`):** Call `faceProc(root)` to start the recursive subdivision and extraction process [S1, unofficial].
3. **QEF Computation (`get_QEF`):** Inside `OctreeNode.__init__`, call `get_QEF()` [S1, unofficial]. Create `Feature_Isolation` object with `base_coords`, `side_length`, `field_values`, `isolevel`, `number_of_points` [S1, unofficial]. Calculate `mid_point = base_coords + side_length / 2` [S1, unofficial]. Set `initial_guess = [isolevel, mid_point.x, mid_point.y, mid_point.z]` [S1, unofficial]. Set bounds: `w` unbounded, `x,y,z` bounded by `[base_coords, base_coords + side_length]` [S1, unofficial]. Call `scipy.optimize.minimize` with `compute_E` as objective [S1, unofficial]. Return `result.fun` (QEF) and `result.x` (cell_params) [S1, unofficial]. If `QEF < QEF_threshold`, set `node.is_leaf = True` [S1, unofficial].
4. **Error Function (`compute_E`):** Sample `number_of_points` random points in `[0,1]^3` (normalized) [S1, unofficial]. For each point `p_i`, compute gradient `grad_i` using `compute_gradient` [S1, unofficial]. Compute error: $E_i = \frac{(w - \text{dot}(\text{grad}_i, \text{point}_i - \text{center}))^2}{1 + |\text{grad}_i|^2}$ [S1, unofficial]. Here, `params = [w, x, y, z]`, `center = [x, y, z]` from `initial_guess` [S1, unofficial]. Return `sum(E_i)` [S1, unofficial].
5. **Gradient and Field Evaluation:** `compute_gradient(field_values, x, y, z, h=1e-5)` computes `dfdx = (compute_f(x+h, y, z) - compute_f(x-h, y, z)) / (2*h)` and similarly for `dfdy`, `dfdz` [S1, unofficial]. `compute_f(field_values, sx, sy, sz)` returns `0` if `sx, sy, sz` out of `[0,1]` [S1, unofficial]. Maps to grid indices: `x = sx*(Dx-1)`, etc. Clamps `i0` to `Dx-2` if `i0 == Dx-1` to avoid out-of-bounds access for `i1` [S1, unofficial]. Computes weights `wx = (x - i0) / (i1 - i0)` (if `i1==i0`, `wx=0`) [S1, unofficial]. Performs trilinear interpolation of 8 corners [S1, unofficial].
6. **Subdivision (`subdivide`):** If `is_leaf`, return [S1, unofficial]. Calculate `half_size = side_length / 2` [S1, unofficial]. Calculate `new_number_of_points = number_of_points // 8` [S1, unofficial]. If `new_number_of_points == 0`, set `is_child_leaf = True` for all children to prevent infinite recursion or invalid sampling [S1, unofficial]. Create 8 children with updated `base_coords`, `half_size`, `new_number_of_points` [S1, unofficial].
7. **Edge Processing (`edgeProc`):** `edgeProc(node1, node2)` returns if either `None` or both are leaves [S1, unofficial]. If `node1.base_coords[0] != node2.base_coords[0]` (X-axis adjacent), define 4 edge pairs of children, recursively call `edgeProc` on these pairs, then call `vertProc` with the 8 children involved in the edge [S1, unofficial]. If not X-axis adjacent (Y or Z adjacency), define 4 different edge pairs of children, recursively call `edgeProc`, then call `vertProc` with the 8 children involved [S1, unofficial].
8. **Vertex Processing (`vertProc`):** `vertProc(node1...node8)` returns if any `None` [S1, unofficial]. If all 8 are leaves, collect `cell_params` into an 8x4 array, call `mod_marching_cubes(cell_params, isolevel, field_values)`, insert `verts` and `fcs` into `dmcoutput`, and return [S1, unofficial]. If not all leaves, recursively call `vertProc` on the children of the 8 nodes, reversing the order (child 7 of node1, child 6 of node2, etc.) [S1, unofficial].
9. **Marching Cubes Extraction (`mod_marching_cubes`):** Extract `vertices_coords` from `cell_params[:, 1:]` [S1, unofficial]. Compute `scalar_field` for each vertex using `compute_f` [S1, unofficial]. Call `marching_cubes(vertices_coords, scalar_field, isolevel)` [S1, unofficial].
10. **Standard Marching Cubes (`marching_cubes`):** Compute `cube_index` by checking if `isovalue < isolevel` for each of 8 vertices [S1, unofficial]. Get edges from `tri_table[cube_index]` [S1, unofficial]. For each triangle (3 edges), get vertex pairs from `EdgeVertexIndices` [S1, unofficial]. Interpolate vertices using `interpolate_vertex` [S1, unofficial]. Store unique vertices in `vert_dict` [S1, unofficial]. Append triangle indices to `triangles` [S1, unofficial].
11. **Interpolation (`interpolate_vertex`):** `interpolate_vertex(p1, p2, val1, val2, isolevel)` returns `p1` if `|isolevel - val1| < 0.00001`, `p2` if `|isolevel - val2| < 0.00001`, `p1` if `|val1 - val2| < 0.00001`, otherwise computes `t = (isolevel - val1) / (val2 - val1)` and returns `p1 + t * (p2 - p1)` [S1, unofficial].
12. **Output Insertion (`DMCOutput.insert`):** Append `verts` to `self.vertices` [S1, unofficial]. For each face `fc`, add `self.vertices_len` to each index [S1, unofficial]. Append modified `fc` to `self.triangles` [S1, unofficial]. Increment `self.vertices_len` by `len(verts)` [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import math
from scipy.optimize import minimize

# Standard MC lookup tables [S1, unofficial]
EdgeVertexIndices = [
    [0, 1], [1, 3], [3, 2], [2, 0], [4, 5], [5, 7], [7, 6], [6, 4],
    [0, 4], [1, 5], [3, 7], [2, 6]
]
tri_table = [...] # Standard 256-entry table mapping cube index to edge triples [S1, unofficial]

def compute_f(field_values, sx, sy, sz):
    if sx < 0 or sx > 1 or sy < 0 or sy > 1 or sz < 0 or sz > 1:
        return 0.0
    dimx, dimy, dimz = field_values.shape
    x, y, z = sx * (dimx - 1), sy * (dimy - 1), sz * (dimz - 1)
    i0, j0, k0 = int(math.floor(x)), int(math.floor(y)), int(math.floor(z))
    if i0 == dimx - 1: i0 -= 1
    if j0 == dimy - 1: j0 -= 1
    if k0 == dimz - 1: k0 -= 1
    i1, j1, k1 = i0 + 1, j0 + 1, k0 + 1
    wx = (x - i0) / (i1 - i0) if i1 != i0 else 0.0
    wy = (y - j0) / (j1 - j0) if j1 != j0 else 0.0
    wz = (z - k0) / (k1 - k0) if k1 != k0 else 0.0
    f000 = field_values[i0, j0, k0]
    f100 = field_values[i1, j0, k0]
    f010 = field_values[i0, j1, k0]
    f110 = field_values[i1, j1, k0]
    f001 = field_values[i0, j0, k1]
    f101 = field_values[i1, j0, k1]
    f011 = field_values[i0, j1, k1]
    f111 = field_values[i1, j1, k1]
    return ((1 - wx) * (1 - wy) * (1 - wz) * f000 +
            wx * (1 - wy) * (1 - wz) * f100 +
            (1 - wx) * wy * (1 - wz) * f010 +
            wx * wy * (1 - wz) * f110 +
            (1 - wx) * (1 - wy) * wz * f001 +
            wx * (1 - wy) * wz * f101 +
            (1 - wx) * wy * wz * f011 +
            wx * wy * wz * f111)

def compute_gradient(field_values, x, y, z, h=1e-5):
    dfdx = (compute_f(field_values, x + h, y, z) - compute_f(field_values, x - h, y, z)) / (2 * h)
    dfdy = (compute_f(field_values, x, y + h, z) - compute_f(field_values, x, y - h, z)) / (2 * h)
    dfdz = (compute_f(field_values, x, y, z + h) - compute_f(field_values, x, y, z - h)) / (2 * h)
    return np.array([dfdx, dfdy, dfdz])

class Feature_Isolation:
    def __init__(self, base_coords, side_length, field_values, isolevel, number_of_points):
        self.base_coords = base_coords
        self.side_length = side_length
        self.number_of_points = number_of_points
        self.QEF, self.cell_params = self.compute_QEF(field_values, isolevel)

    def compute_QEF(self, field_values, isolevel):
        mid_point = np.array([
            self.base_coords[0] + self.side_length / 2,
            self.base_coords[1] + self.side_length / 2,
            self.base_coords[2] + self.side_length / 2
        ])
        initial_guess = np.hstack([isolevel, mid_point])
        bounds = [
            (None, None),
            (self.base_coords[0], self.base_coords[0] + self.side_length),
            (self.base_coords[1], self.base_coords[1] + self.side_length),
            (self.base_coords[2], self.base_coords[2] + self.side_length)
        ]
        result = minimize(self.compute_E, initial_guess, args=(field_values, isolevel, mid_point), bounds=bounds)
        return result.fun, result.x

    def compute_E(self, params, field_values, isolevel, mid_point):
        points = np.random.rand(self.number_of_points, 3)
        w, x, y, z = params
        gradient = np.array([compute_gradient(field_values, p[0], p[1], p[2]) for p in points])
        gr_magnitude = np.array([np.sqrt(g[0]**2 + g[1]**2 + g[2]**2) for g in gradient])
        E_arr = np.array([
            ((w - np.dot(gradient[i], np.array([x - points[i][0], y - points[i][1], z - points[i][2]]))) ** 2) / (1 + gr_magnitude[i]**2)
            for i in range(len(gradient))
        ])
        return np.sum(E_arr)

class OctreeNode:
    def __init__(self, base_coords, side_length, number_of_points, QEF_threshold=0.01):
        self.base_coords = base_coords
        self.side_length = side_length
        self.number_of_points = number_of_points
        self.QEF_threshold = QEF_threshold
        self.is_leaf = False
        self.cell_params = None
        self.QEF = None
        self.children = [None] * 8
        self._get_QEF()

    def _get_QEF(self):
        fi = Feature_Isolation(self.base_coords, self.side_length, field_values, isolevel, self.number_of_points)
        self.QEF = fi.QEF
        self.cell_params = fi.cell_params
        if self.QEF < self.QEF_threshold:
            self.is_leaf = True

    def subdivide(self):
        if self.is_leaf:
            return
        half_size = self.side_length / 2
        new_number_of_points = self.number_of_points // 8
        is_child_leaf = (new_number_of_points == 0)
        offsets = [
            [0, 0, 0], [1, 0, 0], [0, 1, 0], [1, 1, 0],
            [0, 0, 1], [1, 0, 1], [0, 1, 1], [1, 1, 1]
        ]
        for i, offset in enumerate(offsets):
            new_coords = self.base_coords + np.array(offset) * half_size
            child = OctreeNode(new_coords, half_size, new_number_of_points, self.QEF_threshold)
            if is_child_leaf:
                child.is_leaf = True
            self.children[i] = child

class DMCOutput:
    def __init__(self):
        self.vertices = []
        self.triangles = []
        self.vertices_len = 0

    def insert(self, verts, fcs):
        if fcs is not None:
            self.vertices.extend(verts)
            for fc in fcs:
                self.triangles.append([self.vertices_len + fc[0], self.vertices_len + fc[1], self.vertices_len + fc[2]])
            self.vertices_len += len(verts)

def interpolate_vertex(p1, p2, val1, val2, isolevel):
    if abs(isolevel - val1) < 0.00001: return p1
    if abs(isolevel - val2) < 0.00001: return p2
    if abs(val1 - val2) < 0.00001: return p1
    t = (isolevel - val1) / (val2 - val1)
    return p1 + t * (p2 - p1)

def marching_cubes(cube_positions, cube_isovalues, isolevel):
    cube_index = 0
    for i in range(8):
        if cube_isovalues[i] < isolevel:
            cube_index |= (1 << i)
    edges = tri_table[cube_index]
    if not edges:
        return [], []
    vert_dict = {}
    triangles = []
    for tri_edges in range(0, len(edges), 3):
        e0, e1, e2 = edges[tri_edges], edges[tri_edges+1], edges[tri_edges+2]
        v0 = EdgeVertexIndices[e0]
        v1 = EdgeVertexIndices[e1]
        v2 = EdgeVertexIndices[e2]
        verts = []
        for edge_idx in [v0, v1, v2]:
            p1, p2 = cube_positions[edge_idx[0]], cube_positions[edge_idx[1]]
            val1, val2 = cube_isovalues[edge_idx[0]], cube_isovalues[edge_idx[1]]
            verts.append(interpolate_vertex(p1, p2, val1, val2, isolevel))
        tri_indices = []
        for v in verts:
            v_key = tuple(np.round(v, 6))
            if v_key not in vert_dict:
                vert_dict[v_key] = len(vert_dict)
            tri_indices.append(vert_dict[v_key])
        triangles.append(tri_indices)
    return list(vert_dict.values()), triangles

def mod_marching_cubes(cell_params, isolevel, field_values):
    vertices_coords = cell_params[:, 1:]
    scalar_field = np.array([compute_f(field_values, p[0], p[1], p[2]) for p in vertices_coords])
    return marching_cubes(vertices_coords, scalar_field, isolevel)

def faceProc(node):
    if node.is_leaf:
        return
    node.subdivide()
    for child in node.children:
        faceProc(child)
    edge_pairs = [
        (node.children[0], node.children[1]), (node.children[1], node.children[3]),
        (node.children[3], node.children[2]), (node.children[2], node.children[0]),
        (node.children[4], node.children[5]), (node.children[5], node.children[7]),
        (node.children[7], node.children[6]), (node.children[6], node.children[4]),
        (node.children[0], node.children[4]), (node.children[1], node.children[5]),
        (node.children[3], node.children[7]), (node.children[2], node.children[6])
    ]
    for c1, c2 in edge_pairs:
        edgeProc(c1, c2)
    vertProc(*node.children)

def edgeProc(node1, node2):
    if node1 is None or node2 is None or (node1.is_leaf and node2.is_leaf):
        return
    if node1.base_coords[0] != node2.base_coords[0]:
        pairs = [(node1.children[0], node2.children[0]), (node1.children[1], node2.children[1]),
                 (node1.children[4], node2.children[4]), (node1.children[5], node2.children[5])]
        for c1, c2 in pairs:
            edgeProc(c1, c2)
        vertProc(node1.children[0], node1.children[1], node2.children[0], node2.children[1],
                 node1.children[4], node1.children[5], node2.children[4], node2.children[5])
    else:
        pairs = [(node1.children[0], node2.children[0]), (node1.children[2], node2.children[2]),
                 (node1.children[4], node2.children[4]), (node1.children[6], node2.children[6])]
        for c1, c2 in pairs:
            edgeProc(c1, c2)
        vertProc(node1.children[0], node1.children[2], node2.children[0], node2.children[2],
                 node1.children[4], node1.children[6], node2.children[4], node2.children[6])

def vertProc(n1, n2, n3, n4, n5, n6, n7, n8):
    if any(n is None for n in [n1, n2, n3, n4, n5, n6, n7, n8]):
        return
    if all(n.is_leaf for n in [n1, n2, n3, n4, n5, n6, n7, n8]):
        cell_params = np.array([n.cell_params for n in [n1, n2, n3, n4, n5, n6, n7, n8]])
        verts, fcs = mod_marching_cubes(cell_params, isolevel, field_values)
        dmcoutput.insert(verts, fcs)
        return
    vertProc(n1.children[7], n2.children[6], n3.children[5], n4.children[4],
             n5.children[3], n6.children[2], n7.children[1], n8.children[0])
```

## Operating characteristics

## Background

## Sources
[S1] ank1001/DualMarchingCubes, https://github.com/ank1001/DualMarchingCubes, fetched 2026-10-27 (unofficial)
