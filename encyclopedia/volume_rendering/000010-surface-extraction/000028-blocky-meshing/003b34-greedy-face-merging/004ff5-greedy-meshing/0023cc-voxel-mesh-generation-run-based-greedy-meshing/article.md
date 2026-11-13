# Voxel Mesh Generation (Run-based Greedy Meshing)

Also called: Greedy Mesh Generation, Optimised voxel mesh generation, Sector's Edge meshing

## Overview
Voxel Mesh Generation (Run-based Greedy Meshing) is a CPU-based mesh generation algorithm for voxel worlds that combines adjacent faces into runs along the X and Y axes to reduce triangle count, while optimizing for generation speed over minimal triangle count [S2, unofficial]. It extends the 2D greedy meshing concept to 3D by processing chunks of blocks, producing meshes with approximately 20% more triangles than standard greedy meshing but running approximately 390% faster [S2, unofficial]. It is designed for scenarios requiring frequent chunk regeneration, such as multiplayer voxel games [S2, unofficial].

## How it works
The method processes a voxel chunk (typically 32x32x32 blocks) by iterating through visible faces, identifying maximal rectangular runs of identical blocks, and packing vertex data into a compact 32-bit integer format for efficient GPU buffering.

### Inputs and Outputs
**Inputs:**
*   A voxel chunk containing $N = \text{CHUNK\_SIZE}^3$ blocks (default $\text{CHUNK\_SIZE}=32$, $N=32768$) [S2, unofficial].
*   Block metadata: type (texture ID), health, visibility, and material properties [S2, unofficial].
*   Neighbor chunk references for boundary checks [S2, unofficial].

**Outputs:**
*   A vertex buffer containing packed vertex data (position, texture, brightness, normal) [S2, unofficial].
*   Index data (implicitly handled by the quad generation logic in the source implementation, though the primary optimization is vertex packing) [S2, unofficial].

### Data Structures

**1. Packed Vertex Integer (32-bit uint)**
Vertex attributes are packed into a single 32-bit integer to minimize memory bandwidth and simplify GPU vertex attributes [S2, unofficial].

| Bits | Field | Size | Description |
| :--- | :--- | :--- | :--- |
| 0-5 | X Position | 6 bits | Block X coordinate (0-31) [S2, unofficial] |
| 6-11 | Y Position | 6 bits | Block Y coordinate (0-31) [S2, unofficial] |
| 12-17 | Z Position | 6 bits | Block Z coordinate (0-31) [S2, unofficial] |
| 18-22 | Texture ID | 5 bits | Texture index (0-31) [S2, unofficial] |
| 23-26 | Health/Brightness | 4 bits | Block health divided by 16 (0-15) [S2, unofficial] |
| 27-29 | Normal | 3 bits | Face normal direction (0-7) [S2, unofficial] |
| 30-31 | Unused | 2 bits | Reserved [S2, unofficial] |

**2. Chunk Helper Visitation Arrays**
Six boolean arrays of size $\text{CHUNK\_SIZE\_CUBED}$ (32768) track visited blocks for each face direction to prevent duplicate face generation within a chunk [S2, unofficial]:
*   `visitXN`, `visitXP`, `visitYN`, `visitYP`, `visitZN`, `visitZP` [S2, unofficial].

**3. Height Maps**
Two byte arrays of size $\text{CHUNK\_SIZE\_SQUARED}$ (1024) store the minimum and maximum Y-coordinates of solid blocks for each (X, Z) column, allowing the algorithm to skip empty space [S2, unofficial]:
*   `MinY[i]`: Minimum Y index for column $i$.
*   `MaxY[i]`: Maximum Y index for column $i$.

**4. Vertex Buffer**
An integer array (`int[]`) storing the packed vertex data. It supports dynamic extension if the buffer nears capacity [S2, unofficial].

### Algorithm Steps

1.  **Initialization**:
    *   Initialize `visitXN`, `visitXP`, `visitZN`, `visitZP` (and Y-variants if used) to `false` [S2, unofficial].
    *   Initialize `MinY` array with default value $\text{CHUNK\_SIZE}$ (32) [S2, unofficial].
    *   Initialize `vertexBuffer` with a default size (e.g., 4096 integers) or previous size + 1024 [S1, unofficial][S2, unofficial].
    *   Load neighbor chunk references for X-, X+, Y-, Y+, Z-, Z+ directions. Handle boundary conditions where neighbor is `null` if the chunk is at the map edge [S2, unofficial].

2.  **Column Iteration**:
    *   Iterate $k$ (Z-axis) from 0 to $\text{CHUNK\_SIZE}-1$ [S2, unofficial].
    *   Calculate `kCS2 = k * CHUNK_SIZE_SQUARED` [S2, unofficial].
    *   Determine `minZEdge` ($k==0$) and `maxZEdge` ($k==\text{CHUNK\_SIZE}-1$) [S2, unofficial].
    *   Iterate $i$ (X-axis) from 0 to $\text{CHUNK\_SIZE}-1$ [S2, unofficial].
    *   Calculate `heightMapAccess = k * CHUNK_SIZE + i` [S2, unofficial].
    *   Get `j = MinY[heightMapAccess]` and `topJ = MaxY[heightMapAccess]` [S2, unofficial].
    *   Determine `minXEdge` ($i==0$) and `maxXEdge` ($i==\text{CHUNK\_SIZE}-1$) [S2, unofficial].

3.  **Run Generation (Inner Loop)**:
    *   Iterate $j$ (Y-axis) from `MinY` to `MaxY - 1` [S2, unofficial].
    *   Calculate `access = kCS2 + i * CHUNK_SIZE + j` [S2, unofficial].
    *   If `block[access]` is not `EMPTY`:
        *   Call `CreateRun(block, i, j, k, ...)` [S2, unofficial].

4.  **CreateRun Logic**:
    *   Pack block data: `textureHealth16 = indexToTextureShifted[block.index] | ((block.health / 16) << 23)` [S2, unofficial].
    *   Calculate `jS = j << 6` and `jS1 = (j + 1) << 6` [S2, unofficial].
    *   **Check X- Face**:
        *   If `!visitXN[access]` and `DrawFaceXN` returns true:
            *   Mark `visitXN[access] = true` [S2, unofficial].
            *   Loop `length` from `jS1` to `Constants.ChunkSizeShifted` in steps of $(1 \ll 6)$ [S2, unofficial].
            *   Check `DifferentBlock(chunkAccess, block)`. If true, break loop [S2, unofficial].
            *   Mark `visitXN[chunkAccess++] = true` [S2, unofficial].
            *   Append quad using `BlockVertex.AppendQuadX` [S2, unofficial].
    *   **Check X+ Face**:
        *   If `!visitXP[access]` and `DrawFaceXP` returns true:
            *   Mark `visitXP[access] = true` [S2, unofficial].
            *   Loop `length` from `jS1` to `Constants.ChunkSizeShifted` in steps of $(1 \ll 6)$ [S2, unofficial].
            *   Check `DifferentBlock(chunkAccess, block)`. If true, break loop [S2, unofficial].
            *   Mark `visitXP[chunkAccess++] = true` [S2, unofficial].
            *   Append quad using `BlockVertex.AppendQuadX` [S2, unofficial].
    *   **Check Z- Face**:
        *   If `!visitZN[access]` and `DrawFaceZN` returns true:
            *   Mark `visitZN[access] = true` [S2, unofficial].
            *   Loop `length` from `jS1` to `Constants.ChunkSizeShifted` in steps of $(1 \ll 6)$ [S2, unofficial].
            *   Check `DifferentBlock(chunkAccess, block)`. If true, break loop [S2, unofficial].
            *   Mark `visitZN[chunkAccess++] = true` [S2, unofficial].
            *   Append quad using `BlockVertex.AppendQuadZ` [S2, unofficial].
    *   **Check Z+ Face**:
        *   If `!visitZP[access]` and `DrawFaceZP` returns true:
            *   Mark `visitZP[access] = true` [S2, unofficial].
            *   Loop `length` from `jS1` to `Constants.ChunkSizeShifted` in steps of $(1 \ll 6)$ [S2, unofficial].
            *   Check `DifferentBlock(chunkAccess, block)`. If true, break loop [S2, unofficial].
            *   Mark `visitZP[chunkAccess++] = true` [S2, unofficial].
            *   Append quad using `BlockVertex.AppendQuadZ` [S2, unofficial].

5.  **Buffer Management**:
    *   After processing a column, if `vertexBuffer.used > vertexBuffer.data.Length - 2048`, extend the buffer by 2048 integers [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
CHUNK_SIZE = 32
CHUNK_SIZE_SQUARED = 1024
CHUNK_SIZE_CUBED = 32768
CHUNK_SIZE_MINUS_ONE = 31
CHUNK_SIZE_SHIFTED = 32 << 6  # 1024

def generate_chunk_mesh(chunk):
    """
    Generates mesh for a single chunk.
    chunk.data: array of Block objects, size CHUNK_SIZE_CUBED
    chunk.MinY: array of bytes, size CHUNK_SIZE_SQUARED
    chunk.MaxY: array of bytes, size CHUNK_SIZE_SQUARED
    chunk.visitXN, visitXP, visitZN, visitZP: boolean arrays, size CHUNK_SIZE_CUBED
    chunk.vertexBuffer: object with .data (int[]), .used (int), .extend(amount) method
    """
    # Initialize visitation arrays
    for i in range(CHUNK_SIZE_CUBED):
        chunk.visitXN[i] = False
        chunk.visitXP[i] = False
        chunk.visitZN[i] = False
        chunk.visitZP[i] = False

    # Iterate Z axis
    for k in range(CHUNK_SIZE):
        kCS2 = k * CHUNK_SIZE_SQUARED
        minZEdge = (k == 0)
        maxZEdge = (k == CHUNK_SIZE_MINUS_ONE)
        
        # Iterate X axis
        for i in range(CHUNK_SIZE):
            heightMapAccess = k * CHUNK_SIZE + i
            j = chunk.MinY[heightMapAccess]
            topJ = chunk.MaxY[heightMapAccess]
            
            minXEdge = (i == 0)
            maxXEdge = (i == CHUNK_SIZE_MINUS_ONE)
            
            # Iterate Y axis from MinY to MaxY
            access = kCS2 + i * CHUNK_SIZE + j
            for j in range(j, topJ):
                block = chunk.data[access]
                
                if block.kind != EMPTY:
                    create_run(
                        block=block,
                        i=i, j=j, k=k,
                        i1=i+1, k1=k+1,
                        y=j + chunk.chunkPosY * CHUNK_SIZE,
                        access=access,
                        minX=minXEdge, maxX=maxXEdge,
                        minY=(j==0), maxY=(j==CHUNK_SIZE_MINUS_ONE),
                        minZ=minZEdge, maxZ=maxZEdge,
                        iCS=i * CHUNK_SIZE,
                        kCS2=kCS2,
                        chunk=chunk
                    )
                
                access += 1
            
            # Extend buffer if needed
            if chunk.vertexBuffer.used > len(chunk.vertexBuffer.data) - 2048:
                chunk.vertexBuffer.extend(2048)

def create_run(block, i, j, k, i1, k1, y, access, minX, maxX, minY, maxY, minZ, maxZ, iCS, kCS2, chunk):
    """
    Creates runs for X- and Z- faces (and their positive counterparts)
    """
    # Pack texture and health
    # indexToTextureShifted is a lookup table mapping block index to shifted texture ID
    texture_health = chunk.indexToTextureShifted[block.index] | ((block.health // 16) << 23)
    
    jS = j << 6
    jS1 = (j + 1) << 6
    
    # X- Face (Left)
    if not chunk.visitXN[access] and draw_face_xn(j, access, minX, kCS2):
        chunk.visitXN[access] = True
        chunk_access = access + 1
        
        # Loop to find run length
        length = jS1
        while length < CHUNK_SIZE_SHIFTED:
            if different_block(chunk.data[chunk_access], block):
                break
            chunk.visitXN[chunk_access] = True
            chunk_access += 1
            length += (1 << 6)
        
        # Append quad for X- face
        # AppendQuadX(buffer, x, yL, yR, kL, kR, normal, textureHealth)
        # Note: Exact parameters depend on implementation details of AppendQuadX
        # Assuming x is derived from i, kL/kR from loop
        append_quad_x(chunk.vertexBuffer, i, jS, length, jS, length, NORMAL_XN, texture_health)

    # X+ Face (Right)
    if not chunk.visitXP[access] and draw_face_xp(j, access, maxX, kCS2):
        chunk.visitXP[access] = True
        chunk_access = access + 1
        
        length = jS1
        while length < CHUNK_SIZE_SHIFTED:
            if different_block(chunk.data[chunk_access], block):
                break
            chunk.visitXP[chunk_access] = True
            chunk_access += 1
            length += (1 << 6)
            
        append_quad_x(chunk.vertexBuffer, i, jS, length, jS, length, NORMAL_XP, texture_health)

    # Z- Face (Front)
    if not chunk.visitZN[access] and draw_face_zn(j, access, minZ, kCS2):
        chunk.visitZN[access] = True
        chunk_access = access + 1
        
        length = jS1
        while length < CHUNK_SIZE_SHIFTED:
            if different_block(chunk.data[chunk_access], block):
                break
            chunk.visitZN[chunk_access] = True
            chunk_access += 1
            length += (1 << 6)
            
        append_quad_z(chunk.vertexBuffer, i, i1, jS, length, k, NORMAL_ZN, texture_health)

    # Z+ Face (Back)
    if not chunk.visitZP[access] and draw_face_zp(j, access, maxZ, kCS2):
        chunk.visitZP[access] = True
        chunk_access = access + 1
        
        length = jS1
        while length < CHUNK_SIZE_SHIFTED:
            if different_block(chunk.data[chunk_access], block):
                break
            chunk.visitZP[chunk_access] = True
            chunk_access += 1
            length += (1 << 6)
            
        append_quad_z(chunk.vertexBuffer, i, i1, jS, length, k, NORMAL_ZP, texture_health)

def append_quad_x(buffer, x, yL, yR, kL, kR, normal, texture_health):
    """
    Appends 6 vertices for an X-face quad.
    Shared bits: x, texture_health, normal.
    Varying bits: y, k (Z)
    """
    shared = x | texture_health | normal
    
    # Vertex 0: yR, kL
    buffer.data[buffer.used] = yR | kL | shared
    # Vertex 1: yL, kL
    buffer.data[buffer.used + 1] = yL | kL | shared
    # Vertex 2: yR, kR
    buffer.data[buffer.used + 2] = yR | kR | shared
    # Vertex 3: yL, kR
    buffer.data[buffer.used + 3] = yL | kR | shared
    # Vertex 4: yL, kL (duplicate for triangle fan/strip)
    buffer.data[buffer.used + 4] = yL | kL | shared
    # Vertex 5: yR, kR (duplicate)
    buffer.data[buffer.used + 5] = yR | kR | shared
    
    buffer.used += 6

def append_quad_z(buffer, xL, xR, yL, yR, z, normal, texture_health):
    """
    Appends 6 vertices for a Z-face quad.
    Shared bits: z, texture_health, normal.
    Varying bits: x, y
    """
    shared = z | texture_health | normal
    
    # Vertex 0: xR, yR
    buffer.data[buffer.used] = xR | yR | shared
    # Vertex 1: xR, yL
    buffer.data[buffer.used + 1] = xR | yL | shared
    # Vertex 2: xL, yR
    buffer.data[buffer.used + 2] = xL | yR | shared
    # Vertex 3: xL, yL
    buffer.data[buffer.used + 3] = xL | yL | shared
    # Vertex 4: xL, yL (duplicate)
    buffer.data[buffer.used + 4] = xL | yL | shared
    # Vertex 5: xR, yR (duplicate)
    buffer.data[buffer.used + 5] = xR | yR | shared
    
    buffer.used += 6

def different_block(block_a, block_b):
    """
    Returns True if blocks are different in type or visibility.
    """
    return block_a.index != block_b.index or block_a.kind == EMPTY

def draw_face_xn(j, access, minX, kCS2):
    """
    Placeholder for visibility check.
    Returns True if the X- face should be drawn.
    """
    # Implementation depends on neighbor chunk data
    return True # Simplified for pseudocode

def draw_face_xp(j, access, maxX, kCS2):
    return True

def draw_face_zn(j, access, minZ, kCS2):
    return True

def draw_face_zp(j, access, maxZ, kCS2):
    return True
```

### Formulas

**Vertex Position Extraction** (from packed integer `aData`):
*   $x = \text{aData} \ \& \ 63$
*   $y = (\text{aData} \gg 6) \ \& \ 63$
*   $z = (\text{aData} \gg 12) \ \& \ 63$
*   $\text{texID} = (\text{aData} \gg 18) \ \& \ 31$
*   $\text{brightness} = ((\text{aData} \gg 23) \ \& \ 15 + 2) / 8.0$
*   $\text{normal} = (\text{aData} \gg 27) \ \& \ 7$ [S2, unofficial]

**Brightness Modulation**:
*   If $\text{normal} == 0$ (Y-), $\text{brightness} *= 1.3$
*   If $\text{normal} == 1$ (Y+), $\text{brightness} *= 0.85$
*   If $\text{normal} < 4$ (X- or X+), $\text{brightness} *= 1.15$ [S2, unofficial]

### Parameters
*   **Chunk Size**: 32x32x32 blocks [S2, unofficial].
*   **Constants**:
    *   $\text{CHUNK\_SIZE} = 32$
    *   $\text{CHUNK\_SIZE\_SQUARED} = 1024$
    *   $\text{CHUNK\_SIZE\_CUBED} = 32768$
    *   $\text{CHUNK\_SIZE\_SHIFTED} = 1024$ [S2, unofficial]

### Edge Cases and Pitfalls
*   **Buffer Overflow**: The vertex buffer must be extended if `used` exceeds `data.Length - 2048` [S2, unofficial].
*   **Boundary Conditions**: Neighbor chunk references must be handled when the chunk is at the map edge (neighbor is `null`) [S2, unofficial].
*   **Duplicate Faces**: The `visit` arrays prevent generating duplicate faces for adjacent blocks within the same chunk, but cross-chunk face generation relies on correct neighbor access and visibility checks [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: The algorithm iterates through each block in the chunk once, with inner loops extending along runs. The complexity is roughly $O(N)$ where $N$ is the number of blocks, but with a small constant factor due to run extension.
*   **Memory Complexity**: $O(N)$ for the chunk data, height maps, and visitation arrays. The vertex buffer size depends on the number of quads generated.
*   **Performance**: On a Ryzen 5 1600 CPU, average Chunk Mesh Generation Time (CMGT) ranges from 0.278ms to 0.535ms depending on the map density [S2, unofficial].
*   **Strengths**: Extremely fast generation speed (~390% faster than standard greedy meshing) [S2, unofficial]. Suitable for real-time regeneration in multiplayer environments [S2, unofficial].
*   **Limitations**: Produces ~20% more triangles than standard greedy meshing [S2, unofficial]. Not optimal for minimal triangle count.
*   **When to Use**: When mesh generation speed is critical, and a slight increase in triangle count is acceptable (e.g., dynamic voxel worlds with frequent updates) [S2, unofficial].
*   **When Not to Use**: When minimal triangle count is the primary goal, or when static meshes are preferred [S2, unofficial].

## Background
This algorithm was developed for the game *Sector's Edge* (later known as *Vercidium*) and is described in a blog post by Vercidium [S2, unofficial]. It is optimized for multiple chunk mesh regenerations per frame with multiple players [S2, unofficial]. The approach is distinct from standard greedy meshing by prioritizing generation speed over mesh minimality, using a run-based approach along X and Y axes within chunks [S2, unofficial].

## Sources
[S1] Greedy Mesh generation for a voxel-based game, https://gamedev.stackexchange.com/questions/106168/greedy-mesh-generation-for-a-voxel-based-game, fetched 2026-10-26 (unofficial)
[S2] Vercidium/voxel-mesh-generation, https://github.com/Vercidium/voxel-mesh-generation, fetched 2026-10-26 (unofficial)
