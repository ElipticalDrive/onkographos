# Tetrahedral Mesh Connectivity Encoding

Also called: Basic Cut-Border Machine, Opcode-based mesh representation, Segment-based mesh encoding

## Overview
Tetrahedral Mesh Connectivity Encoding, commonly known as the Basic Cut-Border Machine (CBM), is a region-growing compression scheme that encodes the connectivity of a tetrahedral mesh by traversing it and emitting opcodes that describe the position of new vertices relative to a growing boundary called the cut-border [S1]. It solves the problem of efficiently storing and transmitting the topological structure of tetrahedral volumes, which is a prerequisite for rendering methods like Projected Tetrahedra [877-projected-tetrahedra.md] or Marching Tetrahedrons [940-marching-tetrahedrons-algorithm.md]. Unlike sibling methods that may rely on global indexing or implicit structures, CBM uses a local, sequential traversal where each opcode references vertices already encountered or currently being defined, ensuring unique identification without global coordinate lookups [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A tetrahedral mesh (potentially non-manifold, with specific constraints on non-manifold edges).
*   **Output:** A stream of opcodes (connectivity symbols) and a stream of vertex geometry/attributes.

**Data Structures**
1.  **Cut-Border:** A triangle mesh representing the boundary between the encoded (inner) and unencoded (outer) parts of the tetrahedral mesh. It consists of faces of the tetrahedra.
2.  **Gate:** A specific triangle face on the cut-border where the next tetrahedron is attached.
3.  **FIFO Buffer:** A queue storing vertex indices to determine the traversal order.
4.  **Hash Maps:** Two hash maps store face incidencies for reconstruction:
    *   `in_patch_hash_map`: Stores pairs of triangles for faces shared within a single patch/segment.
    *   `cross_patch_hash_map`: Stores pairs of triangles for faces shared between different patches/segments.
    *   Keys in hash maps are triangles defined by unique vertex indices.
5.  **Edge Incident Map:** A hash map storing, for every edge on the cut-border, the set of all incident triangles. This is used to detect non-manifold edges [S1].

**Opcodes**
There are 11 primary opcodes (plus a 12th "Skip" for extreme non-manifold cases) encoded as 4-bit words. They describe the position of the "free vertex" (the vertex of the new tetrahedron not part of the gate) relative to the gate:

1.  **B (Border):** The gate is a border of the whole mesh. No free vertex exists.
2.  **N (New Vertex):** The free vertex is new and not on a border between segments/patches. Parameter: Vertex Index.
3.  **S (New Shared Vertex):** The free vertex is new but lies on a border between segments/patches. Parameters: Vertex Index, Global Shared Index.
4.  **F*i* (Flip*i*):** The free vertex is incident to exactly one adjacent triangle of the gate. *i* (0, 1, or 2) identifies which triangle. No vertex index parameter needed.
5.  **T*i* (Top*i*):** The free vertex is incident to exactly two adjacent triangles of the gate. *i* (0, 1, or 2) identifies the triangle *not* incident to the free vertex. No vertex index parameter needed.
6.  **C (Close):** The free vertex is incident to all three adjacent triangles of the gate. No vertex index parameter needed.
7.  **J (Join):** The free vertex is incident to none of the adjacent triangles of the gate. Parameter: Vertex Index.

**Parameters**
*   **Vertex Index (for N, S, J):** Encoded with 16 bits for performance, though theoretically $\log_2(n)$ bits are sufficient where $n$ is the number of vertices in the segment [S1].
*   **Global Shared Index (for S):** Encoded with 32 bits, theoretically $\log_2(m)$ bits where $m$ is the total vertices in the multi-resolution mesh [S1].

**Algorithm Steps**

1.  **Initialization:**
    *   Select an arbitrary starting tetrahedron.
    *   Initialize the cut-border with the four faces of this tetrahedron.
    *   Initialize the FIFO buffer with the indices of the four vertices of the starting tetrahedron.
    *   Initialize the Edge Incident Map.
    *   Initialize `in_patch_hash_map` and `cross_patch_hash_map` as empty.
    *   Output the indices of the first four vertices.

2.  **Traversal Loop:**
    *   While the cut-border is not empty:
        *   **Select Gate:** Pick a face from the cut-border as the current gate.
        *   **Identify Case:** Determine the relationship of the potential free vertex to the gate.
            *   If the gate has no adjacent unencoded tetrahedron, emit **B**. Remove gate from cut-border.
            *   Else, identify the free vertex $v$.
                *   If $v$ is not yet visited:
                    *   Check if $v$ is on a segment/patch boundary.
                    *   If yes, emit **S** with local index and global shared index.
                    *   If no, emit **N** with local index.
                    *   Add $v$ to FIFO.
                *   If $v$ is already visited:
                    *   Check incidence with the three triangles adjacent to the gate.
                    *   If incident to 1 triangle $t_i$, emit **F*i***.
                    *   If incident to 2 triangles, emit **T*i*** where $i$ is the index of the non-incident triangle.
                    *   If incident to 3 triangles, emit **C**.
                    *   If incident to 0 triangles, emit **J** with index of $v$.
        *   **Update Cut-Border:**
            *   Add the new tetrahedron to the inner part.
            *   Remove the gate from the cut-border.
            *   Add the three new faces of the tetrahedron (excluding the gate) to the cut-border.
            *   Update edge neighbors in the Edge Incident Map.
                *   For each new edge, add the new triangle to the incident set.
                *   **Non-Manifold Detection:** For each edge, check if it is non-manifold.
                    *   An edge is non-manifold if the cut of incident triangle sets of its two end vertices is not empty (or via hash map lookup).
                    *   If an edge is incident to exactly 4 triangles, it is allowed; adjacency is defined by vertex order.
                    *   If an edge is incident to >4 triangles, emit **Skip** (12th symbol), discard the current gate, and proceed to the next gate without updating the cut-border for this tetrahedron [S1].
        *   **Process FIFO:**
            *   If the FIFO is not empty, fix the first vertex $v_{fix}$.
            *   Traverse and encode all tetrahedra incident to $v_{fix}$ that are not yet encoded.
            *   Remove $v_{fix}$ from FIFO.

3.  **Reconstruction (Decoder Side):**
    *   Read first 4 vertices.
    *   For each opcode:
        *   **N/S:** Add new vertex to list. Register face incidencies.
            *   For each face of the new tetrahedron:
                *   Determine if it is an in-patch or cross-patch face.
                *   **In-patch:** Look up `in_patch_hash_map` with triangle indices.
                    *   If empty, store self.
                    *   If one entry exists, update incidence for both and delete entry.
                *   **Cross-patch:** Look up `cross_patch_hash_map` with unique global indices.
                    *   If empty, store self.
                    *   If one entry exists, update incidence and store self as second face.
                    *   If two entries exist, replace the face from the patch that was coarsened/refined.
        *   **F/T/C/J:** Use existing vertices. Register face incidencies using the same hash map logic.
        *   **B:** No new vertex. Update incidencies for the three new faces added to the cut-border.

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_tetrahedral_mesh(mesh):
    # mesh: list of tetrahedra, each tetrahedron is a tuple of 4 vertex indices
    # Returns: opcode_stream, vertex_geometry_stream
    
    if not mesh:
        return [], []

    # Data Structures
    cut_border_faces = set() # Set of face tuples (sorted vertex indices)
    edge_incident_map = {}   # Map: edge_tuple -> list of triangle_indices
    fifo = []                # List of vertex indices
    encoded_tetrahedra = set()
    vertex_list = []         # Ordered list of vertices as they appear
    vertex_to_local_index = {}
    opcode_stream = []
    
    # Initialize with first tetrahedron
    start_tet = mesh[0]
    v0, v1, v2, v3 = start_tet
    
    # Add vertices
    for v in [v0, v1, v2, v3]:
        if v not in vertex_to_local_index:
            vertex_to_local_index[v] = len(vertex_list)
            vertex_list.append(v)
            
    # Add faces to cut border
    faces = [
        tuple(sorted([v0, v1, v2])),
        tuple(sorted([v0, v1, v3])),
        tuple(sorted([v0, v2, v3])),
        tuple(sorted([v1, v2, v3]))
    ]
    cut_border_faces.update(faces)
    
    # Add vertices to FIFO
    fifo.extend([v0, v1, v2, v3])
    
    # Output first 4 vertices
    vertex_geometry_stream = [vertex_list[i] for i in range(4)]
    
    # Helper to get edge from face and vertex
    def get_edge(face, v):
        return tuple(sorted([x for x in face if x != v]))

    # Helper to check non-manifold
    def is_non_manifold_edge(edge, incident_tris):
        # Simplified logic based on source: if incident count > 4, it's skip
        # Source says: "If the encoder detects a non-manifold edge that is incident to more than four triangles, it outputs a Skip symbol"
        return len(incident_tris) > 4

    while cut_border_faces:
        # Select gate (arbitrary from cut_border_faces)
        gate = cut_border_faces.pop()
        
        # Find adjacent unencoded tetrahedron
        # In a real implementation, we'd need a adjacency map. 
        # Assuming we can find a tetrahedron sharing 'gate' that is not encoded.
        # For pseudocode completeness, we assume a function find_adjacent_tet(gate, encoded_tetrahedra, mesh)
        # which returns the tetrahedron or None.
        
        adjacent_tet = find_adjacent_tet(gate, encoded_tetrahedra, mesh)
        
        if adjacent_tet is None:
            # Case B: Border
            opcode_stream.append('B')
            continue
            
        encoded_tetrahedra.add(id(adjacent_tet))
        f0, f1, f2, f3 = adjacent_tet
        
        # Identify free vertex (the one not in gate)
        free_vertex = [v for v in [f0, f1, f2, f3] if v not in gate][0]
        
        # Determine opcode
        opcode = None
        param = None
        
        if free_vertex not in vertex_to_local_index:
            # New Vertex
            # Check if shared (simplified: assume logic exists to detect segment boundary)
            is_shared = check_if_shared_vertex(free_vertex, mesh) # [G] Placeholder for segment boundary logic
            
            if is_shared:
                opcode = 'S'
                # Local index
                vertex_to_local_index[free_vertex] = len(vertex_list)
                vertex_list.append(free_vertex)
                param = vertex_to_local_index[free_vertex]
                # Global shared index would be added here [S1]
            else:
                opcode = 'N'
                vertex_to_local_index[free_vertex] = len(vertex_list)
                vertex_list.append(free_vertex)
                param = vertex_to_local_index[free_vertex]
                
            fifo.append(free_vertex)
        else:
            # Existing Vertex
            # Check incidence with the 3 triangles adjacent to the gate in the cut-border context
            # This requires knowing the geometry/topology of the cut border faces adjacent to the gate
            # Simplified: assume we can determine incidence count and index
            
            # Get triangles adjacent to gate in the current cut-border state
            # This is complex to simulate without full state. 
            # We assume helper functions exist.
            
            incident_triangles = get_adjacent_triangles_to_gate(gate, cut_border_faces)
            incidence_count = count_incident_vertices(free_vertex, incident_triangles)
            
            if incidence_count == 1:
                # Flip
                idx = get_incident_triangle_index(free_vertex, incident_triangles)
                opcode = f'F{idx}'
            elif incidence_count == 2:
                # Top
                idx = get_non_incident_triangle_index(free_vertex, incident_triangles)
                opcode = f'T{idx}'
            elif incidence_count == 3:
                opcode = 'C'
            else:
                opcode = 'J'
                param = vertex_to_local_index[free_vertex]
                
        opcode_stream.append(opcode)
        if param is not None:
            opcode_stream.append(param)
            
        # Update Cut Border
        # Remove gate
        # Add new faces (3 faces of new tet, minus gate)
        new_faces = [
            tuple(sorted([f0, f1, f2])),
            tuple(sorted([f0, f1, f3])),
            tuple(sorted([f0, f2, f3])),
            tuple(sorted([f1, f2, f3]))
        ]
        # Remove gate from new_faces if present
        new_faces = [f for f in new_faces if f != gate]
        
        # Check for non-manifold edges in new faces
        skip = False
        for face in new_faces:
            for i in range(3):
                edge = get_edge(face, face[(i+1)%3]) # Simplified edge extraction
                # Update edge incident map
                if edge not in edge_incident_map:
                    edge_incident_map[edge] = []
                edge_incident_map[edge].append(id(adjacent_tet))
                
                if is_non_manifold_edge(edge, edge_incident_map[edge]):
                    skip = True
                    break
            if skip:
                break
                
        if skip:
            opcode_stream.append('Skip') # 12th symbol
            # Discard current gate and proceed to next
            continue
            
        # Add new faces to cut border
        cut_border_faces.update(new_faces)
        
        # Process FIFO
        if fifo:
            fix_vertex = fifo[0]
            # Traverse all incident tetrahedra of fix_vertex
            # This is handled by the main loop finding adjacent tets
            # We just remove from FIFO after processing? 
            # Source: "The first vertex of the fifo is fixed and all its incident tetrahedra are traversed and encoded... Then, the next vertex of the fifo is fixed"
            # This implies the FIFO controls the order of vertex fixation, but the loop continues until all tets are encoded.
            # The FIFO is primarily for ensuring vertices are processed in order of appearance.
            pass 
            
    return opcode_stream, vertex_geometry_stream

def find_adjacent_tet(gate, encoded_tetrahedra, mesh):
    # [G] Helper to find a tetrahedron sharing 'gate' that is not in encoded_tetrahedra
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
```

## Subtopics

### [Implicit Connectivity Mesh (ICM)](11226-implicit-connectivity-mesh-icm.md)
Implicit Connectivity Mesh (ICM) is a representation of implicit surfaces where a mesh is composed of vertices only, partitioned into cubic cells to enable efficient isosurface extraction and simplification. It sets itself apart from explicit polygon meshes by storing connectivity information implicitly within each vertex's encoding vector, allowing for rapid reconstruction of the explicit mesh and supporting multiresolution modeling via topology-preserving vertex clustering.

### [Streaming Tetrahedral Mesh Connectivity Compression](13244-streaming-tetrahedral-mesh-connectivity-compression.md)
Streaming Tetrahedral Mesh Connectivity Compression is an algorithm for compressing tetrahedral volume meshes by incrementally encoding tetrahedra and vertices as they stream in, allowing memory to be freed immediately after use. It solves the problem of compressing gigabyte-sized meshes that do not fit in main memory by avoiding the need to keep the entire uncompressed mesh in memory. It sets itself apart from in-core methods like Tetrahedral Mesh Connectivity Encoding by processing data in a streaming fashion with a small delay buffer for local reordering, and from Implicit Connectivity Mesh (ICM) by explicitly managing active vertex finalization and dynamic indexing based on stream width rather than global indices.

### Extended Cut-Border Machine (CBM) for Multi-Resolution
The Extended Cut-Border Machine (CBM) for Multi-Resolution is a compression scheme extending the CBM to support cross-segment incidencies and respect pre-computed mesh layouts, storing segments sequentially in a binary file [S1]. It enumerates vertices segment-wise, copying vertices lying on segment boundaries into each segment they belong to, and stores an additional unique global index for every border vertex to identify it uniquely across segments [S1]. It encodes each patch's tetrahedral mesh using 11 opcodes (including Border and New Vertex) that describe the position of the free vertex relative to the cut-border gate, and encodes geometry of vertices interleaved with the symbol stream or in a single block, following the traversal order (FIFO buffer) [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of tetrahedra and vertices, $O(N)$, assuming constant-time hash map operations. The traversal is efficient due to the FIFO-based point-wise processing [S1].
*   **Memory Complexity:** Depends on the cut-border size and hash maps. The cut-border is a triangle mesh of the boundary. Hash maps store face incidencies. For large meshes, the hash maps can consume significant memory, though the streaming variant mitigates this [S1].
*   **Compression Ratio:** Uses 4-bit words for opcodes. The source notes that arithmetic coding was not implemented as file size reduction was sufficient, but could improve ratios [S1]. The use of 16-bit indices for N, S, J is a performance choice that may reduce compression compared to variable-length coding [S1].
*   **Strengths:** Simple implementation, no need for global indexing, supports non-manifold edges (up to 4 triangles), efficient traversal.
*   **Limitations:** Non-manifold edges with >4 triangles require a Skip symbol and do not contribute to the mesh structure in the same way. The reliance on 4-bit opcodes and fixed-size indices may not achieve optimal compression compared to arithmetic coding.
*   **When to use:** When a compact, sequential representation of tetrahedral connectivity is needed for rendering or transmission, and the mesh is not excessively non-manifold.
*   **When not to use:** When optimal compression is critical and arithmetic coding is feasible, or when the mesh contains complex non-manifold structures (>4 triangles per edge).

## Background
The Cut-Border Machine (CBM) was originally proposed for triangle meshes by Gumhold et al. [Gumhold et al. 1999] and extended to tetrahedral meshes in the context of segment-based tetrahedral meshing and rendering [S1]. The work by Yoon et al. [2005] highlighted the importance of mesh layout for rendering performance, which motivated the extension of CBM to support multi-resolution meshes and cross-segment incidencies while respecting pre-computed layouts [S1].

## Sources
[S1] Segment-based tetrahedral meshing and rendering, https://doi.org/10.1145/1174429.1174507, fetched 2026-11-06
