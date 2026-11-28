# ChunkMesh Vertex Packing and Management

Also called: Block Face Culling, Cube Face Mesh Generation

## Overview
ChunkMesh Vertex Packing and Management is a dynamic mesh generation and buffer management technique for blocky voxel terrain, optimizing memory usage and draw calls by packing vertex attributes into compact integer formats and managing face visibility via block-type comparison [S1, unofficial]. It differs from static meshing methods by supporting incremental updates (add/remove blocks) that trigger immediate neighbor face adjustments, and from pure culling methods by actively managing the vertex buffer lifecycle through index reuse and compaction [S1, unofficial].

## How it works
**Inputs:**
- Block world data (type, chunk position, visibility flags: `isInvisible`, `isTranslucent`, `isTransparent`) [S1, unofficial].
- Camera position and rotation (for alpha block sorting) [S1, unofficial].
- Chunk dimensions (block size $n=16$) [S1, unofficial].

**Outputs:**
- GPU vertex buffers (`vertex0`, `vertex1`) and index buffer (`indices`) [S1, unofficial].

**Data Structures:**
- `vertex0`: `Uint32Array` storing packed position [S1, unofficial].
- `vertex1`: `Uint32Array` storing packed texture, texcoord, and lighting [S1, unofficial].
- `indices`: `Uint32Array` storing triangle indices [S1, unofficial].
- `vertexSlots`: Stack of reusable vertex indices (for compaction) [S1, unofficial].
- `update`: Boolean flag triggering buffer upload [S1, unofficial].

**Packed Layout (32-bit unsigned integers):**
- `vertex0`:
  - Bits 0-9: X coordinate (10 bits).
  - Bits 10-19: Y coordinate (10 bits).
  - Bits 20-29: Z coordinate (10 bits).
  - Bits 30-31: Unused.
- `vertex1`:
  - Bits 0-9: Texture layer (10 bits).
  - Bits 10-14: Texcoord X (5 bits).
  - Bits 15-19: Texcoord Y (5 bits).
  - Bits 20-21: Shading/Lighting value (2 bits).
  - Bits 22-31: Unused.

**Numbered Steps:**
1. **Face Generation:** For a block at $(x, y, z)$ and direction $dir \in [0, 5]$ (positive x, negative x, positive y, negative y, positive z, negative z), calculate vertex positions using `cubeFacePosition(x, y, z, dir)` with block size $n=16$ [S1, unofficial]. Calculate texture coordinates using `cubeFaceTexcoord(orientation)`, where `orientation` maps from `dir` via `[2, 3, 0, 0, 3, 2][dir]` [S1, unofficial]. Determine lighting value `dLight` using `[1, 1, 3, 0, 2, 2][dir]` [S1, unofficial].
2. **Vertex Addition:** If a reusable index exists in `vertexSlots`, pop it; otherwise, use the current length of `vertex0` [S1, unofficial]. Pack position and attributes into `v0` and `v1` (see Formulas). Push `v0` and `v1` to `vertex0` and `vertex1` arrays [S1, unofficial]. Push triangle indices `(a, b, c, c, d, a)` to `indices` [S1, unofficial]. Set `update = true` [S1, unofficial].
3. **Face Removal:** Search for the face in `vertex0` using `findFace(facePosition)`, which reconstructs packed positions and compares them, skipping `NaN` entries [S1, unofficial]. If found at index `i`, set `vertex0[i...i+3]` and `vertex1[i...i+3]` to `NaN` [S1, unofficial]. Remove the corresponding 6 indices from `indices` using `splice` [S1, unofficial]. Push `i` to `vertexSlots` for reuse [S1, unofficial]. Set `update = true` [S1, unofficial].
4. **Visibility Logic (Face Interaction):** For adjacent blocks `blockData0` (current) and `blockData1` (neighbor) in direction `dir`: Adding Face: add if neighbor is invisible/translucent/transparent AND block types differ [S1, unofficial]. Removing Face: remove if neighbor is NOT invisible AND current block is NOT translucent/transparent OR block types are identical [S1, unofficial]. When updating neighbors, the neighbor's face direction is `dir + (dir % 2 === 0 ? 1 : -1)` [S1, unofficial].
5. **Buffer Update:** If `update` is true, upload `vertex0`, `vertex1`, and `indices` to GPU buffers using `gl.DYNAMIC_DRAW` [S1, unofficial].

**Formulas:**
- Packing position: $v_0 = x \mid (y \ll 10) \mid (z \ll 20)$ [S1, unofficial].
- Packing attributes: $v_1 = \text{textureLayer} \mid (\text{texcoordX} \ll 10) \mid (\text{texcoordY} \ll 15) \mid (\text{dLight} \ll 20)$ [S1, unofficial].
- Unpacking position: $x = v_0 \ \& \ 1023$, $y = (v_0 \ \& \ 1047552) \gg 10$, $z = v_0 \gg 20$ [S1, unofficial].
- Unpacking attributes: $\text{textureLayer} = v_1 \ \& \ 1023$, $\text{texcoordX} = ((v_1 \ \& \ 31744) \gg 10) \times 0.0625$, $\text{texcoordY} = ((v_1 \ \& \ 1015808) \gg 15) \times 0.0625$, $\text{shadeData} = (v_1 \gg 20) \times 0.2 + 0.4$ [S1, unofficial].

**Parameters:**
- Block size $n = 16$ [S1, unofficial].
- Direction mapping: $0 \to +x$, $1 \to -x$, $2 \to +y$, $3 \to -y$, $4 \to +z$, $5 \to -z$ [S1, unofficial].
- Texcoord mapping: $[2, 3, 0, 0, 3, 2][dir]$ [S1, unofficial].
- Lighting mapping: $[1, 1, 3, 0, 2, 2][dir]$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- When removing a block, the neighbor chunk's mesh must be loaded if it doesn't exist before adding the exposed face to it [S1, unofficial].
- Precision limits exist for coordinate range due to 10-bit storage per axis [S1, unofficial].
- Face search requires scanning the `vertex0` array in steps of 4, skipping `NaN` entries, which can be slow without spatial indexing [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def pack_vertex0(x, y, z):
    return x | (y << 10) | (z << 20)

def pack_vertex1(texture_layer, texcoord_x, texcoord_y, d_light):
    return texture_layer | (texcoord_x << 10) | (texcoord_y << 15) | (d_light << 20)

def unpack_vertex0(v0):
    x = v0 & 0x3FF
    y = (v0 >> 10) & 0x3FF
    z = v0 >> 20
    return x, y, z

def unpack_vertex1(v1):
    texture_layer = v1 & 0x3FF
    texcoord_x = ((v1 >> 10) & 0x1F) * 0.0625
    texcoord_y = ((v1 >> 15) & 0x1F) * 0.0625
    d_light = (v1 >> 20) * 0.2 + 0.4
    return texture_layer, texcoord_x, texcoord_y, d_light

def add_vertex(vertex0, vertex1, indices, vertex_slots, x, y, z, texture_layer, texcoord_x, texcoord_y, d_light):
    if vertex_slots:
        idx = vertex_slots.pop()
    else:
        idx = len(vertex0)
    v0 = pack_vertex0(x, y, z)
    v1 = pack_vertex1(texture_layer, texcoord_x, texcoord_y, d_light)
    vertex0.append(v0)
    vertex1.append(v1)
    return idx

def find_face(vertex0, face_positions):
    for i in range(0, len(vertex0), 4):
        if math.isnan(vertex0[i]):
            continue
        v0_0 = pack_vertex0(*face_positions[0])
        v0_1 = pack_vertex0(*face_positions[1])
        v0_2 = pack_vertex0(*face_positions[2])
        v0_3 = pack_vertex0(*face_positions[3])
        if vertex0[i] == v0_0 and vertex0[i+1] == v0_1 and vertex0[i+2] == v0_2 and vertex0[i+3] == v0_3:
            return i
    return -1

def find_index(indices, start_vertex_idx):
    for i in range(0, len(indices), 6):
        if indices[i] == start_vertex_idx:
            return i
    return -1

def remove_face(vertex0, vertex1, indices, vertex_slots, face_positions):
    found_idx = find_face(vertex0, face_positions)
    if found_idx == -1:
        return
    for j in range(4):
        vertex0[found_idx + j] = float('nan')
        vertex1[found_idx + j] = float('nan')
    face_idx_start = find_index(indices, found_idx)
    if face_idx_start != -1:
        del indices[face_idx_start:face_idx_start+6]
    vertex_slots.append(found_idx)

def face_interaction(block_info0, block_info1, block_data0, block_data1, add):
    add_face = False
    remove_face = False
    if add:
        if block_info0.isInvisible:
            return False, False
        if (block_info1.isInvisible or block_info1.isTranslucent or block_info1.isTransparent) and block_data0.type != block_data1.type:
            add_face = True
        if (not block_info1.isInvisible and not block_info0.isTranslucent and not block_info0.isTransparent) or block_data0.type == block_data1.type:
            remove_face = True
    else:
        if (block_info1.isInvisible or block_info1.isTranslucent or block_info1.isTransparent) and block_data0.type != block_data1.type:
            remove_face = True
        if (not block_info1.isInvisible and not block_info0.isTranslucent and not block_info0.isTransparent) or block_data0.type == block_data1.type:
            add_face = True
    return add_face, remove_face
```

## Subtopics

### [Voxel Face Meshing with Packed Attributes](4153-voxel-face-meshing-with-packed-attributes.md)
A mesh generation technique for voxel terrain where vertex data is packed into a single float attribute to reduce buffer overhead, unpacked in the vertex shader, and used to generate quad faces for visible voxel surfaces. It sets itself apart by optimizing storage layout rather than active vertex count reduction. Choose it when minimizing VRAM bandwidth is critical.

### [Mesh Reindexing and Vertex Compaction](4160-mesh-reindexing-and-vertex-compaction.md)
This method compacts vertex buffers by removing unreferenced vertices and renumbering the remaining ones into a dense, contiguous array. It addresses sparse vertex buffers generated during incremental blocky meshing, where vertices are allocated per voxel face but many are culled or unused [G]. Unlike [Voxel Face Meshing with Packed Attributes](4153-voxel-face-meshing-with-packed-attributes.md), which optimizes storage layout, this method actively reduces vertex count [G]. Choose it when vertex buffer sparsity causes performance bottlenecks.

### [Chunk Mesh Data Structure & Buffer Update](4182-chunk-mesh-data-structure-buffer-update.md)
This method manages the creation, population, and recycling of GPU vertex buffers for meshed chunk geometry, specifically addressing the dynamic allocation of face data in large, contiguous memory regions. It solves the problem of efficiently mapping variable-sized mesh outputs from blocky surface extraction into a fixed-capacity GPU buffer without frequent reallocation. It sets itself apart by focusing on the low-level memory allocator (free-list) and buffer upload mechanics rather than the mesh generation algorithm itself. Choose it when managing dynamic buffer lifecycles is required.

### [Alpha/Translucent Block Rendering](4860-alpha-translucent-block-rendering.md)
Alpha/Translucent Block Rendering renders semi-transparent voxel blocks using standard rasterization with pre-computed vertex attributes and fragment shading. It avoids per-pixel ray-marching by applying distance-based fog and shading in the fragment shader. Choose it when rendering transparent surfaces without ray-tracing overhead.

## Operating characteristics
- **Time Complexity:** Face addition/removal is $O(1)$ for buffer updates but $O(V)$ for face search if not indexed by position [S1, unofficial]. Buffer upload is $O(N)$ where $N$ is the number of vertices/indices [G].
- **Memory Complexity:** $O(V)$ for vertex data, where each vertex uses 8 bytes (two 32-bit integers) [S1, unofficial].
- **Performance:** Reduces memory bandwidth and VRAM usage by packing attributes. Dynamic updates allow for efficient incremental changes without full mesh regeneration [S1, unofficial].
- **Strengths:** Low memory footprint, supports dynamic terrain changes, efficient GPU buffer updates.
- **Limitations:** Face search can be slow without spatial indexing; precision limits for coordinate range (10 bits per axis).
- **When to Use:** Dynamic voxel worlds with frequent block changes.
- **When Not to Use:** Static worlds where precomputed meshes are more efficient; worlds with very large coordinate ranges exceeding 10-bit precision.

## Background
Developed as part of a dynamic voxel terrain rendering system, this method adapts face culling and blocky meshing principles for real-time updates and compact data storage [S1, unofficial]. It builds on the principles of face culling and blocky meshing, adapting them for real-time updates and compact data storage.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
