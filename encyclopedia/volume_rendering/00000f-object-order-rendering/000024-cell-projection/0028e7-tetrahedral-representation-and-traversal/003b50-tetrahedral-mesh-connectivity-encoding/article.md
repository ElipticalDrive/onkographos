# Tetrahedral Mesh Connectivity Encoding

Also called: Basic Cut-Border Machine, Opcode-based mesh representation, Segment-based mesh encoding

## Overview
The Cut-Border Machine (CBM) is a region-growing compression scheme for tetrahedral meshes that encodes connectivity by growing a region bounded by a cut-border [S1]. It solves the problem of efficiently storing and transmitting the topological structure of tetrahedral meshes, which is required for object-order volume rendering methods like [Projected Tetrahedra](877-projected-tetrahedra.md) and [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md). Unlike sibling methods that may rely on explicit geometric sorting or fixed grid structures, CBM encodes the mesh topology purely through a stream of opcodes describing the position of new vertices relative to a growing boundary [S1]. This allows for compact storage and reconstruction of the mesh connectivity without preserving the original vertex ordering, though it requires a specific traversal order for efficiency [S1].

## How it works
The method encodes the connectivity of a tetrahedral mesh segment by traversing the mesh and generating a stream of opcodes. The decoder reconstructs the mesh by interpreting these opcodes and managing face incidencies via hash maps.

### Inputs and Outputs
*   **Input:** A tetrahedral mesh (potentially part of a multi-resolution segment).
*   **Output:** A binary stream containing vertex indices for the initial tetrahedron, followed by a sequence of 4-bit opcodes and associated parameters, and a block of geometry/attribute data.

### Data Structures
1.  **Cut-Border:** A triangle mesh representing the boundary between encoded (inner) and unencoded (outer) tetrahedra. It consists of faces from the tetrahedra.
2.  **Gate:** A specific triangle face on the cut-border where the next tetrahedron is added.
3.  **FIFO Buffer:** Stores vertices in the order they are first encountered during traversal.
4.  **Hash Maps:** Two hash maps store pairs of triangles to represent face incidencies:
    *   `InPatchHash`: For faces shared within the same patch/segment.
    *   `CrossPatchHash`: For faces shared between different patches or segments.
    *   Keys are defined by the unique indices of the triangle vertices.
5.  **Opcode Stream:** A sequence of 4-bit words.

### Opcodes
There are 11 primary opcodes that fully determine the position of the "free" vertex (the vertex of the new tetrahedron not on the gate) relative to the gate. The gate is a triangle defined by three vertices already in the inner region.

1.  **B (Border):** The gate is a border of the whole tetrahedral mesh. No free vertex exists.
2.  **N (New Vertex):** The free vertex is not on the cut-border and is not shared with other segments/patches.
    *   *Parameter:* 16-bit index of the new vertex.
3.  **S (New Shared Vertex):** The free vertex is on a border between patches or segments.
    *   *Parameter 1:* 16-bit local index of the new vertex.
    *   *Parameter 2:* 32-bit unique global index of the shared vertex (to identify it across segments).
4.  **F$i$ (Flip$i$):** The free vertex is incident to exactly one adjacent triangle of the gate. $i \in \{0, 1, 2\}$ identifies which of the three gate edges/triangles is shared.
    *   *Parameter:* None.
5.  **T$i$ (Top$i$):** The free vertex is incident to exactly two adjacent triangles of the gate. $i \in \{0, 1, 2\}$ identifies which of the three gate triangles is *not* incident to the free vertex.
    *   *Parameter:* None.
6.  **C (Close):** The free vertex is incident to all three adjacent triangles of the gate.
    *   *Parameter:* None.
7.  **J (Join):** The free vertex is incident to none of the adjacent triangles of the gate.
    *   *Parameter:* 16-bit index of the new vertex.

*Note: Non-manifold edges incident to more than four triangles trigger a **Skip** opcode (12th symbol), discarding the current gate and proceeding to the next. Non-manifold edges with exactly four triangles are allowed and handled by vertex order [S1].*

### Algorithm Steps

1.  **Initialization:**
    *   Select an arbitrary tetrahedron $T_0$.
    *   Encode the indices of its four vertices ($v_0, v_1, v_2, v_3$) as the start of the stream.
    *   Initialize the cut-border with the four faces of $T_0$.
    *   Initialize the FIFO buffer with $v_0, v_1, v_2, v_3$.
    *   Initialize `InPatchHash` and `CrossPatchHash` as empty.

2.  **Traversal Loop:**
    *   While the cut-border is not empty:
        *   Select the current **gate** $g$ from the cut-border.
        *   Determine the case for the free vertex $v_{free}$ relative to $g$.
        *   **Case 1: Border (B).**
            *   Encode opcode `B`.
            *   Remove $g$ from the cut-border.
            *   If the cut-border is empty, terminate.
            *   Else, select a new gate from the remaining cut-border.
        *   **Case 2: Free vertex is new (N, S, J).**
            *   Identify $v_{free}$.
            *   If $v_{free}$ is shared across segments/patches:
                *   Encode opcode `S`.
                *   Encode 16-bit local index.
                *   Encode 32-bit global unique index.
            *   Else if $v_{free}$ is new and not shared:
                *   Encode opcode `N`.
                *   Encode 16-bit local index.
            *   Else if $v_{free}$ is new and not on cut-border but incident to no gate triangles (Join):
                *   Encode opcode `J`.
                *   Encode 16-bit local index.
            *   Add $v_{free}$ to the FIFO buffer if not already present.
            *   Add the new tetrahedron $(v_{gate0}, v_{gate1}, v_{gate2}, v_{free})$ to the inner region.
            *   Update the cut-border: Remove $g$, add the three new faces formed by $v_{free}$ and the edges of $g$ (excluding those already on the border).
            *   Handle non-manifold edge detection using hash maps if necessary.
        *   **Case 3: Free vertex is on cut-border (F, T, C).**
            *   Identify the relationship between $v_{free}$ and the gate triangles.
            *   Encode the corresponding opcode (`F$i$`, `T$i$`, or `C`).
            *   Add the new tetrahedron to the inner region.
            *   Update the cut-border: Remove $g$, add new faces.
            *   Update face incidencies in hash maps.
        *   **Non-Manifold Handling:**
            *   If the operation creates a non-manifold edge with $>4$ triangles, encode `Skip`, discard gate, and continue.
            *   If $=4$ triangles, adjacency is defined by vertex order; no geometry test needed.

3.  **Vertex Processing Order:**
    *   The traversal is point-wise. Fix the first vertex in the FIFO buffer.
    *   Traverse and encode all tetrahedra incident to this vertex.
    *   Move to the next vertex in the FIFO buffer and repeat until the buffer is empty.

4.  **Face Incidency Reconstruction (Decoder Side):**
    *   **In-Patch Faces:**
        *   If no triangle registered in `InPatchHash` for the face, store the current face.
        *   If one triangle is registered, update incidency for both stored and current face, then delete the hash entry.
    *   **Cross-Patch Faces:**
        *   Look up `CrossPatchHash` using unique vertex indices.
        *   If empty, insert current face.
        *   If one face registered, update incidency and store current face as second.
        *   If two faces registered, replace the face from the patch that was replaced by coarsening/refinement (identified by ceiling/floor segment logic).

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_tetrahedral_mesh(mesh):
    """
    Encodes a tetrahedral mesh using the Cut-Border Machine.
    Returns: (opcode_stream, geometry_block)
    """
    # 1. Initialization
    if not mesh.tetrahedra:
        return [], []

    # Start with an arbitrary tetrahedron
    t0 = mesh.tetrahedra[0]
    v_indices = [t0.v0, t0.v1, t0.v2, t0.v3]
    
    # Cut-border stores faces (tuples of 3 vertex indices)
    # Initially the 4 faces of t0
    cut_border = set()
    cut_border.add(frozenset([t0.v0, t0.v1, t0.v2]))
    cut_border.add(frozenset([t0.v0, t0.v1, t0.v3]))
    cut_border.add(frozenset([t0.v0, t0.v2, t0.v3]))
    cut_border.add(frozenset([t0.v1, t0.v2, t0.v3]))
    
    # FIFO buffer for vertex processing order
    fifo = list(v_indices)
    visited_vertices = set(v_indices)
    
    # Hash maps for face incidencies
    in_patch_hash = {}
    cross_patch_hash = {}
    
    opcode_stream = []
    vertex_counter = 4 # Next new vertex index
    
    # Helper to get unique global index for shared vertices
    # In a real implementation, this requires pre-computed segment data
    def get_global_index(local_idx, is_shared):
        if is_shared:
            # Placeholder for logic retrieving the 32-bit global index
            return 0 
        return local_idx

    # 2. Traversal
    while cut_border:
        # Select gate. In a full implementation, gate selection strategy matters.
        # Here we assume a deterministic selection from the set.
        gate = next(iter(cut_border))
        cut_border.remove(gate)
        
        # Identify the three vertices of the gate
        g_v0, g_v1, g_v2 = list(gate)
        
        # Find the free vertex. 
        # This requires looking up the tetrahedron that shares this gate face
        # and has a 4th vertex not in the inner region (or determining case).
        # For the sake of the algorithm description, we assume we find the 
        # tetrahedron T_new incident to gate.
        
        # Find T_new
        t_new = None
        for t in mesh.tetrahedra:
            if frozenset([t.v0, t.v1, t.v2, t.v3]) == frozenset([g_v0, g_v1, g_v2, ?]):
                # Check if the 4th vertex is the free one
                pass 
        
        # Simplified logic based on CBM cases:
        # We need to determine the case (B, N, S, F, T, C, J)
        
        # Case B: Gate is on the boundary of the mesh (no tetrahedron attached)
        if not is_tetrahedron_attached(mesh, gate):
            opcode_stream.append('B')
            continue
            
        # Find the free vertex v_free
        # The tetrahedron is (g_v0, g_v1, g_v2, v_free)
        # Determine v_free from mesh connectivity
        v_free = get_free_vertex(mesh, gate)
        
        # Determine Case
        # Check if v_free is already in the cut-border or inner region
        is_on_border = is_vertex_on_cut_border(cut_border, v_free)
        is_shared = is_vertex_shared_across_segments(v_free)
        
        if not is_on_border and not is_shared:
            # Case N
            opcode_stream.append('N')
            opcode_stream.append(v_free) # 16-bit index
            # Add to FIFO if new
            if v_free not in visited_vertices:
                fifo.append(v_free)
                visited_vertices.add(v_free)
                
        elif is_shared:
            # Case S
            opcode_stream.append('S')
            opcode_stream.append(v_free) # 16-bit local index
            opcode_stream.append(get_global_index(v_free, True)) # 32-bit global index
            if v_free not in visited_vertices:
                fifo.append(v_free)
                visited_vertices.add(v_free)
                
        elif is_on_border:
            # Check incidence with gate triangles
            # Gate has 3 edges. Check which adjacent triangles share v_free
            # This requires checking the cut-border neighbors
            
            # Count how many gate edges are incident to v_free in the cut-border
            # This determines F, T, C, J
            
            # Simplified: Assume we calculate incidence count
            incidence_count = count_incident_gate_triangles(cut_border, gate, v_free)
            
            if incidence_count == 1:
                # Flip
                # Determine which edge i
                i = get_flip_index(gate, v_free)
                opcode_stream.append(f'F{i}')
            elif incidence_count == 2:
                # Top
                # Determine which triangle is NOT incident
                i = get_top_index(gate, v_free)
                opcode_stream.append(f'T{i}')
            elif incidence_count == 3:
                # Close
                opcode_stream.append('C')
            else:
                # Join
                opcode_stream.append('J')
                opcode_stream.append(v_free) # 16-bit index
                if v_free not in visited_vertices:
                    fifo.append(v_free)
                    visited_vertices.add(v_free)
        else:
            # Should not happen if logic is correct, but fallback to N if new
            opcode_stream.append('N')
            opcode_stream.append(v_free)
            if v_free not in visited_vertices:
                fifo.append(v_free)
                visited_vertices.add(v_free)

        # Update Cut-Border
        # Remove gate, add new faces formed by v_free and gate edges
        new_faces = [
            frozenset([g_v0, g_v1, v_free]),
            frozenset([g_v1, g_v2, v_free]),
            frozenset([g_v2, g_v0, v_free])
        ]
        
        for face in new_faces:
            # If face is not already in cut_border, add it
            # If face is in cut_border, it means it's a shared face between two encoded tetrahedra?
            # No, cut_border is the boundary. If a face is added that is already in cut_border,
            # it means it was on the boundary and is now internal? 
            # Actually, CBM removes the gate and adds the other 3 faces.
            # If a new face is already in cut_border, it means it was a boundary face
            # from another tetrahedron, so it becomes internal and should be removed.
            if face in cut_border:
                cut_border.remove(face)
            else:
                cut_border.add(face)
                
        # Handle Non-Manifold Edges
        # Check if any edge in the new faces is non-manifold (>4 triangles)
        # If so, output Skip and revert? Or just skip gate?
        # According to notes: output Skip, discard gate, proceed to next gate.
        # This implies the tetrahedron is NOT added.
        # This is a complex edge case. For simplicity in pseudocode, we assume
        # standard manifold or 4-triangle non-manifold handling.
        
        # Update Hash Maps for Face Incidencies
        # This is done during decoding, but encoder must ensure consistency.
        # The notes describe decoder-side reconstruction.
        
    # 3. Geometry Block
    # Geometry and attributes stored in a single table per segment
    geometry_block = encode_geometry(mesh, fifo)
    
    return opcode_stream, geometry_block

def decode_tetrahedral_mesh(opcode_stream, geometry_block):
    """
    Reconstructs the tetrahedral mesh from the CBM stream.
    """
    fifo = []
    visited_vertices = set()
    tetrahedra = []
    
    in_patch_hash = {}
    cross_patch_hash = {}
    
    # Decode first 4 vertices
    v0, v1, v2, v3 = decode_vertices(geometry_block, 0, 4)
    fifo.extend([v0, v1, v2, v3])
    visited_vertices.update([v0, v1, v2, v3])
    
    tetrahedra.append((v0, v1, v2, v3))
    
    # Initial cut border
    cut_border = set()
    cut_border.add(frozenset([v0, v1, v2]))
    cut_border.add(frozenset([v0, v1, v3]))
    cut_border.add(frozenset([v0, v2, v3]))
    cut_border.add(frozenset([v1, v2, v3]))
    
    idx = 0
    while opcode_stream:
        opcode = opcode_stream[idx]
        idx += 1
        
        if opcode == 'B':
            # Remove gate from cut_border
            # In decoder, we don't explicitly track cut_border as a set of faces
            # to remove, but we rely on the FIFO and opcodes to rebuild.
            # However, the structure of the cut_border is implicit in the 
            # sequence of opcodes.
            # For reconstruction, we just need to know the topology.
            # The decoder builds the mesh by adding tetrahedra.
            # The 'B' opcode indicates the end of a branch or region?
            # Actually, 'B' means the gate is a border. No tetrahedron added.
            # We just move to the next gate.
            continue
            
        # Determine gate
        # The gate is determined by the previous tetrahedron or the current state.
        # In CBM, the gate is the face where the next tetrahedron is attached.
        # The decoder must track the current gate.
        # This is complex to implement without the encoder's state.
        # Standard CBM decoder assumes the encoder and decoder are synchronized
        # on the traversal order.
        
        # Simplified: Assume we have the current gate vertices
        # g_v0, g_v1, g_v2
        
        if opcode in ['N', 'S', 'J']:
            # Read vertex index
            if opcode == 'S':
                local_idx = opcode_stream[idx]
                idx += 1
                global_idx = opcode_stream[idx]
                idx += 1
                v_new = decode_vertex(geometry_block, local_idx)
                # Handle shared vertex logic
            else:
                local_idx = opcode_stream[idx]
                idx += 1
                v_new = decode_vertex(geometry_block, local_idx)
                
            # Add tetrahedron
            tetrahedra.append((g_v0, g_v1, g_v2, v_new))
            
            # Update FIFO
            if v_new not in visited_vertices:
                fifo.append(v_new)
                visited_vertices.add(v_new)
                
        elif opcode.startswith('F'):
            i = int(opcode[1])
            # v_new is determined by the gate and i
            v_new = get_vertex_from_flip(g_v0, g_v1, g_v2, i, cut_border)
            tetrahedra.append((g_v0, g_v1, g_v2, v_new))
            
        elif opcode.startswith('T'):
            i = int(opcode[1])
            v_new = get_vertex_from_top(g_v0, g_v1, g_v2, i, cut_border)
            tetrahedra.append((g_v0, g_v1, g_v2, v_new))
            
        elif opcode == 'C':
            v_new = get_vertex_from_close(g_v0, g_v1, g_v2, cut_border)
            tetrahedra.append((g_v0, g_v1, g_v2, v_new))
            
        # Update Cut-Border implicitly by adding new faces
        # And removing the gate
        # This logic is mirrored from encoder
        
        # Update Hash Maps for Incidencies
        # ... (as described in notes)
        
    return tetrahedra
```

### Parameters
*   **Opcode Size:** 4 bits per opcode.
*   **Vertex Index (N, S, J):** 16 bits. Theoretically $\log_2(n)$ bits where $n$ is the number of vertices in the segment.
*   **Shared Vertex Global Index (S):** 32 bits. Theoretically $\log_2(m)$ bits where $m$ is the total number of vertices in the multi-resolution mesh.
*   **Total Opcodes:** 11 (plus 1 for Skip in non-manifold cases).

### Edge Cases and Pitfalls
*   **Non-Manifold Edges:** Edges incident to more than 4 triangles are not supported by the standard 11 opcodes. The encoder outputs a `Skip` symbol, discards the current gate, and proceeds to the next. This increases the symbol count to 12.
*   **4-Triangle Non-Manifold Edges:** Allowed. Adjacency is defined by the order of vertices in the triangles, avoiding geometry tests.
*   **Precision:** The use of fixed 16-bit and 32-bit indices may lead to larger file sizes than theoretically optimal ($\log n$), but simplifies implementation and improves performance [S1].
*   **Traversal Order:** The efficiency of the algorithm depends heavily on the traversal order. The FIFO-based point-wise traversal is crucial for cache coherence and compression ratio [S1].

## Subtopics

### Extended Cut-Border Machine (CBM) for Multi-Resolution
A compression scheme extending the CBM to support cross-segment incidencies and respect pre-computed mesh layouts [S1]. It stores segments sequentially in a binary file. Vertices on segment boundaries are copied into each segment they belong to, and an additional unique global index is stored for every border vertex to identify it uniquely across segments. The encoder uses the same 11 opcodes but distinguishes between in-patch vertices (opcode N) and shared vertices (opcode S) to reconstruct face incidencies across patches. This allows for multi-resolution rendering where segments can be refined or coarsened independently [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of tetrahedra, $O(N)$, assuming constant-time hash map operations. The traversal order is crucial for performance.
*   **Memory Complexity:** Linear with respect to the number of vertices and faces for storing the cut-border and hash maps.
*   **Compression Ratio:** Good, but not optimal. The use of 4-bit fixed opcodes and 16/32-bit indices instead of arithmetic coding limits compression. Arithmetic coding could improve ratios [S1].
*   **Strengths:** Simple implementation, supports non-manifold edges (up to 4 triangles), allows for multi-resolution and cross-segment incidencies, efficient traversal.
*   **Limitations:** Does not preserve original vertex ordering, fixed-size indices may waste space, non-manifold edges with $>4$ triangles are skipped.
*   **When to Use:** When compact storage of tetrahedral mesh connectivity is needed for object-order rendering, and the mesh may have non-manifold features or multi-resolution structures.
*   **When Not to Use:** When maximum compression is required (use arithmetic coding), or when the mesh has complex non-manifold structures with $>4$ triangles per edge.

## Background
The Cut-Border Machine was introduced by [S1] as a region-growing compression scheme for tetrahedral meshes. It builds on the work of Gumhold et al. [1999] for triangle meshes, extending it to tetrahedra. The method was developed to support interactive direct volume rendering, isosurface extraction, and vector field visualization using multi-triangulations. It allows for view-dependent rendering by adapting the mesh cut and encoding the connectivity efficiently. The basic CBM was later extended to support multi-resolution meshes with cross-segment incidencies [S1].

## Sources
[S1] Segment-based tetrahedral meshing and rendering, https://doi.org/10.1145/1174429.1174507, fetched 2026-11-06
