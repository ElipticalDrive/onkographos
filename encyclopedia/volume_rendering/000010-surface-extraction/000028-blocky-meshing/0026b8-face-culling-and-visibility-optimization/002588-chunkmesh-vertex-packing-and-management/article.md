# ChunkMesh Vertex Packing and Management

Also called: Block Face Culling, Cube Face Mesh Generation

## Overview
ChunkMesh Vertex Packing and Management is a dynamic mesh generation and update technique for voxel terrain that combines face culling with compact vertex storage and incremental buffer updates [S1, unofficial]. It solves the problem of high memory overhead and frequent GPU buffer uploads in voxel worlds by packing vertex attributes into two 32-bit integers and reusing vertex slots via a free-list allocator [S1, unofficial]. Unlike static meshing methods, it supports incremental addition and removal of blocks, updating only the affected faces and their neighbors [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Voxel world state (block types, positions, chunk boundaries), camera position (for sorting translucent blocks), and block change events (add/remove).
- **Output:** GPU vertex buffers (`vertex0`, `vertex1`, `indices`) containing the mesh geometry for a chunk.

**Data Structures:**
The method maintains three primary arrays in CPU memory, which are uploaded to GPU buffers:
1.  `vertex0`: `Uint32Array` of packed vertex positions.
2.  `vertex1`: `Uint32Array` of packed vertex attributes (texture, texcoords, lighting).
3.  `indices`: `Uint32Array` of triangle indices.
4.  `vertexSlots`: Stack of available vertex indices for reuse.

**Packed Vertex Layout:**
Each vertex is represented by two 32-bit unsigned integers (`vertex0`, `vertex1`).

*   `vertex0` (Position):
    *   Bits 0–9: X coordinate (10 bits, range 0–1023).
    *   Bits 10–19: Y coordinate (10 bits, range 0–1023).
    *   Bits 20–29: Z coordinate (10 bits, range 0–1023).
    *   Bits 30–31: Unused.
*   `vertex1` (Attributes):
    *   Bits 0–9: Texture layer (10 bits).
    *   Bits 10–14: Texture coordinate X (5 bits, range 0–31).
    *   Bits 15–19: Texture coordinate Y (5 bits, range 0–31).
    *   Bits 20–21: Lighting/shading value (2 bits, range 0–3).
    *   Bits 22–31: Unused.

**Parameters:**
-   Block size ($n$): 16 units.
-   Direction mapping: 0=+X, 1=-X, 2=+Y, 3=-Y, 4=+Z, 5=-Z.
-   Lighting values: Array `[1, 1, 3, 0, 2, 2]` corresponding to directions 0–5.
-   Texture coordinate mapping: Orientation mapping from direction `dir` is `[2, 3, 0, 0, 3, 2][dir]`.

**Algorithm Steps:**

1.  **Initialization:**
    -   Initialize `vertex0`, `vertex1`, `indices` as empty arrays.
    -   Initialize `vertexSlots` as empty stack.
    -   Set `update = false`.

2.  **Face Visibility Determination:**
    -   For each block at position $P=(x,y,z)$ with type $T_0$ and neighbor at direction $i$ with type $T_1$:
        -   Determine if a face should be added or removed based on block properties (invisible, translucent, transparent, type equality).
        -   Logic for adding a face (`add=true`):
            -   If $T_0$ is invisible, no face.
            -   If $T_1$ is invisible/translucent/transparent AND $T_0 \neq T_1$, add face.
            -   If $T_1$ is NOT invisible AND ($T_0$ is NOT translucent/transparent OR $T_0 = T_1$), remove face (if it existed).
        -   Logic for removing a face (`add=false`): Reverse the above conditions.

3.  **Vertex Generation and Packing:**
    -   To add a face at $P$ with direction $dir$ and texture $tex$:
        -   Calculate 4 vertex positions using `cubeFacePosition(x, y, z, dir)`.
        -   Calculate 4 texture coordinates using `cubeFaceTexcoord(orientation)`.
        -   Get lighting value $L$ from `[1, 1, 3, 0, 2, 2][dir]`.
        -   For each of the 4 vertices:
            -   If `vertexSlots` is not empty, pop index $idx$; else $idx = current\_count$.
            -   Pack position: $v0 = x | (y \ll 10) | (z \ll 20)$.
            -   Pack attributes: $v1 = tex | (texX \ll 10) | (texY \ll 15) | (L \ll 20)$.
            -   Store $v0$ in `vertex0[idx]`, $v1$ in `vertex1[idx]`.
            -   Push $idx$ to `vertexSlots` if it was reused (or manage separately).
        -   Push indices $(a, b, c, c, d, a)$ to `indices`, where $a,b,c,d$ are the 4 vertex indices.
        -   Set `update = true`.

4.  **Face Removal and Slot Reuse:**
    -   To remove a face at $P$ with direction $dir$:
        -   Find the starting vertex index $idx$ by searching `vertex0` for the packed position of the 4 vertices of the face.
        -   If found:
            -   Set `vertex0[idx...idx+3]` and `vertex1[idx...idx+3]` to `NaN` (mark as invalid).
            -   Remove the 6 corresponding indices from `indices` using `splice`.
            -   Push $idx$ to `vertexSlots` for reuse.
            -   Set `update = true`.

5.  **Buffer Update:**
    -   If `update` is true:
        -   Upload `vertex0`, `vertex1`, `indices` to GPU buffers using `gl.DYNAMIC_DRAW`.
        -   Reset `update = false`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data structures
vertex0 = []  # Uint32Array
vertex1 = []  # Uint32Array
indices = []  # Uint32Array
vertexSlots = []  # Stack
update = False

# Constants
BLOCK_SIZE = 16
LIGHTING = [1, 1, 3, 0, 2, 2]
DIRECTION_MAP = [2, 3, 0, 0, 3, 2]  # Orientation mapping from dir

def pack_vertex0(x, y, z):
    return x | (y << 10) | (z << 20)

def pack_vertex1(texture, texX, texY, light):
    return texture | (texX << 10) | (texY << 15) | (light << 20)

def get_face_vertices(x, y, z, dir):
    # Returns 12 floats: 4 vertices (x,y,z) each
    # Based on cubeFacePosition logic
    nx, ny, nz = x * BLOCK_SIZE, y * BLOCK_SIZE, z * BLOCK_SIZE
    if dir == 0: # +X
        return [nx+BLOCK_SIZE, ny, nz, nx+BLOCK_SIZE, ny+BLOCK_SIZE, nz, nx+BLOCK_SIZE, ny+BLOCK_SIZE, nz+BLOCK_SIZE, nx+BLOCK_SIZE, ny, nz+BLOCK_SIZE]
    elif dir == 1: # -X
        return [nx, ny, nz, nx, ny, nz+BLOCK_SIZE, nx, ny+BLOCK_SIZE, nz+BLOCK_SIZE, nx, ny+BLOCK_SIZE, nz]
    elif dir == 2: # +Y
        return [nx, ny+BLOCK_SIZE, nz, nx, ny+BLOCK_SIZE, nz+BLOCK_SIZE, nx+BLOCK_SIZE, ny+BLOCK_SIZE, nz+BLOCK_SIZE, nx+BLOCK_SIZE, ny+BLOCK_SIZE, nz]
    elif dir == 3: # -Y
        return [nx+BLOCK_SIZE, ny, nz+BLOCK_SIZE, nx, ny, nz+BLOCK_SIZE, nx, ny, nz, nx+BLOCK_SIZE, ny, nz]
    elif dir == 4: # +Z
        return [nx, ny, nz+BLOCK_SIZE, nx+BLOCK_SIZE, ny, nz+BLOCK_SIZE, nx+BLOCK_SIZE, ny+BLOCK_SIZE, nz+BLOCK_SIZE, nx, ny+BLOCK_SIZE, nz+BLOCK_SIZE]
    elif dir == 5: # -Z
        return [nx, ny, nz, nx, ny+BLOCK_SIZE, nz, nx+BLOCK_SIZE, ny+BLOCK_SIZE, nz, nx+BLOCK_SIZE, ny, nz]

def get_face_texcoords(dir):
    # Returns 8 floats: 4 vertices (u,v) each
    # Based on cubeFaceTexcoord logic
    orientation = DIRECTION_MAP[dir]
    if orientation == 0: return [0, 0, 0, BLOCK_SIZE, BLOCK_SIZE, BLOCK_SIZE, BLOCK_SIZE, 0]
    elif orientation == 1: return [BLOCK_SIZE, 0, 0, 0, 0, BLOCK_SIZE, BLOCK_SIZE, BLOCK_SIZE]
    elif orientation == 2: return [0, 0, BLOCK_SIZE, 0, BLOCK_SIZE, BLOCK_SIZE, 0, BLOCK_SIZE]
    elif orientation == 3: return [0, BLOCK_SIZE, BLOCK_SIZE, BLOCK_SIZE, BLOCK_SIZE, 0, 0, 0]
    # Add other cases as needed based on source

def add_vertex(pos, texture, texX, texY, light, reuse_index=-1):
    global vertex0, vertex1, vertexSlots
    if reuse_index >= 0:
        idx = reuse_index
    else:
        idx = len(vertex0)
        vertex0.append(0)
        vertex1.append(0)
    
    x, y, z = pos
    vertex0[idx] = pack_vertex0(x, y, z)
    vertex1[idx] = pack_vertex1(texture, texX, texY, light)
    return idx

def add_cube_face(x, y, z, dir, texture):
    global indices, vertexSlots, update
    positions = get_face_vertices(x, y, z, dir)
    texcoords = get_face_texcoords(dir)
    light = LIGHTING[dir]
    
    # Reuse slots if available
    reuse_indices = []
    for _ in range(4):
        if vertexSlots:
            reuse_indices.append(vertexSlots.pop())
        else:
            reuse_indices.append(-1)
            
    verts = []
    for i in range(4):
        pos = (positions[i*3], positions[i*3+1], positions[i*3+2])
        tx, ty = texcoords[i*2], texcoords[i*2+1]
        # Normalize texcoords to 0-1 range if needed, or keep as 0-16
        # Source uses 0-16 for packing, then shader scales
        verts.append(add_vertex(pos, texture, tx, ty, light, reuse_indices[i]))
        
    indices.extend([verts[0], verts[1], verts[2], verts[2], verts[3], verts[0]])
    update = True

def remove_face(face_positions):
    global vertex0, vertex1, indices, vertexSlots, update
    # face_positions is list of 4 (x,y,z) tuples
    # Find starting index
    start_idx = -1
    for i in range(0, len(vertex0), 4):
        if vertex0[i] == 0: continue # Skip invalid
        packed_positions = [pack_vertex0(p[0], p[1], p[2]) for p in face_positions]
        if vertex0[i:i+4] == packed_positions:
            start_idx = i
            break
            
    if start_idx == -1: return
    
    # Mark as invalid
    for i in range(4):
        vertex0[start_idx + i] = float('nan') # Or a sentinel value
        vertex1[start_idx + i] = float('nan')
        
    # Remove indices
    # Find index in indices array
    idx_in_indices = -1
    for i in range(0, len(indices), 6):
        if indices[i] == start_idx:
            idx_in_indices = i
            break
            
    if idx_in_indices != -1:
        del indices[idx_in_indices:idx_in_indices+6]
        
    # Push to slots
    for i in range(4):
        vertexSlots.append(start_idx + i)
        
    update = True

def update_buffers():
    if not update: return
    # Upload to GPU
    # gl.bufferData(gl.ARRAY_BUFFER, new Uint32Array(vertex0), gl.DYNAMIC_DRAW)
    # ... similar for vertex1 and indices
    update = False
```

**Edge Cases and Pitfalls:**
-   **Neighbor Chunk Loading:** When removing a block, the neighbor chunk's mesh must be loaded if it doesn't exist before adding the exposed face to it [S1, unofficial].
-   **Precision:** Coordinates are limited to 10 bits (0–1023). Blocks outside this range cannot be represented in the packed format.
-   **Degenerate Input:** `NaN` values are used to mark invalid vertices. Search functions must skip `NaN` entries.
-   **Index Consistency:** When removing faces, the `indices` array must be spliced correctly to maintain triangle connectivity.

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
-   **Time Complexity:** Face culling is $O(1)$ per face. Mesh generation is proportional to the number of visible faces. Buffer updates are $O(V + I)$ where $V$ is vertices and $I$ is indices.
-   **Memory Complexity:** $O(V)$ for vertex data, $O(I)$ for indices. Packing reduces vertex size to 8 bytes per vertex.
-   **Performance:** Efficient for dynamic voxel worlds with frequent block changes. Reduces GPU memory bandwidth by minimizing buffer uploads.
-   **Strengths:** Low memory footprint, fast incremental updates, simple implementation.
-   **Limitations:** Limited coordinate range (10 bits), requires careful management of vertex slots and indices, not suitable for highly complex non-voxel geometry.

## Background
The method originates from the Exolands project, an unofficial voxel engine implementation [S1, unofficial]. It builds on the principles of face culling and visibility optimization [3861-face-culling-and-visibility-optimization.md] to create efficient chunk-based meshes.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
