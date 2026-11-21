# Streaming Tetrahedral Mesh Connectivity Compression

Also called: Isenburg et al. connectivity compression, Predictive Geometry Compression for Tetrahedral Meshes, SCTVM, Streaming Tet Mesh Compression, Streaming Vertex Position Compression

## Overview
Streaming Tetrahedral Mesh Connectivity Compression is an algorithm for compressing tetrahedral volume meshes by incrementally encoding tetrahedra and vertices as they stream in, allowing memory to be freed immediately after use [S1, unofficial]. It solves the problem of compressing gigabyte-sized meshes that do not fit in main memory by avoiding the need to keep the entire uncompressed mesh in memory [S1, unofficial]. It sets itself apart from in-core methods like [Tetrahedral Mesh Connectivity Encoding](8957-tetrahedral-mesh-connectivity-encoding.md) by processing data in a streaming fashion with a small delay buffer for local reordering, and from [Implicit Connectivity Mesh (ICM)](11226-implicit-connectivity-mesh-icm.md) by explicitly managing active vertex finalization and dynamic indexing based on stream width rather than global indices [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A stream of tetrahedra and vertices. The stream can be interleaved or block-based, but must be reasonably coherent.
- Output: A bit stream containing connectivity opcodes, dynamic indices, and compressed vertex positions.

**Data Structures:**
- **Active Vertices:** Stored in a dynamic vector. Each vertex is active from the moment it is first referenced until it is finalized.
- **Active Half-Edges:** Linked into units of three describing oriented faces. Each active half-edge stores:
    1. A pointer to its origin vertex.
    2. A pointer to the next half-edge of the tetrahedral face it lives in.
    3. A copy of the position of the tip vertex (the non-incident fourth vertex of the tetrahedron that "created" the face) [S1, unofficial].
- **Vertex Lists:** Each active vertex stores a list of pointers to the active half-edges for which the vertex is the origin [S1, unofficial].
- **Delay Buffer:** A circular buffer holding up to $N$ tetrahedra (typically 10,000) to allow local reordering [S1, unofficial].

**Parameters:**
- **Delay Buffer Size ($N$):** Typically 10,000 tetrahedra. Smaller sizes (100, 1,000) reduce memory but increase bit rates due to less sharing [S1, unofficial].
- **Quantization Precision:** User-defined, typically 16 bits per coordinate [S1, unofficial].
- **Stream Width ($w$):** The maximal number of vertices in memory simultaneously. Dynamic indices use $\lceil \log_2 w \rceil$ bits [S1, unofficial].

**Steps:**
1. **Initialization:** Initialize the delay buffer, active vertex list, and half-edge structures.
2. **Buffering:** Read tetrahedra from the input stream into the delay buffer.
3. **Reordering:** Select the next tetrahedron from the buffer that maximizes sharing (face, edge, or vertex) with the previously compressed tetrahedron [S1, unofficial].
4. **Configuration Detection:** Determine the configuration of the selected tetrahedron relative to the active cut-border. Classify it as one of five labels: START, ADD, JOIN, FLIP, or FILL [S1, unofficial].
5. **Connectivity Encoding:**
    - Encode the configuration label using an arithmetic coder with context switching based on the previous label [S1, unofficial].
    - **START:** Topologically rotate so active vertices come before new ones. Encode the count of active vertices. Address non-shared active vertices via dynamic indexing ($\log_2 w$ bits). Compress new vertices with predictive coding and add to the dynamic vector [S1, unofficial].
    - **ADD:** Ideally, the active face is shared with the previous tetrahedron. If not, share an edge or vertex. Otherwise, address one vertex via dynamic indexing. Address remaining vertices using local indices into active half-edge lists of known vertices or the intersection of two known vertices' lists. Compress the new vertex and add to the dynamic vector [S1, unofficial].
    - **JOIN:** Like ADD, but also needs to address the fourth vertex. Can be done locally through half-edge lists of the active face vertices. Otherwise, use dynamic indexing [S1, unofficial].
    - **FLIP:** One face is addressed like an ADD. The other face is specified locally within the intersection of half-edge lists maintained with the two vertices shared by both faces [S1, unofficial].
    - **FILL:** One face is addressed like an ADD. The other two faces are addressed using half-edge lists of known vertices. This also covers the case where all four faces are active [S1, unofficial].
    - **Local Indexing:** If a vertex is reachable via an active half-edge, reference it locally within the list of half-edges maintained by a known vertex, or within the intersection of lists of two already known vertices. If an intersection results in only one correctly oriented face, no further information is stored for that face [S1, unofficial].
6. **Finalization Encoding:** For each of the four vertices of the current tetrahedron, specify whether it is finalized (referenced for the last time) using context-sensitive arithmetic coding. The context is based on the total number of tetrahedra referencing the vertex and the current number of active half-edges around it [S1, unofficial].
7. **Memory Management:** If a vertex is finalized, remove it from the active vertex list and deallocate its associated half-edges [S1, unofficial].
8. **Geometry Compression:** When a new vertex is encountered, compress its position:
    - Quantize the floating-point position to the user-defined precision (e.g., 16 bits per coordinate) [S1, unofficial].
    - Predict the position based on known neighbors using a predictor (e.g., Midpoint, Mirror, Heightmirror, Baseheight) [S1, unofficial].
    - Calculate the correction vector (actual position - predicted position) [S1, unofficial].
    - Optionally, express the correction vector in a local coordinate system (one axis along the base triangle's normal, origin at the midpoint) to separate normal and tangential errors [S1, unofficial].
    - Encode the correction vector [S1, unofficial].
9. **Loop:** Repeat steps 2-8 until the stream is exhausted.

**Formulas:**
- **Mirror Rule Prediction:** $P = 2M - A$, where $M$ is the midpoint (centroid) of the known base triangle of the tetrahedron, and $A$ is the tip vertex of the already known tetrahedron sharing the base [S1, unofficial].
- **Midpoint Rule Prediction:** $P = M$, where $M$ is the midpoint (centroid) of the known base triangle of the tetrahedron [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def streaming_tet_compression(input_stream, delay_buffer_size=10000, quantization_bits=16):
    active_vertices = []  # List of active vertex objects
    active_half_edges = []  # List of active half-edge objects
    delay_buffer = []  # Circular buffer for tetrahedra
    compressed_bits = []  # Output bit stream
    
    # Helper to get dynamic index bits
    def get_dynamic_index_bits():
        return ceil(log2(len(active_vertices))) if active_vertices else 0

    # Helper to predict vertex position
    def predict_position(base_triangle_vertices, known_tip_vertex=None):
        # Calculate midpoint M of base triangle
        M = sum(base_triangle_vertices) / 3.0
        if known_tip_vertex is not None:
            # Mirror Rule
            return 2 * M - known_tip_vertex
        else:
            # Midpoint Rule
            return M

    # Helper to quantize and encode correction
    def encode_vertex_position(new_vertex_pos, base_triangle_vertices, known_tip_vertex=None):
        predicted = predict_position(base_triangle_vertices, known_tip_vertex)
        correction = new_vertex_pos - predicted
        # Quantize correction to quantization_bits
        quantized_correction = quantize(correction, quantization_bits)
        # Encode quantized_correction to compressed_bits
        arithmetic_encode(quantized_correction)

    while input_stream has more data:
        # 1. Buffering
        if len(delay_buffer) < delay_buffer_size:
            tet = input_stream.read_tetrahedron()
            delay_buffer.append(tet)
        
        # 2. Reordering
        # Select tet from delay_buffer that shares max elements with last_processed_tet
        current_tet = select_best_tet(delay_buffer, last_processed_tet)
        delay_buffer.remove(current_tet)
        
        # 3. Configuration Detection
        config = detect_config(current_tet, active_vertices, active_half_edges)
        
        # 4. Connectivity Encoding
        arithmetic_encode_config(config, prev_config)
        
        if config == START:
            # Rotate so active vertices come first
            active_count = count_active_vertices(current_tet)
            arithmetic_encode(active_count)
            for i in range(active_count):
                if not shared_with_last(current_tet.vertices[i]):
                    idx = get_index_in_active_vertices(current_tet.vertices[i])
                    arithmetic_encode_dynamic_index(idx, get_dynamic_index_bits())
            # Encode new vertices
            for v in current_tet.vertices[active_count:]:
                encode_vertex_position(v, get_base_triangle(current_tet), get_known_tip(current_tet))
                add_to_active_vertices(v)
                
        elif config == ADD:
            # Address active face (shared, edge, vertex, or dynamic)
            address_active_face(current_tet, active_vertices, active_half_edges, compressed_bits)
            # Address remaining vertices via local indices or dynamic
            address_remaining_vertices(current_tet, active_vertices, active_half_edges, compressed_bits)
            # Encode new vertex
            new_v = get_new_vertex(current_tet)
            encode_vertex_position(new_v, get_base_triangle(current_tet), get_known_tip(current_tet))
            add_to_active_vertices(new_v)
            
        elif config == JOIN:
            # Like ADD, but address fourth vertex locally or dynamically
            address_active_face(current_tet, active_vertices, active_half_edges, compressed_bits)
            address_fourth_vertex(current_tet, active_vertices, active_half_edges, compressed_bits)
            # Encode new vertex
            new_v = get_new_vertex(current_tet)
            encode_vertex_position(new_v, get_base_triangle(current_tet), get_known_tip(current_tet))
            add_to_active_vertices(new_v)
            
        elif config == FLIP:
            # Address one face like ADD, other locally via intersection
            address_one_face(current_tet, active_vertices, active_half_edges, compressed_bits)
            address_other_face_locally(current_tet, active_vertices, active_half_edges, compressed_bits)
            
        elif config == FILL:
            # Address one face like ADD, other two via half-edge lists
            address_one_face(current_tet, active_vertices, active_half_edges, compressed_bits)
            address_other_faces_via_lists(current_tet, active_vertices, active_half_edges, compressed_bits)
            
        # 5. Finalization Encoding
        for v in current_tet.vertices:
            is_finalized = is_last_reference(v)
            arithmetic_encode_finalization(v, is_finalized)
            if is_finalized:
                remove_from_active_vertices(v)
                deallocate_half_edges(v)
                
        prev_config = config
        last_processed_tet = current_tet

    return compressed_bits
```

## Subtopics

### [Mesh Layout for Streaming](14444-mesh-layout-for-streaming.md)
Organizes tetrahedral meshes on disk in a coherent, I/O-efficient format to minimize memory footprint during streaming simplification. It prioritizes topological contiguity to reduce the "front width" and "span" of the active mesh portion, enabling the use of fixed-size circular arrays instead of hash tables. Choose this when I/O efficiency and memory footprint are critical constraints for streaming operations.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of tetrahedra, $O(N)$, where $N$ is the number of tetrahedra. The delay buffer reordering adds a constant factor overhead proportional to the buffer size [S1, unofficial].
- **Memory Complexity:** Proportional to the stream width $w$, which is the maximal number of active vertices. This is significantly lower than in-core methods that require $O(V)$ memory for all vertices [S1, unofficial]. For example, on the 'sf1' mesh, it uses 20 MB compared to 645 MB for conventional methods [S1, unofficial].
- **Measured Performance:** On the 'torso' mesh with a 10,000 tetrahedron delay buffer, connectivity compression achieves approximately 3.88 bpt (bits per tetrahedron) with explicit finalization [S1, unofficial]. Total bit rates (connectivity + geometry) are 25-40% larger than Gumhold et al.'s in-core method but achieved with drastically less memory and less than half the processing time (44 seconds vs. 100 seconds for 'sf1') [S1, unofficial].
- **Strengths:** Low memory footprint, fast processing, suitable for meshes larger than main memory, robust to input order with a sufficient delay buffer [S1, unofficial].
- **Limitations:** Connectivity compression rates are higher (worse) than in-core methods (almost twice as much space) [S1, unofficial]. Predictive geometry compression is less effective for grid-aligned data (e.g., 'sf1') where delta coding is better [S1, unofficial]. Unfinalized vertices can lead to significant memory increases if not handled correctly [S1, unofficial].
- **When to Use:** Large tetrahedral meshes that do not fit in main memory, or when streaming I/O is preferred.
- **When Not to Use:** When maximum compression ratio is the primary goal and memory is not a constraint, or for meshes with very large boundaries where guessed finalization leads to high memory usage [S1, unofficial].

## Background
The method was introduced by Isenburg and Lindström in their paper "Streaming compression of tetrahedral volume meshes" [S1, unofficial]. It extends the streaming mesh compression approach for triangle meshes by Isenburg et al. [14] to tetrahedral volume meshes. The work addresses the limitations of in-core compression methods like those by Gumhold et al. [4], which require the entire mesh in memory and global reordering. The algorithm was demonstrated at Graphics Interface 2006 [S1, unofficial].

## Sources
[S1] Streaming compression of tetrahedral volume meshes, https://digital.library.unt.edu/ark:/67531/metadc886884/, fetched 2026-10-26 (unofficial)
