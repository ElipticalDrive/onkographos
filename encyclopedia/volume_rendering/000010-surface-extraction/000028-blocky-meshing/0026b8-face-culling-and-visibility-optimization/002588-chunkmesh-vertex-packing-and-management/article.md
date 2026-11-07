# ChunkMesh Vertex Packing and Management

Also called: Block Face Culling, Cube Face Mesh Generation

## Overview
ChunkMesh Vertex Packing and Management is a dynamic mesh generation and update technique for blocky voxel worlds that stores vertex data in highly compacted 32-bit integer formats and manages incremental mesh changes via a reusable vertex slot pool. It solves the problem of high memory bandwidth and buffer update overhead in real-time voxel rendering by minimizing the number of vertices and indices transferred to the GPU per frame. It differs from static meshing methods by supporting arbitrary block addition and removal with localized face culling, and from simple face culling by explicitly managing vertex reuse and buffer synchronization [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: A 3D grid of voxel blocks (occupancy status, block type, transparency flags), a block size $n=16$, and a set of texture layers.
*   **Output**: Three GPU buffers: `vertex0Buffer` (Uint32Array), `vertex1Buffer` (Uint32Array), and `indexBuffer` (Uint32Array).

**Data Structures**
The mesh data is stored in three JavaScript arrays (or equivalent native arrays) which are uploaded to the GPU as `Uint32Array`:
1.  `vertex0`: An array of 32-bit unsigned integers. Each vertex occupies one entry.
    *   **Layout**:
        *   Bits 0–9: X coordinate (10 bits).
        *   Bits 10–19: Y coordinate (10 bits).
        *   Bits 20–29: Z coordinate (10 bits).
        *   Bits 30–31: Unused.
2.  `vertex1`: An array of 32-bit unsigned integers. Each vertex occupies one entry.
    *   **Layout**:
        *   Bits 0–9: Texture layer index (10 bits).
        *   Bits 10–14: Texture coordinate X (5 bits).
        *   Bits 15–19: Texture coordinate Y (5 bits).
        *   Bits 20–21: Lighting/shading value (2 bits).
        *   Bits 22–31: Unused.
3.  `indices`: An array of 32-bit unsigned integers. Each face consists of 6 indices (two triangles: $a, b, c, c, d, a$).
4.  `vertexSlots`: A stack (array) storing indices of vertices that have been marked as "removed" (NaN) and are available for reuse.

**Numbered Steps**

1.  **Face Visibility Determination**: For each voxel at $(x, y, z)$, determine the visibility of its 6 faces based on neighbor occupancy. A face is visible if the neighbor is empty, transparent, or translucent AND the block types differ [S1, unofficial].
2.  **Vertex Generation**: For each visible face, calculate the 4 vertex positions using `cubeFacePosition(x, y, z, dir)` and texture coordinates using `cubeFaceTexcoord(dir)`.
    *   The block size is $n=16$.
    *   Directions: 0=+X, 1=-X, 2=+Y, 3=-Y, 4=+Z, 5=-Z.
    *   Lighting value $dLight$ is determined by direction: `[1, 1, 3, 0, 2, 2][dir]`.
3.  **Vertex Packing**: Pack the vertex data into `vertex0` and `vertex1` integers.
    *   `vertex0 = x | (y << 10) | (z << 20)`
    *   `vertex1 = textureLayer | (texcoordX << 10) | (texcoordY << 15) | (dLight << 20)`
    *   Note: `texcoordX` and `texcoordY` are stored as 5-bit integers (0–31). In the shader, they are converted to floats by multiplying by $0.0625$ ($1/16$) [S1, unofficial].
4.  **Vertex Allocation**:
    *   If `vertexSlots` is not empty, pop the top index `idx` for reuse.
    *   If `vertexSlots` is empty, push new indices to `vertex0` and `vertex1` arrays.
    *   Store the packed `vertex0` and `vertex1` values at the allocated index.
5.  **Index Generation**: Push the 6 indices for the two triangles of the face to the `indices` array: `[a, b, c, c, d, a]`.
6.  **Buffer Update**: Set `this.update = true`. When `update` is true, the `updateBuffers` function uploads the `vertex0`, `vertex1`, and `indices` arrays to the GPU using `gl.DYNAMIC_DRAW` hints [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
BLOCK_SIZE = 16
DIRECTION_LIGHTING = [1, 1, 3, 0, 2, 2]
DIRECTION_TEXCOORD_MAP = [2, 3, 0, 0, 3, 2] # Maps dir to orientation for texcoord generation

class ChunkMesh:
    def __init__(self):
        self.vertex0 = []
        self.vertex1 = []
        self.indices = []
        self.vertexSlots = [] # Stack of reusable indices
        self.update = False

    def add_vertex(self, x, y, z, texture_layer, texcoord_x, texcoord_y, d_light, index_hint=-1):
        """
        Adds a vertex to the mesh.
        Returns the index of the vertex.
        """
        # Pack vertex0
        v0 = x | (y << 10) | (z << 20)
        
        # Pack vertex1
        # texcoord_x and texcoord_y are expected to be 5-bit integers (0-31)
        v1 = texture_layer | (texcoord_x << 10) | (texcoord_y << 15) | (d_light << 20)
        
        if index_hint >= 0:
            # Reuse existing vertex slot
            self.vertex0[index_hint] = v0
            self.vertex1[index_hint] = v1
            return index_hint
        else:
            # Push new vertex
            self.vertex0.append(v0)
            self.vertex1.append(v1)
            return len(self.vertex0) - 1

    def add_cube_face(self, x, y, z, dir, texture_layer):
        """
        Adds a single face of a cube to the mesh.
        """
        # Get positions and texcoords
        positions = self.cube_face_position(x, y, z, dir)
        texcoords = self.cube_face_texcoord(DIRECTION_TEXCOORD_MAP[dir])
        d_light = DIRECTION_LIGHTING[dir]
        
        # Allocate vertices
        # Try to reuse slots
        if self.vertexSlots:
            base_idx = self.vertexSlots.pop()
        else:
            base_idx = len(self.vertex0)
            
        # Add 4 vertices
        idx_a = self.add_vertex(positions[0], positions[1], positions[2], texture_layer, texcoords[0], texcoords[1], d_light, base_idx)
        idx_b = self.add_vertex(positions[3], positions[4], positions[5], texture_layer, texcoords[2], texcoords[3], d_light, base_idx + 1)
        idx_c = self.add_vertex(positions[6], positions[7], positions[8], texture_layer, texcoords[4], texcoords[5], d_light, base_idx + 2)
        idx_d = self.add_vertex(positions[9], positions[10], positions[11], texture_layer, texcoords[6], texcoords[7], d_light, base_idx + 3)
        
        # Add indices for two triangles: a-b-c, c-d-a
        self.indices.extend([idx_a, idx_b, idx_c, idx_c, idx_d, idx_a])
        self.update = True

    def remove_face(self, face_positions):
        """
        Removes a face defined by 4 vertex positions.
        face_positions: list of 12 floats [x0,y0,z0, x1,y1,z1, x2,y2,z2, x3,y3,z3]
        """
        # Find the starting index of the face
        start_idx = self.find_face(face_positions)
        if start_idx == -1:
            return
            
        # Mark vertices as invalid (NaN)
        self.vertex0[start_idx] = float('nan')
        self.vertex0[start_idx + 1] = float('nan')
        self.vertex0[start_idx + 2] = float('nan')
        self.vertex0[start_idx + 3] = float('nan')
        
        self.vertex1[start_idx] = float('nan')
        self.vertex1[start_idx + 1] = float('nan')
        self.vertex1[start_idx + 2] = float('nan')
        self.vertex1[start_idx + 3] = float('nan')
        
        # Remove indices
        # Find the index in the indices array where the start_idx appears
        idx_arr_pos = self.find_index(start_idx)
        if idx_arr_pos != -1:
            self.indices[idx_arr_pos : idx_arr_pos + 6] = []
            
        # Push the base index back to the slot pool for reuse
        self.vertexSlots.append(start_idx)
        self.update = True

    def find_face(self, face_positions):
        """
        Searches for a face in vertex0 array.
        Returns the starting index or -1.
        """
        # Pack the target face positions into uints for comparison
        target_v0 = [
            face_positions[0] | (face_positions[1] << 10) | (face_positions[2] << 20),
            face_positions[3] | (face_positions[4] << 10) | (face_positions[5] << 20),
            face_positions[6] | (face_positions[7] << 10) | (face_positions[8] << 20),
            face_positions[9] | (face_positions[10] << 10) | (face_positions[11] << 20)
        ]
        
        for i in range(0, len(self.vertex0), 4):
            if self.vertex0[i] != self.vertex0[i]: # Check for NaN
                continue
            if (self.vertex0[i] == target_v0[0] and
                self.vertex0[i+1] == target_v0[1] and
                self.vertex0[i+2] == target_v0[2] and
                self.vertex0[i+3] == target_v0[3]):
                return i
        return -1

    def find_index(self, vertex_idx):
        """
        Finds the position in the indices array where a specific vertex index starts a face.
        """
        for i in range(0, len(self.indices), 6):
            if self.indices[i] == vertex_idx:
                return i
        return -1

    def cube_face_position(self, x, y, z, dir):
        """
        Returns 12 floats representing 4 vertices of a face.
        """
        n = BLOCK_SIZE
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

    def cube_face_texcoord(self, orientation):
        """
        Returns 8 floats representing 4 (u,v) pairs.
        """
        # Simplified mapping based on source logic
        # Source returns [0,0, 0,16, 16,16, 16,0] for orientation 0
        # Assuming texcoords are in range 0-16, stored as 5-bit ints (0-31)
        # We return raw values to be packed into vertex1
        if orientation == 0:
            return [0, 0, 0, 16, 16, 16, 16, 0]
        elif orientation == 1:
            return [16, 0, 0, 0, 0, 16, 16, 16]
        elif orientation == 2:
            return [0, 0, 16, 0, 16, 16, 0, 16]
        elif orientation == 3:
            return [0, 16, 16, 16, 16, 0, 0, 0]
        # Default fallback
        return [0, 0, 0, 16, 16, 16, 16, 0]

    def update_buffers(self, gl):
        """
        Uploads data to GPU.
        """
        gl.bindVertexArray(self.vao)
        gl.bindBuffer(gl.ARRAY_BUFFER, self.vertex0Buffer)
        gl.bufferData(gl.ARRAY_BUFFER, Uint32Array(self.vertex0), gl.DYNAMIC_DRAW)
        gl.bindBuffer(gl.ARRAY_BUFFER, self.vertex1Buffer)
        gl.bufferData(gl.ARRAY_BUFFER, Uint32Array(self.vertex1), gl.DYNAMIC_DRAW)
        gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, self.indexBuffer)
        gl.bufferData(gl.ELEMENT_ARRAY_BUFFER, Uint32Array(self.indices), gl.DYNAMIC_DRAW)
```

**Parameters**
*   `BLOCK_SIZE` ($n$): 16. Determines the world unit scale of a voxel face.
*   `vertex0` bit depth: 10 bits per coordinate. Allows coordinates in range $[0, 1023]$.
*   `vertex1` texcoord bit depth: 5 bits. Allows 32 distinct texture coordinate steps.
*   `vertex1` lighting bit depth: 2 bits. Allows 4 distinct lighting levels.

**Edge Cases and Pitfalls**
*   **Neighbor Chunk Loading**: When removing a block, the neighbor chunk's mesh must be loaded if it does not exist before adding the exposed face to it [S1, unofficial].
*   **NaN Comparison**: `findFace` must skip entries where `vertex0` is NaN to avoid false matches or crashes.
*   **Index Search**: `findIndex` is a linear search because the `indices` array is not sorted after removals. This can be slow for very large meshes but is necessary for correctness in this dynamic allocation scheme.
*   **Coordinate Overflow**: The 10-bit limit for coordinates means chunks larger than $1024 \times 1024 \times 1024$ units cannot be represented in a single mesh without coordinate wrapping or multiple meshes.

## Subtopics

### [Voxel Face Meshing with Packed Attributes](4153-voxel-face-meshing-with-packed-attributes.md)
Defines the specific bit-layout and shader logic to reconstruct geometry from the packed data.

### [Mesh Reindexing and Vertex Compaction](4160-mesh-reindexing-and-vertex-compaction.md)
Compacts the sparse vertex buffer by removing unreferenced vertices and renumbering.

### [Chunk Mesh Data Structure & Buffer Update](4182-chunk-mesh-data-structure-buffer-update.md)
Manages GPU buffers using a MegaSSBO and free-list allocator.

### [Alpha/Translucent Block Rendering](4860-alpha-translucent-block-rendering.md)
Renders translucent blocks using pre-computed vertex attributes and fragment shading.

## Operating characteristics
*   **Time Complexity**:
    *   Adding a face: $O(1)$ for vertex packing and index pushing.
    *   Removing a face: $O(V)$ for `findFace` (linear scan of vertices) and $O(I)$ for `findIndex` (linear scan of indices), where $V$ is the number of vertices and $I$ is the number of indices.
    *   Buffer Update: $O(V + I)$ to copy data to GPU.
*   **Memory Complexity**:
    *   Vertex storage: 8 bytes per vertex (2 x 32-bit ints).
    *   Index storage: 4 bytes per index.
    *   Total memory per face: 32 bytes (4 vertices) + 24 bytes (6 indices) = 56 bytes.
*   **Performance**: The use of `Uint32Array` and `DYNAMIC_DRAW` allows efficient GPU updates. The vertex slot reuse minimizes memory fragmentation and allocation overhead.
*   **Strengths**: Low memory footprint, fast incremental updates for small changes, simple data structure.
*   **Limitations**: Linear search for face removal is slow for large meshes. 10-bit coordinate limit restricts chunk size. 5-bit texcoord limit reduces texture precision.

## Background
ChunkMesh Vertex Packing and Management is implemented in the Exolands voxel engine [S1, unofficial]. It builds on the principles of face culling and visibility optimization to generate efficient meshes for blocky terrain. The technique is designed for real-time rendering in JavaScript-based web environments, where memory management and buffer update efficiency are critical.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
