# ChunkMesh Vertex Packing and Management

Also called: Block Face Culling, Cube Face Mesh Generation

## Overview
ChunkMesh Vertex Packing and Management is a dynamic mesh generation and buffer management technique for voxel terrain, designed to minimize GPU memory overhead and draw calls by packing vertex attributes into compact integer formats and reusing vertex slots [S1, unofficial]. It solves the problem of sparse, high-latency buffer updates in incremental voxel editing by maintaining a contiguous vertex array with a free-list allocator, allowing faces to be added or removed without reallocating GPU memory [S1, unofficial]. Unlike static meshing methods, it supports real-time block placement and removal by updating only the affected chunk and its immediate neighbors, leveraging the neighbor check principles of [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) to determine face visibility [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Voxel world data (block types, chunk positions), camera position (for alpha sorting), and block modification events (add/remove).
*   **Output:** GPU vertex buffers (`vertex0`, `vertex1`) and index buffer (`indices`) representing the visible surface of the chunk.

**Data Structures:**
Vertex data is packed into two 32-bit unsigned integers (`uint`) per vertex to reduce buffer size and improve cache efficiency [S1, unofficial].

*   **`vertex0` (Position):**
    *   Bits 0–9: X coordinate (10 bits, range 0–1023).
    *   Bits 10–19: Y coordinate (10 bits, range 0–1023).
    *   Bits 20–29: Z coordinate (10 bits, range 0–1023).
    *   Bits 30–31: Unused.
*   **`vertex1` (Attributes):**
    *   Bits 0–9: Texture layer index (10 bits).
    *   Bits 10–14: Texture coordinate X (5 bits, range 0–31).
    *   Bits 15–19: Texture coordinate Y (5 bits, range 0–31).
    *   Bits 20–21: Lighting/shading value (2 bits, range 0–3).
    *   Bits 22–31: Unused.

The mesh maintains three arrays:
1.  `vertex0`: Array of `uint` storing packed positions.
2.  `vertex1`: Array of `uint` storing packed attributes.
3.  `indices`: Array of `uint` storing triangle indices (6 indices per quad).
4.  `vertexSlots`: A stack (free-list) storing indices of removed vertices for reuse.
5.  `update`: A boolean flag triggering buffer upload.

**Numbered Steps:**

1.  **Face Visibility Determination:**
    For each block at position $P=(x,y,z)$ with direction $dir \in \{0..5\}$ (positive/negative X, Y, Z), determine if a face is exposed using [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) principles [S1, unofficial].
    *   Calculate neighbor position $P_{neighbor} = P + N_{dir}$.
    *   Retrieve neighbor block data.
    *   Apply `faceInteraction` logic:
        *   If adding a block: Add face if neighbor is invisible/translucent/transparent AND block types differ. Remove face if neighbor is NOT invisible AND current block is NOT translucent/transparent OR block types are identical.
        *   If removing a block: Reverse the logic.
    *   Note: If the neighbor chunk does not exist, it must be loaded before updating its mesh [S1, unofficial].

2.  **Vertex Generation:**
    When a face is exposed, generate 4 vertices for the quad.
    *   **Position:** Calculate using `cubeFacePosition(x, y, z, dir)`. The block size $n=16$. The function returns 12 floats (3 floats $\times$ 4 vertices).
        *   Example for `dir=0` (Positive X): Vertices at $(x+16, y, z), (x+16, y+16, z), (x+16, y+16, z+16), (x+16, y, z+16)$.
    *   **Texture Coordinates:** Calculate using `cubeFaceTexcoord(orientation)`. Orientation mapping from direction: `[2, 3, 0, 0, 3, 2][dir]`. Returns 4 pairs of $(u,v)$.
    *   **Lighting:** Determine `dLight` using `[1, 1, 3, 0, 2, 2][dir]`.

3.  **Vertex Packing:**
    Pack the generated data into `vertex0` and `vertex1` integers [S1, unofficial]:
    $$
    \begin{aligned}
    \text{vertex0} &= x \ | \ (y \ll 10) \ | \ (z \ll 20) \\
    \text{vertex1} &= \text{textureLayer} \ | \ (\text{texcoordX} \ll 10) \ | \ (\text{texcoordY} \ll 15) \ | \ (\text{dLight} \ll 20)
    \end{aligned}
    $$
    Where `texcoordX` and `texcoordY` are integer values 0–31 derived from the float texture coordinates (scaled by 0.0625 in shader).

4.  **Vertex Allocation:**
    *   Pop an index from `vertexSlots`. If empty, append new indices to `vertex0` and `vertex1`.
    *   Call `addVertex` for each of the 4 vertices, storing the packed integers at the allocated indices.
    *   Push the 6 triangle indices ($a, b, c, c, d, a$) to the `indices` array.
    *   Set `this.update = true`.

5.  **Face Removal:**
    When a face is no longer exposed:
    *   Call `findFace(facePosition)` to locate the starting index in `vertex0`.
        *   `findFace` reconstructs the packed `vertex0` for the 4 vertices of the target face and compares them against the array, skipping `NaN` entries.
    *   If found:
        *   Set `vertex0` and `vertex1` at indices $i, i+1, i+2, i+3$ to `NaN`.
        *   Remove the 6 corresponding indices from the `indices` array using `splice`.
        *   Push the starting vertex index $i$ to `vertexSlots` for reuse.
        *   Set `this.update = true`.

6.  **Buffer Update:**
    When `this.update` is true:
    *   Bind VAO.
    *   Upload `vertex0`, `vertex1`, and `indices` to GPU buffers using `gl.DYNAMIC_DRAW` hint.
    *   Reset `this.update = false`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def add_cube_face(x, y, z, dir, texture_layer):
    # 1. Get positions and texcoords
    positions = cube_face_position(x, y, z, dir) # Returns 12 floats
    texcoords = cube_face_texcoord(dir)          # Returns 8 floats (4 pairs)
    d_light = [1, 1, 3, 0, 2, 2][dir]
    
    # 2. Allocate vertex indices
    if vertex_slots is not empty:
        base_index = vertex_slots.pop()
    else:
        base_index = len(vertex0)
        
    # 3. Pack and store vertices
    for k in range(4):
        px, py, pz = positions[k*3], positions[k*3+1], positions[k*3+2]
        tx, ty = texcoords[k*2], texcoords[k*2+1]
        
        # Pack vertex0
        v0 = px | (py << 10) | (pz << 20)
        vertex0.append_or_set(base_index + k, v0)
        
        # Pack vertex1
        # tx, ty are floats 0.0-1.0, convert to 0-31 range for packing
        # Note: Source uses integer texcoords 0-31 directly in packing logic
        # Assuming tx_int, ty_int are pre-scaled integers 0-31
        v1 = texture_layer | (tx_int << 10) | (ty_int << 15) | (d_light << 20)
        vertex1.append_or_set(base_index + k, v1)
        
    # 4. Add indices
    indices.append(base_index)
    indices.append(base_index + 1)
    indices.append(base_index + 2)
    indices.append(base_index + 2)
    indices.append(base_index + 3)
    indices.append(base_index)
    
    update = True

def remove_face(face_position):
    # face_position is 6 floats: 3 floats for vertex 1, 3 for vertex 2, etc.
    start_index = find_face(face_position)
    if start_index == -1:
        return
        
    # Mark vertices as invalid
    for k in range(4):
        vertex0[start_index + k] = NaN
        vertex1[start_index + k] = NaN
        
    # Remove indices
    # Find index in indices array where start_index is the first element
    idx_in_indices = find_index_in_indices_array(start_index)
    if idx_in_indices != -1:
        indices.splice(idx_in_indices, 6)
        
    # Recycle vertex slot
    vertex_slots.append(start_index)
    update = True

def find_face(face_position):
    # face_position: [x1,y1,z1, x2,y2,z2, x3,y3,z3, x4,y4,z4]
    packed_target = []
    for k in range(4):
        x, y, z = face_position[k*3], face_position[k*3+1], face_position[k*3+2]
        packed_target.append(x | (y << 10) | (z << 20))
        
    for i in range(0, len(vertex0), 4):
        if isnan(vertex0[i]):
            continue
        if vertex0[i] == packed_target[0] and \
           vertex0[i+1] == packed_target[1] and \
           vertex0[i+2] == packed_target[2] and \
           vertex0[i+3] == packed_target[3]:
            return i
    return -1

def update_buffers():
    if not update:
        return
    gl.bind_buffer(gl.ARRAY_BUFFER, vertex0_buffer)
    gl.buffer_data(gl.ARRAY_BUFFER, Uint32Array(vertex0), gl.DYNAMIC_DRAW)
    gl.bind_buffer(gl.ARRAY_BUFFER, vertex1_buffer)
    gl.buffer_data(gl.ARRAY_BUFFER, Uint32Array(vertex1), gl.DYNAMIC_DRAW)
    gl.bind_buffer(gl.ELEMENT_ARRAY_BUFFER, index_buffer)
    gl.buffer_data(gl.ELEMENT_ARRAY_BUFFER, Uint32Array(indices), gl.DYNAMIC_DRAW)
    update = False
```

**Parameters:**
*   Block size $n=16$ [S1, unofficial].
*   Vertex coordinate precision: 10 bits (0–1023) [S1, unofficial].
*   Texture coordinate precision: 5 bits (0–31) [S1, unofficial].
*   Lighting levels: 4 discrete values [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Neighbor Chunk Loading:** When updating a neighbor chunk's mesh (e.g., adding a face to an adjacent chunk), the neighbor chunk's mesh must be loaded if it does not already exist [S1, unofficial].
*   **Precision Loss:** Coordinates are limited to 10 bits. If chunk coordinates exceed 1023 units, overflow occurs. This is typically managed by chunking the world into 16x16x16 blocks, where local coordinates fit within 10 bits (0–15 scaled to 0–1023 via multiplication by 16? No, the source says `x *= n` where `n=16`, and `x` is block coordinate. If block coordinate is 0-63, `x*16` is 0-1008, fitting in 10 bits).
*   **Index Search:** `findFace` is $O(V)$ where $V$ is the number of vertices. For large chunks, this may be slow. The source uses a linear scan [S1, unofficial].

## Subtopics

### [Voxel Face Meshing with Packed Attributes](4153-voxel-face-meshing-with-packed-attributes.md)
Optimizes storage by packing position, type, and direction into a single float attribute, unpacked in the vertex shader. Choose when buffer bandwidth is the bottleneck.

### [Mesh Reindexing and Vertex Compaction](4160-mesh-reindexing-and-vertex-compaction.md)
Compacts vertex buffers by removing unreferenced vertices. Choose when vertex count is significantly higher than needed due to incremental updates.

### [Chunk Mesh Data Structure & Buffer Update](4182-chunk-mesh-data-structure-buffer-update.md)
Manages GPU buffers using a single large SSBO with a free-list allocator. Choose for dynamic scenes with frequent block changes.

### [Alpha/Translucent Block Rendering](4860-alpha-translucent-block-rendering.md)
Renders semi-transparent blocks using pre-computed attributes and fragment shading. Choose when transparency is required.

## Operating characteristics
*   **Time Complexity:**
    *   Face Addition/Removal: $O(1)$ for packing and index management, but $O(V)$ for `findFace` if searching by position [S1, unofficial].
    *   Buffer Upload: $O(V + I)$ where $V$ is vertex count and $I$ is index count.
*   **Memory Complexity:** $O(V)$ for vertex arrays and $O(I)$ for index arrays. Packing reduces vertex size from 32+ bytes (standard float3) to 8 bytes (two uint32s) [S1, unofficial].
*   **Performance:** The use of `gl.DYNAMIC_DRAW` and buffer reuse minimizes allocation overhead. The free-list allocator (`vertexSlots`) prevents fragmentation and reuse of memory [S1, unofficial].
*   **Strengths:** Low memory footprint, fast incremental updates, no GPU buffer reallocation.
*   **Limitations:** Linear search for face removal (`findFace`) can be slow for very large chunks. Coordinate precision is limited to 10 bits per axis.
*   **When to Use:** Real-time voxel editors, Minecraft-like games with frequent block changes.
*   **When Not to Use:** Static meshes, scenes requiring high-precision coordinates (>1023 units), or when transparency sorting is complex (requires additional logic as seen in [Alpha/Translucent Block Rendering](4860-alpha-translucent-block-rendering.md)).

## Background
The method originates from the Exolands project, an open-source voxel engine [S1, unofficial]. It was developed to handle dynamic terrain editing efficiently by combining face culling with compact vertex packing and dynamic buffer management. The approach is inspired by traditional voxel meshing techniques but optimized for JavaScript/OpenGL environments where memory allocation overhead is significant.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
