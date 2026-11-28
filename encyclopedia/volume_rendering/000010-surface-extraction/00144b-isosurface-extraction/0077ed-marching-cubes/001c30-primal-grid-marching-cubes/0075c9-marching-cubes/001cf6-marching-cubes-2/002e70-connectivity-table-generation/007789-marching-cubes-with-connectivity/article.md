# Marching Cubes with Connectivity

Also called: Connectivity-aware Marching Cubes, Marching Cubes with Adjacency Arrays

## Overview
Marching Cubes with Connectivity extends the standard Marching Cubes algorithm by explicitly computing and outputting adjacency relationships between mesh primitives (vertices, edges, and faces) during the surface extraction pass [S1]. While standard Marching Cubes produces a list of triangles, this method resolves topological ambiguities and generates adjacency arrays (vertex-to-edge, edge-to-face, face-to-vertex) in a single pass, ensuring a manifold mesh structure [S1]. It is chosen when downstream applications require efficient neighborhood queries, such as mesh smoothing, subdivision, or fluid simulation, without the need for post-processing to rebuild connectivity [S1].

## How it works
**Inputs:**
- A 3D scalar field defined on a voxel grid.
- A scalar threshold value for isosurface extraction.

**Outputs:**
- A list of vertex positions.
- A list of triangle indices (vertex references).
- Adjacency arrays:
  - `vertex_to_edge`: Maps each vertex to its incident edges.
  - `edge_to_face`: Maps each edge to its incident faces.
  - `face_to_vertex`: Maps each face to its constituent vertices.

**Data Structures:**
The method relies on a predefined **Connectivity Table** with 256 entries, one for each possible 8-bit configuration index derived from the voxel corner signs [S1]. Each entry contains:
- Edge intersection flags indicating which of the 12 cube edges are intersected by the isosurface.
- Adjacency pointers defining how the resulting triangles share vertices and edges [S1].

The vertex and edge configurations use a hierarchical indexing scheme. Primary positions are denoted $P_0$ through $P_8$. Neighbor counts or connectivity indices are denoted $N_{18}$ and $N_{26}$. Specific edge/face connectivity cases within the 256 configurations are identified by subscripts such as $P_{2b-1}$, $P_{3c-2}$, and $P_{4g-8}$ [S1].

**Algorithm Steps:**
1. **Voxel Iteration:** Iterate through every voxel in the input grid.
2. **Configuration Index:** Compute the 8-bit configuration index $I$ by comparing the scalar field values at the 8 voxel corners against the threshold. If a corner value is greater than or equal to the threshold, the corresponding bit is set to 1; otherwise, 0 [S1].
3. **Table Lookup:** Retrieve the connectivity entry for index $I$ from the 256-entry table. This entry specifies the intersected edges and the topological arrangement of triangles [S1].
4. **Vertex Generation:** For each intersected edge, compute the intersection point using linear interpolation. Assign a unique global vertex ID.
5. **Triangle Generation:** Construct triangles based on the connectivity entry. The entry defines how vertices are grouped into triangles and how they share edges [S1].
6. **Adjacency Update:** Simultaneously update the adjacency arrays:
   - Record which edges are incident to each new vertex.
   - Record which faces are incident to each new edge.
   - Record which vertices form each new face [S1].
7. **Ambiguity Resolution:** For ambiguous cases (e.g., configurations 10, 26, 115, 121), the connectivity table explicitly resolves the topology to prevent non-manifold edges or holes [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_cubes_with_connectivity(grid, threshold):
    # Initialize adjacency arrays
    # vertex_to_edge: dict mapping vertex_id -> list of edge_ids
    # edge_to_face: dict mapping edge_id -> list of face_ids
    # face_to_vertex: dict mapping face_id -> list of 3 vertex_ids
    
    vertices = []
    triangles = []
    vertex_to_edge = {}
    edge_to_face = {}
    face_to_vertex = {}
    
    # Precomputed connectivity table: 256 entries
    # Each entry: {edges: [edge_indices], triangles: [[v1, v2, v3], ...], adj: {...}}
    conn_table = load_connectivity_table() 
    
    # Helper to get edge intersection point
    def get_edge_intersection(v0, v1, val0, val1):
        if val0 == val1:
            return None # Degenerate case, handled by table or skipped
        t = (threshold - val0) / (val1 - val0)
        return v0 + t * (v1 - v0)

    # Helper to get unique edge ID
    # Edges are identified by the two vertex indices they connect
    def get_edge_id(v_idx0, v_idx1):
        return min(v_idx0, v_idx1), max(v_idx0, v_idx1)

    for voxel in grid:
        # 1. Compute configuration index
        corner_vals = [grid[v] for v in voxel.corners]
        config_index = 0
        for i, val in enumerate(corner_vals):
            if val >= threshold:
                config_index |= (1 << i)
        
        # 2. Lookup connectivity
        entry = conn_table[config_index]
        
        # 3. Generate vertices and triangles
        # Map local vertex indices to global indices
        global_vertices = []
        
        for edge_idx in entry.edges:
            # Determine which two corners define this edge
            c0, c1 = edge_corners[edge_idx]
            v0_pos = voxel.corners[c0].pos
            v1_pos = voxel.corners[c1].pos
            val0 = corner_vals[c0]
            val1 = corner_vals[c1]
            
            pt = get_edge_intersection(v0_pos, v1_pos, val0, val1)
            if pt is None:
                continue
                
            # Create global vertex
            v_id = len(vertices)
            vertices.append(pt)
            global_vertices.append(v_id)
            
            # Update vertex-to-edge adjacency
            e_id = get_edge_id(v0_idx, v1_idx) # Local edge ID logic
            if v_id not in vertex_to_edge:
                vertex_to_edge[v_id] = []
            vertex_to_edge[v_id].append(e_id)

        # 4. Create triangles and update adjacencies
        for tri_verts in entry.triangles:
            # tri_verts contains indices into the generated vertices for this voxel
            g_v0 = global_vertices[tri_verts[0]]
            g_v1 = global_vertices[tri_verts[1]]
            g_v2 = global_vertices[tri_verts[2]]
            
            f_id = len(triangles)
            triangles.append((g_v0, g_v1, g_v2))
            face_to_vertex[f_id] = (g_v0, g_v1, g_v2)
            
            # Update edge-to-face adjacency
            edges = [(g_v0, g_v1), (g_v1, g_v2), (g_v2, g_v0)]
            for e in edges:
                e_id = get_edge_id(e[0], e[1])
                if e_id not in edge_to_face:
                    edge_to_face[e_id] = []
                edge_to_face[e_id].append(f_id)

    return vertices, triangles, vertex_to_edge, edge_to_face, face_to_vertex
```

**Parameters:**
- **Connectivity Table Size:** 256 entries [S1].
- **Threshold:** Scalar value for isosurface extraction.

**Edge Cases and Pitfalls:**
- **Ambiguous Cases:** Configurations 10, 26, 115, and 121 are topologically ambiguous; the vertex signs alone do not uniquely determine the surface topology. The connectivity table must explicitly resolve these to avoid non-manifold edges or holes [S1].
- **Precision:** Linear interpolation assumes linear variation within the voxel. High-frequency fields may require higher-order interpolation or denser grids.
- **Boundaries:** Voxels on the grid boundary have fewer neighbors; the algorithm must handle missing corners or clamp indices appropriately.

## Subtopics

### Standard Marching Cubes
Produces only vertices and triangles without explicit adjacency data. Use when memory is constrained and neighborhood queries are not needed. [Link](5782-marching-cubes.md)

### Dual Contouring
Uses a dual grid and optimization to preserve sharp features. Use when feature preservation is critical over topological simplicity. [Link](5784-dual-contouring.md)

## Operating characteristics
- **Time Complexity:** $O(V)$, where $V$ is the number of voxels. The constant overhead per voxel includes table lookup and adjacency updates [S1].
- **Memory Complexity:** $O(F + E + V)$, where $F$, $E$, and $V$ are the number of faces, edges, and vertices in the output mesh, respectively. This accounts for the storage of adjacency arrays [S1].
- **Strengths:** Provides immediate access to mesh topology, enabling efficient downstream operations like smoothing or subdivision. Resolves ambiguities deterministically.
- **Limitations:** Higher memory usage due to adjacency arrays. Slightly higher computational overhead per voxel compared to standard Marching Cubes due to adjacency bookkeeping.
- **When to Use:** When the mesh topology is required for subsequent processing steps that rely on neighborhood relationships.
- **When Not to Use:** When only a visual representation is needed and memory is extremely limited.

## Background
The method was introduced by [S1] in 1999. It builds upon the foundational Marching Cubes algorithm by adding explicit connectivity information to the output, addressing the need for manifold meshes in applications requiring topological consistency.

## Sources
[S1] Marching cubes method with connectivity, https://doi.org/10.1109/icip.1999.819613, fetched 2026-10-26
