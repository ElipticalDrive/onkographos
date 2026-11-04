# Voxel Face Culling with Chunked Rendering Order

Also called: Camera-relative Chunk Iteration, Chunked Rendering Order, Neighbor-based Face Culling

## Overview
This method optimizes blocky voxel mesh rendering by generating geometry exclusively for voxel faces adjacent to empty space or chunk boundaries, while traversing the world in a camera-relative front-to-back order to improve GPU depth sorting efficiency [S1, unofficial]. It builds on [Chunk-Based Parallel Voxel Culling & Rendering](3994-chunk-based-parallel-voxel-culling-rendering.md) by adding camera-relative chunk traversal logic and boundary-aware face culling [S1, unofficial].

## How it works
**Inputs:** Camera look vector $\vec{L} = (L_x, L_y, L_z)$; array of `Chunk` objects; `Constants.ChunkSize` (linear dimension $N$ in voxels) [S1, unofficial].
**Outputs:** Draw calls for each chunk's `voxelBuffer` [S1, unofficial].

**Data Structures:**
*   `Voxel`: Contains `index` (byte/uint). Empty voxels have `index == 0` [S1, unofficial].
*   `Chunk`: Contains `Allocated` (bool), `voxels` (pointer to flat array of `Voxel`), `WorldPos` (vec3), `IsDirty` (bool), `fake` (bool), `voxelBuffer` (GPU buffer handle) [S1, unofficial].
*   `Constants`:
    *   `ChunkSize`: Linear dimension $N$.
    *   `ChunkSizeM1`: $N - 1$.
    *   `ChunkSizeSquared`: $N^2$.
    *   `ACCESS_STEP_X`: $1$.
    *   `ACCESS_STEP_Y`: $N$.
    *   `ACCESS_STEP_Z`: $N^2$.

**Algorithm:**

1.  **Calculate Chunk Iteration Bounds:**
    Determine traversal order based on the camera look vector $\vec{L} = (L_x, L_y, L_z)$ [S1, unofficial].
    *   For X axis: If $L_x > 0$, $startX=0$, $endX=ChunkAmountX$, $xStep=1$, $xAccessStep=N$. Otherwise, reverse indices [S1, unofficial].
    *   For Z axis: If $L_z > 0$, $startZ=0$, $endZ=ChunkAmountZ$, $zStep=1$, $zAccessStep=N$. Otherwise, reverse indices [S1, unofficial].
    *   Y axis iteration order is implementation-dependent but typically runs inner or outer relative to the X/Z camera-relative loops [S1, unofficial].

2.  **Iterate Chunks:**
    Loop over chunk indices $k$ (Z), $i$ (X), and $j$ (Y) using the calculated bounds and steps [S1, unofficial].
    *   Skip chunks where `chunk.Allocated` is false [S1, unofficial].
    *   If `chunk.IsDirty()` is true, regenerate the mesh [S1, unofficial].
    *   Bind `chunk.voxelBuffer` and issue a draw call if the buffer is not null [S1, unofficial].

3.  **Mesh Chunk (Face Culling):**
    For each voxel in the chunk, determine which faces to render [S1, unofficial]. A face is rendered if the neighbor voxel is empty (`index == 0`) or the neighbor chunk is outside the world [S1, unofficial].
    
    **Boundary Handling:**
    When checking a neighbor at a chunk boundary (e.g., X- direction at $x=0$):
    *   Identify the neighboring chunk (e.g., `chunkXN`) [S1, unofficial].
    *   If `chunkXN.Allocated` is false, treat the neighbor as empty and render the face immediately [S1, unofficial].
    *   If `chunkXN.Allocated` is true, fetch the voxel from the neighbor chunk's data pointer [S1, unofficial]. For the X- boundary, the neighbor voxel is at local index $x=N-1$ in the neighbor chunk [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_chunks(camera_look_vector, chunks, chunk_amounts, constants):
    # 1. Calculate iteration bounds
    L = camera_look_vector
    N = constants.ChunkSize
    N2 = N * N
    N_minus_1 = N - 1
    
    # X axis
    if L.x > 0:
        x_start, x_end, x_step = 0, chunk_amounts.x, 1
    else:
        x_start, x_end, x_step = chunk_amounts.x - 1, -1, -1
        
    # Z axis
    if L.z > 0:
        z_start, z_end, z_step = 0, chunk_amounts.z, 1
    else:
        z_start, z_end, z_step = chunk_amounts.z - 1, -1, -1
        
    # 2. Iterate chunks
    for k in range(z_start, z_end, z_step):
        for i in range(x_start, x_end, x_step):
            for j in range(chunk_amounts.y):
                chunk = chunks[i][j][k]
                
                if not chunk.allocated:
                    continue
                    
                if chunk.is_dirty:
                    mesh_chunk(chunk, constants)
                    
                if chunk.voxel_buffer is not None:
                    gpu_bind_and_draw(chunk.voxel_buffer)

def mesh_chunk(chunk, constants):
    N = constants.ChunkSize
    N2 = N * N
    N_minus_1 = N - 1
    
    voxels = chunk.voxels
    faces = []
    
    for k in range(N):
        for i in range(N):
            for j in range(N):
                idx = i + j * N + k * N2
                voxel = voxels[idx]
                
                # X- Direction
                if i == 0:
                    neighbor_chunk = chunk.chunk_xn
                    if not neighbor_chunk.allocated:
                        faces.append((idx, (-1, 0, 0)))
                    else:
                        neighbor_idx = N_minus_1 + j * N + k * N2
                        if neighbor_chunk.voxels[neighbor_idx].index == 0:
                            faces.append((idx, (-1, 0, 0)))
                else:
                    neighbor_idx = (i - 1) + j * N + k * N2
                    if voxels[neighbor_idx].index == 0:
                        faces.append((idx, (-1, 0, 0)))
                        
                # X+ Direction
                if i == N_minus_1:
                    neighbor_chunk = chunk.chunk_xp
                    if not neighbor_chunk.allocated:
                        faces.append((idx, (1, 0, 0)))
                    else:
                        neighbor_idx = 0 + j * N + k * N2
                        if neighbor_chunk.voxels[neighbor_idx].index == 0:
                            faces.append((idx, (1, 0, 0)))
                else:
                    neighbor_idx = (i + 1) + j * N + k * N2
                    if voxels[neighbor_idx].index == 0:
                        faces.append((idx, (1, 0, 0)))
                        
                # Y- Direction
                if j == 0:
                    neighbor_chunk = chunk.chunk_yn
                    if not neighbor_chunk.allocated:
                        faces.append((idx, (0, -1, 0)))
                    else:
                        neighbor_idx = i + N_minus_1 * N + k * N2
                        if neighbor_chunk.voxels[neighbor_idx].index == 0:
                            faces.append((idx, (0, -1, 0)))
                else:
                    neighbor_idx = i + (j - 1) * N + k * N2
                    if voxels[neighbor_idx].index == 0:
                        faces.append((idx, (0, -1, 0)))
                        
                # Y+ Direction
                if j == N_minus_1:
                    neighbor_chunk = chunk.chunk_yp
                    if not neighbor_chunk.allocated:
                        faces.append((idx, (0, 1, 0)))
                    else:
                        neighbor_idx = i + 0 * N + k * N2
                        if neighbor_chunk.voxels[neighbor_idx].index == 0:
                            faces.append((idx, (0, 1, 0)))
                else:
                    neighbor_idx = i + (j + 1) * N + k * N2
                    if voxels[neighbor_idx].index == 0:
                        faces.append((idx, (0, 1, 0)))
                        
                # Z- Direction
                if k == 0:
                    neighbor_chunk = chunk.chunk_zn
                    if not neighbor_chunk.allocated:
                        faces.append((idx, (0, 0, -1)))
                    else:
                        neighbor_idx = i + j * N + N_minus_1 * N2
                        if neighbor_chunk.voxels[neighbor_idx].index == 0:
                            faces.append((idx, (0, 0, -1)))
                else:
                    neighbor_idx = i + j * N + (k - 1) * N2
                    if voxels[neighbor_idx].index == 0:
                        faces.append((idx, (0, 0, -1)))
                        
                # Z+ Direction
                if k == N_minus_1:
                    neighbor_chunk = chunk.chunk_zp
                    if not neighbor_chunk.allocated:
                        faces.append((idx, (0, 0, 1)))
                    else:
                        neighbor_idx = i + j * N + 0 * N2
                        if neighbor_chunk.voxels[neighbor_idx].index == 0:
                            faces.append((idx, (0, 0, 1)))
                else:
                    neighbor_idx = i + j * N + (k + 1) * N2
                    if voxels[neighbor_idx].index == 0:
                        faces.append((idx, (0, 0, 1)))
                        
    return faces

def gpu_bind_and_draw(buffer):
    # Standard graphics API call to bind vertex array object and issue draw command [G]
    pass
```

**Edge Cases:**
*   **Unallocated Neighbors:** If a neighbor chunk is not allocated, the face is always rendered, treating the out-of-bounds region as empty space [S1, unofficial].
*   **Edge Ownership & Padding:** Some implementations pad the block array by `sectionSide + 1` to handle inter-chunk edges, and explicitly ignore checks at `j == sectionSide` to prevent double-rendering or recursive checks between chunks [S2, unofficial].
*   **Precision:** Use integer arithmetic for indices to avoid floating-point errors in boundary calculations [G].

## Operating characteristics
*   **Time Complexity:** $O(V)$ where $V$ is the number of visible voxels [S1, unofficial]. Chunk iteration is $O(C)$ where $C$ is the number of chunks [S1, unofficial].
*   **Memory Complexity:** $O(C \cdot N^3)$ for voxel data, plus $O(F)$ for mesh faces where $F$ is the number of faces [S1, unofficial].
*   **Performance:** Improves GPU depth sorting by rendering front-to-back relative to the camera [S1, unofficial]. Reduces overdraw and culling overhead by only generating faces for solid-void interfaces [S1, unofficial].
*   **Strengths:** Efficient for large, sparse worlds [S1, unofficial]. Simple boundary handling [S1, unofficial].
*   **Limitations:** Requires neighbor chunk pointers [S1, unofficial]. Mesh regeneration is needed when chunks change [S1, unofficial].

## Background
This method is used in voxel-based game engines and editors, specifically referenced in the vercidium-patreon/meshing project [S1, unofficial]. It addresses the performance bottleneck of rendering large voxel worlds by combining spatial partitioning (chunks) with visibility optimization (culling and rendering order) [S1, unofficial].

## Sources
[S1] vercidium-patreon/meshing, https://github.com/vercidium-patreon/meshing, fetched 2026-10-26 (unofficial)
[S2] Manolo94/UE5MinecraftClone, https://github.com/Manolo94/UE5MinecraftClone, fetched 2026-10-26 (unofficial)
