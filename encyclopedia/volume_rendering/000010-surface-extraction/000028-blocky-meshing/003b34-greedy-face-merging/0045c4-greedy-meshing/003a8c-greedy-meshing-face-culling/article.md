# Greedy Meshing / Face Culling

Also called: Face culling meshing, Face-based Meshing, Block Mesh Generation, Voxel Face Generation, Voxel face extraction, Block Surface Extraction, Voxel Surface Extraction

## Overview
Greedy Meshing / Face Culling is a surface extraction technique that generates a static triangle mesh for a voxel chunk by iterating through occupied voxel positions and emitting geometry only for faces that are adjacent to empty space or transparent blocks [S1, S2, S3, unofficial]. It solves the problem of excessive polygon counts in voxel worlds by skipping interior faces, generating geometry only for exterior (exposed) faces [S4, unofficial]. Unlike its parent method, [Greedy Meshing](1776-greedy-meshing.md), which merges adjacent coplanar faces into larger quads, this approach does not perform merging, resulting in a mesh where each exposed face is represented by a distinct quad [S1, S2, unofficial]. This reduces vertex count and draw calls compared to naive voxel rendering (one quad per voxel face) by excluding internal faces [S2, unofficial].

## How it works
The method iterates over every voxel in a chunk, determines visibility for each of its six faces by checking neighbors, and emits vertex/index data for exposed faces.

### Inputs and Outputs
*   **Input**: A voxel chunk data structure containing block presence information (e.g., a 3D boolean array or bit-packed array) and optionally block type/texture data.
*   **Output**: A vertex buffer and an index buffer containing the triangle mesh for the chunk.

### Data Structures
Sources define several layouts for the voxel data and mesh buffers:

1.  **Voxel Storage**:
    *   A 3D boolean array `filled[512][512][512]` representing block presence [S2, unofficial].
    *   A bit-packed array `active[x][y]` where each integer represents a column of blocks along the Z-axis, allowing neighbor checks via bit shifting [S1, unofficial].
    *   A padded 3D boolean array `blockMap` of size `(size + 2) x (size + 2) x (size + 2)` to simplify boundary checks [S6, unofficial].

2.  **Mesh Buffers**:
    *   **Vertex Buffer**: An interleaved array of `GLfloat`. Each vertex consists of 6 floats: 3 for position (x, y, z) and 3 for normal (nx, ny, nz) [S6, unofficial]. Some implementations include color (RGB) or UV coordinates [S1, S3, unofficial].
    *   **Index Buffer**: An array of `unsigned int`. Indices are 0-based relative to the start of a quad's 4 vertices, typically using the pattern `{0, 1, 2, 2, 3, 0}` to form two triangles [S6, unofficial].

3.  **Face Definitions**:
    *   **Normals**: Six unit vectors representing face directions: LEFT(-1,0,0), RIGHT(1,0,0), BOTTOM(0,-1,0), TOP(0,1,0), BACK(0,0,-1), FRONT(0,0,1) [S5, unofficial].
    *   **Vertex Offsets**: For a unit cube centered at origin, vertices are defined at coordinates like {-0.5,-0.5,0.5}, etc. [S5, unofficial]. Alternatively, face-specific vertex positions are calculated using offsets of 0 or `-blockSize` relative to the block's base position [S1, unofficial].
    *   **Face Indices**: A mapping from face direction to two triangles (arrays of 3 vertex indices). Example for LEFT face: `{{3,7,4}, {3,4,0}}` [S5, unofficial].

### Algorithm Steps

1.  **Initialize Buffers**: Allocate memory for vertices and indices. If using a padded map, initialize `blockMap` with false values [S6, unofficial].
2.  **Iterate Voxels**: Loop through all voxel coordinates $(x, y, z)$ within the chunk bounds. If the voxel at $(x, y, z)$ is empty, skip it [S3, S6, unofficial].
3.  **Check Face Visibility**: For each of the 6 faces, determine if the adjacent neighbor is empty or transparent.
    *   **Neighbor Check**: Calculate neighbor coordinates $(cx, cy, cz)$ using face normals [S3, unofficial].
    *   **Boundary Handling**: If the neighbor coordinate is outside the chunk bounds, treat it as empty (exposed face) [S1, S6, unofficial].
    *   **Data Check**: If the neighbor voxel is present (solid), the face is internal and skipped. If the neighbor is absent (empty), the face is exposed [S3, S4, unofficial].
    *   *Bit-packed Optimization*: If using bit-packed columns, check specific bits in the `active` array for Z-neighbors, and check existence of adjacent column arrays for X/Y neighbors [S1, unofficial].
4.  **Generate Geometry**: For each exposed face:
    *   **Calculate Position**: Determine the world position of the face vertices. For a block at $(x,y,z)$ with block size $B$, positions are offset by $0$ or $-B$ depending on the face direction [S1, unofficial].
    *   **Calculate Normal**: Assign the outward normal vector for the face [S5, unofficial].
    *   **Calculate UV/Color**: If texture mapping is used, compute UV offsets based on texture atlas indices. Formula for UV offset in a grid of $cols \times rows$:
        $$u_{off} = (tex\_idx \mod cols) \times (1.0 / cols)$$
        $$v_{off} = \lfloor tex\_idx / cols \rfloor \times (1.0 / rows)$$
        [S3, unofficial]
    *   **Ambient Occlusion (Optional)**: Sample AO for the 4 vertices. To resolve ambiguous winding order due to AO, compare diagonal AO sums: if $(ao[0] + ao[2]) > (ao[1] + ao[3])$, flip the triangle winding [S3, unofficial].
    *   **Emit Vertices**: Add 4 vertices (position, normal, UV, color) to the vertex buffer [S3, S6, unofficial].
    *   **Emit Indices**: Add 6 indices to the index buffer using the face-specific triangle pattern, adjusting for the current vertex offset [S3, S6, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_face_culled_mesh(chunk_data, chunk_size, block_size):
    vertices = []
    indices = []
    vertex_offset = 0
    
    # Define face normals and corresponding vertex offsets for a unit cube
    # Normals: Left(-1,0,0), Right(1,0,0), Bottom(0,-1,0), Top(0,1,0), Back(0,0,-1), Front(0,0,1)
    face_normals = [
        (-1, 0, 0), (1, 0, 0), (0, -1, 0), (0, 1, 0), (0, 0, -1), (0, 0, 1)
    ]
    
    # Vertex offsets for each face relative to block origin (0,0,0)
    # Each face has 4 vertices. Order must match index pattern {0,1,2,2,3,0}
    face_vertex_offsets = [
        # Left (x-1)
        [(-1,0,0), (-1,0,-1), (-1,-1,-1), (-1,-1,0)],
        # Right (x+1)
        [(1,0,0), (1,0,-1), (1,-1,-1), (1,-1,0)],
        # Bottom (y-1)
        [(0,-1,0), (0,-1,-1), (-1,-1,-1), (-1,-1,0)],
        # Top (y+1)
        [(0,1,0), (0,1,-1), (-1,1,-1), (-1,1,0)],
        # Back (z-1)
        [(0,0,-1), (-1,0,-1), (-1,-1,-1), (0,-1,-1)],
        # Front (z+1)
        [(0,0,0), (-1,0,0), (-1,-1,0), (0,-1,0)]
    ]
    
    # Standard index pattern for two triangles forming a quad
    quad_indices = [0, 1, 2, 2, 3, 0]

    for x in range(chunk_size):
        for y in range(chunk_size):
            for z in range(chunk_size):
                # Skip empty voxels
                if not chunk_data.is_occupied(x, y, z):
                    continue

                for face_idx in range(6):
                    nx, ny, nz = face_normals[face_idx]
                    
                    # Calculate neighbor coordinates
                    neighbor_x = x + nx
                    neighbor_y = y + ny
                    neighbor_z = z + nz
                    
                    # Check if neighbor is empty or out of bounds
                    is_exposed = False
                    if (neighbor_x < 0 or neighbor_x >= chunk_size or
                        neighbor_y < 0 or neighbor_y >= chunk_size or
                        neighbor_z < 0 or neighbor_z >= chunk_size):
                        is_exposed = True
                    elif not chunk_data.is_occupied(neighbor_x, neighbor_y, neighbor_z):
                        is_exposed = True
                    
                    if not is_exposed:
                        continue

                    # Emit geometry for this exposed face
                    for v_idx in range(4):
                        ox, oy, oz = face_vertex_offsets[face_idx][v_idx]
                        
                        # World position
                        wx = (x + ox) * block_size
                        wy = (y + oy) * block_size
                        wz = (z + oz) * block_size
                        
                        # Normal is constant for the face
                        nx_f, ny_f, nz_f = face_normals[face_idx]
                        
                        # Add vertex (Position + Normal)
                        vertices.append((wx, wy, wz, nx_f, ny_f, nz_f))
                    
                    # Add indices
                    for i in quad_indices:
                        indices.append(vertex_offset + i)
                    
                    vertex_offset += 4

    return vertices, indices
```

### Parameters and Edge Cases
*   **Block Size**: Typically 1.0 unit, but can be scaled [S1, S6, unofficial].
*   **Chunk Size**: Common sizes are 16x16x16 or 512x512x512 depending on implementation [S1, S2, unofficial].
*   **Boundary Conditions**: When checking neighbors at chunk boundaries (e.g., $y=0$ for bottom face), the neighbor is considered empty unless cross-chunk data is explicitly available [S1, S6, unofficial].
*   **Precision**: Using integer coordinates for voxel positions avoids floating-point errors in neighbor lookups [S5, unofficial].
*   **Empty Data**: If the voxel data array is empty, the mesh generation should return early to prevent errors [S5, unofficial].

## Subtopics

### [Visible Block Face Meshing](4919-visible-block-face-meshing.md)
Explicitly handles cross-chunk boundary visibility checks to ensure global consistency, differing from the parent by not merging faces.

### [Chunk Mesh Generation](8755-chunk-mesh-generation.md)
Often incorporates dynamic zone management and recalculates merge boundaries based on player movement, unlike static greedy meshing.

### [Chunk Loading and Mesh Generation](8774-chunk-loading-and-mesh-generation.md)
Manages the lifecycle of chunk geometry, caching GPU resources and handling updates when chunks change.

### Voxel Mesh Synthesis
A post-processing step that converts validated leaf voxels from an octree into standard 3D cube meshes for export [S7, unofficial]. It generates 8 unique corner vertices for the cube, constructs 12 counter-clockwise triangular faces with outward-pointing normals, and outputs clean Wavefront .obj data [S7, unofficial].

### Chunk-based Voxel Rendering Pipeline
A pipeline that instantiates a chunk, fills its grid from a world grid, generates the surface mesh, and allocates heap memory for vertex and index arrays. It creates Vertex Buffer Objects (VBO) and Index Buffer Objects (IBO) with specific layouts (e.g., `layout.Push(GL_FLOAT, 4)`) and draws the mesh using `glDrawElements` [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of voxels in the chunk, as each voxel and its 6 neighbors are checked once [S1, S3, unofficial].
*   **Memory Complexity**: $O(F)$ where $F$ is the number of exposed faces. In the worst case (checkerboard pattern), $F \approx 3N$, leading to high memory usage. In typical terrain, $F \ll N$.
*   **Performance**: Reduces triangle count significantly compared to naive rendering (one quad per voxel face) but is higher than greedy meshing which merges faces. Performance is heavily dependent on the surface-to-volume ratio of the chunk [S2, unofficial].
*   **Strengths**: Simple to implement, deterministic, no merging artifacts, good for static chunks [S1, S5, unofficial].
*   **Limitations**: High draw call count if not batched, no quad merging leads to larger vertex buffers than greedy meshing [S1, S2, unofficial].
*   **When to Use**: Static voxel worlds, simple terrain, or when implementation simplicity is prioritized over maximum mesh compression [S1, S4, unofficial].
*   **When Not to Use**: Highly dynamic scenes requiring frequent mesh updates where greedy meshing's reduced triangle count would offer better performance, or when memory bandwidth is a critical bottleneck [S2, unofficial].

## Background
The technique is a fundamental surface extraction method in voxel rendering, originating from early voxel engines and games like Minecraft. It is widely used in WebGL voxel engines [S1, unofficial], C++ voxelizers [S2, unofficial], and game engine plugins [S5, unofficial]. The method is often implemented with optimizations like bit-packing for neighbor checks [S1, unofficial] or multi-threaded slab rendering [S2, unofficial].

## Sources
[S1] Show HN: Basic voxel engine in WebGL/JavaScript with full source, https://github.com/Lallassu/vox2, fetched 2026-10-26 (unofficial)
[S2] 3D-Mesh-Voxelizer - GitHub, https://github.com/Talkative-Banana/3D-Mesh-Voxelizer, fetched 2026-10-26 (unofficial)
[S3] Linirby/HelloVoxel, https://github.com/Linirby/HelloVoxel, fetched 2026-10-27 (unofficial)
[S4] nnarain/VoxelEngine, https://github.com/nnarain/VoxelEngine, fetched 2026-10-27 (unofficial)
[S5] Ry645/voxel-terrain-plugin, https://github.com/Ry645/voxel-terrain-plugin, fetched 2026-10-27 (unofficial)
[S6] FredrikErikJohansson/Proccraft, https://github.com/FredrikErikJohansson/Proccraft, fetched 2026-10-27 (unofficial)
[S7] Joji0/3D-Object-Surface-Voxelizer, https://github.com/Joji0/3D-Object-Surface-Voxelizer, fetched 2026-10-27 (unofficial)
