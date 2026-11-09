# GPU-Based Chunk Culling via Indirect Commands

Also called: Compute Shader Culling, Indirect Draw Culling

## Overview
A rendering optimization that uses a compute shader to filter visible chunks and generates indirect draw arguments on the GPU, reducing CPU-GPU synchronization overhead [S1, unofficial]. It differs from CPU-side culling by offloading the visibility test and command buffer population to the GPU, allowing the CPU to remain free for scene management and chunk loading. It builds on [GPU-driven Visibility Culling](3640-gpu-driven-visibility-culling.md) by utilizing the metadata prepared there to drive the compute dispatch and indirect execution.

## How it works
**Inputs and Outputs:**
*   **Input:** A list of loaded chunks (managed by the CPU) and their associated render resources.
*   **Output:** An indirect command buffer containing `D3D12_DRAW_INDEXED_ARGUMENTS` for only the visible chunks, ready for `ExecuteIndirect`.

**Data Structures:**

1.  **`IndirectCommand`**: The structure written by the compute shader to the output buffer. It contains:
    *   `render_resources`: A struct of type `VoxelRenderResources` containing:
        *   `scene_constant_buffer_index`: Index into the scene constant buffer.
        *   `chunk_constant_buffer_index`: Index into the chunk constant buffer.
    *   `index_buffer_view`: A `D3D12_INDEX_BUFFER_VIEW` describing the index buffer for the chunk mesh.
    *   `draw_arguments`: A `D3D12_DRAW_INDEXED_ARGUMENTS` struct containing:
        *   `IndexCountPerInstance`: Number of indices.
        *   `InstanceCount`: Number of instances (usually 1).
        *   `StartIndexLocation`: Base index in the index buffer.
        *   `BaseVertexLocation`: Base vertex offset.
        *   `StartInstanceLocation`: Base instance ID.

2.  **`GPUCullRenderResources`**: Struct passed to the compute shader via root constants (64 bits / 16 32-bit values). It contains:
    *   `number_of_chunks`: `u32`, the total number of chunks to process.
    *   `indirect_command_srv_index`: `u32`, descriptor heap index for the input command buffer (SRV).
    *   `output_command_uav_index`: `u32`, descriptor heap index for the output command buffer (UAV).
    *   `scene_constant_buffer_index`: `u32`, descriptor heap index for the scene constant buffer (CBV).

**Numbered Steps:**

1.  **Build Indirect Command Vector (CPU):** Iterate through the list of loaded chunks. For each chunk, populate an `IndirectCommand` struct with the necessary render resources and draw arguments. Store these in a contiguous vector (`indirect_command_vector`) on the CPU [S1, unofficial].
2.  **Upload Data (CPU):** Transition the upload resource of the indirect command buffer to `COPY_DEST` state. Copy the `indirect_command_vector` data to the mapped pointer of the upload resource using `memcpy` [S1, unofficial].
3.  **Setup Compute Shader (CPU):**
    *   Set the descriptor heaps containing the shader-visible descriptors [S1, unofficial].
    *   Set the compute root signature (bindless) [S1, unofficial].
    *   Set the pipeline state object (PSO) for the culling compute shader [S1, unofficial].
    *   Set the compute root constants (64 bits) with the `GPUCullRenderResources` struct [S1, unofficial].
4.  **Reset Counter (CPU):** Reset the UAV counter associated with the output buffer. This is done by copying a zeroed 4-byte value from a `zeroed_counter_buffer_resource` to the `counter_offset` of the `default_resource` [S1, unofficial].
5.  **Dispatch Compute Shader (CPU/GPU):**
    *   Transition the `default_resource` of the indirect command buffer from `COPY_DEST` to `UNORDERED_ACCESS` [S1, unofficial].
    *   Dispatch the compute shader. The thread group count is calculated as `(indirect_command_vector.size() + 31) / 32` in the X dimension, with 1 in Y and Z [S1, unofficial].
6.  **Execute Indirect Draw (CPU/GPU):**
    *   Transition the `default_resource` from `UNORDERED_ACCESS` to `INDIRECT_ARGUMENT` [S1, unofficial].
    *   Set the graphics root signature and PSO [S1, unofficial].
    *   Set the primitive topology to `D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST` [S1, unofficial].
    *   Call `ExecuteIndirect` with:
        *   `command_signature`: The signature defining the indirect draw structure.
        *   `max_command_count`: `MAX_CHUNKS_TO_BE_DRAWN` (maximum number of indirect draw calls per frame) [S1, unofficial].
        *   `pIndirectCommandBuffer`: The `default_resource` of the indirect command buffer.
        *   `IndirectCommandOffset`: 0.
        *   `pIndirectArgumentBuffer`: The same `default_resource`.
        *   `IndirectArgumentOffset`: The `counter_offset` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# CPU Side
def gpu_based_chunk_culling(indirect_command_vector, renderer, command_list):
    # 1. Upload data
    # Transition upload resource to COPY_DEST
    # memcpy(upload_resource_mapped_ptr, indirect_command_vector.data(), 
    #        indirect_command_vector.size() * sizeof(IndirectCommand))
    
    # 2. Setup Compute Shader
    command_list.set_descriptor_heaps([shader_visible_descriptor_heaps])
    command_list.set_compute_root_signature(renderer.m_bindless_root_signature)
    command_list.set_pipeline_state(gpu_culling_pso)
    
    gpu_cull_render_resources = {
        'number_of_chunks': len(indirect_command_vector),
        'indirect_command_srv_index': indirect_command_buffer.upload_resource_srv_index,
        'output_command_uav_index': indirect_command_buffer.default_resource_uav_index,
        'scene_constant_buffer_index': scene_buffer.cbv_index
    }
    command_list.set_compute_root_constants(0, 64, gpu_cull_render_resources)
    
    # 3. Reset Counter
    command_list.copy_buffer_region(
        dst=indirect_command_buffer.default_resource,
        dst_offset=indirect_command_buffer.counter_offset,
        src=indirect_command_buffer.zeroed_counter_buffer_resource,
        src_offset=0,
        size=4
    )
    
    # 4. Dispatch Compute Shader
    # Transition default_resource from COPY_DEST to UNORDERED_ACCESS
    command_list.dispatch(
        num_threads_x=(len(indirect_command_vector) + 31) // 32,
        num_threads_y=1,
        num_threads_z=1
    )
    
    # 5. Execute Indirect
    # Transition default_resource from UNORDERED_ACCESS to INDIRECT_ARGUMENT
    command_list.set_descriptor_heaps([shader_visible_descriptor_heaps])
    command_list.set_graphics_root_signature(renderer.m_bindless_root_signature)
    command_list.set_pipeline_state(render_pso)
    command_list.ia_set_primitive_topology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST)
    
    command_list.execute_indirect(
        command_signature=command_signature,
        max_command_count=MAX_CHUNKS_TO_BE_DRAWN,
        p_indirect_command_buffer=indirect_command_buffer.default_resource,
        indirect_command_buffer_offset=0,
        p_indirect_argument_buffer=indirect_command_buffer.default_resource,
        indirect_argument_buffer_offset=indirect_command_buffer.counter_offset
    )
```

**Compute Shader Logic (Conceptual):**
The compute shader processes each chunk. It likely performs a visibility test (e.g., frustum culling or occlusion) using the `scene_constant_buffer` and chunk metadata. If visible, it appends the corresponding `IndirectCommand` to the UAV output buffer using `InterlockedAdd` on the counter. If not visible, it skips writing.

**Parameters:**
*   `MAX_CHUNKS_TO_BE_DRAWN`: Maximum number of indirect draw calls per frame. Used as the second argument to `ExecuteIndirect` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Resource State Transitions:** Must strictly follow D3D12 rules: `INDIRECT_ARGUMENT` -> `COPY_DEST` -> `UNORDERED_ACCESS` -> `INDIRECT_ARGUMENT`. Missing transitions cause validation errors or undefined behavior [S1, unofficial].
*   **Counter Reset:** The UAV counter must be reset before each dispatch to ensure correct indexing in the output buffer.
*   **Thread Group Size:** The dispatch size is calculated based on 32 threads per group. Ensure the vector size is handled correctly for non-multiples of 32.

## Operating characteristics
*   **Time Complexity:** O(N) where N is the number of loaded chunks, performed on the GPU.
*   **Memory Complexity:** O(N) for the indirect command buffer storage.
*   **Performance:** Reduces CPU-GPU synchronization overhead by moving culling logic to the GPU.
*   **Strengths:** Efficient for large numbers of chunks; leverages GPU parallelism for culling.
*   **Limitations:** Requires D3D12 or equivalent API support for indirect commands and compute shaders. State management is complex.
*   **When to Use:** When CPU is a bottleneck and the number of chunks is large.
*   **When Not to Use:** For small scenes where CPU culling is sufficient.

## Background
Originated in modern voxel engines utilizing D3D12 for efficient rendering of large voxel worlds. The technique is demonstrated in the `rtarun9/voxel-engine` repository [S1, unofficial].

## Sources
[S1] rtarun9/voxel-engine, https://github.com/rtarun9/voxel-engine, fetched 2026-10-27 (unofficial)
