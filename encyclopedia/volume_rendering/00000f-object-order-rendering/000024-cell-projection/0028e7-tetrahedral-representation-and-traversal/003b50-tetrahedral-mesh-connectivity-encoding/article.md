# Tetrahedral Mesh Connectivity Encoding

Also called: Basic Cut-Border Machine, Opcode-based mesh representation, Segment-based mesh encoding

## Overview
Tetrahedral Mesh Connectivity Encoding, specifically the Basic Cut-Border Machine (CBM), is a region-growing compression scheme that encodes the connectivity of a tetrahedral mesh by traversing it and generating a stream of opcodes [S1]. It solves the problem of efficiently storing and transmitting mesh topology by describing the position of new vertices relative to a growing boundary (the cut-border) [S1]. It sets itself apart from sibling methods like [Projected Tetrahedra](877-projected-tetrahedra.md) by focusing on lossless connectivity compression rather than direct rasterization, and from [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md) by preserving the explicit mesh structure rather than extracting an isosurface [S1].

## How it works
**Inputs:** A tetrahedral mesh (vertices and connectivity).
**Outputs:** A bitstream containing opcodes and vertex indices, followed by geometry/attribute data.

**Data Structures:**
1.  **Cut-Border:** A triangle mesh representing the boundary between encoded (inner) and unencoded (outer) tetrahedra. It consists of faces of the tetrahedra.
2.  **Gate:** A specific triangle face on the cut-border where the next tetrahedron is added.
3.  **FIFO Buffer:** Stores vertices in the order they are first encountered during traversal.
4.  **Hash Maps:**
    *   *Edge Hash Map:* Stores for every edge of the cut-border all its incident triangles. An edge is uniquely defined by its two end vertex indices. Used to detect non-manifold edges [S1].
    *   *In-Patch Face Hash Map:* Stores pairs of triangles representing in-patch face incidencies. Key: triangle formed by vertex indices. Value: list of faces [S1].
    *   *Cross-Patch Face Hash Map:* Stores pairs of triangles representing cross-patch incidencies. Key: triangle formed by *unique global indices* of vertices. Value: list of faces [S1].
5.  **Opcode Storage:** Opcodes are stored as 4-bit words. Vertex indices for N, S, J opcodes are encoded with 16 bits. Shared vertex index for S is encoded with an additional 32 bits [S1].

**Numbered Steps:**

1.  **Initialization:**
    *   Select an arbitrary tetrahedron from the mesh.
    *   Initialize the cut-border with the four faces of this tetrahedron.
    *   Initialize the FIFO buffer with the four vertices of this tetrahedron.
    *   Initialize the hash maps as empty.
    *   Output the indices of the four initial vertices to the stream [S1].

2.  **Traversal Loop:**
    *   While the cut-border is not empty:
        1.  Select the current **gate** (a face on the cut-border).
        2.  Identify the **free vertex** of the tetrahedron adjacent to the gate that is not on the cut-border.
        3.  Determine the **Opcode** based on the position of the free vertex relative to the gate and the cut-border:
            *   **B (Border):** The gate is a border of the whole mesh (no adjacent tetrahedron). Discard gate, select next gate [S1].
            *   **N (New Vertex):** The free vertex is new (not on cut-border). Output opcode N, then the 16-bit vertex index. Add the new vertex to the FIFO buffer.
            *   **S (New Shared Vertex):** The free vertex is new but lies on a border between patches/segments. Output opcode S, then the 16-bit local vertex index, then the 32-bit unique global index. Add to FIFO.
            *   **F*i* (Flip):** The free vertex is already on the cut-border and is incident to exactly one adjacent triangle of the gate. *i* (0, 1, or 2) indicates which of the three adjacent triangles shares the vertex. Output opcode F*i*.
            *   **T*i* (Top):** The free vertex is already on the cut-border and is incident to exactly two adjacent triangles of the gate. *i* (0, 1, or 2) indicates which of the three adjacent triangles is *not* incident to the vertex. Output opcode T*i*.
            *   **C (Close):** The free vertex is incident to all three adjacent triangles of the gate. Output opcode C.
            *   **J (Join):** The free vertex is incident to none of the adjacent triangles of the gate. Output opcode J, then the 16-bit vertex index.
            *   **Skip:** If a non-manifold edge incident to *more than four* triangles is detected, output a Skip symbol (12th symbol), discard the current gate, and proceed to the next gate [S1].
        4.  **Update Cut-Border:**
            *   Remove the gate from the cut-border.
            *   Add the three new faces of the newly encoded tetrahedron to the cut-border (excluding the gate).
            *   Update edge neighbors for the new faces.
            *   **Non-Manifold Detection:** Use the Edge Hash Map. For each new edge, look up incident triangles. If an edge is already in the cut-border, the sets of incident triangles for its end vertices will have a non-empty intersection (or the lookup finds existing entries). Correctly set edge-neighbors for the new triangles using geometry or encoded indices [S1].
            *   *Note:* Non-manifold edges incident to exactly four triangles are allowed; adjacency is defined by vertex order in triangles [S1].
        5.  **Face Registration (for Decoder/Encoder consistency):**
            *   If the opcode is N, S, or J, a new tetrahedron is formed. Register its faces.
            *   *In-Patch Faces:* Look up the In-Patch Face Hash Map using the triangle's vertex indices.
                *   If empty: Store this face.
                *   If one face exists: Update incidency for both stored and current face, delete hash entry.
            *   *Cross-Patch Faces:* (If all three vertices are shared/unique) Look up Cross-Patch Face Hash Map using unique global indices.
                *   If empty: Store this face.
                *   If one face exists: Update incidency, store current face as second entry.
                *   If two faces exist: A patch replacement occurred (coarsening/refinement). Identify the replaced patch (ceiling/floor segment) and replace its face entry with the current face [S1].
        6.  **FIFO Processing:**
            *   After encoding all tetrahedra incident to the first vertex in the FIFO, remove it.
            *   Fix the next vertex in the FIFO and repeat the traversal for all its incident tetrahedra [S1].

3.  **Geometry Storage:**
    *   Store geometry (coordinates) and attribute values in a single table per segment, in the order of vertex appearance (FIFO order) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_tetrahedral_mesh(mesh):
    """
    Encodes a tetrahedral mesh using the Basic Cut-Border Machine.
    mesh: object with vertices (list of 3D points) and tetrahedra (list of 4 vertex indices).
    Returns: bitstream (opcodes and indices), geometry_stream.
    """
    # Data structures
    cut_border_faces = set() # Set of face tuples (v1, v2, v3)
    edge_hash_map = {}       # Key: (min(v1,v2), max(v1,v2)), Value: list of faces
    in_patch_face_hash = {}  # Key: tuple(sorted(v1,v2,v3)), Value: list of faces
    cross_patch_face_hash = {} # Key: tuple(sorted(unique_indices)), Value: list of faces
    fifo = []                # List of vertex indices
    encoded_tetrahedra = set()
    stream = []
    
    # 1. Initialization
    start_tet = mesh.tetrahedra[0]
    v0, v1, v2, v3 = start_tet
    
    # Add initial faces to cut border
    faces = [
        (v0, v1, v2), (v0, v1, v3), (v0, v2, v3), (v1, v2, v3)
    ]
    for f in faces:
        cut_border_faces.add(f)
        update_edge_hash(f, edge_hash_map)
        
    fifo.extend([v0, v1, v2, v3])
    encoded_tetrahedra.add(start_tet)
    
    # Output initial vertices
    stream.append(('V', v0))
    stream.append(('V', v1))
    stream.append(('V', v2))
    stream.append(('V', v3))
    
    # 2. Traversal
    while fifo:
        current_v = fifo.pop(0)
        
        # Find all unencoded tetrahedra incident to current_v
        incident_tets = [t for t in mesh.tetrahedra 
                         if current_v in t and t not in encoded_tetrahedra]
        
        for tet in incident_tets:
            # Identify gate and free vertex
            # Gate is the face shared with an already encoded tetrahedron
            # Free vertex is the one not in the gate
            gate, free_v = find_gate_and_free_vertex(tet, encoded_tetrahedra)
            
            if gate is None:
                continue # Should not happen if logic is correct
            
            opcode = determine_opcode(gate, free_v, cut_border_faces, edge_hash_map, mesh)
            
            # Output Opcode
            stream.append(('OP', opcode))
            
            if opcode in ['N', 'S', 'J']:
                stream.append(('IDX', mesh.get_vertex_index(free_v)))
                if opcode == 'S':
                    stream.append(('GLOBAL_IDX', mesh.get_global_index(free_v)))
            
            # Update Cut Border
            cut_border_faces.remove(gate)
            remove_edge_from_hash(gate, edge_hash_map)
            
            # Add new faces to cut border
            new_faces = get_new_faces(tet, gate)
            for nf in new_faces:
                cut_border_faces.add(nf)
                update_edge_hash(nf, edge_hash_map)
                
            # Handle Non-Manifold / Skip
            if opcode == 'SKIP':
                continue
                
            # Register Faces
            register_faces(tet, in_patch_face_hash, cross_patch_face_hash, mesh)
            
            encoded_tetrahedra.add(tet)
            
            # Add free vertex to FIFO if new
            if opcode in ['N', 'S']:
                if free_v not in fifo:
                    fifo.append(free_v)
                    
    return stream, mesh.vertices

def determine_opcode(gate, free_v, cut_border_faces, edge_hash_map, mesh):
    """
    Determines the opcode based on the position of free_v relative to gate.
    """
    # Case 1: Gate is a border (no adjacent tetrahedron)
    if not has_adjacent_tet(gate, cut_border_faces):
        return 'B'
    
    # Case 2: Free vertex is on the cut-border
    if free_v in get_vertices_of_cut_border(cut_border_faces):
        # Sub-cases: Flip, Top, Close, Join
        adjacent_triangles = get_adjacent_triangles_to_gate(gate)
        incident_count = count_incident_triangles(free_v, adjacent_triangles)
        
        if incident_count == 1:
            # Flip: find which triangle shares the vertex
            i = find_incident_triangle_index(free_v, adjacent_triangles)
            return f'F{i}'
        elif incident_count == 2:
            # Top: find which triangle does NOT share the vertex
            i = find_non_incident_triangle_index(free_v, adjacent_triangles)
            return f'T{i}'
        elif incident_count == 3:
            return 'C'
        elif incident_count == 0:
            return 'J'
        else:
            # Non-manifold check
            if is_non_manifold_edge(free_v, edge_hash_map, count=4):
                return 'J' # Allowed
            elif is_non_manifold_edge(free_v, edge_hash_map, count>4):
                return 'SKIP'
            else:
                # Fallback or error
                return 'J'
    else:
        # Case 3: Free vertex is new
        if is_shared_vertex(free_v, mesh):
            return 'S'
        else:
            return 'N'

def update_edge_hash(face, edge_hash_map):
    v1, v2, v3 = face
    edges = [(v1, v2), (v2, v3), (v3, v1)]
    for e in edges:
        e_key = (min(e[0], e[1]), max(e[0], e[1]))
        if e_key not in edge_hash_map:
            edge_hash_map[e_key] = []
        edge_hash_map[e_key].append(face)

def is_non_manifold_edge(v, edge_hash_map, count):
    # Check edges incident to v
    # This is a simplified check; actual implementation requires iterating edges
    # and checking if the number of incident triangles exceeds limits
    pass
```

## Subtopics

### [Implicit Connectivity Mesh (ICM)](11226-implicit-connectivity-mesh-icm.md)
Implicit Connectivity Mesh (ICM) is a representation of implicit surfaces where a mesh is composed of vertices only, partitioned into cubic cells to enable efficient isosurface extraction and simplification. It sets itself apart from explicit polygon meshes by storing connectivity information implicitly within each vertex's encoding vector, allowing for rapid reconstruction of the explicit mesh and supporting multiresolution modeling via topology-preserving vertex clustering.

### [Streaming Tetrahedral Mesh Connectivity Compression](13244-streaming-tetrahedral-mesh-connectivity-compression.md)
Streaming Tetrahedral Mesh Connectivity Compression is a method for compressing tetrahedral volume meshes by incrementally encoding tetrahedra and vertices as they stream in, allowing memory to be freed immediately after use. It solves the problem of processing gigabyte-sized meshes that exceed main memory capacity by avoiding the need to keep the entire uncompressed mesh in memory. It sets itself apart from in-core methods like [Gumhold et al.](4567-gumhold-tetrahedral-mesh-compression.md) by processing data in a streaming fashion with a small delay buffer for local reordering, achieving connectivity rates within a factor of two of the state-of-the-art while using drastically less memory.

### Extended Cut-Border Machine (CBM) for Multi-Resolution
The Extended Cut-Border Machine (CBM) for Multi-Resolution is a compression scheme extending the CBM to support cross-segment incidencies and respect pre-computed mesh layouts, storing segments sequentially in a binary file [S1]. It enumerates vertices segment-wise, copying vertices lying on segment boundaries into each segment they belong to, and stores an additional unique global index for every border vertex to identify it uniquely across segments [S1]. It encodes each patch's tetrahedral mesh using 11 opcodes (including Border 'B' and New Vertex 'N') that describe the position of the free vertex relative to the cut-border gate, with geometry interleaved or stored in a single block following the FIFO traversal order [S1].

## Operating characteristics
**Time Complexity:** The traversal is linear with respect to the number of tetrahedra, $O(N)$, where $N$ is the number of tetrahedra, assuming hash map operations are $O(1)$ [S1]. The efficiency depends heavily on the traversal order (FIFO) [S1].
**Memory Complexity:** Requires storage for the cut-border, hash maps, and FIFO buffer. The hash maps store edge incidencies and face incidencies. The extended version requires additional storage for global indices and cross-segment data [S1].
**Performance:** The basic CBM does not implement arithmetic coding for opcodes, relying on 4-bit word storage, which provides sufficient file size reduction for some applications but is suboptimal compared to arithmetic coding [S1]. The extended version supports multi-resolution meshes and cross-segment incidencies [S1].
**Strengths:** Efficient region-growing traversal; handles non-manifold edges (up to 4 triangles) explicitly; supports multi-resolution and cross-segment encoding [S1].
**Limitations:** Non-manifold edges with >4 triangles require a Skip symbol, potentially reducing compression efficiency; fixed 16-bit/32-bit index encoding may be inefficient for very large meshes compared to logarithmic bit encoding; no arithmetic coding in the basic version [S1].

## Background
The Cut-Border Machine (CBM) was introduced by [Gumhold et al.](4567-gumhold-tetrahedral-mesh-compression.md) for triangle meshes and extended to tetrahedral meshes by [S1] (Yoon et al., 2005). The extension aimed to support multi-resolution tetrahedral meshes and view-dependent rendering, addressing the need for efficient storage of complex volumetric data [S1]. It is used in visualization systems supporting interactive direct volume rendering, isosurface extraction, and vector field visualization [S1].

## Sources
[S1] Segment-based tetrahedral meshing and rendering, https://doi.org/10.1145/1174429.1174507, fetched 2026-11-06
