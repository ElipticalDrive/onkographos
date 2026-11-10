# ChunkMesh Vertex Packing and Management

Also called: Block Face Culling, Cube Face Mesh Generation

## Overview
ChunkMesh Vertex Packing and Management is a dynamic mesh generation and memory management technique for voxel terrain that packs vertex attributes into compact 32-bit integers and manages buffer updates via a free-list allocator. It solves the problem of high memory overhead and frequent GPU buffer uploads in blocky meshing by reducing vertex data size and enabling efficient incremental updates [S1, unofficial]. It differs from static meshing by supporting runtime addition/removal of blocks with neighbor synchronization, and from pure culling methods by actively managing the vertex buffer layout and reuse [S1, unofficial].

## How it works
**Inputs:** Voxel world data (block types, chunk positions), camera position (for alpha sorting in variants), and block modification events (add/remove).
**Outputs:** GPU vertex buffers (`vertex0`, `vertex1`) and index buffer (`indices`) representing the visible surface of the chunk.

**Data Structures:**
*   `vertex0`: `Uint32Array` of packed 32-bit integers.
*   `vertex1`: `Uint32Array` of packed 32-bit integers.
*   `indices`: `Uint32Array` of triangle indices.
*   `vertexSlots`: Stack of available vertex indices for reuse.
*   `update`: Boolean flag triggering buffer upload.

**Packed Layout:**
Each vertex is stored in two 32-bit unsigned integers.

*   `vertex0` (Position):
    *   Bits 0-9: X coordinate (10 bits, range 0-1023).
    *   Bits 10-19: Y coordinate (10 bits, range 0-1023).
    *   Bits 20-29: Z coordinate (10 bits, range 0-1023).
    *   Bits 30-31: Unused.
*   `vertex1` (Attributes):
    *   Bits 0-9: Texture layer (10 bits).
    *   Bits 10-14: Texture coordinate X (5 bits, range 0-31).
    *   Bits 15-19: Texture coordinate Y (5 bits, range 0-31).
    *   Bits 20-21: Lighting/shading value (2 bits, range 0-3).
    *   Bits 22-31: Unused.

**Parameters:**
*   Block size ($n$): 16 units [S1, unofficial].
*   Direction mapping: 0=+X, 1=-X, 2=+Y, 3=-Y, 4=+Z, 5=-Z [S1, unofficial].
*   Lighting values: Array `[1, 1, 3, 0, 2, 2]` indexed by direction [S1, unofficial].
*   Texcoord orientation mapping: Array `[2, 3, 0, 0, 3, 2]` indexed by direction [S1, unofficial].

**Steps:**

1.  **Initialize:** Create empty `vertex0`, `vertex1`, `indices` arrays. Initialize `vertexSlots` as empty. Set `update = false`.
2.  **Add Block:** When a block is added at $(x, y, z)$, call `updateBlock(x, y, z, true, blockData)`.
3.  **Remove Block:** When a block is removed at $(x, y, z)$, call `updateBlock(x, y, z, false, prevBlockData)`.
4.  **Update Block Logic:**
    *   Iterate over 6 directions ($i = 0$ to $5$).
    *   Calculate neighbor coordinates $(tx, ty, tz)$.
    *   Retrieve neighbor block data and info.
    *   Call `faceInteraction(blockData0, blockData1, add=true, i, blockInfo0, blockInfo1)` to determine if a face should be added or removed.
    *   If `addFace` is true:
        *   If adding block: Call `addCubeFace(x, y, z, i, texture)` on current chunk.
        *   If removing block: Load neighbor chunk mesh if needed, then call `addCubeFace` on neighbor chunk with direction $i + (i \% 2 === 0 ? 1 : -1)$.
    *   If `removeFace` is true:
        *   If adding block: Call `removeFace` on neighbor chunk.
        *   If removing block: Call `removeFace` on current chunk.
5.  **Face Interaction Logic:**
    *   Let $B_0$ be current block, $B_1$ be neighbor. Let $add$ be boolean indicating if block is being added.
    *   If $add$:
        *   If $B_0$ is invisible: No face change.
        *   If ($B_1$ is invisible/translucent/transparent) AND ($B_0$.type $\neq$ $B_1$.type): Add face to $B_0$.
        *   If ($B_1$ is NOT invisible) AND ($B_0$ is NOT translucent/transparent) OR ($B_0$.type $==$ $B_1$.type): Remove face from $B_0$.
    *   If NOT $add$ (removing):
        *   If ($B_1$ is invisible/translucent/transparent) AND ($B_0$.type $\neq$ $B_1$.type): Remove face from $B_0$.
        *   If ($B_1$ is NOT invisible) AND ($B_0$ is NOT translucent/transparent) OR ($B_0$.type $==$ $B_1$.type): Add face to $B_0$ (exposing neighbor).
6.  **Add Cube Face:**
    *   Calculate 4 vertex positions using `cubeFacePosition(x, y, z, dir)`.
    *   Calculate 4 texture coordinates using `cubeFaceTexcoord(orientation)`.
    *   Get lighting value `dLight` from direction array.
    *   Pop `vertexIndex` from `vertexSlots` if available; otherwise, use next available index.
    *   Call `addVertex` for 4 vertices (a, b, c, d) with `vertexIndex`, `vertexIndex+1`, etc.
    *   Push indices `a, b, c, c, d, a` to `indices`.
    *   Set `update = true`.
7.  **Add Vertex:**
    *   If `vertexIndex` >= 0: Update `vertex0[vertexIndex]` and `vertex1[vertexIndex]`.
    *   Else: Push new packed values to `vertex0` and `vertex1`.
    *   Packed `vertex0`: `x | (y << 10) | (z << 20)`.
    *   Packed `vertex1`: `textureLayer | (texcoordX << 10) | (texcoordY << 15) | (dLight << 20)`.
8.  **Remove Face:**
    *   Call `findFace(facePosition)` to get starting vertex index.
    *   If found:
        *   Set `vertex0` and `vertex1` at indices $i, i+1, i+2, i+3$ to `NaN`.
        *   Remove 6 corresponding indices from `indices` array using `splice`.
        *   Push starting vertex index to `vertexSlots`.
        *   Set `update = true`.
9.  **Find Face:**
    *   Iterate `vertex0` array in steps of 4.
    *   Skip if `vertex0[i]` is `NaN`.
    *   Reconstruct packed position for each of 4 vertices of target face and compare against stored `vertex0` values.
    *   Return starting index or -1.
10. **Update Buffers:**
    *   If `update` is true:
        *   Bind VAO.
        *   Upload `vertex0` to `vertex0Buffer` with `gl.DYNAMIC_DRAW`.
        *   Upload `vertex1` to `vertex1Buffer` with `gl.DYNAMIC_DRAW`.
        *   Upload `indices` to `indexBuffer` with `gl.DYNAMIC_DRAW`.
        *   Set `update = false`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack_vertex0(x, y, z):
    return x | (y << 10) | (z << 20)

def pack_vertex1(texture_layer, texcoord_x, texcoord_y, d_light):
    return texture_layer | (texcoord_x << 10) | (texcoord_y << 15) | (d_light << 20)

def unpack_vertex0(v0):
    x = v0 & 1023
    y = (v0 >> 10) & 1023
    z = v0 >> 20
    return x, y, z

def unpack_vertex1(v1):
    texture_layer = v1 & 1023
    texcoord_x = ((v1 >> 10) & 31) * 0.0625
    texcoord_y = ((v1 >> 15) & 31) * 0.0625
    d_light = (v1 >> 20) * 0.2 + 0.4
    return texture_layer, texcoord_x, texcoord_y, d_light

def cube_face_position(x, y, z, dir, n=16):
    x *= n
    y *= n
    z *= n
    if dir == 0: # +X
        return [x+n, y, z, x+n, y+n, z, x+n, y+n, z+n, x+n, y, z+n]
    elif dir == 1: # -X
        return [x, y, z, x, y, z+n, x, y+n, z+n, x, y+n, z]
    elif dir == 2: # +Y
        return [x, y+n, z, x, y+n, z+n, x+n, y+n, z+n, x+n, y+n, z]
    elif dir == 3: # -Y
        return [x+n, y, z+n, x, y, z+n, x, y, z, x+n, y, z]
    elif dir == 4: # +Z
        return [x, y, z+n, x+n, y, z+n, x+n, y+n, z+n, x, y+n, z+n]
    elif dir == 5: # -Z
        return [x, y, z, x, y+n, z, x+n, y+n, z, x+n, y, z]

def cube_face_texcoord(orientation):
    # Returns 8 floats: u0, v0, u1, v1, u2, v2, u3, v3
    # Mapping from dir: [2, 3, 0, 0, 3, 2]
    # Simplified for dir 0 (positive x) as example: [0,0, 0,16, 16,16, 16,0]
    # Actual implementation uses switch/case for 6 orientations
    pass

def add_vertex(vertex0_arr, vertex1_arr, vertex_slots, x, y, z, texture_layer, texcoord_x, texcoord_y, d_light, vertex_index=-1):
    if vertex_index >= 0:
        vertex0_arr[vertex_index] = pack_vertex0(x, y, z)
        vertex1_arr[vertex_index] = pack_vertex1(texture_layer, texcoord_x, texcoord_y, d_light)
    else:
        vertex_index = len(vertex0_arr)
        vertex0_arr.append(pack_vertex0(x, y, z))
        vertex1_arr.append(pack_vertex1(texture_layer, texcoord_x, texcoord_y, d_light))
    return vertex_index

def add_cube_face(x, y, z, dir, texture_layer, vertex0_arr, vertex1_arr, indices_arr, vertex_slots):
    positions = cube_face_position(x, y, z, dir)
    texcoords = cube_face_texcoord(dir) # Returns 8 floats
    d_light = [1, 1, 3, 0, 2, 2][dir]
    
    # Get reusable index
    if vertex_slots:
        vertex_index = vertex_slots.pop()
    else:
        vertex_index = -1
        
    a = add_vertex(vertex0_arr, vertex1_arr, vertex_slots, positions[0], positions[1], positions[2], texture_layer, texcoords[0], texcoords[1], d_light, vertex_index)
    b = add_vertex(vertex0_arr, vertex1_arr, vertex_slots, positions[3], positions[4], positions[5], texture_layer, texcoords[2], texcoords[3], d_light, vertex_index + 1)
    c = add_vertex(vertex0_arr, vertex1_arr, vertex_slots, positions[6], positions[7], positions[8], texture_layer, texcoords[4], texcoords[5], d_light, vertex_index + 2)
    d = add_vertex(vertex0_arr, vertex1_arr, vertex_slots, positions[9], positions[10], positions[11], texture_layer, texcoords[6], texcoords[7], d_light, vertex_index + 3)
    
    indices_arr.extend([a, b, c, c, d, a])
    return True

def find_face(vertex0_arr, face_positions):
    # face_positions is list of 12 floats (4 vertices * 3 coords)
    for i in range(0, len(vertex0_arr), 4):
        if vertex0_arr[i] != vertex0_arr[i]: # Check for NaN
            continue
        # Reconstruct packed positions for comparison
        v0_packed = pack_vertex0(face_positions[0], face_positions[1], face_positions[2])
        v1_packed = pack_vertex0(face_positions[3], face_positions[4], face_positions[5])
        v2_packed = pack_vertex0(face_positions[6], face_positions[7], face_positions[8])
        v3_packed = pack_vertex0(face_positions[9], face_positions[10], face_positions[11])
        
        if (vertex0_arr[i] == v0_packed and 
            vertex0_arr[i+1] == v1_packed and 
            vertex0_arr[i+2] == v2_packed and 
            vertex0_arr[i+3] == v3_packed):
            return i
    return -1

def remove_face(vertex0_arr, vertex1_arr, indices_arr, vertex_slots, face_positions):
    vertex_index = find_face(vertex0_arr, face_positions)
    if vertex_index == -1:
        return False
    
    # Mark vertices as NaN
    vertex0_arr[vertex_index] = float('nan')
    vertex0_arr[vertex_index+1] = float('nan')
    vertex0_arr[vertex_index+2] = float('nan')
    vertex0_arr[vertex_index+3] = float('nan')
    
    vertex1_arr[vertex_index] = float('nan')
    vertex1_arr[vertex_index+1] = float('nan')
    vertex1_arr[vertex_index+2] = float('nan')
    vertex1_arr[vertex_index+3] = float('nan')
    
    # Remove indices
    # Find start of indices for this face
    # Note: indices are not sorted, so linear search is needed
    for j in range(0, len(indices_arr), 6):
        if indices_arr[j] == vertex_index:
            indices_arr[j:j+6] = []
            break
            
    vertex_slots.append(vertex_index)
    return True
```

**Edge Cases and Pitfalls:**
*   **Neighbor Chunk Loading:** When removing a block, the neighbor chunk's mesh must be loaded if it doesn't exist before adding the exposed face to it [S1, unofficial].
*   **NaN Handling:** `find_face` must skip `NaN` entries in `vertex0` to avoid false matches or crashes [S1, unofficial].
*   **Index Search:** Since indices are not sorted after removals, `find_index` for indices array requires linear search, not binary search [S1, unofficial].
*   **Direction Mapping:** When updating neighbor chunks, the direction index for the neighbor's face is calculated as $i + (i \% 2 === 0 ? 1 : -1)$ to map outward normal to inward normal [S1, unofficial].

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
*   **Time Complexity:** Adding/removing a block involves checking 6 neighbors, each requiring constant-time face interaction logic. Buffer upload is $O(V + I)$ where $V$ is number of vertices and $I$ is number of indices. Finding a face for removal is $O(V)$ due to linear search.
*   **Memory Complexity:** $O(V)$ for vertex data, $O(I)$ for indices. Packing reduces vertex data size by 50% compared to storing 3 floats per vertex.
*   **Performance:** Efficient for dynamic worlds with frequent block changes. The free-list allocator minimizes memory fragmentation. GPU buffer updates are minimized by only uploading when `update` flag is set.
*   **Strengths:** Low memory footprint, fast incremental updates, simple implementation.
*   **Limitations:** Linear search for face removal can be slow for large meshes. No advanced culling beyond face-level. Alpha sorting requires additional logic.

## Background
Originating from the Exolands project, this method was developed to provide efficient dynamic voxel terrain rendering in web-based environments [S1, unofficial]. It builds on the principles of face culling and visibility optimization, focusing on the practical aspects of mesh data management and GPU buffer interaction.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
