# Chunk Rendering with Multidraw Indirect

Also called: GPU Instanced Voxel Rendering via DrawMeshInstancedIndirect, Indirect Voxel Rendering

## Overview
Chunk Rendering with Multidraw Indirect is a technique for rendering voxel worlds that batches chunk geometry using OpenGL's `glMultiDrawElementsIndirect` to reduce draw call overhead [S1, unofficial]. It stores chunk geometry in a large shared Vertex Buffer Object (VBO) and uses indirect commands to specify per-chunk offsets and counts [S1, unofficial]. It is distinguished from its parent [GPU Instancing with Indirect Draw Calls](2205-gpu-instancing-with-indirect-draw-calls.md) by its specific application to voxel chunks, where geometry is aggregated into a single massive buffer rather than relying on per-instance vertex attributes for position, instead using a separate vertex attribute for chunk offsets [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A list of "remeshed chunks" containing visible voxel geometry (quads). Each chunk has a world position (`chunkPos`) and a mesh with a quad count.
*   **Output:** A single GPU draw call rendering all visible chunks.

**Data Structures:**
1.  **`blocksIbo` (Index Buffer):**
    *   Size: `maxIndices` elements.
    *   Layout: Pre-generated indices for a single quad mesh repeated for each potential chunk.
    *   Content: For each chunk index `i` from `0` to `maxIndices / 6`, the buffer contains 6 indices: `{0, 2, 1, 0, 3, 2}` offset by `i * 4`.
    *   Total Indices: `CHUNK_SIZE * 6 * 6` [S1, unofficial].
2.  **`hugeVbo` (Vertex Buffer):**
    *   Capacity: `std::pow(GetSettings()->GetRenderDistance(), 3ULL) * CHUNK_SIZE / 2` elements of type `BlockQuad` [S1, unofficial].
    *   Layout: Concatenated vertex data for all active chunks.
3.  **`multidrawCommands` (Indirect Command Buffer):**
    *   Layout: Array of `DrawElementsIndirectCommand` structures.
    *   Fields per command: `count` (indices to draw), `instanceCount` (1), `firstIndex` (0), `baseVertex` (offset into `hugeVbo`), `baseInstance` (chunk index `i`).
4.  **`multidrawChunkOffsets` (Vertex Buffer):**
    *   Layout: Array of `glm::vec3` (3 floats).
    *   Content: Chunk location offsets relative to the player.
5.  **`ChunkDrawCall` Structure:**
    *   Fields: Chunk pointer, number of indices to draw, chunk's shader position offset [S1, unofficial].

**Numbered Steps:**

1.  **Initialize Index Buffer:**
    *   Create `blocksIbo` with size `maxIndices`.
    *   Loop `i` from `0` to `maxIndices / 6`.
    *   For each `i`, write 6 indices to the buffer: `indices[0] + (i * 4)`, `indices[1] + (i * 4)`, ..., `indices[5] + (i * 4)`.

2.  **Copy Remeshed Chunks:**
    *   Iterate through each chunk in `remeshedChunks`.
    *   If `mesh->GetQuadCount() == 0`, skip the chunk [S1, unofficial].
    *   Ensure GL buffers exist.
    *   Reserve VBO/IBO capacity at `1.5 * mesh_count` [S1, unofficial].
    *   Copy mesh data to mapped buffers via `memcpy`.
    *   Write to `hugeVbo` range via `WriteToRange`.
    *   Swap next buffers.

3.  **Prepare Multidraw Commands:**
    *   Create `darray` for `chunkOffsets` and `commands`.
    *   Loop `i` from `0` to `chunksToDraw`.
    *   For each chunk, create a `DrawElementsIndirectCommand`:
        *   `count`: Number of indices to draw for this chunk.
        *   `instanceCount`: 1.
        *   `firstIndex`: 0.
        *   `baseVertex`: `chunkVboRange->GetOffset() * 4`.
        *   `baseInstance`: `i`.
    *   Add chunk offset to `chunkOffsets`.

4.  **Execute Multidraw:**
    *   Bind `blockMultidrawShader`.
    *   Bind `blocksIbo` and `hugeVbo`.
    *   Bind `multidrawChunkOffsets` as vertex buffer at binding point 4.
    *   Bind `multidrawCommands` as indirect buffer.
    *   Call `glMultiDrawElementsIndirect(GL_TRIANGLES, GL_UNSIGNED_INT, (const void*)0, chunksToDraw, 0)`.

**Formulas:**
*   **Shader Position Offset:** The position offset for a chunk in the shader is calculated as:
    $$ \text{vec3}(\text{chunkLocation} - \text{playerPos}) $$
    Where:
    $$ \text{chunkLocation} = \text{vec3}(\text{chunkPos}.x \times \text{CHUNK\_LENGTH}, \text{chunkPos}.y \times \text{CHUNK\_LENGTH}, \text{chunkPos}.z \times \text{CHUNK\_LENGTH}) $$
    [S1, unofficial]

**Parameters:**
*   **Huge VBO Reserve Capacity:** `std::pow(GetSettings()->GetRenderDistance(), 3ULL) * CHUNK_SIZE / 2` elements of type `BlockQuad` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty Chunks:** Chunks with 0 quads are skipped during buffer copy to avoid wasting buffer space and draw commands [S1, unofficial].
*   **Base Vertex Calculation:** The `baseVertex` in the indirect command must be multiplied by 4 (assuming 4 bytes per vertex component or similar stride logic) to correctly offset into the `hugeVbo` [S1, unofficial].
*   **Buffer Swapping:** Buffers must be swapped after copying to ensure the GPU does not read from a buffer currently being written to by the CPU [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_chunks_with_multidraw_indirect(remeshed_chunks, player_pos, chunk_length, huge_vbo_capacity):
    # Step 1: Initialize Index Buffer
    max_indices = huge_vbo_capacity * 6 # Assuming 6 indices per quad, but source says CHUNK_SIZE * 6 * 6
    # Note: Source specifies total indices as CHUNK_SIZE * 6 * 6. 
    # We will assume max_indices is pre-calculated based on render distance.
    blocks_ibo = create_buffer(max_indices)
    
    indices_base = [0, 2, 1, 0, 3, 2]
    for i in range(max_indices // 6):
        for j in range(6):
            blocks_ibo[i * 6 + j] = indices_base[j] + (i * 4)

    # Step 2: Copy Remeshed Chunks
    huge_vbo = create_buffer(huge_vbo_capacity)
    chunk_offsets = []
    commands = []
    current_vbo_offset = 0
    
    for chunk in remeshed_chunks:
        mesh = chunk.mesh
        if mesh.get_quad_count() == 0:
            continue
            
        # Reserve capacity (source mentions 1.5 * mesh_count, but we are appending)
        # Copy mesh data to huge_vbo
        mesh_data = mesh.get_vertex_data()
        huge_vbo.write_range(current_vbo_offset, mesh_data)
        
        # Step 3: Prepare Multidraw Commands
        # baseVertex is offset in bytes? Source says chunkVboRange->GetOffset() * 4
        # Assuming GetOffset() returns index count, and we multiply by 4 for bytes (vec3? or vec4?)
        # Source says baseVertex = chunkVboRange->GetOffset() * 4
        base_vertex = current_vbo_offset * 4 
        
        command = {
            'count': mesh.get_index_count(),
            'instanceCount': 1,
            'firstIndex': 0,
            'baseVertex': base_vertex,
            'baseInstance': len(commands)
        }
        commands.append(command)
        
        # Calculate chunk offset
        chunk_pos = chunk.pos
        chunk_location = vec3(chunk_pos.x * chunk_length, chunk_pos.y * chunk_length, chunk_pos.z * chunk_length)
        offset = chunk_location - player_pos
        chunk_offsets.append(offset)
        
        current_vbo_offset += mesh.get_vertex_count()

    # Step 4: Execute Multidraw
    multidraw_commands_buf = create_buffer(commands)
    multidraw_chunk_offsets_buf = create_buffer(chunk_offsets)
    
    bind_shader(block_multidraw_shader)
    bind_index_buffer(blocks_ibo)
    bind_vertex_buffer(0, huge_vbo) # Attributes 0-3
    bind_vertex_buffer(4, multidraw_chunk_offsets_buf) # Attribute 4
    
    # Bind indirect buffer
    # Note: glMultiDrawElementsIndirect takes the indirect buffer as an array of commands
    # The source binds multidrawCommands as indirect buffer.
    
    gl_multi_draw_elements_indirect(
        primitive_type=GL_TRIANGLES,
        index_type=GL_UNSIGNED_INT,
        indirect=(const void*)0, # Points to multidraw_commands_buf
        draw_count=len(commands),
        stride=0 # Assuming tightly packed commands
    )
```

## Subtopics

### Chunk Draw Call Preparation
A variant step where the `ChunkDrawCall` structure is initialized with the chunk pointer, the number of indices to draw, and the chunk's shader position offset [S1, unofficial]. This structure is used to aggregate data before creating the final indirect commands.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of visible chunks, as each chunk is processed once to build the command buffer. The draw call itself is $O(1)$ relative to the number of chunks, though the GPU work is proportional to the total number of vertices and indices.
*   **Memory Complexity:** $O(N)$ for the `hugeVbo` and command buffers, where $N$ is the number of chunks. The VBO size is pre-allocated based on render distance [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales well with the number of chunks because it reduces the CPU overhead of issuing many draw calls. The bottleneck shifts to the GPU's ability to process the large vertex buffer and the bandwidth of the VBO.
*   **Strengths:** Significantly reduces draw call overhead compared to individual draw calls or instanced drawing with base vertex [S1, unofficial].
*   **Limitations:** Requires a large contiguous VBO, which may be difficult to allocate for very large render distances. The source disables individual draw calls and instanced drawing with `glDrawElementsInstancedBaseVertexBaseInstance` via `#if false`, suggesting this method is preferred for this specific implementation [S1, unofficial].
*   **When to Use:** When rendering a large number of similar geometry chunks (voxels) where draw call overhead is a bottleneck.
*   **When Not to Use:** When the number of chunks is small, or when memory constraints prevent allocating a large VBO.

## Background
This technique is implemented in the `gabkhanfig/Cube` repository, an unofficial project [S1, unofficial]. It uses OpenGL's `glMultiDrawElementsIndirect` to batch chunk rendering. The source code explicitly disables alternative methods like individual draw calls and `glDrawElementsInstancedBaseVertexBaseInstance`, indicating a design choice to favor multidraw indirect for this voxel engine.

## Sources
[S1] gabkhanfig/Cube, https://github.com/gabkhanfig/Cube, fetched 2026-10-27 (unofficial)
