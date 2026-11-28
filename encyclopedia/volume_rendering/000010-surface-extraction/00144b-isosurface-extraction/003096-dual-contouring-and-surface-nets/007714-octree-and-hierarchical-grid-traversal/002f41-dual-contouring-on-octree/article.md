# Dual Contouring on Octree

Also called: Dual Contouring with Octree, Dual Contouring: Seams & LOD for Chunked Terrain

## Overview
Dual Contouring on Octree is a surface extraction method that generates a triangle mesh from an adaptive octree representation of a scalar field by placing a single vertex inside each leaf cell containing an isosurface intersection, determined by minimizing a Quadratic Error Function (QEF) [S1, S2, S3, S4, unofficial]. It builds on [Dual Contouring](778-dual-contouring.md) but replaces the uniform grid with a hierarchical octree to allow adaptive resolution and mesh simplification based on error thresholds [S3, S4, unofficial]. It differs from its siblings by explicitly processing the octree hierarchy through recursive cell, face, and edge traversal to stitch vertices across different levels of subdivision, ensuring topological consistency without requiring a uniform grid [S1, S2, S3, S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar field function $f(\mathbf{x})$, an isovalue $iso$, a bounding box (Cuboid), and a maximum octree depth [S2, S4, unofficial][S5].
*   **Output:** A list of meshes, each containing a vertex buffer (positions, normals) and an index buffer (triangles) [S2, S3, unofficial].

**Data Structures:**
The method relies on an Octree structure and specific lookup tables for connectivity [S2, S3, S4, unofficial].

1.  **Octree<T>**:
    *   `bounds`: A Cuboid defining the spatial extent (min/max coordinates).
    *   `data`: A generic type `T` (typically `NodeData` or `OctreeDrawInfo`).
    *   `children`: An array of 8 `Octree<T>` pointers. `null` indicates a leaf node.
    *   **Subdivision**: When `SubDivide()` is called, the bounds are split into 8 equal children. Child $i$ (0-7) is offset by `(i&1)*size.x`, `(i&2)*size.y`, `(i&4)*size.z` [S2, unofficial].

2.  **NodeData** (used in [S2, unofficial]):
    *   `qef`: A Quadratic Error Function object.
    *   `vertex`: A 3D float position (the solution to the QEF).
    *   `cornerEncoding`: An integer bitmask representing which corners of the cell are inside/outside the isosurface.
    *   `index`: An integer mesh vertex index, initialized to -1. Used to prevent duplicate vertices [S2, unofficial].
    *   `size`: The diagonal length of the node's bounding box.

3.  **OctreeDrawInfo** (used in [S3, unofficial]):
    *   `position`: Vector3 vertex position.
    *   `normal`: Vector3 vertex normal.
    *   `index`: Integer mesh vertex index.
    *   `corners`: Integer bitmask of corner material states.
    *   `size`: Float diagonal length.
    *   Constants: `MATERIAL_SOLID = 1`, `MATERIAL_AIR = 0` [S3, unofficial].

4.  **Lookup Tables** (Hardcoded integers defining connectivity and processing order) [S3, S4, unofficial]:
    *   `edgevmap` (12x2): Maps edge indices to the two vertex indices they connect. [S3, S4, unofficial]
        *   X-axis edges: `{0,4}, {1,5}, {2,6}, {3,7}` [S3, S4, unofficial]
        *   Y-axis edges: `{0,2}, {1,3}, {4,6}, {5,7}` [S3, S4, unofficial]
        *   Z-axis edges: `{0,1}, {2,3}, {4,5}, {6,7}` [S3, S4, unofficial]
    *   `vertMap` (8x3): Maps vertex index to corner coordinates (0/1 for x,y,z).
    *   `faceMap` (6x4): Maps face index to the 4 vertex corners.
    *   `cellProcFaceMask` (12x3): Maps cell face index to two child indices and a direction.
        *   Example: `{{0, 4, 0}, {1, 5, 0}, {2, 6, 0}, {3, 7, 0}, {0, 2, 1}, {4, 6, 1}, {1, 3, 1}, {5, 7, 1}, {0, 1, 2}, {2, 3, 2}, {4, 5, 2}, {6, 7, 2}}` [S4, unofficial].
    *   `cellProcEdgeMask` (6x5): Maps cell edge index to four child indices and a direction.
        *   Example: `{{0, 2, 1, 3, 0}, {4, 6, 5, 7, 0}, {0, 1, 4, 5, 1}, {2, 3, 6, 7, 1}, {0, 4, 2, 6, 2}, {1, 5, 3, 7, 2}}` [S4, unofficial].
    *   `faceProcFaceMask` (3x4x3): Maps face-level children to sub-children and directions.
    *   `faceProcEdgeMask` (3x4x6): Maps face-level children to sub-edges and directions.
    *   `edgeProcEdgeMask` (3x2x5): Maps edge-level children to sub-edges and directions.
    *   `processEdgeMask` (3x4): Maps edge directions to global edge indices.

**Algorithm Steps:**

1.  **Build Octree**: Recursively subdivide the bounding box. [S2, unofficial] For a node at depth > 1, create 8 children by splitting the domain. [S4, unofficial] For each child, recursively build the subtree. [S4, unofficial] Combine the QEF matrices from all valid children into the parent's grid [S4, unofficial]. Leaf node processing: If depth == 1, check if the grid is signed. [S4, unofficial] If not, delete the node. If signed, sample the QEF from the scalar field, mark as leaf, and solve for the vertex position [S4, unofficial].
    *   *Note*: In adaptive variants, a node is "clusterable" if all its children are clusterable AND the faces between adjacent children allow for cluster merging. [S4, unofficial] If clusterable and approximation error is below a threshold, remove all children and mark the node as a leaf [S4, unofficial].

2.  **Generate Vertex Indices**: Traverse the octree recursively. [S3, unofficial] For each leaf (or pseudo-leaf) node, assign `drawInfo.index = verts.Count`, append `drawInfo.position` to the vertex list, and append `drawInfo.normal` to the normal list [S3, unofficial].

3.  **Contour Cell Proc**: Recursively traverse the octree. [S3, unofficial] For each internal node:
    *   Recursively call `ContourCellProc` on all 8 children. [S2, unofficial]
    *   Iterate 12 times (for each face): Fetch two child indices and a direction from `cellProcFaceMask[i]`. [S3, unofficial] Construct a 2-element node array and call `ContourFaceProc`. [S3, unofficial]
    *   Iterate 6 times (for each edge): Fetch four child indices and a direction from `cellProcEdgeMask[i]`. [S3, unofficial] Construct a 4-element node array and call `ContourEdgeProc` [S2, S3, unofficial].

4.  **Contour Face Proc**: Given two adjacent nodes (`nodes[0]`, `nodes[1]`) and a direction:
    *   If `nodes[0]` or `nodes[1]` is null, return. [S2, unofficial]
    *   If both nodes are leaves, return.
    *   If either node is internal, subdivide the face into 4 child faces. [S4, unofficial] For each child face, determine the corresponding child nodes from the parent nodes using `faceProcFaceMask` and recursively call `ContourFaceProc`. [S3, unofficial]
    *   Iterate 4 times (for each sub-edge): Fetch 4 child indices and a direction from `faceProcEdgeMask` and the orders table. [S3, unofficial] Construct a 4-element node array and call `ContourEdgeProc` [S2, S3, unofficial].

5.  **Contour Edge Proc**: Given four nodes along an edge and a direction:
    *   If any node is null, return.
    *   If all 4 nodes are leaves (or pseudo-nodes), call `ContourProcessEdge`. [S3, unofficial]
    *   Otherwise, subdivide the edge into 2 child edges. [S3, unofficial] Determine the corresponding child nodes using `edgeProcEdgeMask` and recursively call `ContourEdgeProc` [S2, S3, unofficial].

6.  **Contour Process Edge (Triangle Generation)**: Given four leaf nodes along an edge:
    *   Check if `vertices.Count + 4 > MAX_VERTS_PER_MESH` (default 30000). If true, flush the mesh buffers [S2, unofficial].
    *   Initialize `ptrs` array to `{-1, -1, -1, -1}`.
    *   Initialize `signChange` array to `{false, false, false, false}`.
    *   Initialize `minSize` to Infinity, `minIndex` to 0, `flip` to false.
    *   Loop $i$ from 0 to 3:
        *   Get bounds and data for `nodes[i]`.
        *   Get edge index from `processEdgeMask[direction, i]` and encoding from `data.cornerEncoding`.
        *   Check sign change:
            *   $c0Sign = ((encoding \gg edgevmap[edge, 0]) \& 1) > 0$
            *   $c1Sign = ((encoding \gg edgevmap[edge, 1]) \& 1) > 0$
            *   $signChange[i] = c0Sign \neq c1Sign$
        *   If `data.size < minSize`, update `minSize`, `minIndex`, and `flip = c0Sign`.
        *   If `data.index == -1`, set `data.index = vertices.Count`, add `data.vertex` to vertices, and set `ptrs[i] = data.index`.
    *   If `signChange[minIndex]` is true, add triangles to indices:
        *   If `flip` is true: Add `(ptrs[0], ptrs[1], ptrs[3])` and `(ptrs[0], ptrs[3], ptrs[2])`.
        *   Else: Add `(ptrs[0], ptrs[3], ptrs[1])` and `(ptrs[0], ptrs[2], ptrs[3])` [S2, S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MAX_VERTS_PER_MESH = 30000

# Lookup Tables (Exact values from sources)
edgevmap = [
    [0,4], [1,5], [2,6], [3,7], # x-axis
    [0,2], [1,3], [4,6], [5,7], # y-axis
    [0,1], [2,3], [4,5], [6,7]  # z-axis
]

cellProcFaceMask = [
    [0, 4, 0], [1, 5, 0], [2, 6, 0], [3, 7, 0],
    [0, 2, 1], [4, 6, 1], [1, 3, 1], [5, 7, 1],
    [0, 1, 2], [2, 3, 2], [4, 5, 2], [6, 7, 2]
]

cellProcEdgeMask = [
    [0, 2, 1, 3, 0], [4, 6, 5, 7, 0],
    [0, 1, 4, 5, 1], [2, 3, 6, 7, 1],
    [0, 4, 2, 6, 2], [1, 5, 3, 7, 2]
]

processEdgeMask = [
    [0, 1, 2, 3],
    [4, 5, 6, 7],
    [8, 9, 10, 11]
]

# Helper classes
class Cuboid:
    def __init__(self, min_pt, max_pt):
        self.min = min_pt
        self.max = max_pt
        self.size = [max_pt[i] - min_pt[i] for i in range(3)]

class Octree:
    def __init__(self, bounds, data=None):
        self.bounds = bounds
        self.data = data
        self.children = None

    def SubDivide(self):
        if self.children is not None:
            return
        self.children = [None] * 8
        child_size = [self.bounds.size[i] / 2.0 for i in range(3)]
        for i in range(8):
            offset = [
                child_size[0] if (i & 1) else 0.0,
                child_size[1] if (i & 2) else 0.0,
                child_size[2] if (i & 4) else 0.0
            ]
            min_pt = [self.bounds.min[i] + offset[i] for i in range(3)]
            max_pt = [min_pt[i] + child_size[i] for i in range(3)]
            self.children[i] = Octree(Cuboid(min_pt, max_pt))

# Sources do not provide scalar field sampling or QEF solver implementation. [G]

def build_octree(node, scalar_field, iso, max_depth, current_depth):
    if node is None:
        return
    if current_depth >= max_depth or not bounds_intersect_scalar_field(node.bounds, scalar_field, iso):
        node.data = solve_qef(node.bounds, scalar_field, iso)
        node.children = None
        return

    node.SubDivide()
    for i in range(8):
        build_octree(node.children[i], scalar_field, iso, max_depth, current_depth + 1)

def generate_vertex_indices(node, vertices, normals):
    if node is None:
        return
    if node.children is not None:
        for i in range(8):
            generate_vertex_indices(node.children[i], vertices, normals)
    else:
        draw_info = node.data
        draw_info["index"] = len(vertices)
        vertices.append(draw_info["vertex"])
        normals.append(draw_info["normal"])

def contour_cell_proc(node, indices):
    if node is None:
        return
    if node.children is not None:
        for i in range(8):
            contour_cell_proc(node.children[i], indices)
        for i in range(12):
            c = cellProcFaceMask[i]
            face_nodes = [node.children[c[0]], node.children[c[1]]]
            contour_face_proc(face_nodes, c[2], indices)
        for i in range(6):
            c = cellProcEdgeMask[i]
            edge_nodes = [node.children[c[0]], node.children[c[1]], node.children[c[2]], node.children[c[3]]]
            contour_edge_proc(edge_nodes, c[4], indices)

def contour_face_proc(nodes, direction, indices):
    if nodes[0] is None or nodes[1] is None:
        return
    if nodes[0].children is None and nodes[1].children is None:
        return

    if nodes[0].children is not None or nodes[1].children is not None:
        faceProcFaceMask = [
            [[4, 0, 0], [5, 1, 0], [6, 2, 0], [7, 3, 0]],
            [[2, 0, 1], [6, 4, 1], [3, 1, 1], [7, 5, 1]],
            [[1, 0, 2], [3, 2, 2], [5, 4, 2], [7, 6, 2]]
        ]
        for i in range(4):
            c = faceProcFaceMask[direction][i]
            child_nodes = [nodes[c[0]], nodes[c[1]]]
            contour_face_proc(child_nodes, c[2], indices)
        
        faceProcEdgeMask = [
            [[1, 4, 5, 0, 1, 1], [1, 6, 7, 2, 3, 1], [0, 4, 6, 0, 2, 2], [0, 5, 7, 1, 3, 2]],
            [[0, 2, 3, 0, 1, 0], [0, 6, 7, 4, 5, 0], [1, 2, 6, 0, 4, 2], [1, 3, 7, 1, 5, 2]],
            [[1, 1, 3, 0, 2, 0], [1, 5, 7, 4, 6, 0], [0, 1, 5, 0, 4, 1], [0, 3, 7, 2, 6, 1]]
        ]
        orders = [[0, 0, 1, 1], [0, 1, 0, 1]]
        for i in range(4):
            c = faceProcEdgeMask[direction][i]
            idx = c[0]
            edge_nodes = [nodes[orders[idx][j]] if nodes[orders[idx][j]].children is None else nodes[orders[idx][j]].children[c[j+1]] for j in range(4)]
            contour_edge_proc(edge_nodes, c[5], indices)

def contour_edge_proc(nodes, direction, indices):
    if any(n is None for n in nodes):
        return
    if all(n.children is None for n in nodes):
        contour_process_edge(nodes, direction, indices)
    else:
        edgeProcEdgeMask = [
            [[0, 2, 3, 1], [4, 6, 7, 5]],
            [[0, 1, 5, 4], [2, 3, 7, 6]],
            [[0, 4, 6, 2], [1, 5, 7, 2]]
        ]
        for i in range(2):
            c = edgeProcEdgeMask[direction][i]
            child_nodes = [nodes[c[j]] for j in range(4)]
            contour_edge_proc(child_nodes, direction, indices)

def contour_process_edge(nodes, direction, indices):
    if len(vertices) + 4 > MAX_VERTS_PER_MESH:
        vertices.clear()
        normals.clear()
        indices.clear()
        return

    ptrs = [-1, -1, -1, -1]
    sign_change = [False, False, False, False]
    min_size = float('inf')
    min_index = 0
    flip = False

    for i in range(4):
        node = nodes[i]
        data = node.data
        encoding = data["cornerEncoding"]
        global_edge_idx = processEdgeMask[direction][i]
        v0, v1 = edgevmap[global_edge_idx]
        
        c0_sign = (encoding >> v0) & 1
        c1_sign = (encoding >> v1) & 1
        sign_change[i] = (c0_sign != c1_sign)
        
        if data["size"] < min_size:
            min_size = data["size"]
            min_index = i
            flip = (c0_sign == 1)
        
        if data["index"] == -1:
            data["index"] = len(vertices)
            vertices.append(data["vertex"])
        
        ptrs[i] = data["index"]

    if sign_change[min_index]:
        if flip:
            indices.extend([ptrs[0], ptrs[1], ptrs[3]])
            indices.extend([ptrs[0], ptrs[3], ptrs[2]])
        else:
            indices.extend([ptrs[0], ptrs[3], ptrs[1]])
            indices.extend([ptrs[0], ptrs[2], ptrs[3]])

def extract_mesh(scalar_field, iso, max_depth, bounds):
    root = Octree(bounds)
    build_octree(root, scalar_field, iso, max_depth, 0)
    
    vertices = []
    normals = []
    indices = []
    
    generate_vertex_indices(root, vertices, normals)
    contour_cell_proc(root, indices)
    
    return {"vertices": vertices, "normals": normals, "indices": indices}
```

## Subtopics

### [Manifold Dual Contouring](2138-manifold-dual-contouring.md)
Manifold Dual Contouring is an extension of Dual Contouring that guarantees the generated mesh is a closed 2-manifold even under adaptive simplification, using an octree-based topology-preserving vertex-clustering algorithm. It differs from standard Dual Contouring by enforcing manifold constraints through explicit vertex collapsing and edge processing, ensuring the resulting mesh is a valid 2-manifold without non-manifold edges. Unlike Extended Dual Contouring, it places no restriction on the number of contour intersections on each octree cell edge because the vertex tree is separate from the octree, allowing simplification of multiple layers of thin geometry.

### [Adaptive Contouring via Octree Vertex Clustering](2142-adaptive-contouring-via-octree-vertex-clustering.md)
Adaptive Contouring via Octree Vertex Clustering is a surface extraction method that constructs multi-resolution iso-surfaces by clustering vertices of child cells in an octree into a single vertex in a topologically safe manner. It allows an arbitrary number of vertices per cell to significantly reduce polygon count in flat regions, addressing the restrictive single-vertex-per-cell limitation of standard Dual Contouring. Unlike its siblings, it explicitly builds a "vertex tree" to manage hierarchical simplification and uses QEF minimization to determine the position of clustered vertices.

### [Adaptive Dual Contouring](3664-adaptive-dual-contouring.md)
Adaptive Dual Contouring extracts a manifold mesh from a scalar density function by building an adaptive octree, solving Quadratic Error Functions (QEF) at nodes to determine vertex positions, and generating a mesh from the resulting constrained vertices. It solves a Quadratic Error Function (QEF) at each node to determine vertex positions, then recursively collapses internal octree nodes into 'pseudo-leaf' nodes if the resulting QEF error is below a threshold, preserving sharp features defined by the sign configuration of child nodes. Unlike standard Dual Contouring which places one vertex per leaf voxel, this method reduces vertex count by merging internal nodes, producing a coarser but topologically consistent mesh suitable for real-time rendering or further processing.

### [Kdtree-based Iso-surface Extraction](3816-kdtree-based-iso-surface-extraction.md)
Kdtree-based Iso-surface Extraction extracts iso-surfaces from scalar fields using a hierarchical K-d tree built upon an octree, applying Quadratic Error Function (QEF) optimization and clusterability checks for vertex simplification. It differs from sibling methods by explicitly constructing a K-d tree to guide traversal and subdivision, while supporting a fallback octree-only mode for standard extraction.

### [Mesh Extraction from K-D Tree](4607-mesh-extraction-from-k-d-tree.md)
This method extracts a triangle mesh from a `RectilinearGrid` hierarchy by traversing the tree, generating vertex indices, and performing contouring operations on cells, faces, and edges. It processes the grid nodes directly without requiring a separate uniform grid or octree build.

### [Isomesh Octree Build and Clustering](4984-isomesh-octree-build-and-clustering.md)
Isomesh Octree Build and Clustering is a hierarchical mesh generation method that extracts a triangular mesh from an implicit function defined over an octree structure. It solves surface extraction in sparse or hierarchical domains by recursively building an octree, clustering vertices at each node level (leaves, edges, faces, cells) using Disjoint Set Union (DSU) and Quadratic Error Function (QEF) solvers to merge geometric primitives while preserving material properties and minimizing reconstruction error. Unlike uniform grid methods, it adapts resolution locally, allowing coarser branches to store vertices that minimize the sum of their children's Q functions, enabling recursive vertex calculation for all levels of detail. It sets itself apart from siblings like [Manifold Dual Contouring](2138-manifold-dual-contouring.md) by explicitly handling hierarchical LOD stitching via a "skirt" optimization and using DSU for topological correctness across irregular octree subdivisions.

### [Intersection-Free Dual Contouring](5313-intersection-free-dual-contouring.md)
Intersection-Free Dual Contouring is a variant of Dual Contouring that ensures the resulting mesh is manifold and free of self-intersections by detecting intersecting triangles and performing local tessellation or vertex splitting. It addresses the problem of self-intersecting triangles produced by the original Dual Contouring algorithm. It sets itself apart from siblings like [Manifold Dual Contouring](2138-manifold-dual-contouring.md) by using explicit pairwise intersection tests and post-processing tessellation rather than modifying the underlying optimization or constraint formulation to inherently prevent intersections.

### [Octree Construction for Dual Contouring](5913-octree-construction-for-dual-contouring.md)
This method constructs the spatial hierarchy required for Dual Contouring by recursively subdividing a volume into an octree, pruning branches that do not intersect the isosurface. It differs from sibling methods like [Mesh Extraction from K-D Tree](4607-mesh-extraction-from-k-d-tree.md) by using a fixed-depth hierarchical grid structure rather than a binary spatial partition, and from [Isomesh Octree Build and Clustering](4984-isomesh-octree-build-and-clustering.md) by focusing on single-vertex-per-leaf extraction logic rather than clustering [G]. It solves the problem of efficiently identifying which voxels contain surface geometry to minimize memory usage and processing time for sparse volumes [G].

### Seam Generation for Adjacent Chunks
Generates mesh vertices and indices along the boundary between two adjacent octree chunks to eliminate seams common in dual contouring implementations [S6, unofficial]. It initializes a `borderChildren` array of 8 pointers to `nullptr`, and an `idxs` array mapping chunk 0 to n1 and chunk 1 to n2. It iterates over both chunks, retrieves children arrays, and loops 4 times to fetch coordinates and edge indices from `edgeIndexLookup[dir][j][i]`. It computes octree indices, retrieves children, stores them in `borderChildren`, and generates vertex indices. Finally, it dispatches to the appropriate `FaceProc` function based on direction, passing the two octrees in the correct order to handle winding/normal consistency [S6, unofficial]. The `edgeIndexLookup` is a 4D array of ints with shape `[6][2][4][4]`, where dimensions correspond to directions (x+, x-, y+, y-, z+, z-), chunk index (old/new), border points, and `{x, y, z, edgeIdx}` respectively [S6, unofficial].

### Vertex Clustering for Adaptive DC
A vertex clustering approach for simplifying contours in adaptive octree-based Dual Contouring. It maintains disconnected contour components during simplification to improve adaptivity [S7, unofficial]. However, this method can still generate non-manifold edges and vertices in adaptive settings. It also requires non-trivial coding-vector operations and restricts the maximum number of intersections between the contour and each cell edge to 2 [S7, unofficial]. Compared to other methods, it is more complex to implement and has more restrictive simplification limits due to the intersection cap [S7, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of octree nodes, assuming constant-time QEF solving and lookup table access. The recursive traversal visits each node once [G].
*   **Memory Complexity:** $O(N)$ for storing the octree structure and vertex data. The octree size depends on the complexity of the scalar field and the maximum depth [G].
*   **Measured Performance:** For the Stanford Dragon model with an error threshold of $1e0$, the octree method produced 35,836 triangles, compared to 309,962 for a uniform grid [S4, unofficial]. A k-d tree method produced 22,608 triangles, indicating the octree method may produce more triangles than k-d tree variants for the same scene [S4, unofficial].
*   **Strengths:** Adaptive resolution allows for high detail in complex regions and coarser resolution in flat areas, reducing memory and processing time for sparse volumes [G]. It handles hierarchical simplification naturally.
*   **Limitations:** Can produce non-manifold geometry if not carefully constrained (e.g., using Manifold Dual Contouring). Adaptive simplification can be complex to implement correctly [S7, unofficial]. The fixed lookup tables require careful indexing to avoid errors [S2, unofficial].
*   **When to use:** When the scalar field is sparse or has varying levels of detail, and a manifold mesh is required (possibly with extensions).
*   **When not to use:** When a uniform grid is sufficient and simplicity of implementation is prioritized over adaptivity. When strict manifold guarantees are required without additional constraints.

## Background
The octree-based variant extends standard Dual Contouring by using a hierarchical grid structure to represent the scalar field, allowing for adaptive resolution [S3, S4, unofficial]. The method is based on works of Schaefer et al. (Manifold dual contouring) [S3, unofficial]. It is used in real destructible-voxel implementations (e.g., GPU dual contouring in Unity) which use classic octree LOD without meshlets [S1, unofficial].

## Sources
[S1] research(render,world): LOD strategy for terraformable planet terrain, https://github.com/lobinuxsoft/kooch/issues/614, fetched 2026-10-26 (unofficial)
[S2] nivshalomlom/Dual-contouring, https://github.com/nivshalomlom/Dual-contouring, fetched 2026-10-26 (unofficial)
[S3] Alan-Baylis/DualContouring-3, https://github.com/Alan-Baylis/DualContouring-3, fetched 2026-10-26 (unofficial)
[S4] Danielmelody/KdtreeISO, https://github.com/Danielmelody/KdtreeISO, fetched 2026-10-27 (unofficial)
[S5] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
[S6] Tsarpf/procgen, https://github.com/Tsarpf/procgen, fetched 2026-10-26 (unofficial)
[S7] PDF Manifold Dual Contouring, https://people.engr.tamu.edu/schaefer/research/dualsimp_tvcg.pdf, fetched 2026-10-27 (unofficial)
