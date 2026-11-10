# ChunkMesh Vertex Packing and Management

Also called: Block Face Culling, Cube Face Mesh Generation

## Overview
ChunkMesh Vertex Packing and Management is a method for generating and maintaining dynamic surface meshes for voxel-based environments by packing vertex attributes into compact integer buffers and managing face visibility through neighbor checks. It solves the problem of high memory overhead and frequent GPU buffer updates in voxel terrain rendering by using a sparse vertex array with reuse slots and two 32-bit integers per vertex to store position, texture, and lighting data [S1, unofficial]. It sets itself apart from static meshing methods by supporting incremental addition and removal of blocks with immediate neighbor face updates, and from pure face culling methods by integrating the vertex storage layout directly into the generation logic [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Voxel world state (block types, chunk positions), camera position (for alpha sorting variants), and block update events (add/remove at $x, y, z$).
*   **Output:** GPU vertex buffers (`vertex0`, `vertex1`) and index buffer (`indices`) representing the visible surface of the chunk.

**Data Structures:**
The core data structure consists of three parallel arrays:
1.  `vertex0`: `Uint32Array` storing packed position data.
2.  `vertex1`: `Uint32Array` storing packed attribute data.
3.  `indices`: `Uint32Array` storing triangle indices.
4.  `vertexSlots`: Array acting as a free-list for reusable vertex indices.

**Vertex Packing Layout:**
Each vertex is represented by two 32-bit unsigned integers.

*   **`vertex0` (Position):**
    *   Bits 0–9: X coordinate (10 bits, range 0–1023).
    *   Bits 10–19: Y coordinate (10 bits, range 0–1023).
    *   Bits 20–29: Z coordinate (10 bits, range 0–1023).
    *   Bits 30–31: Unused.
    *   *Note:* The coordinate system assumes a block size $n=16$. Coordinates are scaled by $n$ before packing.

*   **`vertex1` (Attributes):**
    *   Bits 0–9: Texture layer index (10 bits).
    *   Bits 10–14: Texture coordinate X (5 bits, range 0–31).
    *   Bits 15–19: Texture coordinate Y (5 bits, range 0–31).
    *   Bits 20–21: Lighting/shading value (2 bits, range 0–3).
    *   Bits 22–31: Unused.

**Numbered Steps:**

1.  **Initialization:**
    *   Initialize `vertex0`, `vertex1`, and `indices` as empty arrays.
    *   Initialize `vertexSlots` as empty.
    *   Set `this.update = false`.

2.  **Face Generation (`addCubeFace`):**
    *   Given block position $(x, y, z)$, direction $dir \in \{0..5\}$, and texture key.
    *   Calculate vertex positions using `cubeFacePosition(x, y, z, dir)`.
        *   Scale coordinates: $X = x \cdot 16, Y = y \cdot 16, Z = z \cdot 16$.
        *   Return 12 floats representing 4 vertices (2 triangles).
        *   Direction mapping: 0=+X, 1=-X, 2=+Y, 3=-Y, 4=+Z, 5=-Z.
    *   Calculate texture coordinates using `cubeFaceTexcoord(dir)`.
        *   Mapping: $[2, 3, 0, 0, 3, 2][dir]$.
        *   Returns 4 floats (2 u,v pairs).
    *   Calculate lighting value $dLight$ using $[1, 1, 3, 0, 2, 2][dir]$.
    *   Reuse vertex indices:
        *   If `vertexSlots` is not empty, pop `vertexIndex` from `vertexSlots`.
        *   Else, `vertexIndex` is the current length of `vertex0`.
    *   Add 4 vertices:
        *   For each vertex $k \in \{0, 1, 2, 3\}$:
            *   Pack position: $v0 = pos_x | (pos_y \ll 10) | (pos_z \ll 20)$.
            *   Pack attributes: $v1 = texture | (texX \ll 10) | (texY \ll 15) | (dLight \ll 20)$.
            *   Store at `vertex0[vertexIndex + k]` and `vertex1[vertexIndex + k]`.
    *   Add indices:
        *   Push $(vertexIndex, vertexIndex+1, vertexIndex+2, vertexIndex+2, vertexIndex+3, vertexIndex)$ to `indices`.
    *   Set `this.update = true`.

3.  **Face Removal (`removeFace`):**
    *   Input: `facePosition` (array of 12 floats representing the 4 vertices of the face to remove).
    *   Find the starting index in `vertex0` using `findFace(facePosition)`.
        *   Iterate `i` from 0 to `vertex0.length` with step 4.
        *   Skip if `vertex0[i]` is NaN.
        *   Reconstruct packed `vertex0` for each of the 4 vertices of the target face and compare against `vertex0[i...i+3]`.
        *   Return `i` if match found, else -1.
    *   If found:
        *   Set `vertex0[i...i+3]` and `vertex1[i...i+3]` to `NaN`.
        *   Remove the 6 corresponding indices from `indices` using `splice`.
        *   Push `i` to `vertexSlots` for reuse.
        *   Set `this.update = true`.

4.  **Neighbor Interaction (`faceInteraction`):**
    *   Determine if a face should be added or removed based on block types and properties.
    *   Inputs: `blockData0` (current), `blockData1` (neighbor), `add` (boolean, true if adding block, false if removing).
    *   Logic:
        *   If `add` is true:
            *   If `blockInfo0.isInvisible`, return `[false, false]`.
            *   If `blockInfo1.isInvisible || blockInfo1.isTranslucent || blockInfo1.isTransparent` AND `blockData0.type != blockData1.type`, set `addFace = true`.
            *   If `!blockInfo1.isInvisible` AND `(!blockInfo0.isTranslucent && !blockInfo0.isTransparent)` OR `blockData0.type == blockData1.type`, set `removeFace = true`.
        *   If `add` is false (removing block):
            *   If `blockInfo1.isInvisible || blockInfo1.isTranslucent || blockInfo1.isTransparent` AND `blockData0.type != blockData1.type`, set `removeFace = true`.
            *   If `!blockInfo1.isInvisible` AND `(!blockInfo0.isTranslucent && !blockInfo0.isTransparent)` OR `blockData0.type == blockData1.type`, set `addFace = true`.
    *   Return `[addFace, removeFace]`.

5.  **Block Update (`updateBlock`):**
    *   Iterate over 6 directions $i \in \{0..5\}$.
    *   Calculate neighbor coordinates $(tx, ty, tz)$.
    *   Get neighbor block data and info.
    *   Call `faceInteraction`.
    *   If `addFace` is true:
        *   If adding block: Add face to current chunk.
        *   If removing block: Add face to neighbor chunk (load neighbor mesh if needed).
    *   If `removeFace` is true:
        *   If adding block: Remove face from neighbor chunk.
        *   If removing block: Remove face from current chunk.
    *   Direction mapping for neighbor: $dir_{neighbor} = i + (i \% 2 === 0 ? 1 : -1)$.

6.  **Buffer Upload:**
    *   When `this.update` is true:
        *   Bind VAO.
        *   Upload `vertex0` to `vertex0Buffer` using `gl.DYNAMIC_DRAW`.
        *   Upload `vertex1` to `vertex1Buffer` using `gl.DYNAMIC_DRAW`.
        *   Upload `indices` to `indexBuffer` using `gl.DYNAMIC_DRAW`.
        *   Set `this.update = false`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def add_vertex(vertex0, vertex1, vertex_slots, pos, texture, texcoord, dlight, reuse_index=-1):
    if reuse_index >= 0:
        idx = reuse_index
    else:
        idx = len(vertex0)
    
    # Pack position
    v0 = pos[0] | (pos[1] << 10) | (pos[2] << 20)
    # Pack attributes
    v1 = texture | (texcoord[0] << 10) | (texcoord[1] << 15) | (dlight << 20)
    
    vertex0.append(v0)
    vertex1.append(v1)
    return idx

def add_cube_face(vertex0, vertex1, indices, vertex_slots, x, y, z, dir, texture):
    n = 16
    # Get positions and texcoords based on dir
    positions = cube_face_position(x, y, z, dir, n)
    texcoords = cube_face_texcoord(dir)
    dlight = [1, 1, 3, 0, 2, 2][dir]
    
    # Reuse index if available
    if vertex_slots:
        vertex_index = vertex_slots.pop()
    else:
        vertex_index = len(vertex0)
        
    # Add 4 vertices
    a = add_vertex(vertex0, vertex1, vertex_slots, positions[0:3], texture, texcoords[0:2], dlight, vertex_index)
    b = add_vertex(vertex0, vertex1, vertex_slots, positions[3:6], texture, texcoords[2:4], dlight, vertex_index + 1)
    c = add_vertex(vertex0, vertex1, vertex_slots, positions[6:9], texture, texcoords[4:6], dlight, vertex_index + 2)
    d = add_vertex(vertex0, vertex1, vertex_slots, positions[9:12], texture, texcoords[6:8], dlight, vertex_index + 3)
    
    # Add indices
    indices.extend([a, b, c, c, d, a])
    
    return True

def remove_face(vertex0, vertex1, indices, vertex_slots, face_position):
    idx = find_face(vertex0, face_position)
    if idx == -1:
        return False
    
    # Mark as removed
    vertex0[idx:idx+4] = [float('nan')] * 4
    vertex1[idx:idx+4] = [float('nan')] * 4
    
    # Remove indices
    # Note: indices are not sorted, so linear search for the start index is required
    start_idx = find_index_in_indices(indices, idx)
    if start_idx != -1:
        del indices[start_idx:start_idx+6]
        
    vertex_slots.append(idx)
    return True

def find_face(vertex0, face_position):
    # Reconstruct packed positions for the target face
    target_v0 = []
    for i in range(0, 12, 3):
        x, y, z = face_position[i], face_position[i+1], face_position[i+2]
        target_v0.append(x | (y << 10) | (z << 20))
        
    for i in range(0, len(vertex0), 4):
        if vertex0[i] != vertex0[i]: # Check for NaN
            continue
        if (vertex0[i] == target_v0[0] and
            vertex0[i+1] == target_v0[1] and
            vertex0[i+2] == target_v0[2] and
            vertex0[i+3] == target_v0[3]):
            return i
    return -1

def find_index_in_indices(indices, vertex_idx):
    for i in range(0, len(indices), 6):
        if indices[i] == vertex_idx:
            return i
    return -1
```

**Edge Cases and Pitfalls:**
*   **NaN Comparison:** When searching for faces, `NaN` values must be explicitly skipped, as `NaN != NaN` in IEEE 754.
*   **Neighbor Chunk Loading:** When removing a block, the neighbor chunk's mesh must be loaded if it does not exist before adding the exposed face to it [S1, unofficial].
*   **Index Splicing:** Removing indices from the middle of the array shifts subsequent indices. The `findIndex` helper must locate the specific face's start index to splice correctly.
*   **Precision:** Coordinates are limited to 10 bits (0-1023). For a block size of 16, this covers a chunk of $16 \times 1024 = 16384$ units, which is sufficient for standard chunk sizes but requires careful scaling.

## Subtopics

### [Voxel Face Meshing with Packed Attributes](4153-voxel-face-meshing-with-packed-attributes.md)
A mesh generation technique for voxel terrain where vertex data (position, type, direction) is packed into a single float attribute to reduce buffer overhead, unpacked in the vertex shader, and used to generate quad faces for visible voxel surfaces.

### [Mesh Reindexing and Vertex Compaction](4160-mesh-reindexing-and-vertex-compaction.md)
This method reduces the memory footprint of a chunk-based mesh by removing unreferenced vertices and renumbering the remaining ones to form a dense, contiguous array. It solves the problem of sparse vertex buffers that arise during incremental blocky meshing, where vertices are allocated per voxel face but many are culled or unused. Unlike [Voxel Face Meshing with Packed Attributes](4153-voxel-face-meshing-with-packed-attributes.md), which focuses on storage layout, this method actively compacts the data. It differs from [Chunk Mesh Data Structure & Buffer Update](4182-chunk-mesh-data-structure-buffer-update.md) by performing the logical remapping rather than just updating buffer pointers.

### [Chunk Mesh Data Structure & Buffer Update](4182-chunk-mesh-data-structure-buffer-update.md)
This method manages the creation, population, and recycling of GPU vertex buffers for meshed chunk geometry by utilizing a single large Shader Storage Buffer (SSBO) known as the "MegaSSBO". It solves the problem of dynamic memory allocation overhead and fragmentation in voxel terrain rendering by implementing a free-list allocator within the SSBO to handle the variable face counts of individual chunks. It differs from per-chunk buffer allocation by centralizing memory management, and from static buffer approaches by supporting incremental updates and defragmentation.

### [Alpha/Translucent Block Rendering](4860-alpha-translucent-block-rendering.md)
Alpha/Translucent Block Rendering is a fragment-shader-based technique for rendering semi-transparent voxel blocks (such as water or glass) within a chunked voxel world. It solves the problem of visualizing translucent geometry without requiring complex ray-marching or volumetric accumulation per pixel, relying instead on standard rasterization with pre-computed vertex attributes. It differs from opaque block meshing by applying per-fragment shading and fog based on world-space depth, and from volumetric raycasting by operating on discrete mesh faces rather than continuous density fields.

## Operating characteristics
*   **Time Complexity:** Face addition/removal is $O(1)$ for buffer updates if indices are reused, but $O(N)$ for face search if the free-list is not used or if the face must be found by position. The `findFace` operation is $O(V)$ where $V$ is the number of vertices, as it scans the array.
*   **Memory Complexity:** $O(F)$ where $F$ is the number of visible faces. Each face uses 4 vertices (8 integers) and 6 indices (6 integers). The packing reduces memory bandwidth compared to floating-point attributes.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales well with chunk size due to localized updates. Neighbor chunk updates are required for boundary faces, which may cause synchronization overhead in multi-threaded environments.
*   **Strengths:** Low memory footprint due to integer packing; efficient reuse of vertex slots; simple neighbor-based culling logic.
*   **Limitations:** Linear search for face removal can be slow for large chunks; limited coordinate range (10 bits); requires careful handling of NaN values.
*   **When to use:** Suitable for dynamic voxel worlds with frequent block updates where memory bandwidth is a bottleneck.
*   **When not to use:** Not suitable for static meshes where compaction is preferred, or for worlds with very large coordinate ranges requiring more than 10 bits for position.

## Background
The method originates from the Exolands project, an open-source voxel engine implementation [S1, unofficial]. It was developed to address the performance issues of naive voxel meshing, where every voxel face is rendered regardless of visibility. The approach of packing vertex data into integers and using a free-list for vertex reuse was introduced to optimize GPU buffer updates and reduce memory usage in JavaScript-based voxel renderers [S1, unofficial].

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
