# Isosurface Extraction with Topological Optimization

Also called: Topological Optimization for Marching Cubes

## Overview
This method extracts a triangle mesh from a 3D scalar field by optimizing global topological properties—specifically triangle count, genus, and number of connected components (shells)—rather than relying on local disambiguation rules [S1]. It solves the ambiguity inherent in Marching Cubes configurations by identifying specific degrees of freedom (X-faces and X-cubes) and making global decisions to minimize topological complexity [S1]. Unlike standard Marching Cubes or its variants [774-marching-cubes.md], [1393-marching-cubes-2.md], which use precomputed lookup tables or local heuristics, this approach uses auxiliary data structures to ensure the resulting mesh is two-manifold and topologically optimal [S1].

## How it works
**Inputs:** A regular, axis-aligned 3D lattice of nodes with scalar values; an isolevel threshold [S1].
**Outputs:** A two-manifold triangle mesh $M$ approximating the isosurface [S1].

**Data Structures:**
1. **Grid:** A 3D array of nodes. Each node is classified as **Black** (inside or on boundary) or **White** (outside) [S1].
2. **Cube:** Spans the interstice between 8 lattice nodes. Has 12 edges and 6 faces [S1].
3. **X-face Graph:** Captures connectivity between ambiguous faces to propagate decisions [S1].
4. **Merge Tree:** Tracks equivalence classes of loops to manage connectivity decisions in X-cubes [S1].

**Definitions:**
* **Border Edge:** An edge of the mesh $M$ lying on a face of a cube. It connects two vertices on that face [S1].
* **Loop:** A simple cycle formed by chaining border edges within a single cube. A cube can contain 0 to 4 loops [S1].
* **Sheet:** A connected component of triangles within a cube, bounded by one or more loops [S1].
* **X-face:** A face with alternating Black and White corners (e.g., B-W-B-W). It contains two mesh vertices and two possible border edges. The algorithm must choose one [S1].
* **X-cube:** A cube with no X-faces but containing more than one loop. This occurs in specific Marching Cubes configurations (e.g., the configuration with two disjoint loops). The algorithm must decide whether to connect the loops (forming an annulus/tunnel) or keep them separate (forming two disks) [S1].

**Steps:**

1. **Classification:** Classify every grid node as Black if $value(node) \ge threshold$, else White [S1].
2. **Vertex Placement:** Place mesh vertices at the midpoints of lattice edges connecting a Black node and a White node. Vertices may be adjusted along the edge for geometric fidelity if no self-intersection occurs, but midpoints are the standard topological choice [S1].
3. **Identify X-faces:** Iterate over all faces of all cubes. Identify faces with alternating corner colors. These are X-faces. Each X-face has two candidate border edges [S1].
4. **Identify X-cubes:** Iterate over all cubes. Identify cubes that have no X-faces but contain $>1$ loop. These are X-cubes [S1].
5. **Global Decision on X-faces:** Use the X-face graph to make consistent slash decisions across the volume. The goal is to optimize the target metric (e.g., minimize genus). This ensures that border edges match between adjacent cubes, preserving the manifold property [S1].
6. **Global Decision on X-cubes:** Use the Merge Tree to decide loop connectivity in X-cubes. For each X-cube, decide whether to connect the loops (reducing the number of shells/components) or keep them separate. This decision is global to optimize the total number of shells or genus [S1].
7. **Triangulation:** For each cube, generate triangles based on the chosen border edges (from X-face decisions) and loop connections (from X-cube decisions). Triangles must lie entirely within a single cube. Triangles lying on the boundary face of a cube are excluded to ensure topological validity and allow vertex sliding [S1].
8. **Assembly:** Combine triangles from all cubes into the final mesh $M$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_isosurface(grid, threshold):
    # Step 1: Classification
    nodes = [[[grid[i,j,k] >= threshold for k in range(grid.shape[2])]
              for j in range(grid.shape[1])]
             for i in range(grid.shape[0])]

    # Step 2: Identify Ambiguities & Compute Border Edges
    x_faces = []
    x_cubes = []
    cubes = []

    for i in range(grid.shape[0]-1):
        for j in range(grid.shape[1]-1):
            for k in range(grid.shape[2]-1):
                cube = Cube(i, j, k, nodes)
                cube.compute_border_edges()
                cube.identify_x_faces()
                cube.compute_loops()
                cubes.append(cube)
                if cube.is_x_cube():
                    x_cubes.append(cube)
                x_faces.extend(cube.x_faces)

    # Step 5: Global Decision on X-faces
    x_face_decisions = optimize_x_faces(x_faces)

    # Step 6: Global Decision on X-cubes
    x_cube_decisions = optimize_x_cubes(x_cubes)

    # Step 7: Triangulation
    mesh_triangles = []
    for cube in cubes:
        cube.apply_decisions(x_face_decisions, x_cube_decisions)
        mesh_triangles.extend(cube.triangulate())

    return mesh_triangles

class Cube:
    # Edge indices for a cube: 0-3 bottom, 4-7 top, 8-11 vertical
    EDGE_MAP = [
        (0,1), (1,2), (2,3), (3,0),
        (4,5), (5,6), (6,7), (7,4),
        (0,4), (1,5), (2,6), (3,7)
    ]
    # Face corner indices (bottom 0, top 1)
    FACE_MAP = [
        [(0,0), (1,0), (2,0), (3,0)], # bottom
        [(0,1), (1,1), (2,1), (3,1)], # top
        [(0,0), (1,0), (1,1), (0,1)], # front
        [(1,0), (2,0), (2,1), (1,1)], # right
        [(2,0), (3,0), (3,1), (2,1)], # back
        [(3,0), (0,0), (0,1), (3,1)]  # left
    ]

    def __init__(self, i, j, k, nodes):
        self.i, self.j, self.k = i, j, k
        self.nodes = nodes
        # Extract 8 corners: [z][y][x] relative to cube origin
        self.corners = [[nodes[i][j][k], nodes[i+1][j][k], nodes[i+1][j+1][k], nodes[i][j+1][k]],
                        [nodes[i][j][k+1], nodes[i+1][j][k+1], nodes[i+1][j+1][k+1], nodes[i][j+1][k+1]]]
        self.border_edges = []
        self.x_faces = []
        self.loops = []
        self.chosen_slash = {}
        self.connect_loops = False

    def compute_border_edges(self):
        for idx, (c1, c2) in enumerate(self.EDGE_MAP):
            n1 = self.corners[0][c1] if idx < 8 else self.corners[1][c1-8]
            n2 = self.corners[0][c2] if idx < 8 else self.corners[1][c2-8]
            if n1 != n2:
                self.border_edges.append(idx)

    def identify_x_faces(self):
        for f_idx, corners in enumerate(self.FACE_MAP):
            c0, c1, c2, c3 = [self.corners[z][x] for z, x in corners]
            if c0 != c1 and c1 != c2 and c2 != c3 and c3 != c0:
                self.x_faces.append(f_idx)

    def compute_loops(self):
        # Chain border edges on cube faces to form loops
        visited_edges = set()
        self.loops = []
        for edge_idx in self.border_edges:
            if edge_idx in visited_edges:
                continue
            loop = []
            curr = edge_idx
            while curr not in visited_edges:
                visited_edges.add(curr)
                loop.append(curr)
                # Find next edge on same face
                next_edge = self._next_edge_on_face(curr)
                if next_edge is None or next_edge in visited_edges:
                    break
                curr = next_edge
            if len(loop) > 0:
                self.loops.append(loop)

    def _next_edge_on_face(self, edge_idx):
        # Determine which face contains this edge and find the next border edge on it
        for f_idx, corners in enumerate(self.FACE_MAP):
            face_edges = self._edges_of_face(f_idx)
            if edge_idx in face_edges:
                for e in face_edges:
                    if e != edge_idx and e in self.border_edges:
                        return e
        return None

    def _edges_of_face(self, f_idx):
        # Map face corners to edge indices
        corners = self.FACE_MAP[f_idx]
        edges = []
        for i in range(4):
            c1 = corners[i]
            c2 = corners[(i+1)%4]
            for e_idx, (n1, n2) in enumerate(self.EDGE_MAP):
                if (c1 == n1 and c2 == n2) or (c1 == n2 and c2 == n1):
                    edges.append(e_idx)
        return edges

    def is_x_cube(self):
        return len(self.x_faces) == 0 and len(self.loops) > 1

    def apply_decisions(self, x_face_decisions, x_cube_decisions):
        # Apply X-face slash decisions
        for f_idx in self.x_faces:
            self.chosen_slash[f_idx] = x_face_decisions.get((self.i, self.j, self.k, f_idx), 0)
        # Apply X-cube loop connection decision
        if self.is_x_cube():
            self.connect_loops = x_cube_decisions.get((self.i, self.j, self.k), False)

    def triangulate(self):
        triangles = []
        # Standard triangulation based on computed loops and slash decisions
        # Exclude triangles on cube boundary faces
        for loop in self.loops:
            if len(loop) >= 3:
                # Fan triangulation from first vertex of loop
                v0 = self._edge_midpoint(loop[0])
                for i in range(1, len(loop)-1):
                    v1 = self._edge_midpoint(loop[i])
                    v2 = self._edge_midpoint(loop[i+1])
                    triangles.append((v0, v1, v2))
        return triangles

    def _edge_midpoint(self, edge_idx):
        # Return midpoint coordinates (normalized 0-1 for simplicity)
        c1, c2 = self.EDGE_MAP[edge_idx]
        z1 = 0 if c1 < 4 else 1
        z2 = 0 if c2 < 4 else 1
        x1, y1 = c1 % 4, c1 // 4
        x2, y2 = c2 % 4, c2 // 4
        return ((x1+x2)/2, (y1+y2)/2, (z1+z2)/2)

def optimize_x_faces(x_faces):
    # Build X-face graph and resolve slashes to optimize genus/triangle count
    # Returns dict mapping (i,j,k,face_idx) -> chosen_slash_index (0 or 1)
    decisions = {}
    for face in x_faces:
        # Simplified resolution: choose slash 0 consistently for implementation
        # In practice, solve graph coloring/optimization problem
        decisions[(face.i, face.j, face.k, face.f_idx)] = 0
    return decisions

def optimize_x_cubes(x_cubes):
    # Build Merge Tree and decide loop connectivity
    # Returns dict mapping (i,j,k) -> connect_loops (bool)
    decisions = {}
    for cube in x_cubes:
        # Simplified resolution: connect loops to minimize shells
        decisions[(cube.i, cube.j, cube.k)] = True
    return decisions
```

## Subtopics

### [Volume-Based Topology Simplification via Augmented Reeb Graph](9674-volume-based-topology-simplification-via-augmented-reeb-grap.md)
Removes topological noise (handles) by sweeping through the volume and encoding topology in an augmented Reeb graph. It differs by using an axis-aligned sweep for out-of-core processing and local cut-graph search, whereas this method uses global X-face/X-cube optimization.

## Operating characteristics
* **Complexity:** Described as "very efficient" due to the use of efficient implementations and auxiliary data structures (X-face graph, Merge Tree) [S1].
* **Strengths:** Guarantees a two-manifold mesh. Optimizes global topological properties (genus, triangle count, number of shells) which local methods cannot control [S1].
* **Limitations:** Requires global data structures and decisions, which may be more complex to implement than local lookup tables [S1].
* **When to use:** When topological correctness and optimization of genus/triangle count are critical, and the input data allows for global analysis [S1].
* **When not to use:** When real-time performance is strictly required and global optimization overhead is prohibitive, or when local artifacts are acceptable [S1].

## Background
Originated in the paper "Optimizing the topological and combinatorial complexity of isosurfaces" (2005) [S1]. It addresses the limitations of local disambiguation strategies in Marching Cubes, which cannot control global properties like genus or triangle count. The method identifies X-faces and X-cubes as the primary sources of topological ambiguity and provides a framework for global optimization [S1].

## Sources
[S1] Optimizing the topological and combinatorial complexity of isosurfaces, https://doi.org/10.1016/j.cad.2004.09.013, fetched 2026-10-30
