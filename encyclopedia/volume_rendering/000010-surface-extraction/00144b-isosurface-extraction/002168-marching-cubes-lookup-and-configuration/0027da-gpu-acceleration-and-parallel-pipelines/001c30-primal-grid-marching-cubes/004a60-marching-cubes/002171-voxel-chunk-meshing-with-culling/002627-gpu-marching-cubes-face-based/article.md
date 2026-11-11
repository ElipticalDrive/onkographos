# GPU Marching Cubes (Face-based)

Also called: Chunk-based Surface Extraction

## Overview
This method extracts a triangle mesh from a 3D voxel grid by processing each voxel independently on the GPU using a compute shader [S1, unofficial]. It outputs vertices and indices into global buffers, using atomic counters to ensure parallel write safety when multiple threads generate faces simultaneously [S1, unofficial]. Unlike CPU-based face culling which iterates sequentially, this approach leverages massive parallelism to handle large voxel chunks (e.g., $80^3$) efficiently [S1, unofficial]. It differs from [Voxel Chunk Meshing with Culling](2523-voxel-chunk-meshing-with-culling.md) by moving the iteration and geometry generation into a GPU compute pipeline, and from [Marching Cubes](2520-marching-cubes.md) by relying on axis-aligned face generation rather than polygon interpolation within voxels [G].

## How it works
### Inputs and Outputs
*   **Input:** A 3D voxel grid stored in a `std430` buffer (`VoxelBuffer`, binding 1). Voxels with a value $>0$ are solid; $0$ is empty [S1, unofficial].
*   **Input:** Chunk feedback data (`ChunkFeedbackBuffer`, binding 7) containing `vertexOffset`, `indexOffset`, `vertexCount`, and `indexCount` for each chunk [S1, unofficial]. The buffer is bound as `readonly std430` [S1, unofficial].
*   **Output:** A `VertexBuffer` (binding 2, writeonly `std430`) storing generated vertices [S1, unofficial].
*   **Output:** An `IndexBuffer` (binding 3, writeonly `std430`) storing generated indices [S1, unofficial].

### Data Structures
**Vertex Layout:** Each vertex consists of 10 floats (40 bytes): `position` (`float3`), `texcoord` (`float2`), and `normal` (`float3`) [S1, unofficial].
**Chunk Feedback Layout:** `vertexOffset` (`uint`), `vertexCount` (`uint`), `indexOffset` (`uint`), `indexCount` (`uint`) [S1, unofficial].
**Draw Command Layout:** `indexCount` (`uint`), `instanceCount` (`uint`), `firstIndex` (`uint`), `vertexOffset` (`int`), `firstInstance` (`uint`) [S1, unofficial].

### Algorithm Steps
1.  **Initialization:** Thread 0 in each workgroup calculates the global chunk index using `gl_WorkGroupID` with the formula `sChunkIndex = gl_WorkGroupID.x + 10 * (gl_WorkGroupID.y + 10 * gl_WorkGroupID.z)` [S1, unofficial]. It then loads the chunk's `vertexOffset`, `indexOffset`, `vertexCount`, and `indexCount` from the `ChunkFeedbackBuffer` into shared memory variables (`sVertexOffset`, `sIndexOffset`, `sVertexCount`, `sIndexCount`) [S1, unofficial].
2.  **Voxel Check:** Each thread calculates its global voxel index within the chunk using the 1D mapping formula [S1, unofficial]. If the voxel value is $>0$, it proceeds to face extraction; otherwise, it skips [S1, unofficial].
3.  **Face Extraction:** For each of the 6 face directions ($\pm X, \pm Y, \pm Z$):
    a. Calculate the neighbor's 3D coordinates [S1, unofficial].
    b. Check if the neighbor is within bounds $[0, \text{CHUNK\_SIZE})$. If out of bounds, treat as empty (exposed face) [S1, unofficial].
    c. If the neighbor is empty (value $0$), allocate space for 4 vertices and 6 indices using `atomicAdd` on the shared `sVertexOffset` and `sIndexOffset` [S1, unofficial].
    d. Write the 4 vertices to the global `VertexBuffer` at the allocated offset. The vertices are positioned relative to the current voxel's coordinate, with the normal pointing outward from the voxel [S1, unofficial].
    e. Write the 6 indices (`0,1,2, 2,3,0`) to the global `IndexBuffer` at the allocated offset [S1, unofficial].
4.  **Feedback Update:** After all threads in the workgroup finish, the updated offsets and counts are written back to the `ChunkFeedbackBuffer` [S1, unofficial].

### Formulas
**1D Index Calculation:**
$$ \text{idx} = x + \text{CHUNK\_SIZE} \times (y + \text{CHUNK\_SIZE} \times z) $$
where $x, y, z$ are the 3D coordinates within the chunk [S1, unofficial].

**3D Coordinate from 1D Index:**
$$ x = \text{idx} \% \text{CHUNK\_SIZE} $$
$$ y = (\text{idx} / \text{CHUNK\_SIZE}) \% \text{CHUNK\_SIZE} $$
$$ z = \text{idx} / (\text{CHUNK\_SIZE} \times \text{CHUNK\_SIZE}) $$
[S1, unofficial]

### Parameters
*   **CHUNK_SIZE:** The dimension of the voxel chunk processed by the compute shader. The source specifies a value of 80 [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gpu_face_extraction_thread(global_voxel_index, chunk_size, voxel_buffer, feedback_buffer):
    """
    Simulates the logic executed by one thread for one voxel.
    """
    # Step 2: Convert 1D index to 3D coordinates
    x = global_voxel_index % chunk_size
    y = (global_voxel_index // chunk_size) % chunk_size
    z = global_voxel_index // (chunk_size * chunk_size)
    
    # Check if voxel is solid
    if voxel_buffer[global_voxel_index] <= 0:
        return

    # Load shared offsets (simulated; in GPU these are shared memory variables)
    current_vertex_offset = feedback_buffer.vertex_offset
    current_index_offset = feedback_buffer.index_offset
    
    # Define 6 face directions: (dx, dy, dz, normal_x, normal_y, normal_z)
    faces = [
        (1, 0, 0, 1, 0, 0),   # +X
        (-1, 0, 0, -1, 0, 0), # -X
        (0, 1, 0, 0, 1, 0),   # +Y
        (0, -1, 0, 0, -1, 0), # -Y
        (0, 0, 1, 0, 0, 1),   # +Z
        (0, 0, -1, 0, 0, -1)  # -Z
    ]
    
    for dx, dy, dz, nx, ny, nz in faces:
        nx_coord = x + dx
        ny_coord = y + dy
        nz_coord = z + dz
        
        # Boundary Check: Treat out-of-bounds as empty
        if 0 <= nx_coord < chunk_size and 0 <= ny_coord < chunk_size and 0 <= nz_coord < chunk_size:
            neighbor_idx = nx_coord + chunk_size * (ny_coord + chunk_size * nz_coord)
            neighbor_is_solid = voxel_buffer[neighbor_idx] > 0
        else:
            neighbor_is_solid = False
            
        if not neighbor_is_solid:
            # Allocate vertices and indices using atomicAdd (simulated here by incrementing)
            # In GPU: atomicAdd(&sVertexOffset, 4)
            # In GPU: atomicAdd(&sIndexOffset, 6)
            
            # Write 4 vertices
            # Vertices are positioned relative to the current voxel's coordinate,
            # with the normal pointing outward from the voxel.
            # The exact positions form a unit quad on the face plane.
            write_vertex(current_vertex_offset, (x + (1 if nx == 1 else 0), y + (1 if ny == 1 else 0), z + (1 if nz == 1 else 0)), (nx, ny, nz))
            write_vertex(current_vertex_offset + 1, (x + (1 if nx == 1 else 0), y + (1 if ny == -1 else 0), z + (1 if nz == 1 else 0)), (nx, ny, nz))
            write_vertex(current_vertex_offset + 2, (x + (1 if nx == 1 else 0), y + (1 if ny == 1 else 0), z + (1 if nz == -1 else 0)), (nx, ny, nz))
            write_vertex(current_vertex_offset + 3, (x + (1 if nx == 1 else 0), y + (1 if ny == -1 else 0), z + (1 if nz == -1 else 0)), (nx, ny, nz))
            
            # Write 6 indices
            write_index(current_index_offset, current_vertex_offset)
            write_index(current_index_offset + 1, current_vertex_offset + 1)
            write_index(current_index_offset + 2, current_vertex_offset + 2)
            write_index(current_index_offset + 3, current_vertex_offset + 2)
            write_index(current_index_offset + 4, current_vertex_offset + 3)
            write_index(current_index_offset + 5, current_vertex_offset)
            
            # Update offsets (simulated)
            current_vertex_offset += 4
            current_index_offset += 6

    # Update feedback buffer (simulated)
    feedback_buffer.vertex_count += current_vertex_offset - feedback_buffer.vertex_offset
    feedback_buffer.index_count += current_index_offset - feedback_buffer.index_offset
    feedback_buffer.vertex_offset = current_vertex_offset
    feedback_buffer.index_offset = current_index_offset

def write_vertex(offset, position, normal):
    """Writes a vertex to the global vertex buffer at the given offset."""
    pass

def write_index(offset, vertex_idx):
    """Writes an index to the global index buffer at the given offset."""
    pass
```

### Edge Cases and Pitfalls
*   **Boundary Check:** The `hasVoxel` function must explicitly check if coordinates are within $[0, \text{CHUNK\_SIZE})$ to prevent out-of-bounds access when checking neighbors at chunk edges [S1, unofficial].
*   **Atomic Contention:** Multiple threads writing to the same global buffers require atomic operations (`atomicAdd`) to avoid race conditions and data corruption [S1, unofficial].
*   **Translucency:** The basic algorithm assumes opaque voxels. Translucent blocks require extended culling logic to handle visibility correctly [G].

### Subtopics
*   **Voxel Mesh Rendering with Texture Atlas & Per-Vertex Lighting:** Uses the generated mesh for rendering with texture atlases and lighting. [Link](2185-voxel-mesh-rendering-with-texture-atlas-per-vertex-lighting.md)
*   **Face Culling Voxel Meshing:** CPU-based face culling method. [Link](2268-face-culling-voxel-meshing.md)
*   **Naive Voxel Meshing:** Generates all faces without culling. [Link](2376-naive-voxel-meshing.md)

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the chunk, as each voxel is processed independently [S1, unofficial].
*   **Memory Complexity:** $O(F)$ where $F$ is the number of exposed faces, as vertices and indices are stored in global buffers [S1, unofficial].
*   **Performance:** The method is highly parallelizable, making it suitable for large voxel chunks (e.g., $80^3$) on modern GPUs [S1, unofficial].
*   **Strengths:** Efficient culling of internal faces, parallel processing, simple logic [S1, unofficial].
*   **Limitations:** Requires atomic operations for buffer writes, which can cause contention; boundary handling must be consistent across chunks to avoid gaps or double geometry [S1, unofficial].

## Background
This method is implemented in the `UnityVoxelMeshGPU` project by artnas [S1, unofficial]. It is designed for use in Unity's High Definition Render Pipeline (HDRP) for unlit materials [S1, unofficial]. The approach leverages GPU compute shaders to accelerate the mesh generation process compared to CPU-based methods [S1, unofficial].

## Sources
[S1] artnas/UnityVoxelMeshGPU, https://github.com/artnas/UnityVoxelMeshGPU, fetched 2026-10-27 (unofficial)
