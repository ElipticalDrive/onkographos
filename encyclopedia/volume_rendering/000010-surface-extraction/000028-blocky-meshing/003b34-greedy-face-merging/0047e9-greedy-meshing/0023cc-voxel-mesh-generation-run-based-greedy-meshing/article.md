# Voxel Mesh Generation (Run-based Greedy Meshing)

Also called: Optimised voxel mesh generation, Sector's Edge meshing

## Overview
A CPU-based mesh generation algorithm for voxel worlds that combines adjacent visible faces into runs along the X and Y axes to reduce triangle count, while optimizing for generation speed over minimal triangle count [S1, unofficial]. It produces approximately 20% more triangles than full greedy meshing but runs approximately 390% faster, making it suitable for real-time scenarios requiring frequent chunk regeneration [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A chunk of voxel data ($32 \times 32 \times 32$ blocks), neighbor chunk references for all 6 directions, and visibility flags for each block face [S1, unofficial].
- **Output:** A vertex buffer containing packed vertex data for all visible faces, optimized into quads via run-length expansion [S1, unofficial].

**Data Structures:**
1. **Block Vertex Data:** Packed into a single 32-bit integer (`uint`) [S1, unofficial].
   - Bits 0–5 (6 bits): X position (0–31) [S1, unofficial].
   - Bits 6–11 (6 bits): Y position (0–31) [S1, unofficial].
   - Bits 12–17 (6 bits): Z position (0–31) [S1, unofficial].
   - Bits 18–22 (5 bits): Texture ID (0–31) [S1, unofficial].
   - Bits 23–26 (4 bits): Brightness/Health (0–15) [S1, unofficial].
   - Bits 27–29 (3 bits): Normal (0–7) [S1, unofficial].
2. **ChunkHelper:** Contains 6 boolean arrays of size $32^3 = 32,768$ (`visitXN`, `visitXP`, `visitZN`, `visitZP`, `visitYN`, `visitYP`) to track visited blocks for each face direction and avoid duplicate face generation [S1, unofficial].
3. **BlockVertexBuffer:** Stores vertex data in an `int[]` array. Uses OpenGL VAO/VBO. Vertex attribute 0 is a single integer (`uint`) with stride equal to `sizeof(uint)` [S1, unofficial].
4. **Height Maps:** `MinY` and `MaxY` arrays of size $32 \times 32 = 1,024$ bytes, storing the minimum and maximum occupied Y indices for each $(X, Z)$ column to skip empty space [S1, unofficial].

**Parameters:**
- Chunk Size: $32 \times 32 \times 32$ blocks [S1, unofficial].
- Constants: `CHUNK_SIZE` = 32, `CHUNK_SIZE_SQUARED` = 1024, `CHUNK_SIZE_CUBED` = 32768, `CHUNK_SIZE_SHIFTED` = $32 \ll 6$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants and lookup tables from source
CHUNK_SIZE = 32
CHUNK_SIZE_SQUARED = 1024
CHUNK_SIZE_CUBED = 32768
CHUNK_SIZE_SHIFTED = 32 << 6  # 2048
EMPTY = 0
indexToTextureShifted = [0] * 256  # Placeholder for block-index-to-texture lookup table

class VertexBuffer:
    def __init__(self, capacity=4096):
        self.data = [0] * capacity
        self.used = 0
    def extend(self, amount):
        self.data += [0] * amount
    def append_quad_x(self, x, yL, yR, kL, kR, normal, textureHealth):
        shared = x | textureHealth | normal
        self.data[self.used]     = yR | kL | shared
        self.data[self.used + 1] = self.data[self.used + 4] = yL | kL | shared
        self.data[self.used + 2] = self.data[self.used + 3] = yR | kR | shared
        self.data[self.used + 5] = yL | kR | shared
        self.used += 6
    def append_quad_z(self, xL, xR, yL, yR, z, normal, textureHealth):
        shared = z | textureHealth | normal
        self.data[self.used]     = xR | yR | shared
        self.data[self.used + 1] = self.data[self.used + 4] = xR | yL | shared
        self.data[self.used + 2] = self.data[self.used + 3] = xL | yR | shared
        self.data[self.used + 5] = xL | yL | shared
        self.used += 6

def draw_face_xn(j, access, minX, k, neighbors):
    if minX:
        return neighbors.get('XN') is None or neighbors['XN'].data[k * CHUNK_SIZE_SQUARED + 31 * CHUNK_SIZE + j].kind == EMPTY
    return chunk.data[access - 1].kind == EMPTY

def draw_face_xp(j, access, maxX, k, neighbors):
    if maxX:
        return neighbors.get('XP') is None or neighbors['XP'].data[k * CHUNK_SIZE_SQUARED + 0 * CHUNK_SIZE + j].kind == EMPTY
    return chunk.data[access + 1].kind == EMPTY

def draw_face_zn(j, access, minZ, i, neighbors):
    if minZ:
        return neighbors.get('ZN') is None or neighbors['ZN'].data[i * CHUNK_SIZE_SQUARED + 31 * CHUNK_SIZE + j].kind == EMPTY
    return chunk.data[access - CHUNK_SIZE_SQUARED].kind == EMPTY

def draw_face_zp(j, access, maxZ, i, neighbors):
    if maxZ:
        return neighbors.get('ZP') is None or neighbors['ZP'].data[i * CHUNK_SIZE_SQUARED + 0 * CHUNK_SIZE + j].kind == EMPTY
    return chunk.data[access + CHUNK_SIZE_SQUARED].kind == EMPTY

def different_block(access, current_block):
    return chunk.data[access].kind != current_block.kind

def generate_chunk_mesh(chunk, neighbors):
    vertex_buffer = VertexBuffer(capacity=4096)
    visitXN = [False] * CHUNK_SIZE_CUBED
    visitXP = [False] * CHUNK_SIZE_CUBED
    visitZN = [False] * CHUNK_SIZE_CUBED
    visitZP = [False] * CHUNK_SIZE_CUBED
    
    chunkY = chunk.chunkPosY * CHUNK_SIZE
    
    for k in range(CHUNK_SIZE):
        kCS2 = k * CHUNK_SIZE_SQUARED
        minZEdge = (k == 0)
        maxZEdge = (k == CHUNK_SIZE - 1)
        
        for i in range(CHUNK_SIZE):
            heightMapAccess = k * CHUNK_SIZE + i
            j = chunk.MinY[heightMapAccess]
            topJ = chunk.MaxY[heightMapAccess]
            minXEdge = (i == 0)
            maxXEdge = (i == CHUNK_SIZE - 1)
            
            iCS = i * CHUNK_SIZE
            access = kCS2 + iCS + j
            
            for j in range(j, topJ):
                b = chunk.data[access]
                if b.kind != EMPTY:
                    create_run(b, i, j, k, access, minXEdge, maxXEdge, minZEdge, maxZEdge, neighbors, visitXN, visitXP, visitZN, visitZP, vertex_buffer, chunkY)
                access += 1
                
                if vertex_buffer.used > len(vertex_buffer.data) - 2048:
                    vertex_buffer.extend(2048)

def create_run(b, i, j, k, access, minX, maxX, minZ, maxZ, neighbors, visitXN, visitXP, visitZN, visitZP, vertex_buffer, chunkY):
    textureHealth16 = indexToTextureShifted[b.index] | ((b.health // 16) << 23)
    jS1 = (j + 1) << 6
    
    # X- Face (Left)
    if not visitXN[access] and draw_face_xn(j, access, minX, k, neighbors):
        visitXN[access] = True
        chunkAccess = access + 1
        length = jS1
        while length < CHUNK_SIZE_SHIFTED:
            if different_block(chunkAccess, b):
                break
            visitXN[chunkAccess] = True
            chunkAccess += 1
            length += (1 << 6)
        vertex_buffer.append_quad_x(i, j << 6, length >> 6, k << 12, k << 12, 3, textureHealth16)

    # X+ Face (Right)
    if not visitXP[access] and draw_face_xp(j, access, maxX, k, neighbors):
        visitXP[access] = True
        chunkAccess = access + 1
        length = jS1
        while length < CHUNK_SIZE_SHIFTED:
            if different_block(chunkAccess, b):
                break
            visitXP[chunkAccess] = True
            chunkAccess += 1
            length += (1 << 6)
        vertex_buffer.append_quad_x(i, j << 6, length >> 6, k << 12, k << 12, 2, textureHealth16)

    # Z- Face (Back)
    if not visitZN[access] and draw_face_zn(j, access, minZ, i, neighbors):
        visitZN[access] = True
        chunkAccess = access + 1
        length = jS1
        while length < CHUNK_SIZE_SHIFTED:
            if different_block(chunkAccess, b):
                break
            visitZN[chunkAccess] = True
            chunkAccess += 1
            length += (1 << 6)
        vertex_buffer.append_quad_z(i, i, j << 6, length >> 6, k << 12, 5, textureHealth16)

    # Z+ Face (Front)
    if not visitZP[access] and draw_face_zp(j, access, maxZ, i, neighbors):
        visitZP[access] = True
        chunkAccess = access + 1
        length = jS1
        while length < CHUNK_SIZE_SHIFTED:
            if different_block(chunkAccess, b):
                break
            visitZP[chunkAccess] = True
            chunkAccess += 1
            length += (1 << 6)
        vertex_buffer.append_quad_z(i, i, j << 6, length >> 6, k << 12, 4, textureHealth16)
```

**Formulas:**
- **Vertex Position Extraction:**
  - $x = \text{aData} \ \& \ 63$
  - $y = (\text{aData} \gg 6) \ \& \ 63$
  - $z = (\text{aData} \gg 12) \ \& \ 63$
  - $\text{texID} = (\text{aData} \gg 18) \ \& \ 31$ [S1, unofficial]
  - $\text{brightness} = ((\text{aData} \gg 23) \ \& \ 15 + 2) / 8.0$ [S1, unofficial]
  - $\text{normal} = (\text{aData} \gg 27) \ \& \ 7$ [S1, unofficial]
- **Brightness Modulation:**
  - If $\text{normal} == 0$ (Y-), $\text{brightness} \times 1.3$ [S1, unofficial]
  - If $\text{normal} == 1$ (Y+), $\text{brightness} \times 0.85$ [S1, unofficial]
  - If $\text{normal} < 4$ (X- or X+), $\text{brightness} \times 1.15$ [S1, unofficial]

**Edge Cases and Pitfalls:**
- **Buffer Overflow:** The vertex buffer must be checked for capacity during generation. If `used > data.Length - 2048`, extend by 2048 integers [S1, unofficial].
- **Boundary Conditions:** Neighbor chunks may be `null` if the current chunk is at the map edge. These must be handled as empty space (visible face) [S1, unofficial].
- **Precision:** All coordinates are integers. No floating-point errors during mesh generation [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:**
  - Hardware: Ryzen 5 1600 CPU [S1, unofficial].
  - Crashed Freighter map (799 chunks): Average Chunk Mesh Generation Time (CMGT) of 0.527ms single-threaded [S1, unofficial].
  - Soltrium Temple map (648 chunks): Average CMGT of 0.432ms [S1, unofficial].
  - Aegis Desert map (441 chunks): Average CMGT of 0.456ms [S1, unofficial].
  - Magma Chamber map (293 chunks): Average CMGT of 0.535ms [S1, unofficial].
  - Arena map (108 chunks): Average CMGT of 0.278ms [S1, unofficial].
- **Scaling:** The algorithm is designed for multi-threading, with multiple chunks initialized on different threads [S1, unofficial].
- **Strengths:** Extremely fast generation speed (~390% faster than greedy meshing) [S1, unofficial]. Suitable for real-time applications with frequent updates [S1, unofficial].
- **Limitations:** Produces ~20% more triangles than greedy meshing [S1, unofficial]. Only merges faces in the Y direction for X and Z faces, leading to less optimal triangle counts than full greedy meshing [S1, unofficial].
- **When to use:** Real-time voxel games with many players and frequent chunk updates (e.g., Sector's Edge) [S1, unofficial].
- **When not to use:** Offline mesh generation where minimal triangle count is prioritized over generation speed [S1, unofficial].

## Background
- **Origin:** Developed for the game *Sector's Edge* by Vercidium [S1, unofficial].
- **History:** The algorithm was published in a blog post by Vercidium [S1, unofficial]. It was designed to handle the performance requirements of a multiplayer voxel game with 16 players, where multiple chunk meshes must be regenerated every frame on older hardware [S1, unofficial].
- **Usage:** Used in *Sector's Edge* [S1, unofficial].

## Sources
[S1] Vercidium/voxel-mesh-generation, https://github.com/Vercidium/voxel-mesh-generation, fetched 2026-10-26 (unofficial)
