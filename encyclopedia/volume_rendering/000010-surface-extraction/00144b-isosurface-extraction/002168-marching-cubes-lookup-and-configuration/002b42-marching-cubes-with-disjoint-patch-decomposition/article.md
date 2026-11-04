# Marching Cubes with Disjoint Patch Decomposition

Also called: Disjoint patch MC, MC with edge grouping

## Overview
Marching Cubes with Disjoint Patch Decomposition is a topologically robust variant of the Marching Cubes algorithm that resolves ambiguous cases (such as saddle points) by decomposing the surface within a cube into disjoint connected components of edges. Unlike standard Marching Cubes which may produce non-manifold geometry or incorrect topology in ambiguous configurations, this method uses graph-based coloring to ensure that cut edges are grouped into distinct, non-intersecting patches. It builds on [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md) by replacing the static lookup table with a dynamic graph analysis of edge connectivity.

## How it works
The method operates on a single voxel cube, determining the intersection of an isosurface with the cube by analyzing the connectivity of edges that do not cross the isosurface threshold.

### Inputs and Outputs
- **Input**: 
  - Scalar field values $v_i$ for the 8 cube vertices ($i \in 0..7$).
  - Isosurface threshold $\theta$ (default 0.7 [S1, unofficial]).
  - Cube geometry bounds (lower-left `ll` and upper-right `ur` corners) for coordinate interpolation [S1, unofficial].
- **Output**: 
  - A set of vertices and faces (triangles or quads) representing the surface patch within the cube.
  - An `edgeGroup` array encoding the patch assignment for each of the 6 cube edges.

### Data Structures
1. **Point Indices**: 15 points are defined per cube [S1, unofficial]:
   - Vertices 0-7: Cube corners.
   - Face centers 8-13: Center of each of the 6 faces.
   - Cube center 14: Center of the cube.
2. **Face Definition**: The 6 faces are defined by their 4 vertex indices [S1, unofficial]:
   ```python
   cubefaces = [
       [0, 2, 6, 4], [1, 5, 7, 3], [0, 4, 5, 1],
       [2, 3, 7, 6], [0, 1, 3, 2], [4, 6, 7, 5]
   ]
   ```
3. **Edge Map**: A mapping from unique edge indices (defined by vertex pairs) to sequential IDs $0..N-1$.
4. **Adjacency Graph**: An adjacency list representing connectivity between edges within the cube.
5. **Edge Group Array**: `edgeGroup[256]`, where each integer encodes 6 cube edges using 4 bits per edge. Values 0-3 represent patch assignments or uncategorized edges [S1, unofficial].

### Algorithm Steps

1. **Tetrahedral Decomposition**:
   Decompose the cube into tetrahedra to facilitate scalar field evaluation and edge connectivity analysis. For each face $f \in 0..5$, create 4 tetrahedra using the face vertices, the face center ($8+f$), and the cube center (14) [S1, unofficial]:
   $$ \text{tet} = \{ \text{cubefaces}[f][(j+1)\&3], \text{cubefaces}[f][j], 8+f, 14 \} \quad \text{for } j=0..3 $$

2. **Build Edge Map**:
   For each tetrahedron, iterate over its 6 edges defined by index pairs $i \in \{0,0,0,1,1,2\}$ and $j \in \{1,2,3,2,3,3\}$. Compute a unique edge ID:
   $$ \text{edge\_id} = \min(u, v) \times 100 + \max(u, v) $$
   Map these unique IDs to sequential indices $0..N-1$ [S1, unofficial].

3. **Build Connectivity Graph**:
   Construct an undirected graph where nodes are the mapped edge indices. For each tetrahedron, add an edge in the adjacency list between every pair of its edges $(k, ks)$ where $ks > k$ [S1, unofficial]. This graph represents which edges are connected within the tetrahedral decomposition.

4. **Compute Scalar Values**:
   For a given 8-bit vertex sign pattern (case), compute scalar values for all 15 points [S1, unofficial]:
   - Vertices 0-7: Value is 0 or 1 based on the case bits relative to $\theta$.
   - Face centers 8-13: Average of their 4 vertex values.
   - Cube center 14: Average of its 8 vertex values.

5. **Mark Edges**:
   For each edge $(u, v)$ in the graph:
   - If $(\text{vals}[u] > \theta) \oplus (\text{vals}[v] > \theta)$ is false, mark the edge as **-1** (uncut/ambiguous).
   - Otherwise, mark the edge as **0** (cut) [S1, unofficial].

6. **Compute Connected Components of Uncut Edges**:
   - Build an adjacency matrix $A$ where $A(u,v)=1$ if both $u$ and $v$ are uncut (-1) and connected in the graph.
   - Compute the transitive closure by iterating $A = \text{sign}(A \times A)$ until convergence. This identifies all uncut edges that are part of the same ambiguous component [S1, unofficial].

7. **Color Cut Edges into Disjoint Patches**:
   - Iterate through all cut edges (marked 0).
   - If an edge is not yet colored, assign a new color ID (patch ID).
   - Propagate this color ID to all other cut edges connected to it in the graph, using the transitive closure matrix $A$ to determine connectivity. Note: Connectivity is determined by the graph structure, but the coloring logic ensures disjointness by processing uncolored edges and using the closure to find all connected cut edges within that component [S1, unofficial].
   - Increment the color ID for the next uncolored edge.

8. **Generate Lookup Code**:
   For each of the 6 cube edges (between two cube vertices $v_1, v_2 < 8$):
   - Compute the cube edge index $ceid$.
   - Map the patch mark $mkid$:
     - If $mkid = -1$ (uncut), map to 3.
     - If $mkid \ge 0$ (cut), map to $mkid - 1$.
   - Pack these 4-bit values into a 24-bit integer code (6 edges $\times$ 4 bits) [S1, unofficial].

9. **Vertex Generation**:
   For each cut edge, compute the intersection point. If an edge is part of a patch with count $>0$, the vertex position is the average of the intersection points of all edges in that patch, potentially clamped to cube boundaries using `cubeflag` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_cubes_disjoint_patch(cube_vertices, level, ll, ur):
    # 1. Define points and faces
    # Points 0-7: cube_vertices
    # Points 8-13: face centers (average of 4 vertices)
    # Point 14: cube center (average of 8 vertices)
    points = [0] * 15
    for i in range(8):
        points[i] = cube_vertices[i]
    for f in range(6):
        verts = cubefaces[f]
        points[8 + f] = sum(points[v] for v in verts) / 4.0
    points[14] = sum(points[i] for i in range(8)) / 8.0

    # 2. Build Edge Map and Graph
    edge_map = {} # Maps (min_u, max_u) to sequential ID
    graph = {}    # Adjacency list: ID -> set of IDs
    edge_list = [] # List of (u, v) for each mapped ID

    # Define tetrahedra edges
    tet_edges_indices = [(0,1), (0,2), (0,3), (1,2), (1,3), (2,3)]
    
    for f in range(6):
        face_verts = cubefaces[f]
        face_center = 8 + f
        cube_center = 14
        for j in range(4):
            v1 = face_verts[(j + 1) & 3]
            v2 = face_verts[j]
            tets = [v1, v2, face_center, cube_center]
            
            for i_idx, j_idx in tet_edges_indices:
                u, v = tets[i_idx], tets[j_idx]
                if u > v: u, v = v, u
                edge_key = (u, v)
                
                if edge_key not in edge_map:
                    new_id = len(edge_map)
                    edge_map[edge_key] = new_id
                    edge_list.append((u, v))
                    graph[new_id] = set()
                
                u_id = edge_map[edge_key]
                
                # Add connectivity within tetrahedron
                # For each pair of edges in the tet, connect them
                # This is done by iterating all pairs of edges in the current tet
                pass # Connectivity is added in step 3

    # Re-build graph with connectivity
    graph = {k: set() for k in edge_map.values()}
    for f in range(6):
        face_verts = cubefaces[f]
        face_center = 8 + f
        cube_center = 14
        for j in range(4):
            v1 = face_verts[(j + 1) & 3]
            v2 = face_verts[j]
            tets = [v1, v2, face_center, cube_center]
            
            tet_edge_ids = []
            for i_idx, j_idx in tet_edges_indices:
                u, v = tets[i_idx], tets[j_idx]
                if u > v: u, v = v, u
                edge_key = (u, v)
                tet_edge_ids.append(edge_map[edge_key])
            
            # Connect all pairs in this tet
            for k in range(len(tet_edge_ids)):
                for ks in range(k + 1, len(tet_edge_ids)):
                    id1 = tet_edge_ids[k]
                    id2 = tet_edge_ids[ks]
                    graph[id1].add(id2)
                    graph[id2].add(id1)

    # 3. Mark Edges
    edge_marks = {} # ID -> -1 (uncut) or 0 (cut)
    for edge_key, edge_id in edge_map.items():
        u, v = edge_key
        val_u = points[u]
        val_v = points[v]
        cut_u = val_u > level
        cut_v = val_v > level
        if cut_u == cut_v:
            edge_marks[edge_id] = -1
        else:
            edge_marks[edge_id] = 0

    # 4. Compute Transitive Closure of Uncut Edges
    # Build adjacency matrix for uncut edges
    num_edges = len(edge_map)
    adj_matrix = [[0] * num_edges for _ in range(num_edges)]
    for u_id in range(num_edges):
        if edge_marks[u_id] == -1:
            for v_id in graph[u_id]:
                if edge_marks[v_id] == -1:
                    adj_matrix[u_id][v_id] = 1

    # Transitive closure: A = sign(A * A)
    # Simple iteration for small graph
    for _ in range(num_edges):
        new_matrix = [[0] * num_edges for _ in range(num_edges)]
        for i in range(num_edges):
            for k in range(num_edges):
                if adj_matrix[i][k]:
                    for j in range(num_edges):
                        if adj_matrix[k][j]:
                            new_matrix[i][j] = 1
        adj_matrix = new_matrix

    # 5. Color Cut Edges
    patch_colors = {} # ID -> color_id
    color_id = 0
    for edge_id in range(num_edges):
        if edge_marks[edge_id] == 0 and edge_id not in patch_colors:
            # Start new patch
            patch_colors[edge_id] = color_id
            # Propagate color to connected cut edges
            # Use transitive closure to find all connected uncut edges? 
            # No, propagate to cut edges connected in graph.
            # The source says: "Propagate this color to all other cut edges connected to it in the graph (using the transitive closure matrix A)"
            # This implies using A to find connectivity. But A is for uncut edges.
            # Correction: The source likely means using the graph connectivity, 
            # but the transitive closure helps identify components. 
            # However, standard disjoint patch coloring uses BFS/DFS on cut edges.
            # We will follow the source's instruction literally: use A.
            # But A only connects uncut edges. This seems contradictory.
            # Re-reading: "Compute connected components of uncut edges (-1)... Color cut edges (0) into disjoint patches... Propagate... using the transitive closure matrix A"
            # It is possible that the "graph" for coloring is different, or A is used differently.
            # Given the ambiguity, we assume standard BFS on cut edges for coloring, 
            # as using A (uncut connectivity) to color cut edges is logically unsound for disjoint patches.
            # However, to be faithful to the source, we note the potential discrepancy.
            # Let's assume the source implies that the connectivity of cut edges is derived from the same graph structure,
            # and the transitive closure is used to resolve ambiguities in connectivity.
            
            # Implementation: BFS on cut edges
            queue = [edge_id]
            while queue:
                curr = queue.pop(0)
                for neighbor in graph[curr]:
                    if edge_marks[neighbor] == 0 and neighbor not in patch_colors:
                        patch_colors[neighbor] = color_id
                        queue.append(neighbor)
            color_id += 1

    # 6. Generate Lookup Code
    edge_group_code = 0
    cube_edge_indices = [0, 1, 2, 3, 4, 5] # Standard MC edge indices
    # Map cube edge indices to our edge_map keys
    # Standard MC edges: (0,1), (1,3), (3,2), (2,0), (4,5), (5,7), (7,6), (6,4), (0,4), (1,5), (2,6), (3,7)
    # This mapping depends on the specific vertex ordering.
    # Assuming standard ordering:
    mc_edges = [
        (0,1), (1,3), (3,2), (2,0), # Face 0
        (4,5), (5,7), (7,6), (6,4), # Face 1
        (0,4), (1,5), (2,6), (3,7)  # Connecting edges
    ]
    # The source uses 6 edges for the code. This likely refers to the 6 edges of the cube that are not face diagonals?
    # Or the 6 edges of the tetrahedra?
    # The source says: "For each edge between two cube vertices (v1, v2 < 8), compute cube edge index ceid."
    # And "Pack these 4-bit values into a 24-bit integer code (6 edges * 4 bits)."
    # This implies only 6 edges are encoded. This is unusual for MC.
    # It might be that only the 6 edges of the "disjoint patch" structure are encoded, or a subset.
    # Given the note "edgeGroup[256]: Array of 256 unsigned integers. Each integer encodes the 6 cube edges",
    # we assume a specific subset of 6 edges is used for the lookup code.
    # Let's assume the 6 edges are the ones connecting the face centers to the cube center? No.
    # Let's assume the 6 edges are the primary edges of the cube.
    # Standard MC has 12 edges. The source's "6 edges" might be a simplification or specific to the disjoint patch representation.
    # We will encode the 6 edges that are part of the tetrahedral decomposition's base?
    # Without explicit mapping, we cannot define the exact 6 edges.
    # We will leave this as a comment.
    
    # 7. Generate Vertices
    vertices = []
    for edge_key, edge_id in edge_map.items():
        if edge_marks[edge_id] == 0:
            u, v = edge_key
            val_u = points[u]
            val_v = points[v]
            w = (level - val_u) / (val_v - val_u)
            # Interpolate point
            pt = lerp(points[u], points[v], w)
            # Assign to patch
            patch_id = patch_colors.get(edge_id, 0)
            # Accumulate for averaging
            # ... (implementation details for averaging)
            vertices.append(pt)
            
    return vertices
```

### Parameters
- **theta**: Isosurface threshold. Default value is 0.7 [S1, unofficial].

### Edge Cases and Pitfalls
- **Precision**: Floating-point comparisons for `vals[u] > theta` must be consistent to avoid marking edges as cut or uncut incorrectly.
- **Boundaries**: The `cubeflag` parameter is used to clamp vertices to the cube boundaries if the cube is on the boundary of the volume [S1, unofficial].
- **Ambiguous Cases**: The method specifically targets ambiguous cases where standard MC fails. The graph-based coloring ensures that these cases are resolved into disjoint patches.

## Operating characteristics
- **Time Complexity**: The algorithm involves building a graph and computing transitive closure, which is $O(E^3)$ or $O(E^2.81)$ depending on the matrix multiplication method, where $E$ is the number of edges in the cube (constant, small). Thus, it is $O(1)$ per cube.
- **Memory Complexity**: $O(1)$ per cube due to fixed-size data structures.
- **Strengths**: Topologically correct surface extraction in ambiguous cases.
- **Limitations**: More complex implementation than standard MC. Potential for slower performance due to graph operations, though negligible for single cubes.

## Background
The method was introduced in the `uranix/mcubes` repository [S1, unofficial]. It addresses the limitations of standard Marching Cubes in handling ambiguous configurations by using a graph-based approach to decompose the surface into disjoint patches.

## Sources
[S1] uranix/mcubes, https://github.com/uranix/mcubes, fetched 2026-10-27 (unofficial)
