# Tetrahedral Mesh Connectivity Encoding

Also called: Basic Cut-Border Machine, Opcode-based mesh representation, Segment-based mesh encoding

## Overview
Tetrahedral Mesh Connectivity Encoding, commonly known as the Basic Cut-Border Machine (CBM), is a region-growing compression scheme that encodes the connectivity of a tetrahedral mesh by traversing it and emitting opcodes that describe the position of new vertices relative to a growing boundary called the cut-border [S1]. It solves the problem of efficiently storing and transmitting the topological structure of tetrahedral volumes for visualization systems supporting interactive direct volume renderings, isosurface extraction, and vector field visualization [S1]. Unlike sibling methods that may rely on global indexing or implicit structures, CBM uses a local, sequential traversal where each opcode references vertices already encountered or currently being defined, ensuring unique identification without global coordinate lookups [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A tetrahedral mesh (potentially non-manifold, with specific constraints on non-manifold edges).
*   **Output:** A stream of opcodes (connectivity symbols) and a stream of vertex geometry/attributes.

**Data Structures**
1.  **Cut-Border:** A triangle mesh representing the boundary between the encoded (inner) and unencoded (outer) parts of the tetrahedral mesh [S1].
2.  **Gate:** A specific triangle face on the cut-border where the next tetrahedron is attached [S1].
3.  **FIFO Buffer:** A queue storing vertex indices to determine the traversal order [S1].
4.  **Hash Maps:** Two hash maps store face incidencies for reconstruction: `in_patch_hash_map` for in-patch incidencies and `cross_patch_hash_map` for cross-patch incidencies [S1]. Keys in hash maps are triangles defined by unique vertex indices [S1].
5.  **Edge Incident Map:** A hash map storing, for every edge on the cut-border, the set of all incident triangles [S1].

**Opcodes**
There are 11 primary opcodes (plus a 12th "Skip" for extreme non-manifold cases) encoded as 4-bit words [S1]. They describe the position of the "free vertex" (the vertex of the new tetrahedron not part of the gate) relative to the gate:

1.  **B (Border):** The gate is a border of the whole mesh [S1].
2.  **N (New Vertex):** The free vertex is new and not on a border between segments/patches [S1].
3.  **S (New Shared Vertex):** The free vertex is new but lies on a border between segments/patches [S1].
4.  **F*i* (Flip*i*):** The free vertex is incident to exactly one adjacent triangle of the gate [S1].
5.  **T*i* (Top*i*):** The free vertex is incident to exactly two adjacent triangles of the gate [S1].
6.  **C (Close):** The free vertex is incident to all three adjacent triangles of the gate [S1].
7.  **J (Join):** The free vertex is incident to none of the adjacent triangles of the gate [S1].

**Parameters**
*   **Vertex Index (for N, S, J):** Encoded with 16 bits for performance, though theoretically $\log_2(n)$ bits are sufficient where $n$ is the number of vertices in the segment [S1].
*   **Global Shared Index (for S):** Encoded with 32 bits, theoretically $\log_2(m)$ bits where $m$ is the total vertices in the multi-resolution mesh [S1].

**Algorithm Steps**

1.  **Initialization:**
    *   Select an arbitrary starting tetrahedron [S1].
    *   Initialize the cut-border with the four faces of this tetrahedron [S1].
    *   Initialize the FIFO buffer with the indices of the four vertices of the starting tetrahedron [S1].
    *   Initialize the Edge Incident Map, `in_patch_hash_map`, and `cross_patch_hash_map` as empty [S1].
    *   Output the indices of the first four vertices [S1].

2.  **Traversal Loop:**
    *   While the cut-border is not empty:
        *   **Select Gate:** Pick a face from the cut-border as the current gate [S1].
        *   **Identify Case:** Determine the relationship of the potential free vertex to the gate [S1].
            *   If the gate has no adjacent unencoded tetrahedron, emit **B** [S1]. Remove gate from cut-border.
            *   Else, identify the free vertex $v$.
                *   If $v$ is not yet visited:
                    *   Check if $v$ is on a segment/patch boundary.
                    *   If yes, emit **S** with local index and global shared index [S1].
                    *   If no, emit **N** with local index [S1].
                    *   Add $v$ to FIFO [S1].
                *   If $v$ is already visited:
                    *   Check incidence with the three triangles adjacent to the gate.
                    *   If incident to 1 triangle $t_i$, emit **F*i*** [S1].
                    *   If incident to 2 triangles, emit **T*i*** where $i$ is the index of the non-incident triangle [S1].
                    *   If incident to 3 triangles, emit **C** [S1].
                    *   If incident to 0 triangles, emit **J** with index of $v$ [S1].
        *   **Update Cut-Border:**
            *   Add the new tetrahedron to the inner part [S1].
            *   Remove the gate from the cut-border [S1].
            *   Add the three new faces of the tetrahedron (excluding the gate) to the cut-border [S1].
            *   Update edge neighbors in the Edge Incident Map.
                *   For each new edge, add the new triangle to the incident set.
                *   **Non-Manifold Detection:** For each edge, check if it is non-manifold. An edge is non-manifold if the cut of incident triangle sets of its two end vertices is not empty [S1].
                *   If an edge is incident to exactly 4 triangles, it is allowed; adjacency is defined by vertex order [S1].
                *   If an edge is incident to >4 triangles, emit **Skip** (12th symbol), discard the current gate, and proceed to the next gate without updating the cut-border for this tetrahedron [S1].
        *   **Process FIFO:**
            *   If the FIFO is not empty, fix the first vertex $v_{fix}$ [S1].
            *   Traverse and encode all tetrahedra incident to $v_{fix}$ that are not yet encoded [S1].
            *   Remove $v_{fix}$ from FIFO [S1].

3.  **Reconstruction (Decoder Side):**
    *   Read first 4 vertices [S1].
    *   For each opcode:
        *   **N/S:** Add new vertex to list. Register face incidencies [S1].
            *   For each face of the new tetrahedron:
                *   Determine if it is an in-patch or cross-patch face [S1].
                *   **In-patch:** Look up `in_patch_hash_map` with triangle indices.
                    *   If empty, store self [S1].
                    *   If one entry exists, update incidence for both and delete entry [S1].
                *   **Cross-patch:** Look up `cross_patch_hash_map` with unique global indices.
                    *   If empty, store self [S1].
                    *   If one entry exists, update incidence and store self as second face [S1].
                    *   If two entries exist, replace the face from the patch that was coarsened/refined [S1].
        *   **F/T/C/J:** Use existing vertices. Register face incidencies using the same hash map logic [S1].
        *   **B:** No new vertex. Update incidencies for the three new faces added to the cut-border [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_tetrahedral_mesh(mesh):
    if not mesh:
        return [], []

    cut_border_faces = set()
    edge_incident_map = {}
    fifo = []
    encoded_tetrahedra = set()
    vertex_list = []
    vertex_to_local_index = {}
    opcode_stream = []
    vertex_geometry_stream = []

    start_tet = mesh[0]
    v0, v1, v2, v3 = start_tet

    for v in [v0, v1, v2, v3]:
        if v not in vertex_to_local_index:
            vertex_to_local_index[v] = len(vertex_list)
            vertex_list.append(v)
            vertex_geometry_stream.append(v)

    faces = [
        tuple(sorted([v0, v1, v2])),
        tuple(sorted([v0, v1, v3])),
        tuple(sorted([v0, v2, v3])),
        tuple(sorted([v1, v2, v3]))
    ]
    cut_border_faces.update(faces)
    fifo.extend([v0, v1, v2, v3])

    def get_edge(face, v):
        return tuple(sorted([x for x in face if x != v]))

    def find_adjacent_tet(gate, encoded_tetrahedra, mesh):
        for tet in mesh:
            if id(tet) not in encoded_tetrahedra:
                tet_faces = [
                    tuple(sorted([tet[0], tet[1], tet[2]])),
                    tuple(sorted([tet[0], tet[1], tet[3]])),
                    tuple(sorted([tet[0], tet[2], tet[3]])),
                    tuple(sorted([tet[1], tet[2], tet[3]]))
                ]
                if gate in tet_faces:
                    return tet
        return None

    while cut_border_faces:
        gate = cut_border_faces.pop()
        adjacent_tet = find_adjacent_tet(gate, encoded_tetrahedra, mesh)

        if adjacent_tet is None:
            opcode_stream.append('B')
            continue

        encoded_tetrahedra.add(id(adjacent_tet))
        f0, f1, f2, f3 = adjacent_tet
        free_vertex = [v for v in [f0, f1, f2, f3] if v not in gate][0]

        opcode = None
        param = None

        if free_vertex not in vertex_to_local_index:
            # Sources do not give the implementation for checking segment boundaries.
            is_shared = False 
            if is_shared:
                opcode = 'S'
                vertex_to_local_index[free_vertex] = len(vertex_list)
                vertex_list.append(free_vertex)
                vertex_geometry_stream.append(free_vertex)
                param = vertex_to_local_index[free_vertex]
                # Global shared index stored here
            else:
                opcode = 'N'
                vertex_to_local_index[free_vertex] = len(vertex_list)
                vertex_list.append(free_vertex)
                vertex_geometry_stream.append(free_vertex)
                param = vertex_to_local_index[free_vertex]
            fifo.append(free_vertex)
        else:
            # Sources do not give the implementation for determining incidence 
            # of free_vertex with the three triangles adjacent to the gate.
            incidence_count = 0 
            idx = 0
            if incidence_count == 1:
                opcode = f'F{idx}'
            elif incidence_count == 2:
                opcode = f'T{idx}'
            elif incidence_count == 3:
                opcode = 'C'
            else:
                opcode = 'J'
                param = vertex_to_local_index[free_vertex]

        opcode_stream.append(opcode)
        if param is not None:
            opcode_stream.append(param)

        new_faces = [
            tuple(sorted([f0, f1, f2])),
            tuple(sorted([f0, f1, f3])),
            tuple(sorted([f0, f2, f3])),
            tuple(sorted([f1, f2, f3]))
        ]
        new_faces = [f for f in new_faces if f != gate]

        skip = False
        for face in new_faces:
            for i in range(3):
                edge = get_edge(face, face[(i+1)%3])
                if edge not in edge_incident_map:
                    edge_incident_map[edge] = []
                edge_incident_map[edge].append(id(adjacent_tet))
                if len(edge_incident_map[edge]) > 4:
                    skip = True
                    break
            if skip:
                break

        if skip:
            opcode_stream.append('Skip')
            continue

        cut_border_faces.update(new_faces)

        if fifo:
            fix_vertex = fifo[0]
            # Sources do not give the implementation for traversing incident tetrahedra 
            # of fix_vertex. This is handled by the main loop finding adjacent tets.
            fifo.pop(0)

    return opcode_stream, vertex_geometry_stream
```

## Subtopics

### [Implicit Connectivity Mesh (ICM)](11226-implicit-connectivity-mesh-icm.md)
Implicit Connectivity Mesh (ICM) is a representation of implicit surfaces where a mesh consists of vertices only, partitioned into cubic cells to enable efficient isosurface extraction and simplification. It differs from explicit polygon meshes by storing connectivity information implicitly within each vertex's encoding vector, allowing rapid reconstruction of the explicit mesh and supporting multiresolution modeling via topology-preserving vertex clustering.

### [Streaming Tetrahedral Mesh Connectivity Compression](13244-streaming-tetrahedral-mesh-connectivity-compression.md)
Streaming Tetrahedral Mesh Connectivity Compression is an algorithm for compressing tetrahedral volume meshes by incrementally encoding tetrahedra and vertices as they stream in, allowing memory to be freed immediately after use. It solves the problem of compressing gigabyte-sized meshes that do not fit in main memory by avoiding the need to keep the entire uncompressed mesh in memory. It sets itself apart from in-core methods like Tetrahedral Mesh Connectivity Encoding by processing data in a streaming fashion with a small delay buffer for local reordering, and from Implicit Connectivity Mesh (ICM) by explicitly managing active vertex finalization and dynamic indexing based on stream width rather than global indices.

### Extended Cut-Border Machine (CBM) for Multi-Resolution
The Extended Cut-Border Machine (CBM) for Multi-Resolution is a compression scheme extending the CBM to support cross-segment incidencies and respect pre-computed mesh layouts, storing segments sequentially in a binary file [S1]. It enumerates vertices segment-wise, copying vertices lying on segment boundaries into each segment they belong to, and stores an additional unique global index for every border vertex to identify it uniquely across segments [S1]. It encodes each patch's tetrahedral mesh using 11 opcodes (including Border and New Vertex) that describe the position of the free vertex relative to the cut-border gate, and encodes geometry of vertices interleaved with the symbol stream or in a single block, following the traversal order (FIFO buffer) [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of tetrahedra and vertices, $O(N)$, assuming constant-time hash map operations. The traversal is efficient due to the FIFO-based point-wise processing [S1].
*   **Memory Complexity:** Depends on the cut-border size and hash maps. The cut-border is a triangle mesh of the boundary. Hash maps store face incidencies.
*   **Compression Ratio:** Uses 4-bit words for opcodes. The source notes that arithmetic coding was not implemented as file size reduction was sufficient, but could improve ratios [S1]. The use of 16-bit indices for N, S, J is a performance choice that may reduce compression compared to variable-length coding [S1].
*   **Strengths:** Simple implementation, no need for global indexing, supports non-manifold edges (up to 4 triangles), efficient traversal.
*   **Limitations:** Non-manifold edges with >4 triangles require a Skip symbol and do not contribute to the mesh structure in the same way. The reliance on 4-bit opcodes and fixed-size indices may not achieve optimal compression compared to arithmetic coding.
*   **When to use:** When a compact, sequential representation of tetrahedral connectivity is needed for rendering or transmission, and the mesh is not excessively non-manifold.
*   **When not to use:** When optimal compression is critical and arithmetic coding is feasible, or when the mesh contains complex non-manifold structures (>4 triangles per edge).

## Background
The Cut-Border Machine (CBM) was extended to tetrahedral meshes in the context of segment-based tetrahedral meshing and rendering [S1], building on data structures originally proposed for triangle meshes [Gumhold et al. 1999]. The work by Yoon et al. [2005] highlighted the importance of mesh layout for rendering performance, which motivated the extension of CBM to support multi-resolution meshes and cross-segment incidencies while respecting pre-computed layouts [S1].

## Sources
[S1] Segment-based tetrahedral meshing and rendering, https://doi.org/10.1145/1174429.1174507, fetched 2026-11-06
