# 8-bit Radix Sort

Also called: Bitwise radix sort

## Overview
8-bit Radix Sort is a parallel sorting algorithm that processes 64-bit sort keys by iterating through eight 8-bit chunks (digits), using a hierarchical Blelloch scan to compute element positions without inter-workgroup spin-waiting [Parent]. It is designed for Vulkan compute shaders, ensuring consistent behavior across heterogeneous GPU schedulers by avoiding race conditions inherent in atomic-based sorting [Parent]. Unlike [Subgroup-based Radix Sort Partition](3368-subgroup-based-radix-sort-partition.md) which minimizes atomic contention via subgroup ballots, or [Radix Sort (GPU)](10404-radix-sort-gpu.md) which separates counting and scattering into distinct shader passes, this method uses two separate compute pipelines per pass (histogram and scatter/scan) coordinated via ping-pong buffers to handle arbitrary data types and key spaces [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Array of 64-bit keys (`sortKBuffer`) and associated 32-bit values (`sortVBuffer`) [S1, unofficial].
- Output: Sorted keys and values in the alternate ping-pong buffer [S1, unofficial].
- Parameters: `numInstances` (number of elements to sort), `numRadixSortBlocksPerWorkgroup` (number of sort blocks per compute workgroup, used to calculate invocation count) [S1, unofficial].

**Data Structures:**
- `RadixSortPushConstants`: A 16-byte structure containing four `uint32_t` fields: `g_num_elements`, `g_num_blocks_per_workgroup`, `g_shift`, `g_num_workgroups` [S1, unofficial].
- Buffers:
  - `sortKBufferEven`, `sortKBufferOdd`: Storage buffers for 64-bit keys, sized `numInstances * sizeof(uint64_t)` [S1, unofficial].
  - `sortVBufferEven`, `sortVBufferOdd`: Storage buffers for 32-bit values, sized `numInstances * sizeof(uint32_t)` [S1, unofficial].
  - `sortHistBuffer`: Histogram buffer for digit counting, sized `numWorkgroups * 256 * sizeof(uint32_t)` [S1, unofficial].

**Algorithm Steps:**
1. **Initialization**: Calculate `globalInvocationSize` as `(numInstances + numRadixSortBlocksPerWorkgroup - 1) / numRadixSortBlocksPerWorkgroup`, then align to 256 threads: `(globalInvocationSize + 255) / 256` [S1, unofficial]. Calculate `numWorkgroups` based on the aligned invocation size [G].
2. **Loop 8 times** (once for each 8-bit chunk of the 64-bit key):
   a. **Histogram Pass**: Bind histogram pipeline `sortHistPipeline` and dispatch with ping-pong index `i % 2 == 0 ? 0 : 1` [S1, unofficial]. Dispatch histogram shader to count digit frequencies per workgroup [S1, unofficial]. Barrier on `sortHistBuffer` to ensure histogram data is visible [S1, unofficial].
   b. **Scan and Scatter Pass**: Bind scatter/scan pipeline `sortPipeline` and dispatch with the same push constants and ping-pong index [S1, unofficial]. This pass performs the parallel prefix sum (scan) on the histogram and scatters elements to their correct positions in the output buffer [S1, unofficial].
   c. **Ping-Pong Barrier**: If `i % 2 == 0`, barrier on `sortKBufferOdd` and `sortVBufferOdd`; else barrier on `sortKBufferEven` and `sortVBufferEven` [S1, unofficial]. This ensures the next iteration reads from the buffer written in the current iteration [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def radix_sort_8bit(num_instances, num_radix_sort_blocks_per_workgroup, sort_buffer_size_multiplier, scene_num_vertices):
    # Dynamic buffer resizing check [S1, unofficial]
    if num_instances > scene_num_vertices * sort_buffer_size_multiplier:
        while num_instances > scene_num_vertices * sort_buffer_size_multiplier:
            sort_buffer_size_multiplier += 1
        # Reallocate buffers [S1, unofficial]
        sort_k_buffer_even = allocate_buffer(scene_num_vertices * 8 * sort_buffer_size_multiplier)
        sort_k_buffer_odd = allocate_buffer(scene_num_vertices * 8 * sort_buffer_size_multiplier)
        sort_v_buffer_even = allocate_buffer(scene_num_vertices * 4 * sort_buffer_size_multiplier)
        sort_v_buffer_odd = allocate_buffer(scene_num_vertices * 4 * sort_buffer_size_multiplier)
        # Recalculate workgroups and histogram buffer size [S1, unofficial]
        global_invocation_size = (num_instances + num_radix_sort_blocks_per_workgroup - 1) // num_radix_sort_blocks_per_workgroup
        global_invocation_size = (global_invocation_size + 255) // 256
        num_workgroups = (global_invocation_size + 255) // 256
        sort_hist_buffer = allocate_buffer(num_workgroups * 256 * 4)
        record_preprocess_command_buffer()
        return False  # Skip current render pass

    # Main sort loop
    for i in range(8):
        shift = i * 8
        
        # Histogram Pass
        bind_pipeline(sort_hist_pipeline)
        dispatch_push_constants(
            g_num_elements=num_instances,
            g_num_blocks_per_workgroup=num_radix_sort_blocks_per_workgroup,
            g_shift=shift,
            g_num_workgroups=num_workgroups,
            ping_pong_index=i % 2
        )
        barrier(sort_hist_buffer)
        
        # Scan and Scatter Pass
        bind_pipeline(sort_pipeline)
        dispatch_push_constants(
            g_num_elements=num_instances,
            g_num_blocks_per_workgroup=num_radix_sort_blocks_per_workgroup,
            g_shift=shift,
            g_num_workgroups=num_workgroups,
            ping_pong_index=i % 2
        )
        
        # Ping-Pong Barrier
        if i % 2 == 0:
            barrier(sort_k_buffer_odd)
            barrier(sort_v_buffer_odd)
        else:
            barrier(sort_k_buffer_even)
            barrier(sort_v_buffer_even)

# Helper definitions for GPU dispatch simulation
def allocate_buffer(size_bytes):
    return [0] * (size_bytes // 4)  # Simulated storage buffer

def calculate_num_workgroups(global_invocation_size):
    return (global_invocation_size + 255) // 256

def record_preprocess_command_buffer():
    pass  # Records command buffer state

def bind_pipeline(pipeline):
    pass  # Sets active compute pipeline

def dispatch_push_constants(g_num_elements, g_num_blocks_per_workgroup, g_shift, g_num_workgroups, ping_pong_index):
    pass  # Dispatches compute shader with push constants

def barrier(buffer):
    pass  # Synchronizes workgroups on buffer visibility
```

**Edge Cases and Pitfalls:**
- **Buffer Overflow**: If `numInstances` exceeds the initial buffer capacity, dynamic resizing is required [S1, unofficial]. Failure to resize leads to out-of-bounds writes [S1, unofficial].
- **Precision**: Uses `uint64_t` for keys and `uint32_t` for values [S1, unofficial].
- **Boundary Conditions**: The alignment to 256 threads ensures workgroup boundaries are respected, but `numInstances` may not be a multiple of the total thread count, requiring careful handling of out-of-bounds threads in shaders [G].

## Subtopics

### [Subgroup-based Radix Sort Partition](3368-subgroup-based-radix-sort-partition.md)
Performs a single pass of radix sort for a partition (typically 4096 elements) using GPU subgroup ballot operations to compute local offsets, minimizing atomic contention on shared memory. It builds on [8-bit Radix Sort](2599-8-bit-radix-sort.md) by replacing per-element atomic updates with subgroup-level exclusive scans, making it suitable for high-throughput sorting in compute shaders where subgroup size is known and fixed.

### [Radix Sort (GPU)](10404-radix-sort-gpu.md)
A GPU-based integer sorting algorithm implemented via four compute shader passes: histogram counting, prefix sum, digit prefix sum, and scatter. It sorts keys by processing digits in a specific order, using parallel prefix sum (scan) as a subroutine to determine output positions. It is distinct from subgroup-based partitioning methods by explicitly separating the counting, offset calculation, and scattering phases into dedicated shader invocations.

### [4-way Parallel Radix Sort](15052-4-way-parallel-radix-sort.md)
A hardware-optimized parallel implementation of the radix sort algorithm for GPUs that processes two bits per iteration (radix 4) to sort elements, lifting limitations on element count and precision compared to previous GPU sorters. It reduces the number of passes by half compared to binary-radix sort, eliminating restrictions on key space and supporting arbitrary data types. Unlike [8-bit Radix Sort](2599-8-bit-radix-sort.md) which processes one bit at a time, this method clusters numbers with the same radix combination bits inside a block to enable coalesced memory writing, addressing the non-coalesced scattering bottleneck of element-wise mapping.

### Dynamic Sort Buffer Resizing
A variant that dynamically scales the sort buffer capacity based on the scene size [S1, unofficial]. It checks if `numInstances > scene->getNumVertices() * sortBufferSizeMultiplier` [S1, unofficial]. If true, it increments `sortBufferSizeMultiplier` in a while loop until the condition fails, then reallocates key buffers (`sortKBufferEven/Odd`) and value buffers (`sortVBufferEven/Odd`) to `scene->getNumVertices() * sizeof(uint64_t) * sortBufferSizeMultiplier` [S1, unofficial]. It recalculates `globalInvocationSize` and `numWorkgroups` for the histogram buffer and reallocates `sortHistBuffer` to `numWorkgroups * 256 * sizeof(uint32_t)` [S1, unofficial]. Finally, it calls `recordPreprocessCommandBuffer()` and returns `false` to skip the current render pass [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(8 \cdot N)$ where $N$ is the number of elements, as it performs 8 passes (one per 8-bit digit) with linear work per pass [S1, unofficial].
- **Memory Complexity**: $O(N)$ for key and value buffers, plus $O(W \cdot 256)$ for the histogram buffer where $W$ is the number of workgroups [S1, unofficial].
- **Performance**: Measured performance depends on GPU architecture and subgroup size [S1, unofficial]. The ping-pong buffer approach avoids global memory contention for intermediate results but requires careful synchronization [S1, unofficial].
- **Strengths**: Works across heterogeneous GPU schedulers without spin-waiting; supports arbitrary 64-bit keys; dynamic buffer resizing handles varying scene sizes [Parent][S1, unofficial].
- **Limitations**: Fixed 8-bit digit size may not be optimal for all data distributions; requires multiple buffer allocations for ping-pong [S1, unofficial].
- **When to Use**: Suitable for general-purpose sorting in Vulkan/WebGPU applications where key size is 64-bit and hardware atomic support is limited or inconsistent [Parent][S1, unofficial].
- **When Not to Use**: For small datasets or when subgroup operations are heavily optimized and available, [Subgroup-based Radix Sort Partition](3368-subgroup-based-radix-sort-partition.md) may offer better throughput [S1, unofficial].

## Background
Originated in the context of performant, cross-platform Gaussian Splatting implementations using Vulkan [S1, unofficial]. It was developed to address the challenges of sorting large numbers of Gaussian splats efficiently on GPUs with varying capabilities, particularly avoiding the pitfalls of inter-workgroup synchronization [S1, unofficial]. The method leverages the Blelloch scan algorithm for position computation, a technique common in parallel prefix sum implementations, adapted for radix sorting [Parent].

## Sources
[S1] Show HN: 3DGS.cpp – performant, cross platform Gaussian Splatting with Vulkan, https://github.com/shg8/3DGS.cpp, fetched 2026-10-26 (unofficial)
