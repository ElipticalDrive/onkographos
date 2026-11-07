# Greedy Meshing / Face Culling

Also called: Face culling meshing, Face-based Meshing, Block Mesh Generation, Voxel Face Generation, Voxel face extraction, Block Surface Extraction, Voxel Surface Extraction

## Overview
Greedy Meshing / Face Culling is a surface extraction technique that generates a static triangle mesh for a voxel chunk by iterating through occupied voxel positions and emitting geometry only for faces that are adjacent to empty space or transparent blocks [S1, S2, S3, unofficial]. It differs from its parent method, [Blocky meshing](31-blocky-meshing.md), by explicitly handling cross-chunk boundary visibility checks to ensure global consistency, and it differs from [Greedy Meshing](52-greedy-meshing.md) by not merging adjacent coplanar faces into larger quads [S1, S2, unofficial]. This approach reduces vertex count and draw calls compared to naive voxel rendering by excluding internal faces, while maintaining a simpler implementation than greedy approaches [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D boolean array or occupancy grid $V$ representing a chunk of voxels, where $V[x][y][z] = 1$ indicates a solid block and $0$ indicates empty space. The grid dimensions are typically fixed (e.g., $16^3$, $32^3$, or $512^3$) [S1, S2, S6, unofficial].
*   **Output:** A vertex buffer and an index buffer containing triangles for all exposed faces. The vertex format typically includes position (3 floats), normal (3 floats), and optionally color/UV data [S6, unofficial].

**Data Structures:**
*   **Voxel Grid:** A 3D array `filled` or `blockMap` of size $W \times H \times D$ [S2, S6, unofficial]. Some implementations use a flattened array or bitmasks for memory efficiency [S1, unofficial].
*   **Bounding Box:** An array `BIndex` of 6 integers `(minx, maxx, miny, maxy, minz, maxz)` used to limit the iteration range to non-empty regions [S2, unofficial].
*   **Face Definitions:**
    *   `face_normals[6][3]`: Unit vectors for the 6 directions: $\pm \hat{x}, \pm \hat{y}, \pm \hat{z}$ [S3, unofficial].
    *   `face_vertices[6][4][3]`: Vertex offsets for the 4 corners of each face relative to the voxel center [S3, unofficial].
    *   `VOXEL_FACE_VERTEX_INDICES`: A mapping from face direction to two triangles (6 indices) forming a quad. Example for LEFT face: `{{3,7,4}, {3,4,0}}` [S5, unofficial].
    *   `VOXEL_VERTEX_LOCATIONS`: 8 vertices of a unit cube centered at origin: $\{-0.5,-0.5,0.5\}, \{0.5,-0.5,0.5\}, \{0.5,-0.5,-0.5\}, \{-0.5,-0.5,-0.5\}, \{-0.5,0.5,0.5\}, \{0.5,0.5,0.5\}, \{0.5,0.5,-0.5\}, \{-0.5,0.5,-0.5\}$ [S5, unofficial].

**Algorithm Steps:**

1.  **Boundary Padding (Optional but Recommended):**
    To simplify neighbor checks, pad the voxel grid with a 1-voxel border of empty space. If the logical chunk size is $N$, allocate a grid of size $(N+2) \times (N+2) \times (N+2)$ and initialize all values to `false` (empty) [S6, unofficial].

2.  **Bounding Box Calculation:**
    Iterate through the voxel grid to find the minimum and maximum indices of occupied voxels. This defines the AABB for mesh generation, skipping empty space [S2, unofficial].
    $$ BIndex = [\min(x), \max(x), \min(y), \max(y), \min(z), \max(z)] $$

3.  **Face Culling Loop:**
    Iterate through every voxel $(x, y, z)$ within the bounding box (or the full chunk if no bounding box is used).
    For each voxel, check if it is occupied. If empty, skip to the next voxel [S3, S6, unofficial].

4.  **Neighbor Check:**
    For each of the 6 faces of the current voxel, determine if the adjacent voxel is empty.
    Let $N_x, N_y, N_z$ be the neighbor coordinates:
    $$ N_x = x + \text{normal}_x, \quad N_y = y + \text{normal}_y, \quad N_z = z + \text{normal}_z $$
    The face is **exposed** if:
    *   The neighbor coordinate is outside the grid bounds (treated as empty) [S1, S6, unofficial].
    *   The neighbor voxel $V[N_x][N_y][N_z]$ is `false` (empty) [S3, S6, unofficial].

    *Note on Bitmask Implementation:* Some implementations store a column of voxels as a bitmask. Checking neighbor $z+1$ involves shifting the bitmask: `(active[x][y] >> (z+1)) & 0x01` [S1, unofficial].

5.  **Geometry Generation:**
    If a face is exposed, generate the geometry for that face.
    *   **Position:** The face vertices are calculated based on the voxel position $(x,y,z)$ and the face normal. For a voxel at integer coordinates, the face lies on the plane defined by the normal.
        Example for Bottom face ($-\hat{y}$): Vertices are at $y = z_{voxel}$ (or $z_{voxel}-1$ depending on coordinate system origin).
        $$ P = (x \cdot S, y \cdot S, z \cdot S) + \text{offset} $$
        where $S$ is the block size [S1, unofficial].
    *   **Normal:** The face normal is the principal axis direction (e.g., $(0, -1, 0)$ for bottom) [S5, unofficial].
    *   **Indices:** Use the pre-defined index array for the quad (e.g., `0, 1, 2, 2, 3, 0`) [S6, unofficial].
    *   **Attributes:** Append position, normal, and color/UV to the vertex buffer. Append indices to the index buffer, adjusting for the current vertex offset [S3, S6, unofficial].

    *Ambient Occlusion (AO) Winding:* If AO is used, the winding order of the triangles may need to be flipped based on AO values to ensure correct shading.
    $$ \text{flip} = (ao[0] + ao[2]) > (ao[1] + ao[3]) $$
    If flip is true, emit triangles $(start, start+1, start+3)$ and $(start+1, start+2, start+3)$. Else, emit $(start, start+1, start+2)$ and $(start+2, start+3, start)$ [S3, unofficial].

6.  **Buffer Assembly:**
    Create the final Vertex Buffer Object (VBO) and Index Buffer Object (IBO).
    *   VBO Layout: Interleaved floats. Stride is $6 \times \text{sizeof(float)}$ (3 for position, 3 for normal) [S6, unofficial].
    *   IBO Layout: Unsigned integers, 0-based relative to the quad start [S6, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_face_culled_mesh(voxel_grid, chunk_size, block_size):
    """
    Generates a mesh for a voxel chunk using face culling.
    
    Args:
        voxel_grid: 3D array (or padded 3D array) of booleans.
        chunk_size: Integer, dimension of the logical chunk (e.g., 16).
        block_size: Float, size of one voxel.
        
    Returns:
        vertices: List of tuples (x, y, z, nx, ny, nz)
        indices: List of integers
    """
    vertices = []
    indices = []
    vertex_offset = 0
    
    # Define the 6 face normals and their corresponding vertex indices for a quad
    # Normals: Right(1,0,0), Left(-1,0,0), Top(0,1,0), Bottom(0,-1,0), Front(0,0,1), Back(0,0,-1)
    face_normals = [
        (1, 0, 0), (-1, 0, 0), (0, 1, 0), (0, -1, 0), (0, 0, 1), (0, 0, -1)
    ]
    
    # Indices for a quad (2 triangles)
    quad_indices = [0, 1, 2, 2, 3, 0]
    
    # Iterate over the chunk bounds
    # Note: If using padded grid, adjust indices accordingly
    for x in range(chunk_size):
        for y in range(chunk_size):
            for z in range(chunk_size):
                # Skip empty voxels
                if not voxel_grid[x][y][z]:
                    continue
                
                # Check all 6 faces
                for nx, ny, nz in face_normals:
                    # Calculate neighbor coordinates
                    nx_pos = x + nx
                    ny_pos = y + ny
                    nz_pos = z + nz
                    
                    # Check if neighbor is empty or out of bounds
                    is_exposed = False
                    
                    if 0 <= nx_pos < chunk_size and 0 <= ny_pos < chunk_size and 0 <= nz_pos < chunk_size:
                        if not voxel_grid[nx_pos][ny_pos][nz_pos]:
                            is_exposed = True
                    else:
                        # Out of bounds is considered empty
                        is_exposed = True
                    
                    if is_exposed:
                        # Generate 4 vertices for this face
                        # Base position of the voxel
                        base_x = x * block_size
                        base_y = y * block_size
                        base_z = z * block_size
                        
                        # Vertex offsets for a unit cube centered at 0, scaled by block_size
                        # Vertices: 0:(-1,-1,1), 1:(1,-1,1), 2:(1,-1,-1), 3:(-1,-1,-1) ... 
                        # This mapping depends on the specific face normal. 
                        # Simplified: We assume a standard quad generation based on normal.
                        
                        # For implementation clarity, we define the 4 corners relative to the face center
                        # Face Center: (base_x + 0.5*block_size, ...) adjusted by normal
                        # However, standard approach uses voxel corner coordinates.
                        
                        # Let's use the standard 8 corners of the voxel and pick the 4 relevant ones.
                        # Corners:
                        # 0: (-0.5, -0.5, 0.5)
                        # 1: (0.5, -0.5, 0.5)
                        # 2: (0.5, -0.5, -0.5)
                        # 3: (-0.5, -0.5, -0.5)
                        # 4: (-0.5, 0.5, 0.5)
                        # 5: (0.5, 0.5, 0.5)
                        # 6: (0.5, 0.5, -0.5)
                        # 7: (-0.5, 0.5, -0.5)
                        
                        # This part requires mapping the specific face normal to the correct 4 vertices.
                        # For brevity, we assume a helper function `get_face_vertices(normal, base_pos)` exists
                        # which returns the 4 world-space positions.
                        
                        face_verts = get_face_vertices((nx, ny, nz), (base_x, base_y, base_z), block_size)
                        
                        for vx, vy, vz in face_verts:
                            vertices.append((vx, vy, vz, nx, ny, nz))
                        
                        # Add indices
                        for idx in quad_indices:
                            indices.append(vertex_offset + idx)
                        
                        vertex_offset += 4
                        
    return vertices, indices

def get_face_vertices(normal, base_pos, block_size):
    """
    Returns the 4 world-space vertices for a face with the given normal,
    attached to the voxel at base_pos.
    """
    bx, by, bz = base_pos
    s = block_size / 2.0
    
    # Offsets for the 8 corners of a voxel centered at (0,0,0) with size 1
    # 0:(-1,-1,1), 1:(1,-1,1), 2:(1,-1,-1), 3:(-1,-1,-1)
    # 4:(-1,1,1), 5:(1,1,1), 6:(1,1,-1), 7:(-1,1,-1)
    # Note: Y is up. Z is forward/back.
    
    # Mapping based on normal:
    # Right (1,0,0): Vertices 1, 5, 6, 2 (x=+1) -> Order matters for winding
    # Left (-1,0,0): Vertices 0, 4, 7, 3 (x=-1)
    # Top (0,1,0): Vertices 4, 5, 6, 7 (y=+1)
    # Bottom (0,-1,0): Vertices 0, 1, 2, 3 (y=-1)
    # Front (0,0,1): Vertices 0, 1, 5, 4 (z=+1)
    # Back (0,0,-1): Vertices 3, 2, 6, 7 (z=-1)
    
    # This is a simplified mapping; actual implementation must ensure consistent winding.
    # For the purpose of this article, we assume the correct 4 vertices are selected.
    
    # Example for Right Face (x=+1):
    # Vertices 1, 5, 6, 2
    # 1: (0.5, -0.5, 0.5)
    # 5: (0.5, 0.5, 0.5)
    # 6: (0.5, 0.5, -0.5)
    # 2: (0.5, -0.5, -0.5)
    
    # We return these scaled and translated.
    # Note: The exact order depends on the desired winding (CW/CCW) and normal direction.
    
    # Placeholder for the specific vertex selection logic which varies by implementation.
    # The key is that the 4 vertices form a quad on the plane defined by the normal.
    pass
```

## Subtopics

### [Visible Block Face Meshing](4919-visible-block-face-meshing.md)
Visible Block Face Meshing is a surface extraction technique that generates a static triangle mesh for a voxel chunk by iterating through occupied voxel positions and emitting geometry only for faces that are adjacent to empty space or transparent blocks. It differs from its parent method, [Voxel Chunk Mesh Generation (Face Culling)](4888-voxel-chunk-mesh-generation-face-culling.md), by explicitly handling cross-chunk boundary visibility checks to ensure global consistency, and it differs from [Greedy Meshing](52-greedy-meshing.md) by not merging adjacent coplanar faces into larger quads. This approach reduces vertex count and draw calls compared to naive voxel rendering by excluding internal faces, while maintaining a simpler implementation than greedy approaches.

### [Chunk Mesh Generation](8755-chunk-mesh-generation.md)
Not given in the sources.

### [Chunk Loading and Mesh Generation](8774-chunk-loading-and-mesh-generation.md)
Not given in the sources.

### Voxel Mesh Synthesis
A post-processing step that converts validated leaf voxels from the octree into standard 3D cube meshes for export [S7, unofficial].
1.  Generate 8 unique corner vertices for the cube, translated by the box's min and max coordinates [S7, unofficial].
2.  Construct 12 counter-clockwise triangular faces (2 triangles per cube face) ensuring outward-pointing surface normals [S7, unofficial].
3.  Output clean, standard Wavefront .obj data compatible with external 3D software [S7, unofficial].

### Chunk-based Voxel Rendering Pipeline
1.  Instantiate chunk: `chunk c = chunk(voxel_dim, glm::vec3(0, 0, 0), true)` [S2, unofficial].
2.  Fill chunk grid from `Grid` using triple nested loops over `dim` [S2, unofficial].
3.  Generate surface mesh by calling `c.Render()` [S2, unofficial].
4.  Extract counts: `vcnt = c.rendervert.size()`, `icnt = c.indices.size()` [S2, unofficial].
5.  Allocate heap memory: `cube_vertices = (GLfloat*)malloc(vcnt * sizeof(GLfloat))`, `cube_indices = (GLuint*)malloc(icnt * sizeof(GLuint))` [S2, unofficial].
6.  Copy data to arrays via loops over `vcnt` and `icnt` [S2, unofficial].
7.  Create VBO: `VertexBuffer vb(cube_vertices, cnt * numofbytespervertex * numofvertexperblock * sizeof(GLfloat))` [S2, unofficial].
8.  Define vertex layout: `layout.Push(GL_FLOAT, 4)` [S2, unofficial].
9.  Create IBO: `IndexBuffer ib(cube_indices, icnt)` [S2, unofficial].
10. Draw mesh: `glDrawElements(GL_TRIANGLES, cntblocks, GL_UNSIGNED_INT, nullptr)` [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of voxels in the chunk. Each voxel is visited once, and 6 neighbor checks are performed.
*   **Memory Complexity:** $O(F)$, where $F$ is the number of exposed faces. In the worst case (checkerboard pattern), $F \approx 3N$. In dense volumes, $F \approx 2N$.
*   **Performance:** Significantly faster than naive voxel rendering (which renders all $6N$ faces) and simpler to implement than greedy meshing. Parallelization is straightforward by splitting the chunk into slabs or tiles [S2, unofficial].
*   **Strengths:** Simple logic, easy to parallelize, low memory overhead compared to greedy meshing, handles arbitrary voxel distributions.
*   **Limitations:** Does not merge coplanar faces, resulting in higher vertex/face counts than greedy meshing. Not suitable for highly optimized real-time rendering of very large, dense worlds without further optimization (like greedy meshing or instancing).
*   **When to Use:** Small to medium chunks, dynamic worlds where voxels change frequently (greedy meshing is expensive to recompute), or when simplicity of implementation is prioritized.
*   **When Not to Use:** Large, static worlds where maximum draw call reduction is critical.

## Background
The method is a fundamental technique in voxel-based game engines, particularly popularized by games like *Minecraft*. It is often referred to as "face culling" or "block meshing." The specific implementation details vary across open-source projects, with notable examples including the WebGL voxel engine by Lallassu [S1, unofficial], the 3D-Mesh-Voxelizer by Talkative-Banana [S2, unofficial], and the HelloVoxel project by Linirby [S3, unofficial]. The technique is also used in Godot voxel terrain plugins [S5, unofficial] and educational OpenGL projects like Proccraft [S6, unofficial].

## Sources
[S1] Show HN: Basic voxel engine in WebGL/JavaScript with full source, https://github.com/Lallassu/vox2, fetched 2026-10-26 (unofficial)
[S2] 3D-Mesh-Voxelizer - GitHub, https://github.com/Talkative-Banana/3D-Mesh-Voxelizer, fetched 2026-10-26 (unofficial)
[S3] Linirby/HelloVoxel, https://github.com/Linirby/HelloVoxel, fetched 2026-10-27 (unofficial)
[S4] nnarain/VoxelEngine, https://github.com/nnarain/VoxelEngine, fetched 2026-10-27 (unofficial)
[S5] Ry645/voxel-terrain-plugin, https://github.com/Ry645/voxel-terrain-plugin, fetched 2026-10-27 (unofficial)
[S6] FredrikErikJohansson/Proccraft, https://github.com/FredrikErikJohansson/Proccraft, fetched 2026-10-27 (unofficial)
[S7] Joji0/3D-Object-Surface-Voxelizer, https://github.com/Joji0/3D-Object-Surface-Voxelizer, fetched 2026-10-27 (unofficial)
