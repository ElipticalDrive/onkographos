# Simplex Noise Chunk Generation

Also called: Procedural Chunk Generation with Simplex Noise

## Overview
Generates a voxel chunk's geometry and data using a 3D simplex noise function to determine voxel presence, allocating vertex and index buffers for rendering [S1, unofficial]. It differs from heightmap-based siblings by evaluating noise in full 3D space to create volumetric terrain rather than surface-only heightmaps, and from cave generation by using a single scalar threshold for solid vs. empty space rather than complex noise blending [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Chunk origin $(x_{pos}, y_{pos}, z_{pos})$, chunk size $S$, noise frequency scalers $(x_{noise}, y_{noise}, z_{noise})$, and amplitude $A$ [S1, unofficial].
*   **Outputs:** A vertex buffer `chunkVertices` and an index buffer `chunkIndices` representing the visible surface of the generated chunk [S1, unofficial].

**Data Structures:**
1.  **`blockMap`**: A 3D boolean grid of dimensions $(S + 2) \times (S + 2) \times (S + 2)$, implemented as `std::vector<std::vector<std::vector<bool>>>` [S1, unofficial]. It is padded by 1 voxel on all sides to facilitate neighbor checks for surface extraction [S1, unofficial].
2.  **`chunkVertices`**: A `GLfloat*` array pre-allocated to size $S \times S \times S \times 144$ [S1, unofficial]. This size accommodates up to 144 floats per voxel (6 faces $\times$ 4 vertices/face $\times$ 6 floats/vertex for position and normal) [S1, unofficial].
3.  **`chunkIndices`**: An index array pre-allocated to size $S \times S \times S \times 36$ [S1, unofficial]. This accommodates up to 36 indices per voxel (6 faces $\times$ 6 indices/face) [S1, unofficial].

**Parameters:**
*   $S = 16$ (chunk dimension in voxels) [S1, unofficial].
*   $x_{noise} = 30.0, y_{noise} = 40.0, z_{noise} = 30.0$ (frequency scaling factors) [S1, unofficial].
*   $A = 10.0$ (amplitude multiplier) [S1, unofficial].

**Steps:**
1.  Initialize `blockMap` with dimensions $(S+2)^3$ [S1, unofficial].
2.  Allocate `chunkVertices` and `chunkIndices` with the sizes specified above [S1, unofficial].
3.  Initialize a vertex counter `counter = 0` [S1, unofficial].
4.  Loop over $x$ from $x_{pos}$ to $x_{pos} + S + 1$ [S1, unofficial].
5.  Loop over $y$ from $y_{pos}$ to $y_{pos} + S + 1$ [S1, unofficial].
6.  Loop over $z$ from $z_{pos}$ to $z_{pos} + S + 1$ [S1, unofficial].
7.  Calculate local coordinates $posX = x - x_{pos}$, $posY = y - y_{pos}$, $posZ = z - z_{pos}$ [S1, unofficial].
8.  Compute noise value $n = \text{simplex}(x / x_{noise}, y / y_{noise}, z / z_{noise})$ [S1, unofficial].
9.  Compute height $h = \lfloor A \times n \rfloor$ [S1, unofficial].
10. If $h > 0$, mark `blockMap[posX][posY][posZ]` as solid (true) [S1, unofficial].
11. After filling `blockMap`, iterate through the inner chunk $(1 \le i, j, k \le S)$ to extract geometry [S1, unofficial]. For each voxel, check its 6 neighbors in `blockMap` [S1, unofficial].
12. If a neighbor is empty (false), append the corresponding face vertices and indices to `chunkVertices` and `chunkIndices`, incrementing `counter` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_chunk(x_pos, y_pos, z_pos, size=16, x_noise=30.0, y_noise=40.0, z_noise=30.0, amplitude=10.0):
    # 1. Initialize data structures
    # blockMap is (size + 2) x (size + 2) x (size + 2)
    block_map = [[[False for _ in range(size + 2)] for _ in range(size + 2)] for _ in range(size + 2)]
    
    # Pre-allocate buffers
    max_vertices = size * size * size * 144
    max_indices = size * size * size * 36
    chunk_vertices = [0.0] * max_vertices
    chunk_indices = [0] * max_indices
    
    counter = 0
    
    # 2. Fill blockMap with noise
    for x in range(x_pos, x_pos + size + 1):
        for y in range(y_pos, y_pos + size + 1):
            for z in range(z_pos, z_pos + size + 1):
                posX = x - x_pos
                posY = y - y_pos
                posZ = z - z_pos
                
                # simplex_noise(x, y, z) is a standard 3D simplex noise function returning a float in [-1, 1] [G]
                noise_val = simplex_noise(x / x_noise, y / y_noise, z / z_noise)
                
                h = int(amplitude * noise_val) if amplitude * noise_val >= 0 else int(amplitude * noise_val) - 1
                if h > 0:
                    block_map[posX][posY][posZ] = True

    # 3. Extract Geometry
    # Standard unit cube face data: 6 faces, each with 4 vertices (pos+normal) and 6 indices
    # Vertices are stored as [x, y, z, nx, ny, nz] per vertex
    FACE_VERTS = [
        [-1, -1,  1,  0,  0,  1,  1, -1,  1,  0,  0,  1,  1,  1,  1,  0,  0,  1, -1,  1,  1,  0,  0,  1], # Z+
        [-1, -1, -1,  0,  0, -1, -1,  1, -1,  0,  0, -1,  1,  1, -1,  0,  0, -1,  1, -1, -1,  0,  0, -1], # Z-
        [-1, -1, -1, -1,  0,  0,  1, -1, -1, -1,  0,  0,  1,  1, -1, -1,  0,  0, -1,  1, -1, -1,  0,  0], # X-
        [ 1, -1, -1,  1,  0,  0, -1, -1, -1,  1,  0,  0, -1,  1, -1,  1,  0,  0,  1,  1, -1,  1,  0,  0], # X+
        [-1,  1, -1,  0,  1,  0,  1,  1, -1,  0,  1,  0,  1,  1,  1,  0,  1,  0, -1,  1,  1,  0,  1,  0], # Y+
        [-1, -1, -1,  0, -1,  0,  1, -1, -1,  0, -1,  0,  1, -1,  1,  0, -1,  0, -1, -1,  1,  0, -1,  0]  # Y-
    ]
    FACE_INDICES = [
        [0, 1, 2, 0, 2, 3], [0, 1, 2, 0, 2, 3], [0, 1, 2, 0, 2, 3],
        [0, 1, 2, 0, 2, 3], [0, 1, 2, 0, 2, 3], [0, 1, 2, 0, 2, 3]
    ]
    
    for x in range(1, size + 1):
        for y in range(1, size + 1):
            for z in range(1, size + 1):
                if not block_map[x][y][z]:
                    continue
                
                # Check 6 neighbors. If neighbor is False (empty), add face.
                neighbors = [
                    (x - 1, y, z, 2), (x + 1, y, z, 3),
                    (x, y - 1, z, 5), (x, y + 1, z, 4),
                    (x, y, z - 1, 1), (x, y, z + 1, 0)
                ]
                
                for nx, ny, nz, face_idx in neighbors:
                    if not block_map[nx][ny][nz]:
                        # Copy face vertices
                        for i in range(24):
                            chunk_vertices[counter * 24 + i] = FACE_VERTS[face_idx][i]
                        # Copy face indices
                        for i in range(6):
                            chunk_indices[counter * 6 + i] = FACE_INDICES[face_idx][i] + 4 * counter
                        counter += 1

    return chunk_vertices, chunk_indices
```

**Edge Cases and Pitfalls:**
*   The source code snippet cuts off after the $h > 0$ check, leaving the exact vertex/normal generation and index offset logic incomplete [S1, unofficial].
*   Precision issues may arise if `simplex_noise` returns values outside the expected range, causing $h$ to be unexpectedly large or negative [G].
*   Boundary conditions for the padded `blockMap` must be handled correctly to avoid out-of-bounds access during neighbor checks [G].

## Operating characteristics
*   **Time Complexity:** $O(S^3)$ for both filling the `blockMap` and extracting geometry, as both steps iterate over the chunk volume [S1, unofficial].
*   **Memory Complexity:** $O(S^3)$ for the `blockMap`, `chunkVertices`, and `chunkIndices` buffers [S1, unofficial].
*   **Performance:** With $S=16$, the chunk size is manageable for real-time generation [S1, unofficial]. The fixed buffer allocation avoids dynamic memory overhead during generation [S1, unofficial].

## Background
Uses 3D simplex noise to generate voxel terrain [S1, unofficial]. It is a foundational technique for procedural voxel worlds, building upon the concept of chunk-based generation [S1, unofficial].

## Sources
[S1] FredrikErikJohansson/Proccraft, https://github.com/FredrikErikJohansson/Proccraft, fetched 2026-10-27 (unofficial)
