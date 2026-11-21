# Naive Voxel Mesh Generation
Also called: Full Cube Meshing, Per-Block Vertex Generation

## Overview
Naive Voxel Mesh Generation is a surface extraction algorithm that constructs a static cube mesh for a voxel chunk by explicitly generating geometry for every solid block, regardless of occlusion [S1, unofficial]. It differs from optimized blocky meshing methods by failing to cull internal faces or share vertices between adjacent blocks, resulting in a dense, redundant mesh [S1, unofficial].

## How it works
**Inputs:** A voxel chunk defined by a 3D grid of block types (where `0` or negative values represent air/empty space) and a chunk size constant `CHUNK_SIZE` (typically 16) [S1, unofficial].
**Outputs:** A `ChunkMesh` structure containing a list of vertices and an index buffer representing the visible faces of the chunk [S1, unofficial].

**Data Structures:**
The algorithm uses two primary structures:
1.  `Vertex`: A struct containing four fields:
    *   `position`: `vec3` (float x, y, z)
    *   `normal`: `vec3` (float x, y, z)
    *   `color`: `vec4` (float r, g, b, a)
    *   `texture`: `vec2` (float u, v)
2.  `ChunkMesh`: A container holding:
    *   `vertices`: A dynamic array (e.g., `std::vector<Vertex>`) storing `Vertex` objects.
    *   `indices`: A dynamic array (e.g., `std::vector<GLushort>`) storing unsigned short indices.

**Algorithm Steps:**
1.  Initialize an empty `ChunkMesh` object.
2.  Iterate through every block coordinate $(x, y, z)$ within the chunk bounds, where $0 \le x, y, z < CHUNK\_SIZE$ [S1, unofficial].
3.  Retrieve the block type at $(x, y, z)$. If the block is air (value $\le 0$), skip to the next iteration [S1, unofficial].
4.  Calculate the world-relative coordinates for the current block:
    $$
    \begin{aligned}
    wx &= (chunk\_pos.x \times CHUNK\_SIZE) + x \\
    wy &= (chunk\_pos.y \times CHUNK\_SIZE) + y \\
    wz &= (chunk\_pos.z \times CHUNK\_SIZE) + z
    \end{aligned}
    $$
5.  Generate 8 vertices for the cube corresponding to the current block. Each vertex is defined by its local corner offset $(0.0 \text{ or } 1.0)$ added to the block's world position $(wx, wy, wz)$ [S1, unofficial].
    *   The normal is set to $(0, 1, 0)$ for all vertices in the reference implementation [S1, unofficial].
    *   The color is set to $(1, 1, 1, 1)$ for all vertices [S1, unofficial].
    *   The texture coordinates are set to $(0, 0)$ for all vertices [S1, unofficial].
    *   Store the returned indices for these 8 vertices (labeled $i_0$ through $i_7$) [S1, unofficial].
6.  For each of the 6 faces of the cube, check if the adjacent block in the world is air or non-existent (value $\le 0$) [S1, unofficial].
    *   **Top (y+):** Check $(wx, wy+1, wz)$. If exposed, add triangles $(i_2, i_3, i_7)$ and $(i_2, i_7, i_6)$ [S1, unofficial].
    *   **Bottom (y-):** Check $(wx, wy-1, wz)$. If exposed, add triangles $(i_0, i_1, i_3)$ and $(i_0, i_3, i_2)$ [S1, unofficial].
    *   **Front (z-):** Check $(wx, wy, wz-1)$. If exposed, add triangles $(i_0, i_1, i_2)$ and $(i_1, i_3, i_2)$ [S1, unofficial].
    *   **Back (z+):** Check $(wx, wy, wz+1)$. If exposed, add triangles $(i_4, i_5, i_7)$ and $(i_4, i_7, i_6)$ [S1, unofficial].
    *   **Right (x+):** Check $(wx+1, wy, wz)$. If exposed, add triangles $(i_1, i_7, i_3)$ and $(i_1, i_5, i_7)$ [S1, unofficial].
    *   **Left (x-):** Check $(wx-1, wy, wz)$. If exposed, add triangles $(i_2, i_6, i_4)$ and $(i_0, i_2, i_4)$ [S1, unofficial].

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

    # Pre-calculate world offset for the chunk origin
    cx, cy, cz = chunk_pos
    chunk_origin_x = cx * chunk_size
    chunk_origin_y = cy * chunk_size
    chunk_origin_z = cz * chunk_size

    # Define local corner offsets for a unit cube
    # Order: 0:(0,0,1), 1:(1,0,1), 2:(0,1,1), 3:(1,1,1), 
    #        4:(0,0,0), 5:(1,0,0), 6:(0,1,0), 7:(1,1,0)
    # Note: This order matches the source code's implicit indexing based on addVertex calls
    local_offsets = [
        (0.0, 0.0, 1.0), (1.0, 0.0, 1.0), (0.0, 1.0, 1.0), (1.0, 1.0, 1.0),
        (0.0, 0.0, 0.0), (1.0, 0.0, 0.0), (0.0, 1.0, 0.0), (1.0, 1.0, 0.0)
    ]

    for x in range(chunk_size):
        for y in range(chunk_size):
            for z in range(chunk_size):
                block_type = chunk_blocks[x][y][z]
                
                # Skip air blocks
                if block_type <= 0:
                    continue

                # Calculate world position
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
                    vertex = {'pos': pos, 'normal': normal, 'color': color, 'tex': tex}
                    vertices.append(vertex)
                    vertex_indices.append(len(vertices) - 1)

                i0, i1, i2, i3, i4, i5, i6, i7 = vertex_indices

                # Helper to check if a world block is air
                def is_air(bx, by, bz):
                    # In a real implementation, this would check world bounds and block data
                    # For this algorithm, we assume access to a global world lookup
                    # If the block doesn't exist or is <= 0, it's air
                    return False # Placeholder for actual world lookup logic

                # Check Top Face (y+)
                # Source checks World::getBlock(wx, wy + 1, wz) <= 0
                if is_air(wx, wy + 1, wz):
                    indices.extend([i2, i3, i7, i2, i7, i6])

                # Check Bottom Face (y-)
                # Source checks World::getBlock(wx, wy - 1, wz) <= 0
                if is_air(wx, wy - 1, wz):
                    indices.extend([i0, i1, i3, i0, i3, i2])

                # Check Front Face (z-)
                # Source checks World::getBlock(wx, wy, wz - 1) <= 0
                if is_air(wx, wy, wz - 1):
                    indices.extend([i0, i1, i2, i1, i3, i2])

                # Check Back Face (z+)
                # Source checks World::getBlock(wx, wy, wz + 1) <= 0
                if is_air(wx, wy, wz + 1):
                    indices.extend([i4, i5, i7, i4, i7, i6])

                # Check Right Face (x+)
                # Source checks World::getBlock(wx + 1, wy, wz) <= 0
                if is_air(wx + 1, wy, wz):
                    indices.extend([i1, i7, i3, i1, i5, i7])

                # Check Left Face (x-)
                # Source checks World::getBlock(wx - 1, wy, wz) <= 0
                if is_air(wx - 1, wy, wz):
                    indices.extend([i2, i6, i4, i0, i2, i4])

    return vertices, indices
```

**Parameters:**
*   `CHUNK_SIZE`: Integer, dimension of the chunk in blocks. Typical value: 16 [S1, unofficial].
*   `MAX_BLOCKS`: Integer, maximum number of blocks in a chunk, used for buffer allocation sizing. Value: $16^3 = 4096$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Occlusion:** The algorithm generates vertices for all 8 corners of every solid block, even if none of the faces are visible. This leads to excessive vertex count and memory allocation overhead [S1, unofficial].
*   **Memory Reallocation:** Using dynamic arrays (like `std::vector`) without prior reservation causes reallocations during `push_back` operations, which is slow for hundreds of calls [S1, unofficial]. Reserving memory based on `numBlocks * 8` for vertices and `numBlocks * 36` for indices is recommended but may not fully resolve performance issues if the generation logic itself is the bottleneck [S1, unofficial].
*   **Boundary Checks:** The algorithm assumes access to a world lookup function that handles out-of-bounds coordinates gracefully (returning air/non-existent) [S1, unofficial].

## Subtopics

### Optimized Voxel Meshing
Generates vertices only for blocks with at least one exposed face and shares vertices between adjacent blocks. Choose this when performance is critical and mesh complexity is high. [Optimized Voxel Meshing](11002-optimized-voxel-meshing.md)

### Marching Cubes
Generates smooth, isosurface meshes from voxel data. Choose this when smooth geometry is required over blocky aesthetics. [Marching Cubes](11003-marching-cubes.md)

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of blocks in the chunk. However, the constant factor is high due to generating 8 vertices per block regardless of visibility.
*   **Memory Complexity:** $O(N)$ for vertices and indices. Specifically, it allocates space for $8 \times N$ vertices and up to $36 \times N$ indices (6 faces $\times$ 2 triangles $\times$ 3 indices) [S1, unofficial].
*   **Measured Performance:** Mesh generation for one chunk (16x16x16) takes approximately 0.36 seconds [S1, unofficial]. Loading 512 chunks (4x2 radius) takes 189 seconds [S1, unofficial].
*   **Strengths:** Simple to implement; no complex data structures or adjacency checks required beyond basic neighbor lookup.
*   **Limitations:** Extremely inefficient for dense worlds due to redundant vertex generation and lack of face culling. The 0.36-second spike per chunk causes significant frame freezes during editing or world loading [S1, unofficial].
*   **When to use:** Only for educational purposes or extremely sparse voxel worlds where performance is not a concern.
*   **When not to use:** For any production voxel engine or dense terrain generation.

## Background
The method originates from early voxel engine implementations, specifically discussed in a 2016 Stack Overflow question regarding performance issues in a C++/OpenGL voxel engine [S1, unofficial]. It represents a baseline approach before the adoption of face culling and vertex sharing techniques.

## Sources
[S1] Creating chunks mesh in a voxel engine is slow, https://stackoverflow.com/questions/38374535/creating-chunks-mesh-in-a-voxel-engine-is-slow, fetched 2026-10-26 (unofficial)
