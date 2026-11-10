# ChunkMesh Vertex Packing and Management

Also called: Block Face Culling, Cube Face Mesh Generation

## Overview
ChunkMesh Vertex Packing and Management is a dynamic mesh generation and update strategy for voxel terrain that packs vertex attributes into compact 32-bit integers and manages buffer state via a free-list allocator. It solves the problem of high memory bandwidth and CPU overhead in incremental voxel rendering by minimizing vertex buffer size and enabling efficient face addition/removal without full mesh regeneration [S1, unofficial]. It sets itself apart from static meshing methods by supporting real-time block insertion and removal with neighbor-aware face culling, and from pure GPU-side approaches by performing the logical remapping and buffer updates on the CPU [S1, unofficial].

## How it works
**Inputs:**
*   Voxel world state (block types, chunk positions).
*   Camera position and orientation (for alpha sorting).
*   Block size $n = 16$ [S1, unofficial].

**Outputs:**
*   GPU vertex buffers (`vertex0Buffer`, `vertex1Buffer`) and index buffer (`indexBuffer`) containing the mesh geometry for a chunk.

**Data Structures:**
*   `vertex0`: `Uint32Array` storing packed position data.
*   `vertex1`: `Uint32Array` storing packed texture, texcoord, and lighting data.
*   `indices`: `Uint32Array` storing triangle indices (6 indices per face).
*   `vertexSlots`: Stack of available vertex indices for reuse (free-list) [S1, unofficial].
*   `update`: Boolean flag triggering buffer upload [S1, unofficial].

**Packed Layout:**
Each vertex is represented by two 32-bit unsigned integers.

*   `vertex0` (Position):
    *   Bits 0-9: X coordinate (10 bits)
    *   Bits 10-19: Y coordinate (10 bits)
    *   Bits 20-29: Z coordinate (10 bits)
    *   Bits 30-31: Unused [S1, unofficial]

*   `vertex1` (Attributes):
    *   Bits 0-9: Texture layer (10 bits)
    *   Bits 10-14: Texcoord X (5 bits)
    *   Bits 15-19: Texcoord Y (5 bits)
    *   Bits 20-21: Shading/Lighting value (2 bits)
    *   Bits 22-31: Unused [S1, unofficial]

**Vertex Generation:**
Positions are calculated based on the voxel coordinates $(x, y, z)$ and face direction $dir \in \{0, 1, 2, 3, 4, 5\}$, where 0-5 correspond to positive X, negative X, positive Y, negative Y, positive Z, negative Z respectively [S1, unofficial].

$$
\begin{aligned}
x_{base} &= x \cdot n \\
y_{base} &= y \cdot n \\
z_{base} &= z \cdot n
\end{aligned}
$$

The 4 vertices of a quad face are generated as follows [S1, unofficial]:
*   Dir 0 (+X): $(x_{base}+n, y_{base}, z_{base}), (x_{base}+n, y_{base}+n, z_{base}), (x_{base}+n, y_{base}+n, z_{base}+n), (x_{base}+n, y_{base}, z_{base}+n)$
*   Dir 1 (-X): $(x_{base}, y_{base}, z_{base}), (x_{base}, y_{base}, z_{base}+n), (x_{base}, y_{base}+n, z_{base}+n), (x_{base}, y_{base}+n, z_{base})$
*   Dir 2 (+Y): $(x_{base}, y_{base}+n, z_{base}), (x_{base}, y_{base}+n, z_{base}+n), (x_{base}+n, y_{base}+n, z_{base}+n), (x_{base}+n, y_{base}+n, z_{base})$
*   Dir 3 (-Y): $(x_{base}+n, y_{base}, z_{base}+n), (x_{base}, y_{base}, z_{base}+n), (x_{base}, y_{base}, z_{base}), (x_{base}+n, y_{base}, z_{base})$
*   Dir 4 (+Z): $(x_{base}, y_{base}, z_{base}+n), (x_{base}+n, y_{base}, z_{base}+n), (x_{base}+n, y_{base}+n, z_{base}+n), (x_{base}, y_{base}+n, z_{base}+n)$
*   Dir 5 (-Z): $(x_{base}, y_{base}, z_{base}), (x_{base}, y_{base}+n, z_{base}), (x_{base}+n, y_{base}+n, z_{base}), (x_{base}+n, y_{base}, z_{base})$

Texture coordinates are determined by orientation mapping $[2, 3, 0, 0, 3, 2][dir]$ [S1, unofficial]. Lighting values are determined by $[1, 1, 3, 0, 2, 2][dir]$ [S1, unofficial].

**Packing Formulas:**
To pack vertex data [S1, unofficial]:
$$
\begin{aligned}
\text{vertex0} &= x \mid (y \ll 10) \mid (z \ll 20) \\
\text{vertex1} &= \text{textureLayer} \mid (\text{texcoordX} \ll 10) \mid (\text{texcoordY} \ll 15) \mid (\text{dLight} \ll 20)
\end{aligned}
$$
Where `texcoordX` and `texcoordY` are integer values in range [0, 31] (derived from 0.0-1.0 range scaled by 32, or 0-15 scaled by 2 depending on specific implementation details, but the source uses 5 bits so range 0-31). The source code shows `texcoordX` and `texcoordY` being packed into 5 bits. The unpacking in the shader uses `* 0.0625f` which is $1/16$, implying the 5-bit value is divided by 32? No, $1/16 = 0.0625$. If the max value is 31, $31/32 \approx 0.96$. If the max value is 15, $15/16 = 0.9375$. The source code for `cubeFaceTexcoord` returns values like 0, 16. This suggests the texture coordinates are in block units (0-16). The 5-bit packing implies a range of 0-31. The shader unpacking: `float(vertex1 & 31744u) >> 10u) * 0.0625f`. $31744 = 0b1111100000000000$. This extracts bits 10-14. The multiplier $0.0625$ is $1/16$. If the packed value is $v$, the UV is $v/16$. If $v$ is 16, UV is 1.0. This fits the 0-16 range of the face size.

**Algorithm Steps:**

1.  **Face Interaction Check:** For a block at $(x,y,z)$ and neighbor in direction $dir$, determine if a face should be added or removed based on block types and properties (invisible, translucent, transparent) [S1, unofficial].
    *   Add Face if: Neighbor is invisible/translucent/transparent AND block types differ.
    *   Remove Face if: Neighbor is NOT invisible AND (Current block is NOT translucent/transparent OR block types are identical).
    *   Reverse logic for block removal [S1, unofficial].

2.  **Add Face:**
    *   Calculate vertex positions and attributes.
    *   Pop a reusable index from `vertexSlots`. If empty, allocate new indices.
    *   Call `addVertex` for 4 vertices, packing data into `vertex0` and `vertex1`.
    *   Push indices $(a, b, c, c, d, a)$ to `indices` array.
    *   Set `update = true` [S1, unofficial].

3.  **Remove Face:**
    *   Search `vertex0` array in steps of 4 to find the starting index of the face matching `facePosition`. Skip NaN entries.
    *   If found, set `vertex0` and `vertex1` at indices $i, i+1, i+2, i+3$ to `NaN`.
    *   Remove the 6 corresponding indices from `indices` using `splice`.
    *   Push starting vertex index $i$ to `vertexSlots` for reuse.
    *   Set `update = true` [S1, unofficial].

4.  **Buffer Update:**
    *   If `update` is true, bind VAO and upload `vertex0`, `vertex1`, and `indices` to GPU buffers using `gl.DYNAMIC_DRAW` hint.
    *   Reset `update` to false [S1, unofficial].

5.  **Neighbor Update:**
    *   When adding/removing a block, iterate over 6 directions.
    *   If a face needs to be added/removed on the neighbor chunk, load the neighbor's mesh if it doesn't exist.
    *   Calculate neighbor face direction as $dir + (dir \% 2 === 0 ? 1 : -1)$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack_vertex0(x, y, z):
    return x | (y << 10) | (z << 20)

def pack_vertex1(texture_layer, texcoord_x, texcoord_y, d_light):
    return texture_layer | (texcoord_x << 10) | (texcoord_y << 15) | (d_light << 20)

def unpack_vertex0(vertex0_val):
    x = vertex0_val & 1023
    y = (vertex0_val & 1047552) >> 10
    z = vertex0_val >> 20
    return x, y, z

def unpack_vertex1(vertex1_val):
    texture_layer = vertex1_val & 1023
    texcoord_x = ((vertex1_val & 31744) >> 10) * 0.0625
    texcoord_y = ((vertex1_val & 1015808) >> 15) * 0.0625
    shade_data = ((vertex1_val >> 20) * 0.2) + 0.4
    return texture_layer, texcoord_x, texcoord_y, shade_data

def get_face_positions(x, y, z, dir, n=16):
    x_base = x * n
    y_base = y * n
    z_base = z * n
    if dir == 0: return [(x_base+n, y_base, z_base), (x_base+n, y_base+n, z_base), (x_base+n, y_base+n, z_base+n), (x_base+n, y_base, z_base+n)]
    if dir == 1: return [(x_base, y_base, z_base), (x_base, y_base, z_base+n), (x_base, y_base+n, z_base+n), (x_base, y_base+n, z_base)]
    if dir == 2: return [(x_base, y_base+n, z_base), (x_base, y_base+n, z_base+n), (x_base+n, y_base+n, z_base+n), (x_base+n, y_base+n, z_base)]
    if dir == 3: return [(x_base+n, y_base, z_base+n), (x_base, y_base, z_base+n), (x_base, y_base, z_base), (x_base+n, y_base, z_base)]
    if dir == 4: return [(x_base, y_base, z_base+n), (x_base+n, y_base, z_base+n), (x_base+n, y_base+n, z_base+n), (x_base, y_base+n, z_base+n)]
    if dir == 5: return [(x_base, y_base, z_base), (x_base, y_base+n, z_base), (x_base+n, y_base+n, z_base), (x_base+n, y_base, z_base)]
    return []

def add_face(x, y, z, dir, texture_layer):
    positions = get_face_positions(x, y, z, dir)
    texcoords = get_face_texcoords(dir) # Returns 4 pairs of u,v
    d_light = get_lighting(dir)
    
    vertex_indices = []
    for i in range(4):
        pos = positions[i]
        tex = texcoords[i]
        # Get reusable index or allocate new
        if vertex_slots:
            idx = vertex_slots.pop()
        else:
            idx = len(vertex0)
        
        v0 = pack_vertex0(pos[0], pos[1], pos[2])
        v1 = pack_vertex1(texture_layer, tex[0], tex[1], d_light)
        
        vertex0.append(v0)
        vertex1.append(v1)
        vertex_indices.append(idx)
        
    indices.extend([vertex_indices[0], vertex_indices[1], vertex_indices[2],
                    vertex_indices[2], vertex_indices[3], vertex_indices[0]])
    update = True

def remove_face(face_position):
    # face_position is a list of 4 packed vertex0 values
    start_idx = find_face(face_position)
    if start_idx == -1: return
    
    # Mark as invalid
    for i in range(4):
        vertex0[start_idx + i] = float('nan')
        vertex1[start_idx + i] = float('nan')
        
    # Remove indices
    idx_start = find_index_in_indices(start_idx)
    if idx_start != -1:
        indices[idx_start:idx_start+6] = []
        
    # Return to free list
    vertex_slots.append(start_idx)
    update = True

def find_face(face_position):
    for i in range(0, len(vertex0), 4):
        if isnan(vertex0[i]): continue
        if vertex0[i] == face_position[0] and \
           vertex0[i+1] == face_position[1] and \
           vertex0[i+2] == face_position[2] and \
           vertex0[i+3] == face_position[3]:
            return i
    return -1

def find_index_in_indices(start_vertex_idx):
    for i in range(0, len(indices), 6):
        if indices[i] == start_vertex_idx:
            return i
    return -1
```

**Edge Cases and Pitfalls:**
*   **Neighbor Chunk Loading:** When removing a block, the neighbor chunk's mesh must be loaded if it doesn't exist before adding the exposed face to it [S1, unofficial].
*   **Precision:** The 10-bit coordinate packing limits the chunk size to $2^{10} = 1024$ units. With block size 16, this supports chunks of $64 \times 64 \times 64$ blocks.
*   **NaN Handling:** Removing faces sets vertex data to `NaN`. The `find_face` function must skip these entries.

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
*   **Time Complexity:** Face addition/removal is $O(1)$ for allocation (amortized) and $O(N)$ for face search, where $N$ is the number of faces in the chunk. Buffer upload is $O(F)$ where $F$ is the number of faces.
*   **Memory Complexity:** Vertex data is packed into 8 bytes per vertex (2x 32-bit ints). Index data is 4 bytes per index. This is significantly more compact than standard float-based vertex buffers.
*   **Performance:** Efficient for dynamic worlds with frequent block changes. The free-list allocator reduces memory fragmentation.
*   **Strengths:** Low memory footprint, fast incremental updates, simple implementation.
*   **Limitations:** Limited coordinate range due to 10-bit packing. Search for face removal is linear in the worst case. Not suitable for very large chunks without subdivision.

## Background
The method originates from the Exolands voxel engine, an open-source project by kvxy [S1, unofficial]. It is designed for real-time voxel terrain rendering in web browsers, leveraging JavaScript and WebGL. The approach emphasizes simplicity and efficiency for dynamic voxel worlds.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
