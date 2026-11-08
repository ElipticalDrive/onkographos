# ChunkMesh Vertex Packing and Management

Also called: Block Face Culling, Cube Face Mesh Generation

## Overview
ChunkMesh Vertex Packing and Management is a method for generating and maintaining dynamic meshes in voxel terrain by packing vertex attributes into compact 32-bit integers and managing buffer updates via a free-list of reusable vertex slots [S1, unofficial]. It solves the problem of high memory bandwidth and buffer upload overhead in dynamic voxel worlds by minimizing the data transferred to the GPU per vertex and avoiding frequent reallocation [S1, unofficial]. It builds on [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) by providing the concrete data layout and incremental update logic required to implement face culling efficiently in a chunk-based architecture.

## How it works
**Inputs and Outputs:**
*   **Input:** Voxel world state (block types, chunk positions), camera position (for alpha sorting), and block modification events (add/remove).
*   **Output:** GPU vertex buffers (`vertex0`, `vertex1`) and index buffer (`indices`) representing the visible surface of the chunk.

**Data Structures:**
The method uses three primary arrays in CPU memory, which are uploaded to GPU buffers as `Uint32Array`:
1.  `vertex0`: Array of 32-bit unsigned integers. Each entry stores the position of a vertex.
    *   **Layout:**
        *   Bits 0–9: X coordinate (10 bits).
        *   Bits 10–19: Y coordinate (10 bits).
        *   Bits 20–29: Z coordinate (10 bits).
        *   Bits 30–31: Unused.
    *   **Capacity:** Coordinates are stored in block units (0–1023). The block size $n$ is 16, so the coordinate range covers $16 \times 1024 = 16384$ units.
2.  `vertex1`: Array of 32-bit unsigned integers. Each entry stores texture and lighting attributes.
    *   **Layout:**
        *   Bits 0–9: Texture layer index (10 bits).
        *   Bits 10–14: Texture coordinate X (5 bits).
        *   Bits 15–19: Texture coordinate Y (5 bits).
        *   Bits 20–21: Shading/Lighting value (2 bits).
        *   Bits 22–31: Unused.
3.  `indices`: Array of 32-bit unsigned integers. Stores triangle indices.
4.  `vertexSlots`: Stack of reusable vertex indices. When a face is removed, its starting index is pushed here for reuse, preventing buffer growth.

**Numbered Steps:**

1.  **Initialize Buffers:** Create empty arrays `vertex0`, `vertex1`, `indices`, and `vertexSlots`. Set `this.update = false`.
2.  **Determine Face Visibility:** For each block in the chunk, check its 6 neighbors. Use the logic from [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) to determine if a face is visible.
    *   *Note:* The specific interaction logic in [S1, unofficial] adds conditions for block types and transparency:
        *   Add face if: Neighbor is invisible/translucent/transparent AND current block type $\neq$ neighbor block type.
        *   Remove face if: Neighbor is NOT invisible AND (Current block is NOT translucent/transparent OR Block types are identical).
3.  **Generate Face Geometry:** For each visible face at $(x, y, z)$ with direction $dir \in \{0, 1, 2, 3, 4, 5\}$ (representing +X, -X, +Y, -Y, +Z, -Z):
    *   Calculate vertex positions using `cubeFacePosition(x, y, z, dir)`. The block size $n=16$.
    *   Calculate texture coordinates using `cubeFaceTexcoord(orientation)`. The orientation mapping is `[2, 3, 0, 0, 3, 2][dir]`.
    *   Determine lighting value `dLight` using `[1, 1, 3, 0, 2, 2][dir]`.
4.  **Pack and Add Vertices:**
    *   Pop a reusable index from `vertexSlots` if available; otherwise, use the current length of `vertex0`.
    *   Pack the 4 vertices of the quad into `vertex0` and `vertex1`.
    *   Push the 6 indices (two triangles: $a, b, c, c, d, a$) to `indices`.
    *   Set `this.update = true`.
5.  **Remove Face (Incremental Update):**
    *   When a block is removed or hidden, identify the face to remove.
    *   Search `vertex0` in steps of 4 to find the starting index of the face matching the target position. Skip entries where `vertex0[i]` is `NaN`.
    *   If found, set `vertex0[i...i+3]` and `vertex1[i...i+3]` to `NaN`.
    *   Remove the corresponding 6 indices from `indices` using `splice`.
    *   Push the starting index $i$ to `vertexSlots` for reuse.
    *   Set `this.update = true`.
6.  **Update GPU Buffers:**
    *   If `this.update` is true, bind the VAO and upload `vertex0`, `vertex1`, and `indices` to GPU buffers using `gl.DYNAMIC_DRAW`.
    *   Reset `this.update = false`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
BLOCK_SIZE = 16
DIR_MAP = [2, 3, 0, 0, 3, 2] # Orientation mapping for texcoords
LIGHT_MAP = [1, 1, 3, 0, 2, 2] # Lighting values per direction

class ChunkMesh:
    def __init__(self):
        self.vertex0 = [] # Packed positions
        self.vertex1 = [] # Packed attributes
        self.indices = [] # Triangle indices
        self.vertexSlots = [] # Reusable index stack
        self.update = False

    def add_vertex(self, x, y, z, texture_layer, texcoord_x, texcoord_y, d_light, index_hint=-1):
        # Pack position into vertex0
        packed_pos = x | (y << 10) | (z << 20)
        
        # Pack attributes into vertex1
        # texcoord_x/y are 0-15, stored in 5 bits (0-31)
        # d_light is 0-3, stored in 2 bits (0-3)
        packed_attr = texture_layer | (texcoord_x << 10) | (texcoord_y << 15) | (d_light << 20)
        
        if index_hint >= 0:
            self.vertex0[index_hint] = packed_pos
            self.vertex1[index_hint] = packed_attr
            return index_hint
        else:
            self.vertex0.append(packed_pos)
            self.vertex1.append(packed_attr)
            return len(self.vertex0) - 1

    def get_face_positions(self, x, y, z, dir):
        # Returns 12 floats: 4 vertices (x,y,z) each
        # Coordinates are in world units (block * 16)
        bx, by, bz = x * BLOCK_SIZE, y * BLOCK_SIZE, z * BLOCK_SIZE
        if dir == 0: # +X
            return [bx+16, by, bz, bx+16, by+16, bz, bx+16, by+16, bz+16, bx+16, by, bz+16]
        elif dir == 1: # -X
            return [bx, by, bz, bx, by, bz+16, bx, by+16, bz+16, bx, by+16, bz]
        elif dir == 2: # +Y
            return [bx, by+16, bz, bx, by+16, bz+16, bx+16, by+16, bz+16, bx+16, by+16, bz]
        elif dir == 3: # -Y
            return [bx+16, by, bz+16, bx, by, bz+16, bx, by, bz, bx+16, by, bz]
        elif dir == 4: # +Z
            return [bx, by, bz+16, bx+16, by, bz+16, bx+16, by+16, bz+16, bx, by+16, bz+16]
        elif dir == 5: # -Z
            return [bx, by, bz, bx, by+16, bz, bx+16, by+16, bz, bx+16, by, bz]
        return []

    def get_face_texcoords(self, dir):
        # Returns 4 floats: 2 pairs of (u, v)
        # Values are 0-16, mapped to 0-1 in shader via * 0.0625
        orientation = DIR_MAP[dir]
        if orientation == 0: return [0, 0, 0, 16, 16, 16, 16, 0]
        elif orientation == 1: return [16, 0, 0, 0, 0, 16, 16, 16]
        elif orientation == 2: return [0, 16, 16, 16, 16, 0, 0, 0]
        elif orientation == 3: return [0, 0, 16, 0, 16, 16, 0, 16]
        # Fallback for other orientations if needed, based on source logic
        return [0, 0, 0, 16, 16, 16, 16, 0]

    def add_cube_face(self, x, y, z, dir, texture_layer):
        positions = self.get_face_positions(x, y, z, dir)
        texcoords = self.get_face_texcoords(dir)
        d_light = LIGHT_MAP[dir]
        
        # Get reusable index
        if self.vertexSlots:
            idx = self.vertexSlots.pop()
        else:
            idx = len(self.vertex0)
            
        # Add 4 vertices
        v0 = self.add_vertex(positions[0], positions[1], positions[2], texture_layer, texcoords[0], texcoords[1], d_light, idx)
        v1 = self.add_vertex(positions[3], positions[4], positions[5], texture_layer, texcoords[2], texcoords[3], d_light, idx + 1)
        v2 = self.add_vertex(positions[6], positions[7], positions[8], texture_layer, texcoords[4], texcoords[5], d_light, idx + 2)
        v3 = self.add_vertex(positions[9], positions[10], positions[11], texture_layer, texcoords[6], texcoords[7], d_light, idx + 3)
        
        # Add indices
        self.indices.extend([v0, v1, v2, v2, v3, v0])
        self.update = True

    def remove_face(self, face_positions):
        # face_positions is a list of 12 floats
        # Search for the face in vertex0
        target_packed = []
        for i in range(0, 12, 3):
            x, y, z = face_positions[i], face_positions[i+1], face_positions[i+2]
            # Pack as in add_vertex
            packed = x | (y << 10) | (z << 20)
            target_packed.append(packed)
            
        start_idx = -1
        for i in range(0, len(self.vertex0), 4):
            if self.vertex0[i] == target_packed[0] and \
               self.vertex0[i+1] == target_packed[1] and \
               self.vertex0[i+2] == target_packed[2] and \
               self.vertex0[i+3] == target_packed[3]:
                start_idx = i
                break
                
        if start_idx == -1:
            return
            
        # Mark as removed
        self.vertex0[start_idx:start_idx+4] = [float('nan')] * 4
        self.vertex1[start_idx:start_idx+4] = [float('nan')] * 4
        
        # Remove indices
        # Find the index in self.indices that matches start_idx
        # Assuming indices are stored in groups of 6
        for i in range(0, len(self.indices), 6):
            if self.indices[i] == start_idx:
                self.indices[i:i+6] = []
                break
                
        # Push to slots for reuse
        self.vertexSlots.append(start_idx)
        self.update = True

    def update_buffers(self, gl):
        if not self.update:
            return
            
        gl.bind_buffer(gl.ARRAY_BUFFER, self.vertex0_buffer)
        gl.buffer_data(gl.ARRAY_BUFFER, Uint32Array(self.vertex0), gl.DYNAMIC_DRAW)
        
        gl.bind_buffer(gl.ARRAY_BUFFER, self.vertex1_buffer)
        gl.buffer_data(gl.ARRAY_BUFFER, Uint32Array(self.vertex1), gl.DYNAMIC_DRAW)
        
        gl.bind_buffer(gl.ELEMENT_ARRAY_BUFFER, self.index_buffer)
        gl.buffer_data(gl.ELEMENT_ARRAY_BUFFER, Uint32Array(self.indices), gl.DYNAMIC_DRAW)
        
        self.update = False
```

**Edge Cases and Pitfalls:**
*   **Neighbor Chunk Loading:** When updating a face that crosses a chunk boundary, the neighbor chunk's mesh must be loaded if it does not already exist before adding the exposed face to it [S1, unofficial].
*   **NaN Handling:** Removed vertices are marked with `NaN` in `vertex0`. The `findFace` function must explicitly skip these entries to avoid false positives during face removal.
*   **Index Reuse:** The `vertexSlots` stack ensures that removed vertices are reused, preventing unbounded growth of the vertex arrays. However, this means indices are not contiguous, requiring the `findIndex` helper to locate the correct position in the `indices` array for removal.

## Subtopics

### [Voxel Face Meshing with Packed Attributes](4153-voxel-face-meshing-with-packed-attributes.md)
Defines the specific bit-layout and shader logic to unpack the `vertex0` and `vertex1` attributes into position, texture, and lighting data for rendering.

### [Mesh Reindexing and Vertex Compaction](4160-mesh-reindexing-and-vertex-compaction.md)
Compacts the sparse vertex buffer by removing `NaN` entries and renumbering indices, reducing memory footprint at the cost of higher CPU overhead during updates.

### [Chunk Mesh Data Structure & Buffer Update](4182-chunk-mesh-data-structure-buffer-update.md)
Manages the GPU-side memory using a large Shader Storage Buffer (SSBO) and a free-list allocator, centralizing memory management for multiple chunks.

### [Alpha/Translucent Block Rendering](4860-alpha-translucent-block-rendering.md)
Uses the packed attributes to sort translucent faces back-to-front based on camera direction, ensuring correct alpha blending.

## Operating characteristics
*   **Time Complexity:** Adding a face is $O(1)$ amortized (push to array). Removing a face is $O(V)$ where $V$ is the number of vertices, due to the linear search in `findFace` and `findIndex`. Buffer upload is $O(V + I)$ where $I$ is the number of indices.
*   **Memory Complexity:** $O(V)$ for vertex data and $O(I)$ for indices. The packing reduces vertex size from 3 floats (12 bytes) + attributes to 2 uints (8 bytes), saving 33% of vertex buffer memory compared to standard float layouts.
*   **Performance:** Efficient for dynamic worlds with frequent block changes due to the reuse of vertex slots and compact data layout. The linear search for face removal can become a bottleneck in very large chunks with many faces.
*   **Strengths:** Low memory overhead, fast addition of faces, simple implementation.
*   **Limitations:** Slow removal of faces due to linear search. Not suitable for static meshes where reindexing would be more efficient.

## Background
The method originates from the Exolands voxel engine, an open-source project by kvxy [S1, unofficial]. It was designed to provide efficient dynamic meshing for voxel terrain in web-based environments, leveraging JavaScript's typed arrays and WebGL's buffer management capabilities. The approach emphasizes simplicity and low memory usage, making it suitable for client-side rendering in browser-based games.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
