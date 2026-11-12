# ChunkMesh Vertex Packing and Management

Also called: Block Face Culling, Cube Face Mesh Generation

## Overview
ChunkMesh Vertex Packing and Management is a dynamic mesh generation and buffer management technique for voxel terrain that packs vertex attributes into compact 32-bit integers and manages incremental updates via a free-list of reusable vertex indices [S1, unofficial]. It solves the problem of high memory overhead and frequent GPU buffer uploads in blocky meshing by reducing vertex data size and avoiding reallocation [S1, unofficial]. It differs from [Voxel Face Meshing with Packed Attributes](4153-voxel-face-meshing-with-packed-attributes.md) by managing the host-side buffer state and reuse logic, whereas the sibling focuses on the shader-side unpacking layout; it differs from [Mesh Reindexing and Vertex Compaction](4160-mesh-reindexing-and-vertex-compaction.md) by reusing indices in place rather than compacting the array [G].

## How it works
**Inputs and Outputs:**
- Input: Voxel world state (block types, chunk positions), camera position (for alpha sorting variants), block addition/removal events.
- Output: GPU vertex buffers (`vertex0`, `vertex1`) and index buffer (`indices`) containing only visible faces.

**Data Structures:**
The method maintains three JavaScript arrays (or equivalent host-side buffers) that are uploaded to the GPU as `Uint32Array`:
1.  `vertex0`: Array of 32-bit unsigned integers. Each vertex uses one entry.
    -   Bits 0–9: X coordinate (10 bits).
    -   Bits 10–19: Y coordinate (10 bits).
    -   Bits 20–29: Z coordinate (10 bits).
    -   Bits 30–31: Unused.
2.  `vertex1`: Array of 32-bit unsigned integers. Each vertex uses one entry.
    -   Bits 0–9: Texture layer index (10 bits).
    -   Bits 10–14: Texture coordinate X (5 bits).
    -   Bits 15–19: Texture coordinate Y (5 bits).
    -   Bits 20–21: Lighting/shading value (2 bits).
    -   Bits 22–31: Unused.
3.  `indices`: Array of 32-bit unsigned integers. Stores triangle indices. Each face adds 6 indices (2 triangles).
4.  `vertexSlots`: A stack (array) storing available vertex indices for reuse.

**Parameters:**
-   Block size ($n$): 16 units.
-   Direction mapping: 0=+X, 1=-X, 2=+Y, 3=-Y, 4=+Z, 5=-Z.
-   Lighting values ($dLight$): Array `[1, 1, 3, 0, 2, 2]` indexed by direction.
-   Texture coordinate mapping: Orientation index derived from direction `dir` as `[2, 3, 0, 0, 3, 2][dir]`.

**Algorithm Steps:**

1.  **Vertex Packing:**
    When adding a vertex at $(x, y, z)$ with texture layer $t$, texture coordinates $(u, v)$, and lighting $l$:
    $$ \text{vertex0} = x \mid (y \ll 10) \mid (z \ll 20) $$
    $$ \text{vertex1} = t \mid (u \ll 10) \mid (v \ll 15) \mid (l \ll 20) $$
    [S1, unofficial]

2.  **Face Generation (Add):**
    To add a face for a block at $(x, y, z)$ in direction $dir$ with texture $tex$:
    a.  Calculate 4 vertex positions using `cubeFacePosition(x, y, z, dir)`. The positions are scaled by $n=16$.
    b.  Calculate 4 texture coordinate pairs using `cubeFaceTexcoord(orientation)`.
    c.  Determine lighting $dLight$ from the direction array.
    d.  Reuse a vertex index from `vertexSlots` if available; otherwise, use the current length of `vertex0`.
    e.  Call `addVertex` for the 4 corners (a, b, c, d). This pushes packed `vertex0` and `vertex1` values to the arrays.
    f.  Push indices $(a, b, c, c, d, a)$ to the `indices` array.
    g.  Set `this.update = true` to trigger buffer refresh.
    [S1, unofficial]

3.  **Face Removal:**
    To remove a face at `facePosition` (array of 6 floats representing 2 triangles):
    a.  Call `findFace(facePosition)` to locate the starting index $i$ in `vertex0`.
    b.  If found ($i \neq -1$):
        i.  Set `vertex0[i...i+3]` and `vertex1[i...i+3]` to `NaN` (or a sentinel value indicating invalidity).
        ii. Find the index of the first index of this face in the `indices` array using `findIndex(i)`.
        iii. Remove the 6 corresponding indices from `indices` using `splice`.
        iv. Push $i$ to `vertexSlots` for reuse.
        v.  Set `this.update = true`.
    [S1, unofficial]

4.  **Face Search (findFace):**
    Iterate through `vertex0` in steps of 4. Skip entries where `vertex0[i]` is NaN. Reconstruct the packed integer for the 4 vertices of the target face and compare against stored values. Return the starting index or -1.
    [S1, unofficial]

5.  **Block Interaction Logic:**
    When a block is added or removed, iterate over its 6 neighbors. For each neighbor, determine if the shared face should be added or removed based on block types and properties (invisible, translucent, transparent).
    -   **Adding a block:**
        -   Add face if neighbor is invisible/translucent/transparent AND block types differ.
        -   Remove face from neighbor if neighbor is NOT invisible AND current block is NOT translucent/transparent OR block types are identical.
    -   **Removing a block:**
        -   Reverse the logic above.
    [S1, unofficial]

6.  **Neighbor Update:**
    When updating a neighbor chunk, the direction index for the neighbor's face is calculated as $dir + (dir \% 2 === 0 ? 1 : -1)$ to map the outward normal of the current block to the inward normal of the neighbor. If the neighbor chunk mesh does not exist, it must be loaded before adding the face.
    [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def add_vertex(x, y, z, texture_layer, texcoord_x, texcoord_y, d_light, vertex_index=-1):
    """
    Adds a vertex to the mesh. If vertex_index >= 0, updates existing arrays.
    Otherwise, pushes new values.
    """
    packed_v0 = x | (y << 10) | (z << 20)
    packed_v1 = texture_layer | (texcoord_x << 10) | (texcoord_y << 15) | (d_light << 20)
    
    if vertex_index >= 0:
        vertex0[vertex_index] = packed_v0
        vertex1[vertex_index] = packed_v1
    else:
        vertex0.append(packed_v0)
        vertex1.append(packed_v1)
        vertex_index = len(vertex0) - 1
        
    return vertex_index

def cube_face_position(x, y, z, dir):
    """
    Returns 6 floats (2 triangles) for a cube face at block coords (x,y,z) in direction dir.
    Block size n = 16.
    """
    n = 16
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
    """
    Returns 4 floats (2 pairs of u,v) for texture coordinates.
    """
    if orientation == 0:
        return [0, 0, 0, 16, 16, 16, 16, 0]
    elif orientation == 1:
        return [16, 0, 0, 0, 0, 16, 16, 16]
    # Other orientations handled similarly based on source [S1, unofficial]
    return [0, 0, 0, 16, 16, 16, 16, 0] # Placeholder for other cases

def add_cube_face(x, y, z, dir, texture):
    """
    Adds a single face to the mesh.
    """
    positions = cube_face_position(x, y, z, dir)
    orientation = [2, 3, 0, 0, 3, 2][dir]
    texcoords = cube_face_texcoord(orientation)
    d_light = [1, 1, 3, 0, 2, 2][dir]
    
    # Reuse index if available
    if vertex_slots:
        vertex_index = vertex_slots.pop()
    else:
        vertex_index = -1
        
    a = add_vertex(positions[0], positions[1], positions[2], texture, texcoords[0], texcoords[1], d_light, vertex_index)
    b = add_vertex(positions[3], positions[4], positions[5], texture, texcoords[2], texcoords[3], d_light, vertex_index + 1)
    c = add_vertex(positions[6], positions[7], positions[8], texture, texcoords[4], texcoords[5], d_light, vertex_index + 2)
    d = add_vertex(positions[9], positions[10], positions[11], texture, texcoords[6], texcoords[7], d_light, vertex_index + 3)
    
    indices.extend([a, b, c, c, d, a])
    update = True

def find_face(face_position):
    """
    Finds the starting index of a face in vertex0.
    face_position is an array of 6 floats.
    """
    for i in range(0, len(vertex0), 4):
        if isnan(vertex0[i]):
            continue
        # Reconstruct packed v0 for the 4 vertices of the target face
        v0_0 = face_position[0] | (face_position[1] << 10) | (face_position[2] << 20)
        v0_1 = face_position[3] | (face_position[4] << 10) | (face_position[5] << 20)
        v0_2 = face_position[6] | (face_position[7] << 10) | (face_position[8] << 20)
        v0_3 = face_position[9] | (face_position[10] << 10) | (face_position[11] << 20)
        
        if (vertex0[i] == v0_0 and 
            vertex0[i+1] == v0_1 and 
            vertex0[i+2] == v0_2 and 
            vertex0[i+3] == v0_3):
            return i
    return -1

def remove_face(face_position):
    """
    Removes a face from the mesh.
    """
    vertex_index = find_face(face_position)
    if vertex_index == -1:
        return
    
    # Mark vertices as invalid
    vertex0[vertex_index] = float('nan')
    vertex0[vertex_index+1] = float('nan')
    vertex0[vertex_index+2] = float('nan')
    vertex0[vertex_index+3] = float('nan')
    
    vertex1[vertex_index] = float('nan')
    vertex1[vertex_index+1] = float('nan')
    vertex1[vertex_index+2] = float('nan')
    vertex1[vertex_index+3] = float('nan')
    
    # Remove indices
    idx_start = find_index(vertex_index)
    if idx_start != -1:
        del indices[idx_start : idx_start+6]
        
    vertex_slots.append(vertex_index)
    update = True

def update_buffers():
    """
    Uploads data to GPU.
    """
    gl.bind_buffer(gl.ARRAY_BUFFER, vertex0_buffer)
    gl.buffer_data(gl.ARRAY_BUFFER, Uint32Array(vertex0), gl.DYNAMIC_DRAW)
    gl.bind_buffer(gl.ARRAY_BUFFER, vertex1_buffer)
    gl.buffer_data(gl.ARRAY_BUFFER, Uint32Array(vertex1), gl.DYNAMIC_DRAW)
    gl.bind_buffer(gl.ELEMENT_ARRAY_BUFFER, index_buffer)
    gl.buffer_data(gl.ELEMENT_ARRAY_BUFFER, Uint32Array(indices), gl.DYNAMIC_DRAW)
```

## Subtopics

### [Voxel Face Meshing with Packed Attributes](4153-voxel-face-meshing-with-packed-attributes.md)
Optimizes the GPU-side storage and shader unpacking of the packed vertex attributes defined here.

### [Mesh Reindexing and Vertex Compaction](4160-mesh-reindexing-and-vertex-compaction.md)
Compacts the vertex buffer by removing unused vertices, whereas this method reuses them via a free list.

### [Chunk Mesh Data Structure & Buffer Update](4182-chunk-mesh-data-structure-buffer-update.md)
Manages the GPU memory allocation (SSBO) for these buffers, handling the dynamic updates triggered by this method.

### [Alpha/Translucent Block Rendering](4860-alpha-translucent-block-rendering.md)
Uses the generated mesh but applies additional sorting and fragment shading for transparency.

## Operating characteristics
-   **Time Complexity:** Adding/removing a face is $O(1)$ for buffer updates (amortized) but $O(N)$ for `findFace` where $N$ is the number of vertices, as it scans the array. This makes frequent removals expensive without spatial indexing [S1, unofficial].
-   **Memory Complexity:** Vertex data is reduced to 8 bytes per vertex (2x uint32) plus 4 bytes per index. This is significantly smaller than standard float-based vertex buffers [S1, unofficial].
-   **Performance:** The use of `DYNAMIC_DRAW` and frequent `bufferData` calls can cause GPU driver overhead. The method relies on host-side CPU processing for mesh updates [S1, unofficial].
-   **Strengths:** Low memory footprint, simple implementation, effective for static or slowly changing terrain.
-   **Limitations:** `findFace` linear scan is a bottleneck for dynamic removals. Not suitable for highly dynamic scenes with frequent block changes without optimization (e.g., spatial hashing for face lookup) [G].

## Background
This method originates from the Exolands voxel engine implementation, specifically the `ChunkMesh` class [S1, unofficial]. It is designed for JavaScript-based web voxel engines where memory efficiency and simple buffer management are prioritized. The packing scheme allows for efficient transmission to WebGL shaders using integer attributes.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
