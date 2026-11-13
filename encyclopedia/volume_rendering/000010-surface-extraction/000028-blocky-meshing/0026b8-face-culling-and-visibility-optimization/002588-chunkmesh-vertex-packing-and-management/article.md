# ChunkMesh Vertex Packing and Management

Also called: Block Face Culling, Cube Face Mesh Generation

## Overview
ChunkMesh Vertex Packing and Management is a dynamic mesh generation and buffer management technique for voxel terrain, designed to minimize GPU memory bandwidth and draw calls by tightly packing vertex attributes into integer buffers and efficiently handling incremental mesh updates [S1, unofficial]. It solves the problem of sparse, high-overhead vertex buffers in blocky meshing by encoding position, texture, lighting, and texture coordinates into two 32-bit integers per vertex, while maintaining a free-list of reusable vertex indices to avoid reallocation [S1, unofficial]. It sets itself apart from static meshing methods by supporting incremental add/remove operations with neighbor-aware face culling, and from pure GPU-based culling by performing visibility logic and buffer compaction on the CPU side before uploading to the GPU [S1, unofficial].

## How it works
**Inputs:** Voxel world data (block types, chunk positions, visibility flags), camera position and orientation (for translucent sorting), and texture atlas mappings.
**Outputs:** GPU vertex buffers (`vertex0`, `vertex1`) and index buffer (`indices`) representing the visible surface of a chunk.

**Data Structures:**
The mesh data is stored in three JavaScript arrays (or equivalent typed arrays in implementation):
1.  `vertex0`: `Uint32Array` storing packed position data.
2.  `vertex1`: `Uint32Array` storing packed attribute data.
3.  `indices`: `Uint32Array` storing triangle indices.
4.  `vertexSlots`: Stack of reusable vertex indices (free list).

**Packed Layout:**
Each vertex uses two 32-bit unsigned integers.

*   `vertex0` (Position):
    *   Bits 0-9: X coordinate (10 bits, range 0-1023).
    *   Bits 10-19: Y coordinate (10 bits, range 0-1023).
    *   Bits 20-29: Z coordinate (10 bits, range 0-1023).
    *   Bits 30-31: Unused.
*   `vertex1` (Attributes):
    *   Bits 0-9: Texture layer (10 bits).
    *   Bits 10-14: Texture Coordinate X (5 bits, range 0-31).
    *   Bits 15-19: Texture Coordinate Y (5 bits, range 0-31).
    *   Bits 20-21: Shading/Lighting value (2 bits, range 0-3).
    *   Bits 22-31: Unused.

**Parameters:**
*   Block size ($n$): 16 units.
*   Direction mapping: 0=+X, 1=-X, 2=+Y, 3=-Y, 4=+Z, 5=-Z.
*   Lighting values: Array `[1, 1, 3, 0, 2, 2]` corresponding to directions 0-5.
*   Texture coordinate scaling: 0.0625 (1/16).
*   Shading formula: $shade = value \times 0.2 + 0.4$.

**Algorithm Steps:**

1.  **Face Generation:**
    For a block at chunk coordinates $(x, y, z)$ and direction $dir$:
    *   Calculate world positions for 4 vertices using `cubeFacePosition(x, y, z, dir)`.
    *   Calculate texture coordinates using `cubeFaceTexcoord(orientation)`, where $orientation = [2, 3, 0, 0, 3, 2][dir]$.
    *   Determine lighting value $dLight$ from the lighting array based on $dir$.

2.  **Vertex Addition (`addVertex`):**
    *   If a reusable index is available (`vertexSlots` not empty), pop it as `vertexIndex`. Otherwise, assign a new index equal to `vertex0.length`.
    *   Pack position into `vertex0`:
        $$ v0 = x \mid (y \ll 10) \mid (z \ll 20) $$
    *   Pack attributes into `vertex1`:
        $$ v1 = textureLayer \mid (texcoordX \ll 10) \mid (texcoordY \ll 15) \mid (dLight \ll 20) $$
    *   Store $v0$ and $v1$ at `vertex0[vertexIndex]` and `vertex1[vertexIndex]`.
    *   Set `this.update = true`.

3.  **Face Addition (`addCubeFace`):**
    *   Get 4 vertex positions and 4 texture coordinate pairs.
    *   Call `addVertex` for each of the 4 vertices (a, b, c, d).
    *   Push indices to `indices`: `[a, b, c, c, d, a]`.

4.  **Face Removal (`removeFace`):**
    *   Search for the face in `vertex0` using `findFace(facePosition)`.
    *   `findFace` iterates `vertex0` in steps of 4. It reconstructs the packed `vertex0` uint for each of the 4 vertices of the target face and compares them against stored values. It skips entries where `vertex0[i]` is NaN.
    *   If found at `vertexIndex`:
        *   Set `vertex0[i...i+3]` and `vertex1[i...i+3]` to `NaN`.
        *   Remove the 6 corresponding indices from `indices` using `splice`.
        *   Push `vertexIndex` to `vertexSlots` for reuse.
        *   Set `this.update = true`.

5.  **Visibility Logic (`faceInteraction`):**
    Determine if a face between `blockData0` (current) and `blockData1` (neighbor) should be added or removed based on `add` (true if adding block, false if removing).
    *   **Adding Face (`add=true`):**
        *   Add if: Neighbor is invisible/translucent/transparent AND block types differ.
        *   Remove if: Neighbor is NOT invisible AND (Current is NOT translucent/transparent OR block types are identical).
    *   **Removing Face (`add=false`):**
        *   Reverse the logic: Remove if neighbor is invisible/translucent/transparent AND types differ; Add if neighbor is NOT invisible AND (Current is NOT translucent/transparent OR types are identical).

6.  **Chunk Update (`updateBlock`):**
    *   Iterate over 6 directions ($i=0$ to $5$).
    *   Calculate neighbor coordinates $(tx, ty, tz)$.
    *   Get neighbor block data and info.
    *   Call `faceInteraction`.
    *   If `addFace` is true:
        *   If adding block: Add face to current chunk.
        *   If removing block: Add face to neighbor chunk (load neighbor mesh if needed).
    *   If `removeFace` is true:
        *   If adding block: Remove face from neighbor chunk.
        *   If removing block: Remove face from current chunk.
    *   Neighbor direction mapping: $dir_{neighbor} = i + (i \% 2 === 0 ? 1 : -1)$.

7.  **Buffer Upload:**
    *   Bind VAO.
    *   Upload `vertex0`, `vertex1`, and `indices` to GPU buffers using `gl.DYNAMIC_DRAW`.
    *   Assign attributes `vertex0` and `vertex1` as integer attributes with stride 0.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class ChunkMesh:
    def __init__(self):
        self.vertex0 = []
        self.vertex1 = []
        self.indices = []
        self.vertexSlots = []
        self.update = False
        self.n = 16
        self.lighting = [1, 1, 3, 0, 2, 2]

    def add_vertex(self, x, y, z, texture_layer, texcoord_x, texcoord_y, d_light, vertex_index=-1):
        if vertex_index < 0:
            if self.vertexSlots:
                vertex_index = self.vertexSlots.pop()
            else:
                vertex_index = len(self.vertex0)
        
        # Pack vertex0
        v0 = x | (y << 10) | (z << 20)
        # Pack vertex1
        v1 = texture_layer | (texcoord_x << 10) | (texcoord_y << 15) | (d_light << 20)
        
        # Ensure arrays are large enough
        while len(self.vertex0) <= vertex_index:
            self.vertex0.append(0)
            self.vertex1.append(0)
            
        self.vertex0[vertex_index] = v0
        self.vertex1[vertex_index] = v1
        self.update = True
        return vertex_index

    def cube_face_position(self, x, y, z, dir):
        x *= self.n
        y *= self.n
        z *= self.n
        # Return 12 floats: 4 vertices * 3 coords
        if dir == 0: return [x+self.n, y, z, x+self.n, y+self.n, z, x+self.n, y+self.n, z+self.n, x+self.n, y, z+self.n]
        if dir == 1: return [x, y, z, x, y, z+self.n, x, y+self.n, z+self.n, x, y+self.n, z]
        if dir == 2: return [x, y+self.n, z, x, y+self.n, z+self.n, x+self.n, y+self.n, z+self.n, x+self.n, y+self.n, z]
        if dir == 3: return [x+self.n, y, z+self.n, x, y, z+self.n, x, y, z, x+self.n, y, z]
        if dir == 4: return [x, y, z+self.n, x+self.n, y, z+self.n, x+self.n, y+self.n, z+self.n, x, y+self.n, z+self.n]
        if dir == 5: return [x, y, z, x, y+self.n, z, x+self.n, y+self.n, z, x+self.n, y, z]

    def cube_face_texcoord(self, orientation):
        # Returns 8 floats: 4 vertices * 2 coords
        if orientation == 0: return [0, 0, 0, 16, 16, 16, 16, 0]
        if orientation == 1: return [16, 0, 0, 0, 0, 16, 16, 16]
        # Other orientations handled similarly based on source [S1, unofficial]
        return [0, 0, 0, 16, 16, 16, 16, 0]

    def add_cube_face(self, x, y, z, dir, texture):
        pos = self.cube_face_position(x, y, z, dir)
        tex = self.cube_face_texcoord([2, 3, 0, 0, 3, 2][dir])
        d_light = self.lighting[dir]
        
        a = self.add_vertex(pos[0], pos[1], pos[2], texture, tex[0], tex[1], d_light)
        b = self.add_vertex(pos[3], pos[4], pos[5], texture, tex[2], tex[3], d_light)
        c = self.add_vertex(pos[6], pos[7], pos[8], texture, tex[4], tex[5], d_light)
        d = self.add_vertex(pos[9], pos[10], pos[11], texture, tex[6], tex[7], d_light)
        
        self.indices.extend([a, b, c, c, d, a])

    def find_face(self, face_position):
        # face_position is 12 floats
        for i in range(0, len(self.vertex0), 4):
            if self.vertex0[i] == self.vertex0[i]: # Skip NaN check logic: if isNaN(vertex0[i]) continue
                # Reconstruct packed v0 for comparison
                v0_0 = face_position[0] | (face_position[1] << 10) | (face_position[2] << 20)
                v0_1 = face_position[3] | (face_position[4] << 10) | (face_position[5] << 20)
                v0_2 = face_position[6] | (face_position[7] << 10) | (face_position[8] << 20)
                v0_3 = face_position[9] | (face_position[10] << 10) | (face_position[11] << 20)
                
                if self.vertex0[i] == v0_0 and self.vertex0[i+1] == v0_1 and self.vertex0[i+2] == v0_2 and self.vertex0[i+3] == v0_3:
                    return i
        return -1

    def remove_face(self, face_position):
        vertex_index = self.find_face(face_position)
        if vertex_index == -1: return
        
        # Mark as NaN (using a special value or flag, here we use None/NaN concept)
        # In JS, NaN is a float, but arrays are Uint32. Source uses NaN in Uint32Array which becomes 0 or specific bit pattern?
        # Source says: this.vertex0[vertexIndex] = ... = NaN. 
        # In JS, assigning NaN to Uint32Array results in 0. 
        # However, findFace checks isNaN(vertex0[i]). 
        # This implies vertex0 might be a standard Array or Float32Array in the source, 
        # but layout section says Uint32Array. 
        # We assume a mechanism to mark invalid vertices exists, e.g., a separate boolean array or using a specific invalid uint.
        # For implementation fidelity to source logic:
        self.vertex0[vertex_index] = float('nan') # Conceptual mark
        self.vertex0[vertex_index+1] = float('nan')
        self.vertex0[vertex_index+2] = float('nan')
        self.vertex0[vertex_index+3] = float('nan')
        
        # Remove indices
        # Find index in indices array where indices[i] == vertex_index
        idx_in_indices = -1
        for i in range(0, len(self.indices), 6):
            if self.indices[i] == vertex_index:
                idx_in_indices = i
                break
        
        if idx_in_indices != -1:
            self.indices[idx_in_indices:idx_in_indices+6] = []
            
        self.vertexSlots.append(vertex_index)
        self.update = True

    def update_buffers(self):
        # Upload to GPU
        pass
```

## Subtopics

### [Voxel Face Meshing with Packed Attributes](4153-voxel-face-meshing-with-packed-attributes.md)
Optimizes storage layout by packing vertex data into a single float attribute, unpacked in the vertex shader. Choose when GPU shader complexity is acceptable and buffer overhead is the primary concern.

### [Mesh Reindexing and Vertex Compaction](4160-mesh-reindexing-and-vertex-compaction.md)
Compacts vertex buffers by removing unreferenced vertices and renumbering them. Choose when vertex buffer sparsity is high and memory bandwidth is critical.

### [Chunk Mesh Data Structure & Buffer Update](4182-chunk-mesh-data-structure-buffer-update.md)
Manages GPU buffers using a large SSBO and free-list allocator. Choose for dynamic scenes requiring incremental updates without frequent reallocation.

### [Alpha/Translucent Block Rendering](4860-alpha-translucent-block-rendering.md)
Renders semi-transparent blocks using pre-computed attributes and fragment shading. Choose when transparency is required without ray-marching.

## Operating characteristics
**Time Complexity:**
*   Face Addition/Removal: $O(1)$ for vertex packing, $O(N)$ for face search (`findFace`) where $N$ is the number of vertices, though optimized with free-lists for reuse.
*   Buffer Upload: $O(V + I)$ where $V$ is vertex count and $I$ is index count.

**Memory Complexity:**
*   8 bytes per vertex (two 32-bit integers).
*   4 bytes per index.
*   Free-list overhead is minimal.

**Performance:**
*   Reduces vertex buffer size significantly compared to standard float32 positions.
*   Dynamic draw hint (`gl.DYNAMIC_DRAW`) allows GPU to optimize for frequent updates.
*   CPU-side culling reduces triangle count sent to GPU.

**Strengths:**
*   Low memory footprint.
*   Efficient incremental updates.
*   Simple implementation.

**Limitations:**
*   Limited precision (10-bit coordinates).
*   CPU-bound visibility logic.
*   Not suitable for highly dynamic, non-voxel geometry.

## Background
Originating from the Exolands project [S1, unofficial], this method was developed to handle efficient voxel terrain rendering in web-based environments. It builds on basic face culling principles [Face Culling](8735-face-culling.md) and extends them with compact data structures and dynamic buffer management. It is commonly used in voxel games and simulations where memory efficiency and update frequency are critical.

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
