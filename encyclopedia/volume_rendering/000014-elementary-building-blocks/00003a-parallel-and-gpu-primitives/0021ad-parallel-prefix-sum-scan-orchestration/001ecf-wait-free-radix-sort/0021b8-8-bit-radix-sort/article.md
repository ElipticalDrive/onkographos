# 8-bit Radix Sort

Also called: Bitwise radix sort

## Overview
This method implements the 8-bit digit extraction phase of a wait-free parallel radix sort, specifically designed for GPU execution via Vulkan compute shaders. It sorts 64-bit keys by processing them in eight sequential passes, one for each byte. It distinguishes itself from sibling methods by explicitly managing ping-pong buffer swapping and dynamic buffer resizing to handle variable scene sizes without pre-allocation limits, building on the hierarchical scan orchestration of [Wait-Free Radix Sort](1879-wait-free-radix-sort.md).

## How it works
The algorithm processes 64-bit sort keys through eight passes, extracting one 8-bit chunk per pass. It relies on a histogram-based distribution sort where each pass computes a local histogram, performs a global prefix sum (scan) to determine output positions, and scatters elements to their sorted locations.

### Inputs and Outputs
*   **Inputs**:
    *   `numInstances`: The number of elements to sort.
    *   `sortKBufferEven/Odd`: Storage buffers for 64-bit keys.
    *   `sortVBufferEven/Odd`: Storage buffers for 32-bit values associated with keys.
    *   `sortHistBuffer`: Storage buffer for histogram data.
*   **Outputs**:
    *   Sorted keys in `sortKBufferEven` or `sortKBufferOdd` (depending on pass parity).
    *   Sorted values in `sortVBufferEven` or `sortVBufferOdd`.

### Data Structures
**RadixSortPushConstants**
A push constant structure passed to the compute shader, containing four `uint32_t` fields:
1.  `g_num_elements`: Total number of elements to sort.
2.  `g_num_blocks_per_workgroup`: Number of sort blocks per compute workgroup.
3.  `g_shift`: The bit shift amount for the current 8-bit digit extraction (0, 8, 16, ..., 56).
4.  `g_num_workgroups`: The total number of workgroups dispatched.

**Buffers**
*   `sortKBufferEven/Odd`: `uint64_t` arrays. Size: `scene->getNumVertices() * sizeof(uint64_t) * sortBufferSizeMultiplier`.
*   `sortVBufferEven/Odd`: `uint32_t` arrays. Size: `scene->getNumVertices() * sizeof(uint32_t) * sortBufferSizeMultiplier`.
*   `sortHistBuffer`: `uint32_t` arrays. Size: `numWorkgroups * 256 * sizeof(uint32_t)`. The dimension 256 corresponds to the 256 possible values of an 8-bit digit (0-255) per block.

### Algorithm Steps
The core logic executes an 8-pass loop. Each pass consists of a histogram computation phase followed by a scatter/scan phase.

1.  **Initialization**:
    *   Calculate `invocationSize` per pass:
        $$ \text{invocationSize} = \lceil \frac{\text{numInstances} + \text{numRadixSortBlocksPerWorkgroup} - 1}{\text{numRadixSortBlocksPerWorkgroup}} \rceil $$
    *   Align `invocationSize` to 256 threads:
        $$ \text{alignedInvocationSize} = \lceil \frac{\text{invocationSize} + 255}{256} \rceil $$
    *   Determine `numWorkgroups` based on the aligned invocation size and hardware workgroup limits.

2.  **Loop for each byte `i` from 0 to 7**:
    *   **Determine Ping-Pong Index**:
        $$ \text{pingPongIdx} = i \pmod 2 $$
    *   **Phase 1: Histogram**:
        *   Bind `sortHistPipeline`.
        *   Set push constants with `g_shift = i * 8`.
        *   Dispatch histogram shader.
        *   Execute barrier on `sortHistBuffer` to ensure histogram data is visible.
    *   **Phase 2: Scatter and Scan**:
        *   Bind `sortPipeline` (scatter/scan).
        *   Set push constants with `g_shift = i * 8`.
        *   Dispatch scatter/scan shader.
    *   **Synchronization**:
        *   If `i % 2 == 0`:
            *   Barrier on `sortKBufferOdd` and `sortVBufferOdd`.
        *   Else:
            *   Barrier on `sortKBufferEven` and `sortVBufferEven`.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def execute_8_bit_radix_sort(num_instances, num_radix_sort_blocks_per_workgroup, scene_num_vertices, sort_buffer_size_multiplier):
    # Initial buffer allocation
    sort_k_buffer_even = allocate_storage_buffer(scene_num_vertices * 8 * sort_buffer_size_multiplier)
    sort_k_buffer_odd = allocate_storage_buffer(scene_num_vertices * 8 * sort_buffer_size_multiplier)
    sort_v_buffer_even = allocate_storage_buffer(scene_num_vertices * 4 * sort_buffer_size_multiplier)
    sort_v_buffer_odd = allocate_storage_buffer(scene_num_vertices * 4 * sort_buffer_size_multiplier)
    
    # Calculate initial workgroup parameters
    invocation_size = (num_instances + num_radix_sort_blocks_per_workgroup - 1) // num_radix_sort_blocks_per_workgroup
    aligned_invocation_size = (invocation_size + 255) // 256
    
    # Note: num_workgroups calculation depends on specific GPU workgroup size limits, 
    # typically derived from aligned_invocation_size. Not given in sources.
    num_workgroups = calculate_num_workgroups(aligned_invocation_size) 
    
    sort_hist_buffer = allocate_storage_buffer(num_workgroups * 256 * 4)

    for i in range(8):
        ping_pong_idx = i % 2
        
        # Phase 1: Histogram
        bind_pipeline(sort_hist_pipeline)
        set_push_constants(
            g_num_elements=num_instances,
            g_num_blocks_per_workgroup=num_radix_sort_blocks_per_workgroup,
            g_shift=i * 8,
            g_num_workgroups=num_workgroups
        )
        dispatch_histogram_shader()
        barrier(sort_hist_buffer)
        
        # Phase 2: Scatter and Scan
        bind_pipeline(sort_pipeline)
        set_push_constants(
            g_num_elements=num_instances,
            g_num_blocks_per_workgroup=num_radix_sort_blocks_per_workgroup,
            g_shift=i * 8,
            g_num_workgroups=num_workgroups
        )
        dispatch_scatter_scan_shader()
        
        # Ping-Pong Barriers
        if ping_pong_idx == 0:
            barrier(sort_k_buffer_odd)
            barrier(sort_v_buffer_odd)
        else:
            barrier(sort_k_buffer_even)
            barrier(sort_v_buffer_even)

    return sort_k_buffer_even if ping_pong_idx == 0 else sort_k_buffer_odd
```

### Parameters
*   `numRadixSortBlocksPerWorkgroup`: Number of sort blocks per compute workgroup. Used to calculate invocation count. [S1, unofficial]
*   `sortBufferSizeMultiplier`: Dynamic scaling factor for sort buffer capacity, starts at 1 and grows as needed. [S1, unofficial]

### Edge Cases and Pitfalls
*   **Buffer Overflow**: If `numInstances` exceeds the initially allocated buffer size, the algorithm must detect this and resize buffers dynamically. [S1, unofficial]
*   **Precision**: Keys are 64-bit, values are 32-bit. The shift operation must correctly extract the 8-bit chunk without sign-extension issues if keys are treated as signed integers (though typically unsigned for radix sort). [S1, unofficial]
*   **Synchronization**: Failure to barrier between ping-pong buffers leads to race conditions where a workgroup reads data that is being written by another pass. [S1, unofficial]

## Subtopics

### Dynamic Sort Buffer Resizing
This variant handles cases where the number of instances exceeds the pre-allocated buffer capacity. It checks if `numInstances > scene->getNumVertices() * sortBufferSizeMultiplier`. If true, it increments `sortBufferSizeMultiplier` in a while loop until the condition fails. It then reallocates `sortKBufferEven/Odd` to `scene->getNumVertices() * sizeof(uint64_t) * sortBufferSizeMultiplier` and `sortVBufferEven/Odd` to `scene->getNumVertices() * sizeof(uint32_t) * sortBufferSizeMultiplier`. It recalculates `globalInvocationSize` and `numWorkgroups` for the histogram buffer and reallocates `sortHistBuffer` to `numWorkgroups * 256 * sizeof(uint32_t)`. Finally, it calls `recordPreprocessCommandBuffer()` and returns `false` to skip the current render pass, forcing a retry with the new buffers. [S1, unofficial]

## Operating characteristics
*   **Time Complexity**: $O(8 \cdot N)$, where $N$ is the number of elements. The constant factor 8 is fixed for 64-bit keys.
*   **Memory Complexity**: $O(N \cdot M)$, where $M$ is the `sortBufferSizeMultiplier`. Buffers scale linearly with the number of elements and the dynamic multiplier.
*   **Performance**: Performance is heavily dependent on the efficiency of the histogram and scatter/scan kernels. The ping-pong buffer strategy allows for overlapping computation and memory access but doubles the memory footprint for keys and values.
*   **Strengths**: Handles variable input sizes dynamically; avoids inter-workgroup spin-waiting by relying on the parent's wait-free scan orchestration.
*   **Limitations**: Requires significant memory for ping-pong buffers; dynamic resizing incurs overhead and may require restarting the pass.
*   **When to use**: When sorting large, variable-sized datasets on GPU where memory allocation flexibility is required.
*   **When not to use**: When memory is constrained or the dataset size is known and fixed to a small value.

## Background
This method is part of the implementation of 3DGS.cpp, a performant, cross-platform Gaussian Splatting renderer using Vulkan. It was developed to handle the sorting of Gaussian splats efficiently on the GPU. The specific implementation details, including buffer management and pipeline setup, are derived from the 3DGS.cpp source code. [S1, unofficial]

## Sources
[S1] Show HN: 3DGS.cpp – performant, cross platform Gaussian Splatting with Vulkan, https://github.com/shg8/3DGS.cpp, fetched 2026-10-26 (unofficial)
