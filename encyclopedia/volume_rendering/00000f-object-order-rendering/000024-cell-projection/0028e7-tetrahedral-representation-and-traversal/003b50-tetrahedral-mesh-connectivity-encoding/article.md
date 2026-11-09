# Tetrahedral Mesh Connectivity Encoding

Also called: Basic Cut-Border Machine, Opcode-based mesh representation, Segment-based mesh encoding

## Overview
The Cut-Border Machine (CBM) is a region-growing compression scheme for tetrahedral meshes that encodes connectivity by growing a region bounded by a cut-border [S1]. It solves the problem of efficiently storing and transmitting the topological structure of a tetrahedral mesh by generating a stream of opcodes that describe the position of new vertices relative to the current boundary [S1]. Unlike sibling methods that may rely on global sorting or fixed grid structures, CBM uses a local, incremental traversal that ensures unique vertex identification across multi-resolution segments [S1]. It is distinct from [Projected Tetrahedra](877-projected-tetrahedra.md) in that it focuses on the compact representation of the mesh topology itself, which is then used for rendering, rather than the rasterization process directly.

## How it works
**Inputs:** A tetrahedral mesh (potentially multi-resolution with segments and patches).
**Outputs:** A binary stream containing 4-bit opcodes, vertex indices, and geometry/attribute data.

**Data Structures:**
1.  **Cut-Border:** A triangle mesh consisting of faces of the tetrahedra, dividing the mesh into an "inner" part (encoded) and an "outer" part (to-be-encoded) [S1].
2.  **Gate:** A specific triangle face on the cut-border where the next tetrahedron is added [S1].
3.  **FIFO Buffer:** Stores vertices in the order of their appearance during traversal to determine the processing order [S1].
4.  **Hash Maps:**
    *   *In-Patch Hash Map:* Stores pairs of triangles representing in-patch face-incidencies. Key: Triangle formed by vertex indices. Value: List of incident triangles.
    *   *Cross-Patch Hash Map:* Stores pairs of triangles representing cross-patch/incidency. Key: Triangle formed by *unique global indices* of vertices. Value: List of incident triangles.
    *   *Edge Hash Map (Implementation Detail):* Stores for every edge of the cut-border all its incident triangles. An edge is defined by the pair of its end vertex indices [S1].

**Opcodes (11 total):**
The opcodes describe the location of the "free vertex" (the vertex of the new tetrahedron not part of the gate) relative to the gate [S1].
*   **B (Border):** The gate is a border of the whole mesh. No free vertex.
*   **N (New Vertex):** The free vertex is new and not on a segment/patch border. Parameter: Vertex index.
*   **S (New Shared Vertex):** The free vertex is new and lies on a border between patches/segments. Parameters: Local vertex index (16-bit), Global unique index (32-bit).
*   **F*i* (Flip*i*):** The free vertex is incident to exactly one adjacent triangle of the gate. *i* identifies which of the three adjacent triangles shares the vertex.
*   **T*i* (Top*i*):** The free vertex is incident to exactly two adjacent triangles of the gate. *i* identifies which of the three adjacent triangles is *not* incident to the free vertex.
*   **C (Close):** The free vertex is incident to all three adjacent triangles of the gate.
*   **J (Join):** The free vertex is incident to none of the adjacent triangles of the gate.
*   **Skip:** Used for non-manifold edges incident to >4 triangles. Discards current gate, proceeds to next [S1].

**Algorithm Steps:**

1.  **Initialization:**
    *   Select an arbitrary tetrahedron.
    *   Initialize the cut-border with the 4 faces of this tetrahedron.
    *   Initialize the FIFO buffer with the 4 vertices of this tetrahedron.
    *   Initialize Hash Maps as empty.
    *   Output the 4 vertex indices of the first tetrahedron.

2.  **Traversal Loop:**
    *   While the outer part is not empty:
        *   Dequeue vertex $v$ from FIFO.
        *   Fix $v$ as the current vertex.
        *   Traverse all incident tetrahedra of $v$ that are not yet encoded.
        *   For each incident tetrahedron, identify the current **Gate** (a face on the cut-border shared by the tetrahedron and the inner region).
        *   Determine the case for the free vertex $v_{free}$ relative to the Gate.
        *   **Non-Manifold Check:** If the Gate involves a non-manifold edge (detected via Edge Hash Map), check the number of incident triangles.
            *   If > 4: Output **Skip**. Discard Gate. Continue to next Gate.
            *   If == 4: Proceed normally (adjacency defined by vertex order).
        *   **Encode Opcode:**
            *   If Gate is a mesh border: Output **B**.
            *   If $v_{free}$ is new:
                *   If on segment/patch border: Output **S**, then local index (16-bit), then global index (32-bit).
                *   Else: Output **N**, then local index (16-bit).
            *   If $v_{free}$ is old (on cut-border):
                *   Determine sub-case (Flip, Top, Close, Join).
                *   Output Opcode (**F*i***, **T*i***, **C**, or **J**).
        *   **Update Cut-Border:**
            *   Add the new tetrahedron to the inner region.
            *   Remove the Gate from the cut-border.
            *   Add the 3 new faces of the tetrahedron (excluding the Gate) to the cut-border.
            *   Update Edge Hash Map for new edges.
        *   **Update Face Incidencies (Hash Maps):**
            *   Identify the 4 faces of the new tetrahedron.
            *   For each face, determine if it is **In-Patch** or **Cross-Patch**.
                *   *In-Patch:* All 3 vertices are local to the current segment/patch.
                *   *Cross-Patch:* At least one vertex is a shared vertex (identified by global index).
            *   *In-Patch Logic:*
                *   Look up face in In-Patch Hash Map.
                *   If empty: Store current face.
                *   If occupied: Update incidence for both stored and current face. Delete hash entry.
            *   *Cross-Patch Logic:*
                *   Look up face in Cross-Patch Hash Map using unique global indices.
                *   If empty: Store current face.
                *   If one occupied: Update incidence. Store current face as second entry.
                *   If two occupied: A replacement occurred (coarsening/refinement). Identify the patch being replaced (ceiling/floor segment) and replace its face entry with the current face.
        *   If $v_{free}$ is new, enqueue $v_{free}$ into FIFO.

3.  **Geometry Encoding:**
    *   Store geometry and attribute values in a single table per segment, ordered by the traversal order (FIFO appearance) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_tetrahedral_mesh(mesh):
    # 1. Initialization
    first_tet = mesh.get_arbitrary_tetrahedron()
    cut_border = CutBorder(first_tet.faces)
    fifo = FIFOBuffer(first_tet.vertices)
    in_patch_map = HashMap()
    cross_patch_map = HashMap()
    edge_map = HashMap() # Edge -> List of Triangles
    
    output_stream = []
    
    # Output first 4 vertices
    output_stream.extend(first_tet.vertices)
    
    # 2. Traversal
    while not cut_border.is_empty():
        v = fifo.dequeue()
        
        # Get all unencoded tetrahedra incident to v
        incident_tets = mesh.get_incident_tetrahedra(v)
        
        for tet in incident_tets:
            if tet.is_encoded():
                continue
                
            gate = tet.get_face_on_cut_border()
            
            # Non-manifold check
            if is_non_manifold_edge(gate, edge_map):
                incident_count = get_incident_triangle_count(gate, edge_map)
                if incident_count > 4:
                    output_stream.append('Skip')
                    continue # Discard gate, proceed to next
                # If == 4, proceed normally
            
            free_vertex = tet.get_vertex_not_in_gate(gate)
            
            # Determine Opcode
            opcode = determine_opcode(free_vertex, gate, cut_border, mesh)
            
            # Encode Opcode and Parameters
            output_stream.append(opcode)
            
            if opcode == 'N':
                output_stream.append(free_vertex.local_index) # 16-bit
            elif opcode == 'S':
                output_stream.append(free_vertex.local_index) # 16-bit
                output_stream.append(free_vertex.global_index) # 32-bit
            elif opcode.startswith('F') or opcode.startswith('T'):
                output_stream.append(opcode.index) # i parameter
            
            # Update Cut-Border
            cut_border.remove_face(gate)
            for face in tet.faces:
                if face != gate:
                    cut_border.add_face(face)
                    update_edge_map(face, edge_map)
            
            # Update Face Incidencies
            for face in tet.faces:
                if face.is_cross_patch():
                    update_cross_patch_map(face, cross_patch_map)
                else:
                    update_in_patch_map(face, in_patch_map)
            
            tet.mark_encoded()
            
            # Enqueue new vertex if new
            if opcode in ['N', 'S']:
                fifo.enqueue(free_vertex)
                
    # 3. Geometry
    geometry_data = encode_geometry_by_fifo_order(fifo, mesh)
    output_stream.append(geometry_data)
    
    return output_stream

def determine_opcode(free_vertex, gate, cut_border, mesh):
    # Case 1: Gate is a border
    if gate.is_mesh_border():
        return 'B'
    
    # Case 2: Free vertex is on cut-border (old)
    if cut_border.contains_vertex(free_vertex):
        # Check adjacency to gate's adjacent triangles
        adj_triangles = gate.get_adjacent_triangles_on_cut_border()
        incident_count = count_incident_adj_triangles(free_vertex, adj_triangles)
        
        if incident_count == 1:
            # Flip: find which triangle shares the vertex
            i = find_shared_triangle_index(free_vertex, adj_triangles)
            return f'F{i}'
        elif incident_count == 2:
            # Top: find which triangle does NOT share the vertex
            i = find_non_shared_triangle_index(free_vertex, adj_triangles)
            return f'T{i}'
        elif incident_count == 3:
            return 'C'
        elif incident_count == 0:
            return 'J'
            
    # Case 3: Free vertex is new
    if mesh.is_shared_vertex(free_vertex):
        return 'S'
    else:
        return 'N'

def update_cross_patch_map(face, cross_patch_map):
    key = tuple(sorted(face.vertices.global_indices))
    if key not in cross_patch_map:
        cross_patch_map[key] = [face]
    elif len(cross_patch_map[key]) == 1:
        cross_patch_map[key].append(face)
    elif len(cross_patch_map[key]) == 2:
        # Replacement logic: identify patch being replaced
        # This requires context of the multi-resolution hierarchy
        # For simplicity, we assume the implementation handles the
        # identification of ceiling/floor segments as per [S1]
        replace_patch_face(cross_patch_map, key, face)
```

## Subtopics

### Extended Cut-Border Machine (CBM) for Multi-Resolution
A compression scheme extending the CBM to support cross-segment incidencies and respect pre-computed mesh layouts [S1]. It stores segments sequentially in a binary file. Vertices on segment boundaries are copied into each segment they belong to, and an additional unique global index is stored for every border vertex to identify it uniquely across segments [S1]. It uses the same 11 opcodes but distinguishes between in-patch vertices (N) and shared vertices (S) to reconstruct face-incidencies using two hash maps (in-patch and cross-patch) [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of tetrahedra and vertices, assuming constant-time hash map operations. The traversal order (FIFO) is crucial for efficiency [S1].
*   **Memory Complexity:** Proportional to the mesh size. Requires storage for the cut-border, FIFO buffer, and two hash maps for face incidencies. The Edge Hash Map is an implementation detail to avoid set-cutting operations [S1].
*   **Compression:** Uses 4-bit words for opcodes. Indices for N, S, J are fixed at 16 bits (theoretically $\log_2 n$). Shared vertex S requires an additional 32-bit global index (theoretically $\log_2 m$) [S1]. Arithmetic coding is not implemented in the basic version, relying on fixed-width storage [S1].
*   **Strengths:** Efficient region-growing traversal; handles non-manifold edges (up to 4 triangles) without geometry tests; supports multi-resolution and cross-segment incidencies [S1].
*   **Limitations:** Fixed 16-bit indices may be inefficient for very large segments; non-manifold edges with >4 triangles require a Skip symbol and gate discard, potentially reducing compression efficiency [S1].

## Background
The Cut-Border Machine was introduced by Gumhold et al. [S1] as a region-growing compression scheme for tetrahedral meshes. It was extended to support multi-resolution meshes and cross-segment incidencies to facilitate view-dependent rendering and interactive visualization [S1]. The method builds on the principles of the [Tetrahedral representation and traversal](4396-tetrahedral-representation-and-traversal.md) family, specifically providing the connectivity encoding necessary for object-order rendering of tetrahedral volumes.

## Sources
[S1] Segment-based tetrahedral meshing and rendering, https://doi.org/10.1145/1174429.1174507, fetched 2026-11-06
