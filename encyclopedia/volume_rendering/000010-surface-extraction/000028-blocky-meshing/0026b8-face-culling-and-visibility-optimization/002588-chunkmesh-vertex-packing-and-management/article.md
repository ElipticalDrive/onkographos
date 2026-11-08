# ChunkMesh Vertex Packing and Management

Also called: Block Face Culling, Cube Face Mesh Generation

## Overview
ChunkMesh Vertex Packing and Management is a method for generating and maintaining dynamic triangle meshes from voxel data by packing vertex attributes into compact integer formats and managing buffer updates via a free-list allocator. It solves the problem of high memory bandwidth and GPU buffer upload overhead in voxel terrain rendering by minimizing vertex data size and enabling efficient incremental updates. It sets itself apart from sibling methods by focusing on the low-level data layout (bit-packing) and the specific logic for adding/removing faces between adjacent chunks, rather than just culling visibility or reindexing vertices.

## How it works
**Inputs and Outputs:**
*   **Input:** Voxel world state (block types, positions), camera position (for sorting in alpha variants), and chunk boundaries.
*   **Output:** GPU vertex buffers (`vertex0`, `vertex1`) and index buffer (`indices`) representing the visible surface of a chunk.

**Data Structures:**
The core data structure consists of three parallel arrays, converted to `Uint32Array` for GPU upload:
1.  `vertex0`: Array of 32-bit unsigned integers. Each vertex uses one `uint`.
    *   **Layout:**
        *   Bits 0–9: X coordinate (10 bits, range 0–1023).
        *   Bits 10–19: Y coordinate (10 bits, range 0–1023).
        *   Bits 20–29: Z coordinate (10 bits, range 0–1023).
        *   Bits 30–31: Unused.
    *   **Encoding:** `vertex0 = x | (y << 10) | (z << 20)` [S1, unofficial].
2.  `vertex1`: Array of 32-bit unsigned integers. Each vertex uses one `uint`.
    *   **Layout:**
        *   Bits 0–9: Texture layer index (10 bits).
        *   Bits 10–14: Texture coordinate U (5 bits, range 0–31).
        *   Bits 15–19: Texture coordinate V (5 bits, range 0–31).
        *   Bits 20–21: Lighting/Shading value (2 bits, range 0–3).
        *   Bits 22–31: Unused.
    *   **Encoding:** `vertex1 = textureLayer | (texcoordX << 10) | (texcoordY << 15) | (dLight << 20)` [S1, unofficial].
3.  `indices`: Array of 32-bit unsigned integers. Stores triangle indices.
4.  `vertexSlots`: A stack (array) storing available vertex indices for reuse, implementing a free-list allocator [S1, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Allocate `vertex0`, `vertex1`, and `indices` arrays.
    *   Initialize `vertexSlots` as empty.
    *   Set `this.update = false`.

2.  **Face Visibility Determination (Face Interaction):**
    *   For each block at $P_{current}$ and each of its 6 neighbors at $P_{neighbor}$ (direction $dir \in [0, 5]$):
        *   Determine if a face should be added or removed based on block types and properties (invisible, translucent, transparent) [S1, unofficial].
        *   Logic:
            *   If adding a face: Add if neighbor is invisible/translucent/transparent AND block types differ.
            *   If adding a face: Remove if neighbor is NOT invisible AND (current block is NOT translucent/transparent OR block types are identical).
            *   If removing a face: Reverse the above logic [S1, unofficial].

3.  **Adding a Face:**
    *   Calculate vertex positions using `cubeFacePosition(x, y, z, dir)`. The block size $n$ is 16. Coordinates are scaled by $n$.
        *   Directions: 0=+X, 1=-X, 2=+Y, 3=-Y, 4=+Z, 5=-Z [S1, unofficial].
    *   Calculate texture coordinates using `cubeFaceTexcoord(orientation)`. Orientation mapping from direction $dir$ is `[2, 3, 0, 0, 3, 2][dir]`. Returns 4 floats (2 pairs of u,v) [S1, unofficial].
    *   Determine lighting value `dLight` using `[1, 1, 3, 0, 2, 2][dir]` [S1, unofficial].
    *   Get a reusable vertex index from `vertexSlots`. If empty, use current length of `vertex0`.
    *   Call `addVertex` for 4 vertices (a, b, c, d) with the calculated positions, texture, texcoords, and lighting.
        *   `addVertex` packs data into `vertex0` and `vertex1` using the bit layouts above.
        *   If an index was provided, update existing arrays at that index; otherwise, push new values.
        *   Set `this.update = true` [S1, unofficial].
    *   Push indices `(a, b, c, c, d, a)` to the `indices` array [S1, unofficial].

4.  **Removing a Face:**
    *   Call `findFace(facePosition)` to locate the starting index of the face in `vertex0`.
        *   `findFace` iterates through `vertex0` in steps of 4.
        *   It skips entries where `vertex0[i]` is NaN.
        *   It reconstructs the packed `vertex0` uint for each of the 4 vertices of the target face and compares them against stored values.
        *   Returns the starting index or -1 [S1, unofficial].
    *   If found:
        *   Set `vertex0` and `vertex1` values at indices `i, i+1, i+2, i+3` to `NaN`.
        *   Remove the 6 corresponding indices from the `indices` array using `splice`.
        *   Push the starting vertex index `i` to `vertexSlots` for reuse [S1, unofficial].
        *   Set `this.update = true` [S1, unofficial].

5.  **Buffer Update:**
    *   If `this.update` is true:
        *   Bind VAO.
        *   Upload `vertex0` to `vertex0Buffer` using `gl.DYNAMIC_DRAW`.
        *   Upload `vertex1` to `vertex1Buffer` using `gl.DYNAMIC_DRAW`.
        *   Upload `indices` to `indexBuffer` using `gl.DYNAMIC_DRAW`.
        *   Set `this.update = false` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack_vertex0(x, y, z):
    return x | (y << 10) | (z << 20)

def pack_vertex1(texture_layer, texcoord_u, texcoord_v, d_light):
    return texture_layer | (texcoord_u << 10) | (texcoord_v << 15) | (d_light << 20)

def unpack_vertex0(v0):
    x = v0 & 1023
    y = (v0 & 1047552) >> 10
    z = v0 >> 20
    return x, y, z

def unpack_vertex1(v1):
    texture_layer = v1 & 1023
    texcoord_u = ((v1 & 31744) >> 10) * 0.0625
    texcoord_v = ((v1 & 1015808) >> 15) * 0.0625
    d_light = (v1 >> 20) * 0.2 + 0.4
    return texture_layer, texcoord_u, texcoord_v, d_light

def get_direction_coords(dir):
    # Returns lighting value and orientation mapping
    lights = [1, 1, 3, 0, 2, 2]
    orientations = [2, 3, 0, 0, 3, 2]
    return lights[dir], orientations[dir]

def add_cube_face(x, y, z, dir, texture_layer):
    n = 16
    x_scaled = x * n
    y_scaled = y * n
    z_scaled = z * n
    
    # Get positions for 4 vertices (2 triangles)
    positions = get_cube_face_positions(x_scaled, y_scaled, z_scaled, dir)
    
    # Get texcoords
    orientation = get_direction_coords(dir)[1]
    texcoords = get_cube_face_texcoords(orientation)
    
    # Get lighting
    d_light = get_direction_coords(dir)[0]
    
    # Get reusable index
    if vertex_slots:
        idx = vertex_slots.pop()
    else:
        idx = len(vertex0)
        
    # Add 4 vertices
    for i in range(4):
        pos = positions[i*3 : (i+1)*3]
        tc = texcoords[i*2 : (i+1)*2]
        
        v0 = pack_vertex0(pos[0] // n, pos[1] // n, pos[2] // n)
        v1 = pack_vertex1(texture_layer, tc[0] // 16, tc[1] // 16, d_light)
        
        if idx + i < len(vertex0):
            vertex0[idx + i] = v0
            vertex1[idx + i] = v1
        else:
            vertex0.append(v0)
            vertex1.append(v1)
            
    # Add indices
    indices.extend([idx, idx+1, idx+2, idx+2, idx+3, idx])
    update = True

def remove_face(face_positions):
    # face_positions is a list of 4 (x,y,z) tuples
    target_v0s = [pack_vertex0(p[0]//16, p[1]//16, p[2]//16) for p in face_positions]
    
    start_idx = -1
    for i in range(0, len(vertex0), 4):
        if isnan(vertex0[i]): continue
        if vertex0[i:i+4] == target_v0s:
            start_idx = i
            break
            
    if start_idx == -1: return
    
    # Mark as NaN
    for i in range(4):
        vertex0[start_idx + i] = NaN
        vertex1[start_idx + i] = NaN
        
    # Remove indices
    # Find index in indices array corresponding to start_idx
    idx_in_indices = find_index_in_indices(start_idx)
    if idx_in_indices != -1:
        indices.splice(idx_in_indices, 6)
        
    vertex_slots.append(start_idx)
    update = True

def find_index_in_indices(start_vertex_idx):
    for i in range(0, len(indices), 6):
        if indices[i] == start_vertex_idx:
            return i
    return -1
```

**Edge Cases and Pitfalls:**
*   **Neighbor Chunk Loading:** When updating neighbor chunks (e.g., adding a face to a neighbor when removing a block), the neighbor chunk's mesh must be loaded if it doesn't exist before adding the exposed face to it [S1, unofficial].
*   **Direction Mapping:** When adding/removing faces on neighbor chunks, the direction index must be inverted. For direction $i$, the neighbor's direction is $i + (1 \text{ if } i \text{ is even else } -1)$ [S1, unofficial].
*   **Precision:** Coordinates are limited to 10 bits (0–1023). This implies a maximum chunk or block coordinate range. If blocks exceed this, packing will overflow [S1, unofficial].
*   **NaN Handling:** Removed vertices are marked with `NaN` in `vertex0`. The `findFace` function must skip these entries to avoid false matches [S1, unofficial].

## Subtopics

### [Voxel Face Meshing with Packed Attributes](4153-voxel-face-meshing-with-packed-attributes.md)
Defines the specific bit-layout and shader logic required to reconstruct geometry from the packed data.

### [Mesh Reindexing and Vertex Compaction](4160-mesh-reindexing-and-vertex-compaction.md)
Actively compacts the data by removing unreferenced vertices and renumbering the remaining ones.

### [Chunk Mesh Data Structure & Buffer Update](4182-chunk-mesh-data-structure-buffer-update.md)
Manages the creation, population, and recycling of GPU vertex buffers using a "MegaSSBO" and free-list allocator.

### [Alpha/Translucent Block Rendering](4860-alpha-translucent-block-rendering.md)
Applies per-fragment shading and fog based on world-space depth for translucent geometry.

## Operating characteristics
*   **Time Complexity:** Adding/removing a face is $O(1)$ for buffer updates, but `findFace` is $O(V)$ where $V$ is the number of vertices, as it scans the `vertex0` array. This can be a bottleneck for large chunks with frequent updates.
*   **Memory Complexity:** Each vertex uses 8 bytes (2 x 32-bit ints). Indices use 4 bytes per triangle (3 indices). The free-list allocator helps reuse memory, reducing fragmentation.
*   **Performance:** The use of `gl.DYNAMIC_DRAW` and integer attributes reduces bandwidth compared to floating-point attributes. The bit-packing allows storing more data per vertex, reducing the total number of vertices needed for complex textures or lighting.
*   **Strengths:** Efficient memory usage, fast incremental updates for small changes, simple data layout.
*   **Limitations:** $O(V)$ search for face removal, limited coordinate range (10 bits), requires careful management of vertex slots and indices.

## Background
The method originates from the Exolands voxel engine, an unofficial open-source project. It is designed for real-time voxel terrain rendering in web-based environments, leveraging WebGL capabilities. The approach of packing vertex data into integers and using a free-list allocator is a common optimization in voxel engines to handle dynamic world changes efficiently.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
