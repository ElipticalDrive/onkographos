# ChunkMesh Vertex Packing and Management

Also called: Block Face Culling, Cube Face Mesh Generation

## Overview
ChunkMesh Vertex Packing and Management is a dynamic mesh generation and update strategy for voxel terrain that combines face culling with compact vertex storage and incremental buffer updates [S1, unofficial]. It solves the problem of high memory bandwidth and draw call overhead in voxel rendering by packing vertex attributes into two 32-bit integers and managing mesh topology changes (addition/removal of blocks) through local face interaction logic [S1, unofficial]. It sets itself apart from static meshing methods by supporting real-time incremental updates via a free-list vertex allocator, and from pure geometry shaders by performing all topology calculations on the CPU [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Voxel world data (block types, chunk positions), camera position (for alpha sorting), and block modification events (add/remove).
*   **Output:** GPU vertex buffers (`vertex0`, `vertex1`) and index buffer (`indices`) representing the visible surface of a chunk.

**Data Structures:**
Each vertex is stored as two 32-bit unsigned integers (`uint`). The arrays `vertex0` and `vertex1` store these packed values. The `indices` array stores triangle indices. A free-list `vertexSlots` tracks reusable vertex indices.

*   **vertex0 (32-bit):**
    *   Bits 0–9: X coordinate (10 bits, range 0–1023).
    *   Bits 10–19: Y coordinate (10 bits, range 0–1023).
    *   Bits 20–29: Z coordinate (10 bits, range 0–1023).
    *   Bits 30–31: Unused.
*   **vertex1 (32-bit):**
    *   Bits 0–9: Texture layer index (10 bits).
    *   Bits 10–14: Texture coordinate U (5 bits, range 0–31).
    *   Bits 15–19: Texture coordinate V (5 bits, range 0–31).
    *   Bits 20–21: Lighting/shading value (2 bits, range 0–3).
    *   Bits 22–31: Unused.
*   **Indices:** Array of vertex indices forming triangles (6 indices per quad).

**Parameters:**
*   Block size $n = 16$ [S1, unofficial].
*   Direction mapping: 0=+X, 1=-X, 2=+Y, 3=-Y, 4=+Z, 5=-Z [S1, unofficial].
*   Lighting values: `[1, 1, 3, 0, 2, 2]` corresponding to directions 0–5 [S1, unofficial].
*   Texture coordinate mapping: `[2, 3, 0, 0, 3, 2]` for orientation mapping [S1, unofficial].

**Algorithm Steps:**

1.  **Face Visibility Determination:**
    For a voxel at $P_{current}$ and neighbor $P_{neighbor} = P_{current} + \mathbf{n}$, determine if the face is visible using `faceInteraction`.
    *   If adding a face: Add if neighbor is invisible/translucent/transparent AND block types differ. Remove if neighbor is NOT invisible AND current block is NOT translucent/transparent OR block types are identical.
    *   If removing a face: Reverse the logic [S1, unofficial].

2.  **Vertex Packing:**
    When adding a vertex, pack coordinates and attributes into `vertex0` and `vertex1`:
    $$
    \begin{aligned}
    \text{vertex0} &= x \mid (y \ll 10) \mid (z \ll 20) \\
    \text{vertex1} &= \text{textureLayer} \mid (\text{texcoordX} \ll 10) \mid (\text{texcoordY} \ll 15) \mid (\text{dLight} \ll 20)
    \end{aligned}
    $$
    [S1, unofficial]

3.  **Mesh Assembly (Add Face):**
    *   Calculate 4 vertex positions using `cubeFacePosition(x, y, z, dir)` [S1, unofficial].
    *   Calculate texture coordinates using `cubeFaceTexcoord(orientation)` [S1, unofficial].
    *   Retrieve a reusable vertex index from `vertexSlots` if available; otherwise, use the next available index [S1, unofficial].
    *   Call `addVertex` for each of the 4 vertices, packing data into `vertex0` and `vertex1` arrays [S1, unofficial].
    *   Push 6 indices (forming 2 triangles: a,b,c and c,d,a) to the `indices` array [S1, unofficial].
    *   Set `this.update = true` to trigger buffer refresh [S1, unofficial].

4.  **Mesh Update (Remove Face):**
    *   Search for the face in `vertex0` array using `findFace`. This reconstructs the packed `vertex0` uint for each of the 4 vertices and compares them against stored values, skipping NaN entries [S1, unofficial].
    *   If found, set `vertex0` and `vertex1` values at indices $i, i+1, i+2, i+3$ to `NaN` [S1, unofficial].
    *   Remove the 6 corresponding indices from the `indices` array using `splice` [S1, unofficial].
    *   Push the starting vertex index $i$ to `vertexSlots` for reuse [S1, unofficial].
    *   Set `this.update = true` [S1, unofficial].

5.  **Buffer Upload:**
    *   Bind VAO and buffers.
    *   Upload `vertex0`, `vertex1`, and `indices` as `Uint32Array` with `gl.DYNAMIC_DRAW` hint [S1, unofficial].
    *   Assign attributes `vertex0` and `vertex1` as integer attributes with stride 0 and offset 0 [S1, unofficial].

6.  **Alpha Sorting (Optional Extension):**
    For translucent blocks, sort indices based on camera position. Determine the "greatest direction" (X, Y, or Z) based on camera vector magnitude. Loop through blocks in reverse order of that direction to ensure correct depth sorting [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack_vertex0(x, y, z):
    return x | (y << 10) | (z << 20)

def pack_vertex1(texture_layer, texcoord_x, texcoord_y, d_light):
    return texture_layer | (texcoord_x << 10) | (texcoord_y << 15) | (d_light << 20)

def unpack_vertex0(v0):
    x = v0 & 1023
    y = (v0 & 1047552) >> 10
    z = v0 >> 20
    return x, y, z

def unpack_vertex1(v1):
    texture_layer = v1 & 1023
    texcoord_x = ((v1 & 31744) >> 10) * 0.0625
    texcoord_y = ((v1 & 1015808) >> 15) * 0.0625
    shade_data = ((v1 >> 20) * 0.2) + 0.4
    return texture_layer, texcoord_x, texcoord_y, shade_data

def cube_face_position(x, y, z, dir, n=16):
    x *= n
    y *= n
    z *= n
    if dir == 0: return [x+n, y, z, x+n, y+n, z, x+n, y+n, z+n, x+n, y, z+n]
    if dir == 1: return [x, y, z, x, y, z+n, x, y+n, z+n, x, y+n, z]
    if dir == 2: return [x, y+n, z, x, y+n, z+n, x+n, y+n, z+n, x+n, y+n, z]
    if dir == 3: return [x+n, y, z+n, x, y, z+n, x, y, z, x+n, y, z]
    if dir == 4: return [x, y, z+n, x+n, y, z+n, x+n, y+n, z+n, x, y+n, z+n]
    if dir == 5: return [x, y, z, x, y+n, z, x+n, y+n, z, x+n, y, z]

def cube_face_texcoord(orientation):
    # Returns 8 floats (4 pairs of u,v)
    if orientation == 0: return [0, 0, 0, 16, 16, 16, 16, 0]
    if orientation == 1: return [16, 0, 0, 0, 0, 16, 16, 16]
    # ... other orientations ...
    return [0, 0, 0, 16, 16, 16, 16, 0] # Placeholder for brevity

def add_vertex(vertex0_arr, vertex1_arr, vertex_slots, pos, texture, texcoord, d_light, index_hint=-1):
    if index_hint >= 0:
        idx = index_hint
    elif len(vertex_slots) > 0:
        idx = vertex_slots.pop()
    else:
        idx = len(vertex0_arr)
    
    v0 = pack_vertex0(pos[0], pos[1], pos[2])
    v1 = pack_vertex1(texture, texcoord[0], texcoord[1], d_light)
    
    vertex0_arr[idx] = v0
    vertex1_arr[idx] = v1
    return idx

def add_cube_face(x, y, z, dir, texture, vertex0_arr, vertex1_arr, indices_arr, vertex_slots):
    pos = cube_face_position(x, y, z, dir)
    texcoord = cube_face_texcoord([2, 3, 0, 0, 3, 2][dir])
    d_light = [1, 1, 3, 0, 2, 2][dir]
    
    a = add_vertex(vertex0_arr, vertex1_arr, vertex_slots, pos[0:3], texture, texcoord[0:2], d_light)
    b = add_vertex(vertex0_arr, vertex1_arr, vertex_slots, pos[3:6], texture, texcoord[2:4], d_light, a+1)
    c = add_vertex(vertex0_arr, vertex1_arr, vertex_slots, pos[6:9], texture, texcoord[4:6], d_light, a+2)
    d = add_vertex(vertex0_arr, vertex1_arr, vertex_slots, pos[9:12], texture, texcoord[6:8], d_light, a+3)
    
    indices_arr.extend([a, b, c, c, d, a])

def find_face(vertex0_arr, face_pos):
    # face_pos is 12 floats (4 vertices)
    for i in range(0, len(vertex0_arr), 4):
        if math.isnan(vertex0_arr[i]): continue
        v0_0 = pack_vertex0(face_pos[0], face_pos[1], face_pos[2])
        v0_1 = pack_vertex0(face_pos[3], face_pos[4], face_pos[5])
        v0_2 = pack_vertex0(face_pos[6], face_pos[7], face_pos[8])
        v0_3 = pack_vertex0(face_pos[9], face_pos[10], face_pos[11])
        
        if vertex0_arr[i] == v0_0 and vertex0_arr[i+1] == v0_1 and \
           vertex0_arr[i+2] == v0_2 and vertex0_arr[i+3] == v0_3:
            return i
    return -1

def remove_face(vertex0_arr, vertex1_arr, indices_arr, vertex_slots, face_pos):
    idx = find_face(vertex0_arr, face_pos)
    if idx == -1: return
    
    vertex0_arr[idx] = math.nan
    vertex0_arr[idx+1] = math.nan
    vertex0_arr[idx+2] = math.nan
    vertex0_arr[idx+3] = math.nan
    
    vertex1_arr[idx] = math.nan
    vertex1_arr[idx+1] = math.nan
    vertex1_arr[idx+2] = math.nan
    vertex1_arr[idx+3] = math.nan
    
    # Remove 6 indices starting from the index corresponding to vertex idx
    # Note: indices_arr is not sorted, so linear search for the start of the face's indices is needed
    # Assuming find_index finds the position in indices_arr where indices_arr[k] == idx
    k = find_index(indices_arr, idx)
    if k != -1:
        indices_arr[k:k+6] = []
    
    vertex_slots.append(idx)
```

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
*   **Time Complexity:** Face culling is $O(1)$ per voxel face. Adding/removing a block requires checking 6 neighbors, so $O(1)$ per block modification. Buffer upload is $O(V + I)$ where $V$ is vertices and $I$ is indices. Finding a face for removal is $O(V)$ in the worst case due to linear search, though optimizations like binary search on sorted indices can improve this [S1, unofficial].
*   **Memory Complexity:** $O(V)$ for vertex data, where each vertex uses 8 bytes (2x uint32). Index data is $O(I)$, where $I = 2V$ (2 triangles per quad). The free-list `vertexSlots` adds minimal overhead.
*   **Performance:** The use of `Uint32Array` and `gl.DYNAMIC_DRAW` allows efficient GPU updates for dynamic scenes. Packing attributes reduces memory bandwidth compared to storing floats. The method is suitable for real-time voxel terrain with frequent block modifications [S1, unofficial].
*   **Strengths:** Low memory footprint, efficient incremental updates, simple implementation.
*   **Limitations:** Linear search for face removal can be slow for large chunks. Alpha sorting adds CPU overhead. The method assumes a fixed block size and simple geometry (quads) [S1, unofficial].
*   **When to use:** Real-time voxel games with dynamic terrain modification.
*   **When not to use:** Static scenes where pre-computed meshes are better, or complex geometry requiring more than simple quads.

## Background
The method originates from the Exolands project, a voxel engine implementation [S1, unofficial]. It builds on the principles of face culling and visibility optimization [3861-face-culling-and-visibility-optimization.md]. The specific vertex packing scheme and incremental update logic were developed to address the performance bottlenecks of naive voxel meshing in JavaScript-based web environments [S1, unofficial].

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
