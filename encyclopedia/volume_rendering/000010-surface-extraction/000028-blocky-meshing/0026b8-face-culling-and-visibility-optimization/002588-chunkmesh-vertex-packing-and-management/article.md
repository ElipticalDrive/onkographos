# ChunkMesh Vertex Packing and Management

Also called: Block Face Culling, Cube Face Mesh Generation

## Overview
ChunkMesh Vertex Packing and Management is a dynamic mesh generation and update technique for voxel terrain that minimizes GPU memory bandwidth and buffer upload overhead by packing vertex attributes into two 32-bit unsigned integers per vertex. It solves the problem of sparse, fragmented vertex buffers in incremental blocky meshing by reusing vertex slots and compacting data in-place. It sets itself apart from sibling methods like [Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md) by focusing on the internal data structure and memory management of the mesh itself, rather than just the visibility logic. Unlike [Voxel Face Meshing with Packed Attributes](4153-voxel-face-meshing-with-packed-attributes.md), which describes the shader-side unpacking, this method details the CPU-side generation, packing, and dynamic buffer update loop.

## How it works
The method manages a chunk's geometry by maintaining three arrays: `vertex0` and `vertex1` (packed vertex data) and `indices` (triangle indices). It supports incremental updates where blocks are added or removed, requiring the mesh to detect exposed faces and update the GPU buffers accordingly.

### Inputs and Outputs
*   **Inputs**: Block grid data (occupancy, type, transparency), camera position (for sorting in alpha variants), and block change events (add/remove at $x, y, z$).
*   **Outputs**: Updated GPU vertex buffers (`vertex0Buffer`, `vertex1Buffer`) and index buffer (`indexBuffer`).

### Data Structures
Vertex data is packed into two `Uint32` values per vertex. The block size (voxel dimension) is $n=16$.

**`vertex0` Layout (32-bit unsigned int):**
*   Bits 0–9: X coordinate (10 bits, range 0–1023).
*   Bits 10–19: Y coordinate (10 bits, range 0–1023).
*   Bits 20–29: Z coordinate (10 bits, range 0–1023).
*   Bits 30–31: Unused.

**`vertex1` Layout (32-bit unsigned int):**
*   Bits 0–9: Texture layer index (10 bits).
*   Bits 10–14: Texture coordinate U (5 bits, range 0–31).
*   Bits 15–19: Texture coordinate V (5 bits, range 0–31).
*   Bits 20–21: Lighting/Shading value (2 bits, range 0–3).
*   Bits 22–31: Unused.

**Other Arrays:**
*   `indices`: Array of integers storing triangle indices (6 indices per quad).
*   `vertexSlots`: Stack of available vertex indices for reuse (free list).
*   `update`: Boolean flag indicating buffers need upload.

### Numbered Steps

1.  **Face Visibility Determination**:
    For a block change at $(x, y, z)$, iterate over the 6 neighbors (directions $dir \in [0, 5]$). Directions map to normals: 0: +X, 1: -X, 2: +Y, 3: -Y, 4: +Z, 5: -Z.
    Determine if a face needs to be added or removed based on the neighbor's state:
    *   **Adding a block**: Add face if neighbor is invisible/translucent/transparent AND block types differ. Remove face if neighbor is NOT invisible AND (current block is NOT translucent/transparent OR block types are identical).
    *   **Removing a block**: Reverse the logic.
    *   If a face is added to the current chunk, call `addCubeFace`. If added to a neighbor chunk, load the neighbor's mesh if necessary and call `addCubeFace` on it. The neighbor's face direction is $dir + (dir \% 2 === 0 ? 1 : -1)$.

2.  **Vertex Packing and Addition**:
    When `addCubeFace(x, y, z, dir, texture)` is called:
    a.  Calculate vertex positions using `cubeFacePosition(x, y, z, dir)`. This returns 12 floats (4 vertices $\times$ 3 coords).
    b.  Calculate texture coordinates using `cubeFaceTexcoord(dir)`. The orientation mapping is $[2, 3, 0, 0, 3, 2][dir]$. This returns 8 floats (4 vertices $\times$ 2 coords).
    c.  Determine lighting value $dLight$ using $[1, 1, 3, 0, 2, 2][dir]$.
    d.  Pop a reusable index from `vertexSlots`. If empty, allocate new indices.
    e.  For each of the 4 vertices, pack the data:
        $$ v0 = x | (y \ll 10) | (z \ll 20) $$
        $$ v1 = textureLayer | (texcoordX \ll 10) | (texcoordY \ll 15) | (dLight \ll 20) $$
        Store $v0$ in `vertex0` and $v1$ in `vertex1` at the allocated index.
    f.  Push the 6 triangle indices ($a, b, c, c, d, a$) to `indices`.

3.  **Face Removal**:
    When `removeFace(facePosition)` is called:
    a.  Search for the face in `vertex0` using `findFace`. This reconstructs the packed `vertex0` uints for the 4 vertices of the target face and compares them against the array, skipping `NaN` entries.
    b.  If found at index $i$, set `vertex0[i...i+3]` and `vertex1[i...i+3]` to `NaN`.
    c.  Remove the 6 corresponding indices from `indices` using `splice`.
    d.  Push $i$ to `vertexSlots` for reuse.
    e.  Set `update = true`.

4.  **Buffer Update**:
    When `update` is true, upload the arrays to GPU:
    a.  Bind `vertex0Buffer` and call `bufferData` with `new Uint32Array(vertex0)` and hint `gl.DYNAMIC_DRAW`.
    b.  Bind `vertex1Buffer` and call `bufferData` with `new Uint32Array(vertex1)` and hint `gl.DYNAMIC_DRAW`.
    c.  Bind `indexBuffer` and call `bufferData` with `new Uint32Array(indices)` and hint `gl.DYNAMIC_DRAW`.
    d.  Reset `update = false`.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
BLOCK_SIZE = 16
DIRECTION_LIGHTING = [1, 1, 3, 0, 2, 2]
DIRECTION_TEXCOORD_MAP = [2, 3, 0, 0, 3, 2]

class ChunkMesh:
    def __init__(self):
        self.vertex0 = []
        self.vertex1 = []
        self.indices = []
        self.vertexSlots = []
        self.update = False

    def add_vertex(self, x, y, z, texture_layer, tex_u, tex_v, light, index=None):
        # Pack vertex0
        v0 = x | (y << 10) | (z << 20)
        # Pack vertex1
        v1 = texture_layer | (tex_u << 10) | (tex_v << 15) | (light << 20)
        
        if index is not None and index < len(self.vertex0):
            self.vertex0[index] = v0
            self.vertex1[index] = v1
        else:
            if self.vertexSlots:
                index = self.vertexSlots.pop()
                # Ensure arrays are large enough
                while len(self.vertex0) <= index:
                    self.vertex0.append(0)
                    self.vertex1.append(0)
            else:
                index = len(self.vertex0)
                self.vertex0.append(v0)
                self.vertex1.append(v1)
        return index

    def add_cube_face(self, x, y, z, dir, texture_layer):
        positions = self.cube_face_position(x, y, z, dir)
        texcoords = self.cube_face_texcoord(dir)
        light = DIRECTION_LIGHTING[dir]
        
        # Get reusable index
        if self.vertexSlots:
            base_index = self.vertexSlots.pop()
        else:
            base_index = len(self.vertex0)
            # Expand arrays if necessary
            for _ in range(4):
                self.vertex0.append(0)
                self.vertex1.append(0)

        # Add 4 vertices
        v_a = self.add_vertex(positions[0], positions[1], positions[2], texture_layer, texcoords[0], texcoords[1], light, base_index)
        v_b = self.add_vertex(positions[3], positions[4], positions[5], texture_layer, texcoords[2], texcoords[3], light, base_index + 1)
        v_c = self.add_vertex(positions[6], positions[7], positions[8], texture_layer, texcoords[4], texcoords[5], light, base_index + 2)
        v_d = self.add_vertex(positions[9], positions[10], positions[11], texture_layer, texcoords[6], texcoords[7], light, base_index + 3)
        
        self.indices.extend([v_a, v_b, v_c, v_c, v_d, v_a])
        self.update = True

    def remove_face(self, face_positions):
        # face_positions is a list of 12 floats (4 vertices)
        idx = self.find_face(face_positions)
        if idx == -1:
            return
        
        # Mark as removed
        for i in range(4):
            self.vertex0[idx + i] = float('nan')
            self.vertex1[idx + i] = float('nan')
            
        # Remove indices
        # Find start of this face in indices array
        start_idx = self.find_index(idx)
        if start_idx != -1:
            self.indices[start_idx : start_idx + 6] = []
            
        self.vertexSlots.append(idx)
        self.update = True

    def find_face(self, face_positions):
        # Reconstruct packed v0 for comparison
        target_v0s = []
        for i in range(0, 12, 3):
            x, y, z = face_positions[i], face_positions[i+1], face_positions[i+2]
            target_v0s.append(x | (y << 10) | (z << 20))
            
        for i in range(0, len(self.vertex0), 4):
            if self.vertex0[i] != self.vertex0[i]: # Check for NaN
                continue
            match = True
            for j in range(4):
                if self.vertex0[i + j] != target_v0s[j]:
                    match = False
                    break
            if match:
                return i
        return -1

    def find_index(self, vertex_idx):
        for i in range(0, len(self.indices), 6):
            if self.indices[i] == vertex_idx:
                return i
        return -1

    def cube_face_position(self, x, y, z, dir):
        n = BLOCK_SIZE
        x *= n; y *= n; z *= n
        if dir == 0: return [x+n, y, z, x+n, y+n, z, x+n, y+n, z+n, x+n, y, z+n]
        if dir == 1: return [x, y, z, x, y, z+n, x, y+n, z+n, x, y+n, z]
        if dir == 2: return [x, y+n, z, x, y+n, z+n, x+n, y+n, z+n, x+n, y+n, z]
        if dir == 3: return [x+n, y, z+n, x, y, z+n, x, y, z, x+n, y, z]
        if dir == 4: return [x, y, z+n, x+n, y, z+n, x+n, y+n, z+n, x, y+n, z+n]
        if dir == 5: return [x, y, z, x, y+n, z, x+n, y+n, z, x+n, y, z]
        return []

    def cube_face_texcoord(self, dir):
        # Simplified mapping based on source logic
        # Source: orientation mapping from direction dir is [2, 3, 0, 0, 3, 2][dir]
        # And specific returns for each orientation. 
        # For brevity, assuming a helper that returns 8 floats based on dir
        return self._get_texcoords_for_dir(dir)

    def _get_texcoords_for_dir(self, dir):
        # Placeholder for specific texture coordinate logic from source
        # Source provides specific arrays for cases 0-5
        if dir == 0: return [0, 0, 0, 16, 16, 16, 16, 0]
        if dir == 1: return [16, 0, 0, 0, 0, 16, 16, 16]
        if dir == 2: return [0, 0, 0, 16, 16, 16, 16, 0] # Example, source continues
        if dir == 3: return [16, 0, 0, 0, 0, 16, 16, 16] # Example
        if dir == 4: return [0, 0, 0, 16, 16, 16, 16, 0] # Example
        if dir == 5: return [16, 0, 0, 0, 0, 16, 16, 16] # Example
        return [0,0,0,0,0,0,0,0]

    def update_buffers(self, gl):
        gl.bind_buffer(gl.ARRAY_BUFFER, self.vertex0_buffer)
        gl.buffer_data(gl.ARRAY_BUFFER, Uint32Array(self.vertex0), gl.DYNAMIC_DRAW)
        gl.bind_buffer(gl.ARRAY_BUFFER, self.vertex1_buffer)
        gl.buffer_data(gl.ARRAY_BUFFER, Uint32Array(self.vertex1), gl.DYNAMIC_DRAW)
        gl.bind_buffer(gl.ELEMENT_ARRAY_BUFFER, self.index_buffer)
        gl.buffer_data(gl.ELEMENT_ARRAY_BUFFER, Uint32Array(self.indices), gl.DYNAMIC_DRAW)
        self.update = False
```

### Parameters and Edge Cases
*   **Block Size**: Fixed at 16 units. Coordinates are scaled by this factor before packing.
*   **Precision**: Coordinates are limited to 10 bits (0–1023). This implies a chunk size of $1023/16 \approx 63$ voxels, or potentially larger if coordinates are relative. The source uses 10 bits for X, Y, Z, allowing positions up to 1023.
*   **NaN Handling**: Removed vertices are marked with `NaN` in `vertex0` to allow `find_face` to skip them efficiently.
*   **Neighbor Loading**: When adding a face to a neighbor chunk, the neighbor's mesh must be loaded if it does not exist [S1, unofficial].

## Subtopics

### [Voxel Face Meshing with Packed Attributes](4153-voxel-face-meshing-with-packed-attributes.md)
Focuses on the shader-side unpacking of the packed attributes generated by this method.

### [Mesh Reindexing and Vertex Compaction](4160-mesh-reindexing-and-vertex-compaction.md)
A more aggressive compaction method that removes unused vertices entirely, unlike the slot-reuse approach here.

### [Chunk Mesh Data Structure & Buffer Update](4182-chunk-mesh-data-structure-buffer-update.md)
Uses a MegaSSBO and free-list allocator for centralized memory management, differing from the per-chunk array approach here.

### [Alpha/Translucent Block Rendering](4860-alpha-translucent-block-rendering.md)
Extends this meshing with depth-based sorting and fragment-shader transparency.

## Operating characteristics
*   **Time Complexity**: Face addition/removal is $O(1)$ for packing, but `find_face` is $O(V)$ where $V$ is the number of vertices in the chunk, as it scans the array. Buffer upload is $O(V + I)$ where $I$ is the number of indices.
*   **Memory Complexity**: $O(V)$ for vertex arrays and $O(I)$ for indices. The packing reduces memory bandwidth by 50% compared to storing 4 floats per vertex.
*   **Strengths**: Efficient reuse of vertex slots reduces fragmentation. Packed attributes reduce GPU memory usage.
*   **Limitations**: `find_face` linear scan is slow for large chunks. The 10-bit coordinate limit restricts chunk size or coordinate range.
*   **When to use**: Suitable for dynamic voxel worlds with frequent block updates where buffer upload overhead is a bottleneck.

## Background
This method originates from the Exolands project, an unofficial voxel engine implementation. It was designed to handle dynamic terrain updates efficiently by combining face culling with compact vertex storage.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
