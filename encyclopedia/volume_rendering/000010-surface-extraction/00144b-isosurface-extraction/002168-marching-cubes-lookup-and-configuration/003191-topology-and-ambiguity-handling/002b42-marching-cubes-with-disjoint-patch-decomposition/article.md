# Marching Cubes with Disjoint Patch Decomposition

Also called: Disjoint patch MC, MC with edge grouping

## Overview
Marching Cubes with Disjoint Patch Decomposition resolves topological ambiguities in isosurface extraction by decomposing the surface within a voxel into disjoint connected components (patches) of cut edges. Unlike standard lookup tables that may produce non-manifold geometries in ambiguous cases, this method ensures topological correctness through graph-based coloring of edges [S1, unofficial]. It differs from its siblings by explicitly grouping edges into patches based on their connectivity in a tetrahedral decomposition graph, allowing for consistent manifold generation even in complex saddle-point scenarios.

## How it works
The method processes each voxel independently to determine the isosurface intersection. It relies on a specific geometric decomposition of the cube and a graph-based coloring algorithm to assign edges to disjoint patches.

### Inputs and Outputs
**Inputs:**
- Scalar field values $v_i$ for the 8 vertices of the voxel cube, indexed $0 \dots 7$.
- Threshold value $\theta$ (default $0.7$) [S1, unofficial].
- Voxel geometry (corner positions and dimensions) to compute interpolated points.

**Outputs:**
- A set of triangles (or quads, depending on implementation) representing the isosurface intersection within the voxel.
- An `edgeGroup` code for the voxel, encoding the patch assignment for each of the 6 cube edges.

### Data Structures
1.  **Point Indices**: 15 points are defined per cube:
    -   Vertices $0 \dots 7$.
    -   Face centers $8 \dots 13$ (one per face).
    -   Cube center $14$.
2.  **Face Definition**: The 6 faces are defined by their vertex indices:
    $$ \text{cubefaces}[6][4] = \begin{cases} \{0, 2, 6, 4\} \\ \{1, 5, 7, 3\} \\ \{0, 4, 5, 1\} \\ \{2, 3, 7, 6\} \\ \{0, 1, 3, 2\} \\ \{4, 6, 7, 5\} \end{cases} $$
3.  **Tetrahedral Decomposition**: The cube is decomposed into 24 tetrahedra. For each face $f \in \{0 \dots 5\}$, 4 tetrahedra are formed using the face vertices, the face center ($8+f$), and the cube center ($14$). Specifically, for $j \in \{0 \dots 3\}$:
    $$ \text{tet} = \{ \text{cubefaces}[f][(j+1)\&3], \text{cubefaces}[f][j], 8+f, 14 \} $$
4.  **Edge Map**: A mapping from unique geometric edges to sequential indices $0 \dots N-1$. Edges are identified by pairs of point indices $(u, v)$ with $u < v$. The edge ID is computed as $\min(u,v) \times 100 + \max(u,v)$ to ensure uniqueness across the local graph.
5.  **Adjacency Graph**: An adjacency list representing connectivity between edges. Two edges are connected if they belong to the same tetrahedron.
6.  **Edge Group Code**: An array `edgeGroup[256]` where each entry is a 24-bit integer. Each of the 6 cube edges is encoded in 4 bits. Values $0 \dots 2$ represent distinct patch IDs, and $3$ represents an uncut/ambiguous edge.

### Algorithm Steps

1.  **Scalar Evaluation**:
    Compute scalar values for all 15 points based on the vertex signs (case index) and threshold $\theta$:
    -   Vertices $0 \dots 7$: Value is $1$ if vertex bit is set, $0$ otherwise.
    -   Face centers $8 \dots 13$: Average of the 4 vertex values of the corresponding face.
    -   Cube center $14$: Average of the 8 vertex values.

2.  **Edge Marking**:
    For each of the 12 cube edges $(u, v)$:
    -   If $(\text{vals}[u] > \theta) \oplus (\text{vals}[v] > \theta)$ is true, mark the edge as **cut** (type $0$).
    -   Otherwise, mark the edge as **uncut** (type $-1$).

3.  **Connected Components of Uncut Edges**:
    -   Construct an adjacency matrix $A$ for the graph of all edges (from the tetrahedral decomposition). $A(u,v) = 1$ if edges $u$ and $v$ are connected in the graph AND both are uncut.
    -   Compute the transitive closure of $A$ by iterating $A = \text{sign}(A \times A)$ until convergence. This identifies connected components of uncut edges.

4.  **Patch Coloring (Disjoint Decomposition)**:
    -   Initialize all cut edges as uncolored.
    -   Iterate through all cut edges. If an edge is uncolored:
        -   Assign a new unique color ID (patch ID).
        -   Propagate this color ID to all other cut edges connected to it in the graph. Connectivity is determined by the transitive closure matrix $A$ (i.e., if two cut edges are in the same connected component of the graph, they receive the same color).
    -   Increment the color ID for the next new patch.

5.  **Lookup Code Generation**:
    -   For each of the 6 cube edges, determine its patch mark $mkid$.
    -   If the edge is uncut ($mkid = -1$), map to $3$.
    -   If the edge is cut, map to $mkid - 1$ (since patch IDs start at $0$).
    -   Pack these 4-bit values into a 24-bit integer code (6 edges $\times$ 4 bits).

6.  **Mesh Generation**:
    -   Use the `edgeGroup` code to look up the triangulation pattern.
    -   For each cut edge, compute the intersection point using linear interpolation:
        $$ w = \frac{\theta - v_u}{v_v - v_u} $$
        $$ P_{\text{intersect}} = P_u + w (P_v - P_u) $$
    -   If an edge belongs to a specific patch, the intersection point is associated with that patch. In some implementations, if multiple cut edges belong to the same patch, their intersection points are averaged to form a single vertex for that patch, ensuring manifold connectivity [S1, unofficial].
    -   Generate triangles/quads based on the patch assignments. If `NON_SYMMETRIC_SPLIT` is enabled, triangle quality metrics may be used to decide between diagonal splits in quad faces.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_cubes_disjoint_patch(vertices, level, voxel_geometry):
    """
    vertices: array of 8 scalar values for cube corners.
    level: threshold theta.
    voxel_geometry: provides positions for corners, face centers, and cube center.
    """
    # 1. Define 15 points
    points = [None] * 15
    for i in range(8):
        points[i] = voxel_geometry.corners[i]
    for f in range(6):
        points[8 + f] = voxel_geometry.face_centers[f]
    points[14] = voxel_geometry.center

    # 2. Compute scalar values for 15 points
    vals = [0.0] * 15
    for i in range(8):
        vals[i] = 1.0 if vertices[i] > level else 0.0
    
    # Face centers: average of 4 vertices
    for f in range(6):
        face_verts = cubefaces[f]
        vals[8 + f] = sum(vals[v] for v in face_verts) / 4.0
    
    # Cube center: average of 8 vertices
    vals[14] = sum(vals[i] for i in range(8)) / 8.0

    # 3. Build graph of edges from tetrahedral decomposition
    # Edges are identified by (u, v) with u < v
    edge_to_id = {}
    id_to_edge = {}
    graph = {} # adjacency list
    
    # Helper to add edge to graph
    def add_graph_edge(u, v):
        if u > v: u, v = v, u
        edge_key = (u, v)
        if edge_key not in edge_to_id:
            new_id = len(edge_to_id)
            edge_to_id[edge_key] = new_id
            id_to_edge[new_id] = edge_key
            graph[new_id] = set()
        else:
            new_id = edge_to_id[edge_key]
        
        # Add connectivity for all pairs of edges in a tet
        # This is handled by iterating tets below
    
    # Create tets and build graph
    for f in range(6):
        face_verts = cubefaces[f]
        fc = 8 + f
        cc = 14
        for j in range(4):
            v1 = face_verts[(j + 1) & 3]
            v2 = face_verts[j]
            tet_verts = [v1, v2, fc, cc]
            # Add all 6 edges of this tet to the graph
            for i in range(4):
                for k in range(i + 1, 4):
                    u, v = tet_verts[i], tet_verts[k]
                    if u > v: u, v = v, u
                    edge_key = (u, v)
                    if edge_key not in edge_to_id:
                        new_id = len(edge_to_id)
                        edge_to_id[edge_key] = new_id
                        id_to_edge[new_id] = edge_key
                        graph[new_id] = set()
                    else:
                        new_id = edge_to_id[edge_key]
                    
                    # Connect this edge to other edges in the tet
                    for m in range(4):
                        for n in range(m + 1, 4):
                            if i == m and k == n: continue
                            u2, v2 = tet_verts[m], tet_verts[n]
                            if u2 > v2: u2, v2 = v2, u2
                            edge_key2 = (u2, v2)
                            if edge_key2 not in edge_to_id:
                                new_id2 = len(edge_to_id)
                                edge_to_id[edge_key2] = new_id2
                                id_to_edge[new_id2] = edge_key2
                                graph[new_id2] = set()
                            else:
                                new_id2 = edge_to_id[edge_key2]
                            
                            graph[new_id].add(new_id2)
                            graph[new_id2].add(new_id)

    # 4. Mark cube edges
    cube_edge_indices = [0, 1, 2, 3, 4, 5] # Standard MC edge indices
    cube_edges = [
        (0, 1), (1, 3), (3, 2), (2, 0), # Bottom
        (4, 5), (5, 7), (7, 6), (6, 4), # Top
        (0, 4), (1, 5), (2, 6), (3, 7)  # Vertical
    ]
    
    # Map cube edge index to graph edge ID
    cube_edge_graph_ids = []
    for u, v in cube_edges:
        if u > v: u, v = v, u
        edge_key = (u, v)
        if edge_key in edge_to_id:
            cube_edge_graph_ids.append(edge_to_id[edge_key])
        else:
            cube_edge_graph_ids.append(None) # Should not happen

    edge_marks = [-1] * len(edge_to_id) # -1: uncut, 0: cut
    for gid in edge_to_id:
        u, v = id_to_edge[gid]
        if (vals[u] > level) ^ (vals[v] > level):
            edge_marks[gid] = 0
        else:
            edge_marks[gid] = -1

    # 5. Compute connected components of uncut edges
    # Build adjacency matrix for uncut edges
    n_edges = len(edge_to_id)
    adj_matrix = [[0] * n_edges for _ in range(n_edges)]
    for u in range(n_edges):
        if edge_marks[u] != -1: continue
        for v in graph[u]:
            if edge_marks[v] != -1: continue
            adj_matrix[u][v] = 1
            
    # Transitive closure
    for k in range(n_edges):
        for i in range(n_edges):
            for j in range(n_edges):
                adj_matrix[i][j] = 1 if (adj_matrix[i][j] or (adj_matrix[i][k] and adj_matrix[k][j])) else 0

    # 6. Color cut edges into disjoint patches
    patch_colors = [-1] * n_edges
    current_patch_id = 0
    
    for gid in range(n_edges):
        if edge_marks[gid] != 0: continue
        if patch_colors[gid] != -1: continue
        
        # Start new patch
        patch_colors[gid] = current_patch_id
        
        # Propagate color to all connected cut edges
        for other_gid in range(n_edges):
            if edge_marks[other_gid] != 0: continue
            if patch_colors[other_gid] != -1: continue
            if adj_matrix[gid][other_gid]:
                patch_colors[other_gid] = current_patch_id
        
        current_patch_id += 1

    # 7. Generate Lookup Code
    edge_group_code = 0
    for i, gid in enumerate(cube_edge_graph_ids):
        if gid is None:
            patch_val = 3
        else:
            mkid = patch_colors[gid]
            if mkid == -1:
                patch_val = 3
            else:
                patch_val = mkid
        edge_group_code |= (patch_val << (4 * i))

    # 8. Generate Mesh
    # Use edge_group_code to determine triangles
    # Interpolate points for cut edges
    new_points = {} # patch_id -> list of points
    for i, gid in enumerate(cube_edge_graph_ids):
        if gid is None: continue
        if edge_marks[gid] != 0: continue
        
        u, v = id_to_edge[gid]
        w = (level - vals[u]) / (vals[v] - vals[u])
        p = interpolate_point(points[u], points[v], w)
        
        patch_id = patch_colors[gid]
        if patch_id not in new_points:
            new_points[patch_id] = []
        new_points[patch_id].append(p)

    # Average points per patch for manifold connectivity
    final_points = {}
    for pid, pts in new_points.items():
        avg_p = sum(pts) / len(pts)
        final_points[pid] = avg_p

    # Generate triangles based on edge_group_code and final_points
    # This part depends on the specific lookup table implementation
    # which maps the 4-bit patch codes to triangle indices.
    triangles = generate_triangles_from_code(edge_group_code, final_points)
    
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
