# GPU Decompression Pipeline

Also called: Asynchronous data transfer, Multi-threaded loading and LZO decompression

## Overview
The GPU Decompression Pipeline is a hybrid CPU/GPU method for streaming and decompressing time-varying volume data, serving as the execution engine for the [Two-stage compression approach](6805-two-stage-compression-approach.md). It solves the bottleneck of transferring and unpacking compressed volume bricks from disk to GPU memory by overlapping I/O, decompression, and data upload. It sets itself apart from pure CPU decompression by offloading the resolution of variable-length codes, delta encoding, and brick assembly to CUDA kernels, utilizing asynchronous memory copies to hide latency.

## How it works
The pipeline operates as a multi-threaded producer-consumer system where independent threads handle loading, decompression, and GPU transfer.

### Inputs and Outputs
*   **Input:** Compressed brick data packets on disk, organized by time steps.
*   **Output:** A CUDA array (3D texture) in GPU global memory ready for raycasting.

### Data Structures
*   **Transfer Buffer:** A page-locked (pinned) memory region in host RAM. It holds a single large memory block containing all uncompressed bricks for a batch of time steps.
*   **Shadow Copy:** A 1D or 3D array in GPU global memory where decompression kernels write intermediate results.
*   **CUDA Array:** The final 3D texture object in GPU memory, used for rendering.

### Parameters
*   **Loader Queue Size:** Up to 5 time steps [S1].
*   **Bit Lengths Supported:** 16, 8, 4, and 0 bits per voxel [S1].
*   **CUDA Block Size:** 64 threads per block [S1].

### Steps
1.  **Loading:** A loader thread reads compressed data from disk, buffering up to five time steps in advance. It feeds this data to the decompression thread when the latter is idle [S1].
2.  **Asynchronous Copy:** The uncompressed brick data is copied from the page-locked transfer buffer to the GPU using an asynchronous memory copy operation. This allows the copy to run in parallel with CPU operations and GPU kernel executions [S1].
3.  **Decompression Kernels:** The GPU executes specific kernels to resolve the compression:
    *   **Variable-Length Coding:** Separate kernels handle bricks encoded with 16, 8, 4, or 0 bits. These kernels expand the compressed bit streams into full voxel values [S1].
    *   **Delta Encoding:** A kernel adds the delta values to the previous time step's data to reconstruct the current absolute values [S1].
    *   **Brick Assembly:** Kernels assemble the decompressed voxels into the shadow copy [S1].
4.  **Texture Upload:** The host code calls `cudaMemcpy3D()` to copy the shadow copy from global memory to the CUDA array (3D texture). This on-device copy utilizes full GPU memory bandwidth [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global state
transfer_buffer = allocate_pinned_memory(size=total_brick_size * num_brick_batches)
shadow_copy = allocate_gpu_global_memory(size=total_volume_size)
cuda_array = create_cuda_array_3d(dimensions=volume_dims)

def loader_thread(queue):
    """
    Loads compressed data from disk into the transfer buffer.
    """
    while running:
        # Load up to 5 time steps in advance
        compressed_data = read_from_disk(time_steps=queue.max_size)
        
        # Decompress on CPU if necessary, or pass compressed packets
        # Note: The source implies LZO decompression happens here or in a separate thread
        # For this pipeline, we assume the buffer is prepared for GPU consumption
        unpack_compressed_packets(compressed_data, transfer_buffer)
        
        # Signal decompression thread
        queue.put(transfer_buffer)

def decompression_thread(queue, gpu_stream):
    """
    Manages GPU kernel launches for decompression.
    """
    while running:
        buffer = queue.get()
        
        # Launch asynchronous memory copy to GPU
        # Copy large block containing all bricks
        cuda_stream_async_memcpy(gpu_stream, dest_gpu_ptr, buffer, size=total_brick_size)
        
        # Launch kernels for variable-length decoding
        # Specific kernels for 16, 8, 4, 0 bits
        launch_vl_decode_kernel(gpu_stream, input_ptr=dest_gpu_ptr, output_ptr=shadow_copy, bit_depth=16)
        launch_vl_decode_kernel(gpu_stream, input_ptr=dest_gpu_ptr, output_ptr=shadow_copy, bit_depth=8)
        launch_vl_decode_kernel(gpu_stream, input_ptr=dest_gpu_ptr, output_ptr=shadow_copy, bit_depth=4)
        launch_vl_decode_kernel(gpu_stream, input_ptr=dest_gpu_ptr, output_ptr=shadow_copy, bit_depth=0)
        
        # Launch delta encoding kernel
        launch_delta_decode_kernel(gpu_stream, prev_time_step_ptr, delta_ptr, output_ptr=shadow_copy)
        
        # Launch brick assembly kernel
        launch_assembly_kernel(gpu_stream, input_ptr=shadow_copy, output_ptr=shadow_copy)

def assembly_kernel(thread_block_size=64):
    """
    CUDA Kernel for brick assembly.
    Distributes work across blocks of 64 threads.
    Each thread handles one x-coordinate for all voxels in that column.
    """
    # Thread indexing
    thread_id = threadIdx.x + blockIdx.x * blockDim.x
    x_coord = threadIdx.x # Each thread in a block handles one x-coordinate
    
    # Process all voxels for this x-coordinate across the brick
    for y in range(brick_depth_y):
        for z in range(brick_depth_z):
            # Calculate memory index
            # Memory layout ensures adjacent threads access adjacent memory
            index = calculate_index(x_coord, y, z)
            
            # Read compressed/decoded value
            val = read_voxel(index)
            
            # Write to shadow copy
            write_voxel(index, val)

def finalize_rendering():
    """
    Host function called after decompression is complete.
    """
    # Copy shadow copy to CUDA array
    cudaMemcpy3D_params = {
        'srcPtr': shadow_copy,
        'dstPtr': cuda_array,
        'widthInBytes': volume_width * voxel_size,
        'height': volume_height,
        'depth': volume_depth
    }
    cudaMemcpy3D(cudaMemcpy3D_params)
```

### Edge Cases and Pitfalls
*   **Skipping Invisible Bricks:** Optimizations that skip loading bricks based on transfer function opacity are not implemented. Skipping breaks delta encoding because decoding requires access to all previous time steps' absolute values. It also complicates user interaction when transfer functions change dynamically [S1].
*   **Direct Texture Writing:** Current GPUs and drivers do not support direct writing to 3D textures (CUDA arrays) from kernel code. A shadow copy in global memory is required as an intermediate step. The OpenCL extension `cl_khr_3d_image_writes` supports this feature but is not yet supported by current hardware/drivers [S1].

## Operating characteristics
*   **Time Complexity:** The pipeline overlaps I/O, decompression, and transfer, effectively hiding latency. The bottleneck is typically data loading [S1].
*   **Memory Complexity:** Requires page-locked memory for the transfer buffer and a shadow copy in GPU global memory.
*   **Performance:** Achieves maximum transfer bandwidth by copying one large memory block. The assembly kernel achieves full coalescing by assigning each thread an x-coordinate and processing all voxels in that column [S1].
*   **Strengths:** High throughput via asynchronous operations; efficient memory access patterns in CUDA kernels.
*   **Limitations:** Requires a shadow copy due to hardware limitations on direct texture writes; cannot skip invisible bricks without breaking delta encoding.

## Background
The method was introduced by J. Mensmann, T. Ropinski, and K. Hinrichs in 2010 as part of a lossless compression scheme for time-varying volume data [S1]. It addresses the limitations of CPU-only decompression by leveraging CUDA for parallel unpacking and assembly.

## Sources
[S1] A GPU-Supported Lossless Compression Scheme for Rendering Time-Varying Volume Data, https://doi.org/10.2312/vg/vg10/109-116, fetched 2026-10-26
