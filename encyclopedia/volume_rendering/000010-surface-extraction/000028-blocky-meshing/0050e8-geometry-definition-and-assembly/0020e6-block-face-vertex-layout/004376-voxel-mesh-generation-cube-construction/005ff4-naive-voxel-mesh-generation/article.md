# Naive Voxel Mesh Generation
Also called: Debug BlockMesher, Full Cube Meshing, Naive Mesh Generation, Per-Block Vertex Generation, Voxel Chunk Meshing

## Overview
Naive Voxel Mesh Generation is a block-based meshing algorithm that iterates through a voxel chunk's grid, checks each voxel's occupancy and its six immediate neighbors, and generates a full 8-vertex cube for every non-air block, emitting geometry only for faces adjacent to inactive blocks or world boundaries [S2, S3, unofficial]. It differs from optimized blocky meshing methods by generating vertices for all solid blocks regardless of occlusion, resulting in a dense, redundant mesh [S2, unofficial].

## How it works
**Inputs:** A voxel chunk defined by a 3D grid of block types (where values $\le 0$ or `T::default()` represent air/empty space) and a chunk size constant `CHUNK_SIZE` (typically 16) [S2, S3, unofficial].
**Outputs:** A `ChunkMesh` structure containing vectors for vertices and indices [S2, unofficial].

**Data Structures:**
1. `Vertex`: Contains `position` (`vec3`), `normal` (`vec3`), `color` (`vec4`), and `texture` (`vec2`) [S2, unofficial].
2. `ChunkMesh`: Contains `std::vector<Vertex> vertices` and `std::vector<GLushort> indices` [S2, unofficial]. The `addVertex` function pushes to vertices and returns the index. The `addFace` function pushes 3 indices to indices [S2, unofficial].

**Algorithm Steps:**
1. Initialize an empty `ChunkMesh` object.
2. Iterate through every block coordinate $(x, y, z)$ within the chunk bounds, where $0 \le x, y, z < CHUNK\_SIZE$ [S2, unofficial].
3. Retrieve the block type at $(x, y, z)$. If the block is air (value $\le 0$ or `T::default()`), skip to the next iteration [S2, S3, unofficial].
4. Calculate the world-relative coordinates for the current block:
   $$wx = (chunk\_pos.x \times CHUNK\_SIZE) + x$$
   $$wy = (chunk\_pos.y \times CHUNK\_SIZE) + y$$
   $$wz = (chunk\_pos.z \times CHUNK\_SIZE) + z$$
   [S2, unofficial]
5. Generate 8 vertices for the cube corresponding to the current block. Each vertex is defined by its local corner offset $(0.0 \text{ or } 1.0)$ added to the block's world position $(wx, wy, wz)$ [S2, unofficial].
   - The normal is set to $(0, 1, 0)$ for all vertices in the reference implementation [S2, unofficial].
   - The color is set to $(1, 1, 1, 1)$ for all vertices [S2, unofficial].
   - The texture coordinates are set to $(0, 0)$ for all vertices [S2, unofficial].
   - Store the returned indices for these 8 vertices (labeled $i_0$ through $i_7$) [S2, unofficial].
6. For each of the 6 faces of the cube, check if the adjacent block in the world is air or non-existent (value $\le 0$) [S2, unofficial].
   - **Top (y+):** Check $(wx, wy+1, wz)$. If exposed, add triangles $(i_2, i_3, i_7)$ and $(i_2, i_7, i_6)$ [S2, unofficial].
   - **Bottom (y-):** Check $(wx, wy-1, wz)$. If exposed, add triangles $(i_0, i_1, i_3)$ and $(i_0, i_3, i_2)$ [S2, unofficial].
   - **Front (z-):** Check $(wx, wy, wz-1)$. If exposed, add triangles $(i_0, i_1, i_2)$ and $(i_1, i_3, i_2)$ [S2, unofficial].
   - **Back (z+):** Check $(wx, wy, wz+1)$. If exposed, add triangles $(i_4, i_5, i_7)$ and $(i_4, i_7, i_6)$ [S2, unofficial].
   - **Right (x+):** Check $(wx+1, wy, wz)$. If exposed, add triangles $(i_1, i_7, i_3)$ and $(i_1, i_5, i_7)$ [S2, unofficial].
   - **Left (x-):** Check $(wx-1, wy, wz)$. If exposed, add triangles $(i_2, i_6, i_4)$ and $(i_0, i_2, i_4)$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def naive_voxel_mesh_generation(chunk_pos, chunk_blocks, chunk_size=16):
    """
    chunk_pos: tuple (cx, cy, cz) representing the chunk's origin in world blocks
    chunk_blocks: 3D array [chunk_size][chunk_size][chunk_size] of block types
    Returns: (vertices, indices)
    """
    vertices = []
    indices = []

    cx, cy, cz = chunk_pos
    chunk_origin_x = cx * chunk_size
    chunk_origin_y = cy * chunk_size
    chunk_origin_z = cz * chunk_size

    # Local corner offsets matching the reference C++ implementation
    # i0:(0,0,1), i1:(1,0,1), i2:(0,1,1), i3:(1,1,1),
    # i4:(0,0,0), i5:(1,0,0), i6:(0,1,0), i7:(1,1,0)
    local_offsets = [
        (0.0, 0.0, 1.0), (1.0, 0.0, 1.0), (0.0, 1.0, 1.0), (1.0, 1.0, 1.0),
        (0.0, 0.0, 0.0), (1.0, 0.0, 0.0), (0.0, 1.0, 0.0), (1.0, 1.0, 0.0)
    ]

    for x in range(chunk_size):
        for y in range(chunk_size):
            for z in range(chunk_size):
                block_type = chunk_blocks[x][y][z]
                
                # Skip air/empty blocks (value <= 0 or T::default())
                if block_type <= 0:
                    continue

                wx = chunk_origin_x + x
                wy = chunk_origin_y + y
                wz = chunk_origin_z + z

                # Generate 8 vertices for this block
                vertex_indices = []
                for dx, dy, dz in local_offsets:
                    pos = (wx + dx, wy + dy, wz + dz)
                    normal = (0.0, 1.0, 0.0)
                    color = (1.0, 1.0, 1.0, 1.0)
                    tex = (0.0, 0.0)
                    vertices.append({'pos': pos, 'normal': normal, 'color': color, 'tex': tex})
                    vertex_indices.append(len(vertices) - 1)

                i0, i1, i2, i3, i4, i5, i6, i7 = vertex_indices

                # Note: Actual implementation requires a world lookup function to check 
                # if the adjacent block at the specified coordinates is air/non-existent.
                # The sources do not provide the exact world lookup implementation.
                
                # Check Top Face (y+)
                # if world_is_air(wx, wy + 1, wz):
                #     indices.extend([i2, i3, i7, i2, i7, i6])

                # Check Bottom Face (y-)
                # if world_is_air(wx, wy - 1, wz):
                #     indices.extend([i0, i1, i3, i0, i3, i2])

                # Check Front Face (z-)
                # if world_is_air(wx, wy, wz - 1):
                #     indices.extend([i0, i1, i2, i1, i3, i2])

                # Check Back Face (z+)
                # if world_is_air(wx, wy, wz + 1):
                #     indices.extend([i4, i5, i7, i4, i7, i6])

                # Check Right Face (x+)
                # if world_is_air(wx + 1, wy, wz):
                #     indices.extend([i1, i7, i3, i1, i5, i7])

                # Check Left Face (x-)
                # if world_is_air(wx - 1, wy, wz):
                #     indices.extend([i2, i6, i4, i0, i2, i4])

    return vertices, indices
```

**Parameters:**
*   `CHUNK_SIZE`: Integer, dimension of the chunk in blocks. Typical value: 16 [S2, unofficial].
*   `MAX_BLOCKS`: Integer, maximum number of blocks in a chunk, used for buffer allocation sizing. Value: $16^3 = 4096$ [S2, unofficial].
*   `voxelSize`: A scalar used to scale the unit cubes emitted by the mesher [S3, unofficial].
*   `density`: A scalar value where density < 0 indicates a solid voxel [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Occlusion:** The algorithm generates vertices for all 8 corners of every solid block, even if none of the faces are visible. This leads to excessive vertex count and memory allocation overhead [S2, unofficial].
*   **Memory Reallocation:** Using dynamic arrays (like `std::vector`) without prior reservation causes reallocations during `push_back` operations, which is slow for hundreds of calls [S2, unofficial]. Reserving memory based on `numBlocks * 8` for vertices and `numBlocks * 36` for indices is recommended [S2, unofficial].
*   **Boundary Checks:** The algorithm assumes access to a world lookup function that handles out-of-bounds coordinates gracefully (returning air/non-existent) [S2, unofficial].

## Subtopics

### [Voxel Mesh Generation with Adjacency Culling](19348-voxel-mesh-generation-with-adjacency-culling.md)
Voxel Mesh Generation with Adjacency Culling is a surface extraction technique that constructs a mesh from a voxel grid by generating geometry only for faces adjacent to inactive blocks or world boundaries, thereby preventing triangles from connecting across voxel boundaries. It improves upon Naive Voxel Mesh Generation by eliminating internal, occluded faces, which reduces the vertex and index count and prevents visual artifacts caused by rendering hidden geometry. [Voxel Mesh Generation with Adjacency Culling](19348-voxel-mesh-generation-with-adjacency-culling.md)

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of blocks in the chunk. The constant factor is high due to generating 8 vertices per block regardless of visibility [S2, unofficial].
*   **Memory Complexity:** $O(N)$ for vertices and indices. Specifically, it allocates space for $8 \times N$ vertices and up to $36 \times N$ indices (6 faces $\times$ 2 triangles $\times$ 3 indices) [S2, unofficial].
*   **Measured Performance:** Mesh generation for one chunk (16x16x16) takes approximately 0.36 seconds [S2, unofficial]. Loading 512 chunks (4x2 radius) takes 189 seconds [S2, unofficial].
*   **Strengths:** Simple to implement; no complex data structures or adjacency checks required beyond basic neighbor lookup [G].
*   **Limitations:** Extremely inefficient for dense worlds due to redundant vertex generation and lack of face culling. The 0.36-second spike per chunk causes significant frame freezes during editing or world loading [S2, unofficial].
*   **When to use:** Only for educational purposes or debugging SDF and voxel correctness [S3, unofficial].
*   **When not to use:** For any production voxel engine or dense terrain generation [G].

## Background
The method originates from early voxel engine implementations, specifically discussed in a 2016 Stack Overflow question regarding performance issues in a C++/OpenGL voxel engine [S2, unofficial]. It represents a baseline approach before the adoption of face culling and vertex sharing techniques [G].

## Sources
[S1] WildPixelGames/voxelis, https://github.com/WildPixelGames/voxelis, fetched 2026-10-26 (unofficial)
[S2] Creating chunks mesh in a voxel engine is slow, https://stackoverflow.com/questions/38374535/creating-chunks-mesh-in-a-voxel-engine-is-slow, fetched 2026-10-26 (unofficial)
[S3] 4.2.1 – Implement Debug BlockMesher, https://github.com/Hritvik16/VoxelTerraria/issues/81, fetched 2026-10-27 (unofficial)
