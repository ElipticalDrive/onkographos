# Marching Cubes with Disjoint Patch Decomposition

Also called: Disjoint patch MC, MC with edge grouping

## Overview
Marching Cubes with Disjoint Patch Decomposition resolves topological ambiguities in isosurface extraction by decomposing the surface within a voxel into disjoint connected components (patches) of cut edges [S1, unofficial]. Unlike standard lookup tables that may produce non-manifold geometries in ambiguous cases, this method ensures topological correctness through graph-based coloring of edges [S1, unofficial]. It differs from its siblings by explicitly grouping edges into patches based on their connectivity in a tetrahedral decomposition graph, allowing for consistent manifold generation even in complex saddle-point scenarios [S1, unofficial].

## How it works
The method processes each voxel independently to determine the isosurface intersection [S1, unofficial]. It relies on a specific geometric decomposition of the cube and a graph-based coloring algorithm to assign edges to disjoint patches [S1, unofficial].

### Inputs and Outputs
**Inputs:**
- Scalar field values $v_i$ for the 8 vertices of the voxel cube, indexed $0 \dots 7$ [S1, unofficial].
- Threshold value $\theta$ (default $0.7$) [S1, unofficial].
- Voxel geometry (corner positions) to compute interpolated points [S1, unofficial].

**Outputs:**
- A set of triangles representing the isosurface intersection within the voxel [S1, unofficial].
- An `edgeGroup` code for the voxel, encoding the patch assignment for each of the 6 cube edges [S1, unofficial].

### Data Structures
1.  **Point Indices**: 15 points are defined per cube: vertices $0 \dots 7$, face centers $8 \dots 13$ (one per face), and cube center $14$ [S1, unofficial].
2.  **Face Definition**: The 6 faces are defined by their vertex indices:
    $$ \text{cubefaces}[6][4] = \begin{cases} \{0, 2, 6, 4\} \\ \{1, 5, 7, 3\} \\ \{0, 4, 5, 1\} \\ \{2, 3, 7, 6\} \\ \{0, 1, 3, 2\} \\ \{4, 6, 7, 5\} \end{cases} $$
3.  **Edge Group Code**: An array `edgeGroup[256]` where each entry is a 24-bit integer. Each of the 6 cube edges is encoded in 4 bits. Values $0 \dots 2$ represent distinct patch IDs, and $3$ represents an uncut/ambiguous edge [S1, unofficial].

### Algorithm Steps

1.  **Scalar Evaluation**:
    Compute scalar values for all 15 points based on the vertex signs (case index) and threshold $\theta$:
    -   Vertices $0 \dots 7$: Value is $1$ if vertex bit is set, $0$ otherwise.
    -   Face centers $8 \dots 13$: Average of the 4 vertex values of the corresponding face.
    -   Cube center $14$: Average of the 8 vertex values [S1, unofficial].

2.  **Edge Marking**:
    For each of the 12 cube edges $(u, v)$:
    -   If $(\text{vals}[u] > \theta) \oplus (\text{vals}[v] > \theta)$ is true, mark the edge as **cut** (type $0$).
    -   Otherwise, mark the edge as **uncut** (type $-1$) [S1, unofficial].

3.  **Connected Components of Uncut Edges**:
    -   Construct an adjacency matrix $A$ for the graph of all edges (from the tetrahedral decomposition). $A(u,v) = 1$ if edges $u$ and $v$ are connected in the graph AND both are uncut.
    -   Compute the transitive closure of $A$ by iterating $A = \text{sign}(A \times A)$ until convergence. This identifies connected components of uncut edges [S1, unofficial].

4.  **Patch Coloring (Disjoint Decomposition)**:
    -   Initialize all cut edges as uncolored.
    -   Iterate through all cut edges. If an edge is uncolored:
        -   Assign a new unique color ID (patch ID).
        -   Propagate this color ID to all other cut edges connected to it in the graph. Connectivity is determined by the transitive closure matrix $A$ (i.e., if two cut edges are in the same connected component of the graph, they receive the same color).
    -   Increment the color ID for the next new patch [S1, unofficial].

5.  **Lookup Code Generation**:
    -   For each of the 6 cube edges, determine its patch mark $mkid$.
    -   If the edge is uncut ($mkid = -1$), map to $3$.
    -   If the edge is cut, map to $mkid - 1$ (since patch IDs start at $0$).
    -   Pack these 4-bit values into a 24-bit integer code (6 edges $\times$ 4 bits) [S1, unofficial].

6.  **Mesh Generation**:
    -   Use the `edgeGroup` code to look up the triangulation pattern.
    -   For each cut edge, compute the intersection point using linear interpolation:
        $$ w = \frac{\theta - v_u}{v_v - v_u} $$
        $$ P_{\text{intersect}} = P_u + w (P_v - P_u) $$
    -   If an edge belongs to a specific patch, the intersection point is associated with that patch. In some implementations, if multiple cut edges belong to the same patch, their intersection points are averaged to form a single vertex for that patch, ensuring manifold connectivity [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_cubes_disjoint_patch(vertices, level, voxel_geometry):
    """
    vertices: list of 8 scalar values for cube corners.
    level: threshold theta (default 0.7).
    voxel_geometry: object with .corners[8] (list of 3-tuples).
    """
    cubefaces = [
        [0, 2, 6, 4], [1, 5, 7, 3], [0, 4, 5, 1],
        [2, 3, 7, 6], [0, 1, 3, 2], [4, 6, 7, 5]
    ]
    cube_edges = [
        (0, 1), (1, 3), (3, 2), (2, 0),
        (4, 5), (5, 7), (7, 6), (6, 4),
        (0, 4), (1, 5), (2, 6), (3, 7)
    ]

    # 1. Compute scalar values for 15 points
    vals = [0.0] * 15
    for i in range(8):
        vals[i] = 1.0 if vertices[i] > level else 0.0
    for f in range(6):
        vals[8 + f] = sum(vals[v] for v in cubefaces[f]) / 4.0
    vals[14] = sum(vals[i] for i in range(8)) / 8.0

    # 2. Build edge map and graph from tetrahedral decomposition
    edge_to_id = {}
    id_to_edge = {}
    graph = {}

    def add_edge(u, v):
        if u > v: u, v = v, u
        eid = min(u, v) * 100 + max(u, v)
        if eid not in edge_to_id:
            idx = len(edge_to_id)
            edge_to_id[eid] = idx
            id_to_edge[idx] = (u, v)
            graph[idx] = set()
        return edge_to_id[eid]

    for f in range(6):
        fc = 8 + f
        cc = 14
        for j in range(4):
            tet = [cubefaces[f][(j + 1) & 3], cubefaces[f][j], fc, cc]
            i_edges = [0, 0, 0, 1, 1, 2]
            j_edges = [1, 2, 3, 2, 3, 3]
            tet_edge_ids = []
            for k in range(6):
                u, v = tet[i_edges[k]], tet[j_edges[k]]
                tet_edge_ids.append(add_edge(u, v))
            for k in range(6):
                for ks in range(k + 1, 6):
                    graph[tet_edge_ids[k]].add(tet_edge_ids[ks])
                    graph[tet_edge_ids[ks]].add(tet_edge_ids[k])

    # 3. Mark cube edges
    edge_marks = {}
    for idx in edge_to_id:
        u, v = id_to_edge[idx]
        if (vals[u] > level) ^ (vals[v] > level):
            edge_marks[idx] = 0
        else:
            edge_marks[idx] = -1

    # 4. Connected components of uncut edges via transitive closure
    n_edges = len(edge_to_id)
    adj = [[0] * n_edges for _ in range(n_edges)]
    for u in range(n_edges):
        if edge_marks[u] != -1: continue
        for v in graph[u]:
            if edge_marks[v] != -1: continue
            adj[u][v] = 1

    changed = True
    while changed:
        changed = False
        new_adj = [[0] * n_edges for _ in range(n_edges)]
        for i in range(n_edges):
            for j in range(n_edges):
                val = 0
                for k in range(n_edges):
                    if adj[i][k] and adj[k][j]:
                        val = 1
                        break
                new_adj[i][j] = val
                if val != adj[i][j]: changed = True
        adj = new_adj

    # 5. Color cut edges into disjoint patches
    patch_colors = {}
    current_patch_id = 0
    for idx in range(n_edges):
        if edge_marks[idx] != 0: continue
        if idx in patch_colors: continue
        patch_colors[idx] = current_patch_id
        for other in range(n_edges):
            if edge_marks[other] != 0: continue
            if other in patch_colors: continue
            if adj[idx][other]:
                patch_colors[other] = current_patch_id
        current_patch_id += 1

    # 6. Generate lookup code
    edge_group_code = 0
    for i, (u, v) in enumerate(cube_edges):
        if u > v: u, v = v, u
        eid = min(u, v) * 100 + max(u, v)
        gid = edge_to_id[eid]
        mkid = patch_colors.get(gid, -1)
        patch_val = 3 if mkid == -1 else mkid
        edge_group_code |= (patch_val << (4 * i))

    # 7. Generate mesh
    patch_points = {}
    for i, (u, v) in enumerate(cube_edges):
        if u > v: u, v = v, u
        eid = min(u, v) * 100 + max(u, v)
        gid = edge_to_id[eid]
        if edge_marks[gid] != 0: continue
        w = (level - vals[u]) / (vals[v] - vals[u])
        pu = voxel_geometry.corners[u]
        pv = voxel_geometry.corners[v]
        p = (pu[0] + w * (pv[0] - pu[0]), pu[1] + w * (pv[1] - pu[1]), pu[2] + w * (pv[2] - pu[2]))
        pid = patch_colors[gid]
        if pid not in patch_points: patch_points[pid] = []
        patch_points[pid].append(p)

    final_points = {}
    for pid, pts in patch_points.items():
        avg = (sum(p[0] for p in pts)/len(pts), sum(p[1] for p in pts)/len(pts), sum(p[2] for p in pts)/len(pts))
        final_points[pid] = avg

    # Standard lookup table expansion using edge_group_code and final_points
    triangles = []
    # ... expand edge_group_code into triangle indices referencing final_points ...
    return triangles, edge_group_code
```

### Edge Cases and Pitfalls
-   **Precision**: Linear interpolation assumes the scalar field is linear within the voxel. For highly non-linear fields, this may introduce artifacts.
-   **Boundary Conditions**: The method assumes a regular grid. Handling boundaries requires special care to ensure patch consistency across voxel edges.
-   **Degenerate Input**: If the threshold $\theta$ exactly equals a vertex value, the sign is ambiguous. A small epsilon should be added to the comparison.

## Subtopics

### [Ambiguous Face Handling with Connected Components](6727-ambiguous-face-handling-with-connected-components.md)
Resolves ambiguities by selecting connected components based on the count of ambiguous faces, enforcing a white/blue selection rule.

## Operating characteristics
-   **Time Complexity**: The graph construction and transitive closure are performed per voxel. The transitive closure is $O(E^3)$ where $E$ is the number of edges in the local graph (constant for a cube). Thus, it is $O(1)$ per voxel.
-   **Memory Complexity**: $O(1)$ per voxel for the graph and coloring arrays.
-   **Strengths**: Guarantees manifold topology in ambiguous cases. Handles saddle points correctly.
-   **Limitations**: More complex to implement than standard Marching Cubes. Requires careful handling of patch averaging to avoid geometric distortion.
-   **When to Use**: When topological correctness is critical and the scalar field contains ambiguous configurations (e.g., medical imaging, scientific visualization).

## Background
The method was introduced as a variant of Marching Cubes to address the topological ambiguities inherent in the standard lookup table approach. It uses a tetrahedral decomposition of the cube to define a graph of edge connectivity, allowing for a robust coloring algorithm to identify disjoint surface patches.

## Sources
[S1] uranix/mcubes, https://github.com/uranix/mcubes, fetched 2026-10-27 (unofficial)
