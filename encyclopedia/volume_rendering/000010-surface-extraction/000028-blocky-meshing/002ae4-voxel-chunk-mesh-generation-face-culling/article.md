# Voxel Chunk Mesh Generation (Face Culling)

Also called: Block Mesh Generation, Marching Cubes-like Block Meshing, Static voxel meshing, Voxel Surface Extraction, Voxel face generation

## Overview
Voxel Chunk Mesh Generation (Face Culling) is a surface extraction technique that generates a static triangle mesh for a discrete 3D grid of voxels by iterating through occupied positions and emitting geometry only for faces adjacent to empty space or different materials [S1, S2, S3, unofficial]. It differs from its parent method, [Blocky meshing](31-blocky-meshing.md), by explicitly implementing the logic to cull internal faces shared between solid voxels, and it differs from [Greedy Meshing](1776-greedy-meshing.md) by not merging adjacent coplanar faces into larger quads [S1, S2, unofficial]. This approach reduces vertex count and draw calls compared to naive voxel rendering by excluding internal faces, while maintaining a simpler implementation than greedy approaches [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D array or hash map representing voxel occupancy. In the primary implementation [S2, unofficial], this is a boolean array `blockMap` of size $(size+2) \times (size+2) \times (size+2)$. In the Godot plugin implementation [S1, unofficial], this is a dictionary mapping `Vector3` positions to material data.
*   **Output:** A static triangle mesh consisting of vertex positions, normals, and indices. The mesh represents only the exposed surfaces of the voxel chunk.

**Data Structures:**
1.  **Voxel Grid:**
    *   `blockMap[x][y][z]`: Boolean indicating occupancy. Size is typically $N+2$ in each dimension to handle boundary checks easily [S2, unofficial].
    *   Alternatively, a sparse map `voxel_data` mapping `Vector3` to `Color` or material ID [S1, unofficial].
2.  **Vertex Buffer:**
    *   Interleaved array of floats. Each vertex contains 6 floats: 3 for position $(x, y, z)$ and 3 for normal $(n_x, n_y, n_z)$ [S2, unofficial].
    *   Stride: $6 \times \text{sizeof(float)}$ [S2, unofficial].
    *   Pre-allocated size: $size^3 \times 144$ floats for vertices and $size^3 \times 36$ unsigned ints for indices in the naive worst-case allocation [S2, unofficial].
3.  **Index Buffer:**
    *   Array of `unsigned int`. Indices are 0-based relative to the start of the quad's 4 vertices [S2, unofficial].
    *   Standard quad triangulation: `{0, 1, 2, 2, 3, 0}` [S2, unofficial].
4.  **Face Definitions:**
    *   **Normals:** Hardcoded per face direction [S2, unofficial]:
        *   LEFT: $(-1, 0, 0)$, RIGHT: $(1, 0, 0)$
        *   BOTTOM: $(0, -1, 0)$, TOP: $(0, 1, 0)$
        *   BACK: $(0, 0, -1)$, FRONT: $(0, 0, 1)$
    *   **Vertices (Unit Cube):** 8 vertices for a unit cube centered at origin, used as a template [S1, unofficial]:
        *   `{-0.5,-0.5,0.5}, {0.5,-0.5,0.5}, {0.5,-0.5,-0.5}, {-0.5,-0.5,-0.5}, {-0.5,0.5,0.5}, {0.5,0.5,0.5}, {0.5,0.5,-0.5}, {-0.5,0.5,-0.5}`
    *   **Indices per Face:** Two triangles per face. Example for LEFT face: `{{3,7,4}, {3,4,0}}` [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:**
    *   If using a dense array [S2, unofficial], initialize `blockMap` of size $(size+2)^3$ with `false`.
    *   If using a sparse map [S1, unofficial], check if input data is empty; if so, return early to prevent errors.
    *   Allocate vertex and index buffers.
2.  **Iteration:**
    *   Iterate through every voxel position $(x, y, z)$ in the grid.
    *   If using the dense array with padding [S2, unofficial]:
        *   Loop $x$ from $x_{pos}$ to $size + x_{pos} + 2$, $y$ from $y_{pos}$ to $size + y_{pos} + 2$, $z$ from $z_{pos}$ to $size + z_{pos} + 2$.
        *   Calculate local coordinates: $posX = x - x_{pos}$, $posY = y - y_{pos}$, $posZ = z - z_{pos}$.
        *   If $posX > 1$ and $posY > 1$ and $posZ > 1$, decrement indices by 1 to reference the actual block index in the logical grid.
        *   If `blockMap[posX][posY][posZ]` is `false`, skip this voxel.
    *   If using the sparse map [S1, unofficial]:
        *   Iterate through keys in `voxel_data`.
3.  **Face Culling and Emission:**
    *   For each occupied voxel, check its 6 neighbors (X-, X+, Y-, Y+, Z-, Z+).
    *   For each neighbor direction (Face):
        *   Determine the neighbor's position: $neighbor\_pos = voxel\_pos + VOXEL\_FACE\_NORMALS[face]$.
        *   Check if the neighbor is empty (or different material).
            *   In dense array [S2, unofficial]: Check if `blockMap` at neighbor coordinates is `false`.
            *   In sparse map [S1, unofficial]: Check if `voxel_data` does not contain `neighbor_pos`.
        *   If the neighbor is empty (face is exposed):
            *   Generate the 4 vertices for this face.
            *   Position: Template vertex + voxel center position.
            *   Normal: Hardcoded normal for the face direction.
            *   Append 6 floats (3 pos, 3 norm) to `chunkVertices` at offset $24 \times counter$.
            *   Append 6 indices to `chunkIndices` at offset $6 \times counter$, adding $4 \times counter$ to each index value to maintain relative addressing.
            *   Increment `counter`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_voxel_mesh(block_map, size, origin_offset=(0,0,0)):
    """
    Generates a mesh from a 3D boolean block map with padding.
    block_map: 3D array of size (size+2)x(size+2)x(size+2)
    size: logical dimension of the voxel chunk
    origin_offset: (xPos, yPos, zPos) for coordinate translation
    """
    vertices = []
    indices = []
    counter = 0
    
    # Define face normals and corresponding vertex offsets for a unit cube
    # Normals: LEFT(-1,0,0), RIGHT(1,0,0), BOTTOM(0,-1,0), TOP(0,1,0), BACK(0,0,-1), FRONT(0,0,1)
    # Unit cube vertices (centered at 0,0,0)
    unit_verts = [
        (-0.5, -0.5, 0.5), (0.5, -0.5, 0.5), (0.5, -0.5, -0.5), (-0.5, -0.5, -0.5),
        (-0.5, 0.5, 0.5), (0.5, 0.5, 0.5), (0.5, 0.5, -0.5), (-0.5, 0.5, -0.5)
    ]
    
    # Face definitions: (normal, list of 4 vertex indices forming the quad)
    # Indices correspond to unit_verts
    faces = [
        {"normal": (-1, 0, 0), "verts": [3, 7, 4, 0]}, # LEFT
        {"normal": (1, 0, 0), "verts": [1, 5, 6, 2]}, # RIGHT
        {"normal": (0, -1, 0), "verts": [0, 1, 2, 3]}, # BOTTOM
        {"normal": (0, 1, 0), "verts": [4, 5, 6, 7]}, # TOP
        {"normal": (0, 0, -1), "verts": [2, 6, 7, 3]}, # BACK
        {"normal": (0, 0, 1), "verts": [0, 4, 5, 1]}  # FRONT
    ]
    
    # Quad triangulation
    quad_indices = [0, 1, 2, 2, 3, 0]
    
    # Iterate through the padded grid
    for x in range(origin_offset[0], size + origin_offset[0] + 2):
        for y in range(origin_offset[1], size + origin_offset[1] + 2):
            for z in range(origin_offset[2], size + origin_offset[2] + 2):
                
                # Adjust indices to access logical grid if using padding
                local_x = x - origin_offset[0]
                local_y = y - origin_offset[1]
                local_z = z - origin_offset[2]
                
                # Skip boundary padding layers
                if local_x <= 0 or local_y <= 0 or local_z <= 0:
                    continue
                
                # Check if voxel is occupied
                if not block_map[local_x][local_y][local_z]:
                    continue
                
                # Voxel center position in world space
                # Assuming voxel size is 1.0
                voxel_center = (x, y, z)
                
                # Check each face
                for face in faces:
                    nx, ny, nz = face["normal"]
                    neighbor_x = x + nx
                    neighbor_y = y + ny
                    neighbor_z = z + nz
                    
                    # Determine neighbor occupancy
                    # If neighbor is outside logical bounds or empty, face is exposed
                    # Note: In a padded array, boundary checks are implicit if padding is false
                    is_neighbor_empty = True
                    
                    # Check bounds for logical grid access
                    if 0 <= neighbor_x < size + 2 and 0 <= neighbor_y < size + 2 and 0 <= neighbor_z < size + 2:
                        if block_map[neighbor_x][neighbor_y][neighbor_z]:
                            is_neighbor_empty = False
                    else:
                        # Outside logical grid, assume empty
                        is_neighbor_empty = True
                        
                    if is_neighbor_empty:
                        # Emit face
                        face_indices = face["verts"]
                        for vi in face_indices:
                            vx, vy, vz = unit_verts[vi]
                            # Position = voxel_center + unit_vertex
                            pos = (voxel_center[0] + vx, voxel_center[1] + vy, voxel_center[2] + vz)
                            # Normal is hardcoded
                            norm = face["normal"]
                            vertices.extend([pos[0], pos[1], pos[2], norm[0], norm[1], norm[2]])
                        
                        # Add indices with offset
                        base_idx = 4 * counter
                        for idx in quad_indices:
                            indices.append(base_idx + idx)
                        
                        counter += 1
                        
    return vertices, indices
```

**Parameters:**
*   `size`: The dimension of the voxel grid chunk being generated [S2, unofficial].
*   `amplitude`: Scaling factor for noise generation (if generating input data) [S2, unofficial].
*   `xNoise`, `yNoise`, `zNoise`: Divisors for coordinates passed to noise functions to control frequency [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty Data:** If the voxel data is empty, the mesh generation should return early to prevent errors [S1, unofficial].
*   **Boundary Access:** When using a padded array, loops must ensure valid access. The check `posX > 1` etc. ensures that after decrementing, the index is valid for the logical grid, preventing out-of-bounds access on the first layer [S2, unofficial].
*   **Array Resizing:** In some engines (e.g., Godot), the mesh array must be resized to `Mesh::ArrayType::ARRAY_MAX` before assigning vertex/normal/color arrays to prevent out-of-bounds access [S1, unofficial].
*   **Normals:** Normals are hardcoded per face rather than calculated via cross product to save computation and ensure consistent lighting for blocky aesthetics [S2, unofficial].

## Subtopics

### [Visible Block Face Meshing](4919-visible-block-face-meshing.md)
Visible Block Face Meshing is a surface extraction technique that generates a static triangle mesh for a voxel chunk by iterating through occupied voxel positions and emitting geometry only for faces that are adjacent to empty space or transparent blocks. It differs from its parent method, [Voxel Chunk Mesh Generation (Face Culling)](4888-voxel-chunk-mesh-generation-face-culling.md), by explicitly handling cross-chunk boundary visibility checks to ensure global consistency, and it differs from [Greedy Meshing](52-greedy-meshing.md) by not merging adjacent coplanar faces into larger quads. This approach reduces vertex count and draw calls compared to naive voxel rendering by excluding internal faces, while maintaining a simpler implementation than greedy approaches.

### Voxel Mesh Synthesis
Voxel Mesh Synthesis is a post-processing step that converts validated leaf voxels from an octree into standard 3D cube meshes for export [S3, unofficial]. It generates 8 unique corner vertices for the cube, translated by the box's min and max coordinates, and constructs 12 counter-clockwise triangular faces (2 triangles per cube face) ensuring outward-pointing surface normals [S3, unofficial]. The output is clean, standard Wavefront .obj data compatible with external 3D software [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of voxels in the chunk. Each voxel is visited once, and 6 neighbor checks are performed.
*   **Memory Complexity:** $O(F)$, where $F$ is the number of exposed faces. In the worst case (checkerboard pattern), $F \approx 3N$, leading to high memory usage. In dense volumes, $F \approx 2N$ (surface area scaling).
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Performance scales linearly with chunk size. Memory usage scales with the surface area of the solid voxels.
*   **Strengths:** Simple to implement; produces a static mesh suitable for batching; significantly reduces primitive count compared to rendering individual voxel cubes.
*   **Limitations:** Does not merge coplanar faces (unlike Greedy Meshing), resulting in more vertices and indices than necessary for large flat surfaces. Not suitable for dynamic voxel changes without regenerating the mesh.
*   **When to use:** Static voxel environments where implementation simplicity is prioritized over mesh optimization.
*   **When not to use:** Dynamic voxel environments requiring frequent updates (consider GPU-based or incremental methods); scenes requiring maximum draw call reduction (consider Greedy Meshing).

## Background
The method is widely used in voxel-based games and terrain generation systems. The Godot Voxel Terrain Plugin [S1, unofficial] implements this approach for real-time terrain rendering. The Proccraft project [S2, unofficial] demonstrates a CPU-based implementation for procedural terrain generation. The 3D Object Surface Voxelizer [S3, unofficial] uses a variant for exporting voxelized 3D models.

## Sources
[S1] Ry645/voxel-terrain-plugin, https://github.com/Ry645/voxel-terrain-plugin, fetched 2026-10-27 (unofficial)
[S2] FredrikErikJohansson/Proccraft, https://github.com/FredrikErikJohansson/Proccraft, fetched 2026-10-27 (unofficial)
[S3] Joji0/3D-Object-Surface-Voxelizer, https://github.com/Joji0/3D-Object-Surface-Voxelizer, fetched 2026-10-27 (unofficial)
