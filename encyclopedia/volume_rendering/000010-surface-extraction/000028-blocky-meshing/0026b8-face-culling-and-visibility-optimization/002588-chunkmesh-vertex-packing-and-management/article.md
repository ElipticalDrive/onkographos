# ChunkMesh Vertex Packing and Management

Also called: Block Face Culling, Cube Face Mesh Generation

## Overview
ChunkMesh Vertex Packing and Management is a dynamic mesh generation and buffer management technique for voxel terrain that packs vertex attributes into dense 32-bit integer formats to minimize GPU memory bandwidth and storage overhead [S1, unofficial]. It solves the problem of sparse, high-latency vertex buffer updates in blocky meshing by using free-list allocators and bitwise packing, allowing incremental face addition and removal without reallocation [S1, unofficial]. It sets itself apart from static meshing by supporting runtime modification of the voxel world and from pure face culling by managing the underlying vertex data structure and GPU buffer synchronization [S1, unofficial].

## How it works
**Inputs:** Voxel world state (block types, chunk positions), camera position (for translucent sorting), and block modification events (add/remove).
**Outputs:** GPU vertex buffers (`vertex0`, `vertex1`) and index buffer (`indices`) representing the visible surface of a chunk.

**Data Structures:**
The mesh maintains three JavaScript arrays (mapped to GPU buffers):
1.  `vertex0`: `Uint32Array` storing packed position data.
2.  `vertex1`: `Uint32Array` storing packed texture, texcoord, and lighting data.
3.  `indices`: `Uint32Array` storing triangle indices.
4.  `vertexSlots`: Stack (free-list) storing available indices in `vertex0`/`vertex1` for reuse.

**Bit Packing Layout:**
Each vertex is represented by two 32-bit unsigned integers.

*   `vertex0` (Position):
    *   Bits 0–9: X coordinate (10 bits, range 0–1023).
    *   Bits 10–19: Y coordinate (10 bits, range 0–1023).
    *   Bits 20–29: Z coordinate (10 bits, range 0–1023).
    *   Bits 30–31: Unused.
*   `vertex1` (Attributes):
    *   Bits 0–9: Texture layer index (10 bits).
    *   Bits 10–14: Texture coordinate U (5 bits, range 0–31).
    *   Bits 15–19: Texture coordinate V (5 bits, range 0–31).
    *   Bits 20–21: Shading/Lighting value (2 bits, range 0–3).
    *   Bits 22–31: Unused.

**Vertex Generation:**
A chunk consists of blocks of size $n=16$. For a block at integer coordinates $(x, y, z)$ and face direction $d \in \{0, \dots, 5\}$ (0: +X, 1: -X, 2: +Y, 3: -Y, 4: +Z, 5: -Z), the four vertices of the quad are calculated. The direction mapping for lighting is $L = [1, 1, 3, 0, 2, 2][d]$ [S1, unofficial]. The texture coordinate orientation mapping is $O = [2, 3, 0, 0, 3, 2][d]$ [S1, unofficial].

**Face Interaction Logic:**
When a block is added or removed, the algorithm checks the 6 neighbors. A face is generated or culled based on the neighbor's properties:
*   **Add Face:** If the neighbor is invisible/translucent/transparent AND the current block type differs from the neighbor type.
*   **Remove Face:** If the neighbor is NOT invisible AND (the current block is NOT translucent/transparent OR the block types are identical).
*   **Invisible Blocks:** If the current block is invisible, no faces are generated for any direction [S1, unofficial].

**Buffer Update:**
The GPU buffers are updated using `gl.DYNAMIC_DRAW` hint. The vertex attributes are bound as integer attributes (`assignAttribI`) with stride 0 [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
BLOCK_SIZE = 16
DIRECTION_LIGHTING = [1, 1, 3, 0, 2, 2]
DIRECTION_TEXCOORD = [2, 3, 0, 0, 3, 2]

class ChunkMesh:
    def __init__(self):
        self.vertex0 = []  # Packed positions
        self.vertex1 = []  # Packed attributes
        self.indices = []  # Triangle indices
        self.vertexSlots = []  # Free list for vertex indices
        self.update = False

    def add_vertex(self, x, y, z, texture_layer, tex_u, tex_v, light, index_hint=-1):
        """
        Add a vertex to the mesh.
        Returns the index of the vertex in the vertex0/vertex1 arrays.
        """
        # Pack position into vertex0
        packed_pos = x | (y << 10) | (z << 20)
        
        # Pack attributes into vertex1
        # tex_u, tex_v are 0-15 (5 bits). Scale to 0-31 for packing if needed, 
        # but source uses direct bit shift. Assuming input is 0-15 range for 5 bits.
        # Source: (texcoordX << 10). texcoordX is 0-15.
        packed_attr = texture_layer | (tex_u << 10) | (tex_v << 15) | (light << 20)
        
        if index_hint >= 0:
            # Reuse existing slot
            self.vertex0[index_hint] = packed_pos
            self.vertex1[index_hint] = packed_attr
            return index_hint
        else:
            # Push new
            self.vertex0.append(packed_pos)
            self.vertex1.append(packed_attr)
            return len(self.vertex0) - 1

    def get_cube_face_vertices(self, x, y, z, dir):
        """
        Returns 4 vertices (each as [x,y,z]) for the face of the block at x,y,z in direction dir.
        Coordinates are in world space (block_coord * 16).
        """
        bx = x * BLOCK_SIZE
        by = y * BLOCK_SIZE
        bz = z * BLOCK_SIZE
        
        if dir == 0: # +X
            return [
                [bx + BLOCK_SIZE, by, bz],
                [bx + BLOCK_SIZE, by + BLOCK_SIZE, bz],
                [bx + BLOCK_SIZE, by + BLOCK_SIZE, bz + BLOCK_SIZE],
                [bx + BLOCK_SIZE, by, bz + BLOCK_SIZE]
            ]
        elif dir == 1: # -X
            return [
                [bx, by, bz],
                [bx, by, bz + BLOCK_SIZE],
                [bx, by + BLOCK_SIZE, bz + BLOCK_SIZE],
                [bx, by + BLOCK_SIZE, bz]
            ]
        elif dir == 2: # +Y
            return [
                [bx, by + BLOCK_SIZE, bz],
                [bx, by + BLOCK_SIZE, bz + BLOCK_SIZE],
                [bx + BLOCK_SIZE, by + BLOCK_SIZE, bz + BLOCK_SIZE],
                [bx + BLOCK_SIZE, by + BLOCK_SIZE, bz]
            ]
        elif dir == 3: # -Y
            return [
                [bx + BLOCK_SIZE, by, bz + BLOCK_SIZE],
                [bx, by, bz + BLOCK_SIZE],
                [bx, by, bz],
                [bx + BLOCK_SIZE, by, bz]
            ]
        elif dir == 4: # +Z
            return [
                [bx, by, bz + BLOCK_SIZE],
                [bx + BLOCK_SIZE, by, bz + BLOCK_SIZE],
                [bx + BLOCK_SIZE, by + BLOCK_SIZE, bz + BLOCK_SIZE],
                [bx, by + BLOCK_SIZE, bz + BLOCK_SIZE]
            ]
        elif dir == 5: # -Z
            return [
                [bx, by, bz],
                [bx, by + BLOCK_SIZE, bz],
                [bx + BLOCK_SIZE, by + BLOCK_SIZE, bz],
                [bx + BLOCK_SIZE, by, bz]
            ]

    def get_face_texcoords(self, dir):
        """
        Returns 4 texture coordinates (u, v) for the face.
        Mapping from source: case 0 -> [0,0, 0,16, 16,16, 16,0] etc.
        Assuming texcoords are 0-15 range for 5-bit packing.
        """
        # Source mapping: [2, 3, 0, 0, 3, 2][dir]
        # Case 0: [0, 0, 0, 16, 16, 16, 16, 0] -> (0,0), (0,1), (1,1), (1,0) scaled by 16?
        # Source code: float texcoordX = float((vertex1 & 31744u) >> 10u) * 0.0625f;
        # 0.0625 is 1/16. So packed value 16 -> 1.0. Packed value 0 -> 0.0.
        # So we pack 0-15.
        
        if dir == 0:
            return [(0,0), (0,15), (15,15), (15,0)]
        elif dir == 1:
            return [(15,0), (0,0), (0,15), (15,15)]
        elif dir == 2:
            return [(0,0), (0,15), (15,15), (15,0)] # Simplified based on source pattern, exact mapping depends on texture atlas
        elif dir == 3:
            return [(15,0), (0,0), (0,15), (15,15)]
        elif dir == 4:
            return [(0,0), (15,0), (15,15), (0,15)]
        elif dir == 5:
            return [(0,0), (0,15), (15,15), (15,0)]
        
        # Note: The source code snippet for texcoords was truncated. 
        # The logic relies on the orientation index [2,3,0,0,3,2][dir].
        # We assume standard UV mapping for the specific orientation.

    def add_cube_face(self, x, y, z, dir, texture_layer):
        """
        Adds a visible face to the mesh.
        """
        vertices = self.get_cube_face_vertices(x, y, z, dir)
        texcoords = self.get_face_texcoords(dir)
        light = DIRECTION_LIGHTING[dir]
        
        # Get reusable index
        if len(self.vertexSlots) > 0:
            start_index = self.vertexSlots.pop()
        else:
            start_index = len(self.vertex0)
            
        # Add 4 vertices
        idx_a = self.add_vertex(vertices[0][0], vertices[0][1], vertices[0][2], texture_layer, texcoords[0][0], texcoords[0][1], light, start_index)
        idx_b = self.add_vertex(vertices[1][0], vertices[1][1], vertices[1][2], texture_layer, texcoords[1][0], texcoords[1][1], light, start_index + 1)
        idx_c = self.add_vertex(vertices[2][0], vertices[2][1], vertices[2][2], texture_layer, texcoords[2][0], texcoords[2][1], light, start_index + 2)
        idx_d = self.add_vertex(vertices[3][0], vertices[3][1], vertices[3][2], texture_layer, texcoords[3][0], texcoords[3][1], light, start_index + 3)
        
        # Add indices (2 triangles: a,b,c and c,d,a)
        self.indices.extend([idx_a, idx_b, idx_c, idx_c, idx_d, idx_a])
        self.update = True

    def find_face(self, face_positions):
        """
        Finds the starting index of a face in vertex0 array.
        face_positions: list of 4 vertices [v0, v1, v2, v3] where each v is [x,y,z].
        Returns starting index or -1.
        """
        # Pack the expected positions
        expected = []
        for v in face_positions:
            expected.append(v[0] | (v[1] << 10) | (v[2] << 20))
            
        for i in range(0, len(self.vertex0), 4):
            if self.vertex0[i] == float('nan'): # Skip removed vertices
                continue
            if (self.vertex0[i] == expected[0] and
                self.vertex0[i+1] == expected[1] and
                self.vertex0[i+2] == expected[2] and
                self.vertex0[i+3] == expected[3]):
                return i
        return -1

    def remove_face(self, face_positions):
        """
        Removes a face from the mesh.
        """
        start_index = self.find_face(face_positions)
        if start_index == -1:
            return
            
        # Mark vertices as NaN (removed)
        self.vertex0[start_index] = float('nan')
        self.vertex0[start_index+1] = float('nan')
        self.vertex0[start_index+2] = float('nan')
        self.vertex0[start_index+3] = float('nan')
        
        self.vertex1[start_index] = float('nan')
        self.vertex1[start_index+1] = float('nan')
        self.vertex1[start_index+2] = float('nan')
        self.vertex1[start_index+3] = float('nan')
        
        # Remove indices
        # Find index in indices array where start_index is the first element
        # Source uses a linear scan for this
        idx_in_indices = -1
        for i in range(0, len(self.indices), 6):
            if self.indices[i] == start_index:
                idx_in_indices = i
                break
                
        if idx_in_indices != -1:
            self.indices[idx_in_indices : idx_in_indices+6] = [] # Splice 6 elements
            
        # Push to free list
        self.vertexSlots.append(start_index)
        self.update = True

    def update_block(self, x, y, z, is_adding, block_data):
        """
        Called when a block is added or removed.
        is_adding: True if block is being added, False if removed.
        block_data: Data of the block being added/removed.
        """
        for dir in range(6):
            # Calculate neighbor coordinates
            if dir == 0: tx, ty, tz = x + 1, y, z
            elif dir == 1: tx, ty, tz = x - 1, y, z
            elif dir == 2: tx, ty, tz = x, y + 1, z
            elif dir == 3: tx, ty, tz = x, y - 1, z
            elif dir == 4: tx, ty, tz = x, y, z + 1
            elif dir == 5: tx, ty, tz = x, y, z - 1
            
            # Get neighbor block info
            # Note: In a real implementation, this requires access to the world state
            neighbor_data = get_block_data(tx, ty, tz)
            if neighbor_data is None: continue
            
            neighbor_info = get_block_info(neighbor_data.type)
            current_info = get_block_info(block_data.type)
            
            # Determine if face needs to be added/removed
            add_face, remove_face = face_interaction(block_data, neighbor_data, is_adding, dir, current_info, neighbor_info)
            
            # Opposite direction for neighbor
            opp_dir = dir + 1 if dir % 2 == 0 else dir - 1
            
            if add_face:
                if is_adding:
                    # Add face to current chunk
                    self.add_cube_face(x, y, z, dir, get_texture_key(current_info.texture, dir))
                else:
                    # Remove face from neighbor chunk (exposed face)
                    # Load neighbor mesh if needed
                    if not chunk_mesh_exists(neighbor_data.chunk_pos):
                        load_chunk_mesh(neighbor_data.chunk_pos)
                    neighbor_mesh = get_chunk_mesh(neighbor_data.chunk_pos)
                    neighbor_mesh.add_cube_face(tx, ty, tz, opp_dir, get_texture_key(neighbor_info.texture, opp_dir))
                    
            if remove_face:
                if is_adding:
                    # Remove face from neighbor chunk (covered face)
                    if not chunk_mesh_exists(neighbor_data.chunk_pos):
                        load_chunk_mesh(neighbor_data.chunk_pos)
                    neighbor_mesh = get_chunk_mesh(neighbor_data.chunk_pos)
                    neighbor_mesh.remove_face(get_cube_face_vertices(tx, ty, tz, opp_dir))
                else:
                    # Remove face from current chunk (block removed, face gone)
                    self.remove_face(get_cube_face_vertices(x, y, z, dir))

    def update_buffers(self):
        """
        Uploads data to GPU.
        """
        if not self.update:
            return
            
        # Bind buffers and upload
        # gl.bindBuffer(gl.ARRAY_BUFFER, this.vertex0Buffer);
        # gl.bufferData(gl.ARRAY_BUFFER, new Uint32Array(this.vertex0), gl.DYNAMIC_DRAW);
        # ... repeat for vertex1 and indices
        
        self.update = False
```

**Edge Cases and Pitfalls:**
*   **Neighbor Chunk Loading:** When a block is removed, the neighbor chunk's mesh must be loaded if it does not already exist, before adding the newly exposed face to it [S1, unofficial].
*   **NaN Handling:** Removed vertices are marked with `NaN` in `vertex0` and `vertex1`. The `findFace` function must skip these entries.
*   **Index Search:** The `findFace` function performs a linear scan of `vertex0` in steps of 4. This is $O(V)$ where $V$ is the number of vertices. For large chunks, this may be slow. The source mentions a binary search helper `indexBinarySearch` but notes it is not used for face finding because indices are not sorted [S1, unofficial].
*   **Precision:** The 10-bit coordinate packing limits block positions to a $1024 \times 1024 \times 1024$ range per chunk. If blocks are larger than 16 units, the coordinate range shrinks. The source uses $n=16$, so coordinates are $0 \dots 15$ per block, fitting in 4 bits, but the packing uses 10 bits, allowing for larger chunks or offsets [S1, unofficial].

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
*   **Time Complexity:** Face addition/removal is $O(1)$ for buffer push/pop, but face removal requires $O(V)$ search to find the face in `vertex0` [S1, unofficial]. Buffer upload is $O(F)$ where $F$ is the number of faces.
*   **Memory Complexity:** $O(F)$ for vertices and indices. The packing reduces memory bandwidth by 50% compared to storing positions and attributes separately as floats [S1, unofficial].
*   **Performance:** The use of `Uint32Array` and `gl.DYNAMIC_DRAW` allows efficient incremental updates. The free-list allocator prevents memory fragmentation [S1, unofficial].
*   **Strengths:** Low memory footprint, fast incremental updates, simple implementation.
*   **Limitations:** Linear search for face removal is slow for large meshes. 10-bit coordinate limit restricts chunk size or block size. No support for complex geometry beyond quads.

## Background
The method originates from the Exolands project, an unofficial voxel engine implementation [S1, unofficial]. It is designed for real-time voxel terrain rendering where dynamic modification of the world is required. The technique builds on the principles of face culling and visibility optimization [3861-face-culling-and-visibility-optimization.md].

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
