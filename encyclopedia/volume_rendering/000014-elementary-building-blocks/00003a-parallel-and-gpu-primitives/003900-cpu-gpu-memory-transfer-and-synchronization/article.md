# CPU-GPU Memory Transfer and Synchronization

Also called: Staging Buffer Transfer

## Overview
This method manages the transfer of volumetric data (specifically Gaussian splatting pages) from CPU-accessible memory to the GPU's discrete memory via PCIe, ensuring rendering correctness while minimizing pipeline stalls. It differs from direct memory mapping used by integrated GPUs by explicitly utilizing host-visible staging buffers for discrete architectures [S1]. It addresses the problem of independent memory spaces in discrete GPUs, where CPU cannot directly write to GPU VRAM [S1].

## How it works
**Inputs:**
- Gaussian data buffer (memory-mapped on CPU) [S1].
- Page table data [S1].
- Command buffers for graphics and compute queues [S1].

**Outputs:**
- Updated GPU memory with Gaussian data [S1].
- Updated index buffer for depth-sorted rendering [S1].
- Presented image frame [S1].

**Data Structures:**
- **Staging Buffer:** Host-visible memory on GPU, used for writing data from CPU. Size limits the copy budget [S1].
- **Uniform Buffer:** Stores metadata for compute shader dispatches [S1].
- **Push Constant:** Defines parameters for the compute shader [S1].
- **Index Buffer:** Receives depth results post-dispatch [S1].

**Steps:**
1. **Page Update Initiation:** Record commands to draw a proxy mesh and reduce page IDs. Insert a pipeline barrier to ensure memory is flushed before processing results [S1].
2. **Command Submission:** Submit commands to a graphics and compute-capable queue [S1].
3. **CPU Wait:** Use a fence to wait for GPU completion before updating the page table on the CPU [S1].
4. **Data Transfer:** Begin transferring data to the staging buffer during the page table update. Multi-threaded transfers are handled by drivers [S1].
5. **Second Command Buffer:** After the page table update, copy Gaussians to a second command buffer. This buffer also contains depth sorting and drawing commands, separated by a barrier [S1].
6. **Concurrent Recording:** Record commands for the second buffer in separate threads while data transfer occurs [S1].
7. **Depth Sorting:** Execute depth sorting via compute shaders. This requires nine dispatches. Metadata is stored in a uniform buffer, defined by a push constant [S1].
8. **Index Buffer Update:** Post-dispatch, move depth results to the index buffer [S1].
9. **Rendering:** Render the final scene using the updated index buffer [S1].
10. **Presentation:** After updating the page table and completing transfers, submit the command buffer. This triggers a semaphore to present the image [S1].
11. **Double Buffering:** Process two frames simultaneously. Perform CPU work while the GPU presents the previous frame [S1].

**Parameters:**
- **Hardware:** NVIDIA GeForce GTX 1070 GPU, Intel Core i7-4770k CPU, DDR3 system memory, PCIe 3.0 [S1].
- **Proxy Mesh Resolution:** Low, Medium, High. Affects quality and performance [S1].

**Pitfalls:**
- **Pipeline Stalls:** Strict synchronization during frame rendering causes the GPU to wait for task completion, reducing parallelism [S1].
- **Copy Budget:** Limited by staging buffer size and transfer time. Prioritize important copies and delay LOD transitions under high load [S1].
- **Integrated GPUs:** This method is specific to discrete GPUs. Integrated GPUs share memory with the CPU and do not require staging buffers [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_frame(cpu_gaussian_buffer, page_table, command_queue):
    # Step 1: Record commands for proxy mesh and page ID reduction
    record_commands(command_queue, "draw_proxy_mesh")
    record_commands(command_queue, "reduce_page_ids")
    
    # Step 2: Insert barrier to flush memory
    insert_pipeline_barrier(command_queue)
    
    # Step 3: Submit commands and wait with fence
    submit_commands(command_queue)
    wait_for_fence(command_queue)
    
    # Step 4: Update page table on CPU
    update_page_table(page_table)
    
    # Step 5: Start data transfer to staging buffer (multi-threaded by driver)
    start_data_transfer(cpu_gaussian_buffer, staging_buffer)
    
    # Step 6: Record second command buffer in separate thread
    record_second_buffer_thread = Thread(target=record_second_buffer, args=(command_queue,))
    record_second_buffer_thread.start()
    
    # Step 7: Wait for transfer and recording
    record_second_buffer_thread.join()
    
    # Step 8: Insert barrier before depth sorting
    insert_pipeline_barrier(command_queue)
    
    # Step 9: Depth sorting via compute shader (9 dispatches)
    dispatch_compute_shader(command_queue, num_dispatches=9, 
                            uniform_buffer=metadata, 
                            push_constant=push_const)
    
    # Step 10: Update index buffer with depth results
    update_index_buffer(command_queue, depth_results)
    
    # Step 11: Render scene
    render_scene(command_queue, index_buffer)
    
    # Step 12: Submit and present
    submit_commands(command_queue)
    trigger_semaphore(command_queue)
    present_image()
```

## Subtopics

### [GPU Buffer Upload via Transfer Buffer](9032-gpu-buffer-upload-via-transfer-buffer.md)
A variant focusing on the specific mechanism of uploading buffers via transfer buffers.

### [GPU Asynchronous Streaming](9054-gpu-asynchronous-streaming.md)
A technique for truly asynchronous data transfer using separate OpenGL contexts for download and drawing threads, leveraging the GPU's Copy Engine (CE) on Fermi architecture.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Limited by staging buffer size [S1].
- **Performance:** Tested on NVIDIA GeForce GTX 1070, Intel Core i7-4770k, DDR3, PCIe 3.0 [S1]. Performance is crucial for 3DGS adoption on consumer hardware [S1].
- **Scaling:** Two frames can be processed simultaneously, allowing CPU work while GPU presents previous frame [S1].
- **Strengths:** Enables rendering on discrete GPUs with independent memory; allows concurrent CPU/GPU work [S1].
- **Limitations:** Staging buffer size limits copy budget; strict synchronization causes pipeline stalls [S1].
- **When to Use:** Discrete GPUs with independent memory.
- **When Not to Use:** Integrated GPUs (shared memory) [S1].

## Background
Originated in the context of 3D Gaussian Splatting (3DGS) rendering on virtual memory systems [S1]. The method addresses the specific hardware constraints of discrete GPUs, which have independent memory connected via PCIe, unlike integrated GPUs found in mobile devices [S1]. The approach was evaluated on consumer-grade hardware (GTX 1070, i7-4770k) to ensure broad compatibility [S1].

## Sources
[S1] Virtual memory for 3D Gaussian Splatting, https://zenodo.org/records/19690968, fetched 2026-10-26
