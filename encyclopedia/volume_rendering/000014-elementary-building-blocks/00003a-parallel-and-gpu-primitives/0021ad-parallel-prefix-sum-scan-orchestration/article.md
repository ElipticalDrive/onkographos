# Parallel Prefix Sum (Scan) Orchestration

Also called: Exclusive/Inclusive Scan via Ping-Pong, Parallel Prefix Sum, Prefix Sum

## Overview
Parallel Prefix Sum (Scan) Orchestration is a GPU-based method for computing prefix sums (scans) of an input array using a multi-timestep iterative reduction strategy with ping-pong buffers [S1, unofficial]. It solves the problem of efficiently aggregating data across parallel threads without race conditions, serving as a foundational primitive for operations like voxel compaction and vertex indexing in volume rendering pipelines [S1, unofficial]. It sets itself apart from sibling methods like [Tiled Parallel Compositing](1192-tiled-parallel-compositing.md) by focusing on the low-level data movement and synchronization required for global aggregation, rather than high-level compositing logic [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: An array of values (e.g., vertex counts or boolean flags) stored in a storage buffer [S1, unofficial].
- Output: The computed prefix sum array, with the total sum available in the last element of the active buffer [S1, unofficial].

**Data Structures:**
- `prefixSumPingBuffer` and `prefixSumPongBuffer`: Two `std430` storage buffers used for intermediate scan results, swapped each iteration with explicit read/write barriers to prevent race conditions [S1, unofficial].
- `totalSumBufferHost`: A host-accessible buffer receiving the final prefix sum total from the last element `(N-1)` of the active ping/pong buffer after the loop completes [S1, unofficial].
- Push Constant: A 32-bit integer `timestep` indicating the current iteration step of the scan algorithm [S1, unofficial].

**Numbered Steps:**
1. Initialize two ping-pong storage buffers and a host-accessible buffer [S1, unofficial].
2. Determine the number of vertices $N$ in the scene [S1, unofficial].
3. Calculate the iteration count: $I = \lceil \log_2(N) \rceil$ [S1, unofficial].
4. Loop for $t$ from $0$ to $I$ inclusive [S1, unofficial].
5. Inside the loop, push the current timestep as a 32-bit constant and dispatch the compute shader [S1, unofficial].
6. Dispatch the compute shader with workgroup count $\lceil N / 256 \rceil$ and local group size $256 \times 1 \times 1$ [S1, unofficial].
7. Swap ping-pong buffers and apply synchronization barriers based on timestep parity [S1, unofficial].
8. After the loop, copy the final sum from the active ping/pong buffer to `totalSumBufferHost` [S1, unofficial].

**Formulas:**
For a given timestep $t$ and global invocation index $i$:
- If $t$ is even:
  $$
  dst[i] = \begin{cases} 
  src[i] & \text{if } i < 2^t \\
  src[i] + src[i - 2^t] & \text{otherwise}
  \end{cases}
  $$
- If $t$ is odd:
  $$
  src[i] = \begin{cases} 
  dst[i] & \text{if } i < 2^t \\
  dst[i] + dst[i - 2^t] & \text{otherwise}
  \end{cases}
  $$
  Where $src$ and $dst$ refer to the read and write buffers for the current timestep, respectively [S1, unofficial].

**Parameters:**
- `timestep`: Integer push constant, current iteration step $t$ [S1, unofficial].
- `numGroups`: Number of compute shader workgroups, calculated as $\lceil N / 256 \rceil$ [S1, unofficial].
- `iters`: Number of iterations, calculated as $\lceil \log_2(N) \rceil$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def parallel_prefix_sum_orchestration(scene_num_vertices):
    N = scene_num_vertices
    iters = math.ceil(math.log2(N))
    
    # Initialize buffers [G]
    ping_buffer = [0] * N
    pong_buffer = [0] * N
    total_sum_host = [0]
    
    # Assume input data is loaded into ping_buffer initially [G]
    
    for t in range(iters + 1):
        # Dispatch compute shader for current timestep [S1, unofficial]
        dispatch_compute(
            workgroups=(N + 255) // 256,
            local_size=(256, 1, 1),
            push_constants={'timestep': t},
            src_buffer=ping_buffer,
            dst_buffer=pong_buffer
        )
        
        # Swap ping-pong buffers based on timestep parity [S1, unofficial]
        # Even timesteps write to pong/read from ping; odd timesteps write to ping/read from pong
        ping_buffer, pong_buffer = pong_buffer, ping_buffer
        
        # Apply synchronization barriers to ensure memory visibility [S1, unofficial]
        memory_barrier()
        
    # Copy final result from the active buffer to host-accessible buffer [S1, unofficial]
    last_index = N - 1
    src_offset = last_index * 4  # sizeof(uint32)
    dst_offset = 0
    size = 4
    total_sum_host[0] = ping_buffer[last_index]
    
    return total_sum_host[0]

def dispatch_compute(workgroups, local_size, push_constants, src_buffer, dst_buffer):
    timestep = push_constants['timestep']
    power_of_two = 1 << timestep  # Optimized power of 2 calculation [S1, unofficial]
    
    # Simulate parallel thread execution [G]
    for idx in range(workgroups * local_size[0]):
        if idx >= len(src_buffer):
            continue
            
        if timestep % 2 == 0:
            if idx < power_of_two:
                dst_buffer[idx] = src_buffer[idx]
            else:
                index2 = idx - power_of_two
                dst_buffer[idx] = src_buffer[idx] + src_buffer[index2]
        else:
            if idx < power_of_two:
                src_buffer[idx] = dst_buffer[idx]
            else:
                index2 = idx - power_of_two
                src_buffer[idx] = dst_buffer[idx] + dst_buffer[index2]

def memory_barrier():
    pass  # Simulates GPU memory barrier synchronization [G]
```

**Edge Cases and Pitfalls:**
- **Power Calculation:** Using a naive loop for power calculation is inefficient; bit shifting (`1 << timestep`) should be used [S1, unofficial].
- **Barrier Ordering:** Synchronization barriers must strictly match the ping-pong swap direction to avoid GPU read/write conflicts [S1, unofficial].
- **Boundary Conditions:** Ensure bounds checking (`index < N`) is performed before accessing buffer elements to prevent out-of-bounds memory access [S1, unofficial].

## Subtopics

### [CUDA-based Exclusive Prefix Sum for Vertex Indexing](1112-cuda-based-exclusive-prefix-sum-for-vertex-indexing.md)
Solves duplicate vertex calculation in GPU-based parallel isosurface extraction by applying scan-based indexing to both vertices and triangles, ensuring unique global indices without post-processing merging.

### [Wait-Free Radix Sort](1879-wait-free-radix-sort.md)
A parallel sorting algorithm for WebGPU that uses a hierarchical Blelloch scan to compute element positions, avoiding inter-workgroup spin-waiting by decoupling workgroup synchronization via multi-level scan dispatch.

### [Group Prefix Sum for Shell Lists](2746-group-prefix-sum-for-shell-lists.md)
A parallel compaction method for reorganizing shell list data structures in GPU-based isosurface extraction, replacing sequential thread-0 processing with a parallel prefix sum approach that coordinates with the octree for block offsets.

### [Exclusive Prefix Sum (Scan)](3007-exclusive-prefix-sum-scan.md)
The fundamental parallel primitive mapping an input list to an output list where each element is the sum of all preceding elements, primarily used for voxel compaction and differing from inclusive scans by excluding the current element.

## Operating characteristics
- **Time Complexity:** $O(\log N)$ timesteps, with $O(N)$ work per timestep, resulting in $O(N \log N)$ total work [S1, unofficial].
- **Memory Complexity:** $O(N)$ for the ping-pong buffers [S1, unofficial].
- **Performance:** Performance depends on the efficiency of the compute shader dispatch and memory synchronization; the naive power calculation can impact performance for large $N$ [S1, unofficial].
- **Strengths:** Simple implementation, robust synchronization via ping-pong buffers, suitable for irregular data sizes [S1, unofficial].
- **Limitations:** $O(N \log N)$ work is less efficient than $O(N)$ algorithms like Blelloch scan; requires multiple kernel launches and synchronization barriers [S1, unofficial].
- **When to Use:** When implementing a simple, robust prefix sum on Vulkan/OpenGL compute shaders without access to optimized libraries [S1, unofficial].
- **When Not to Use:** When performance is critical and $O(N)$ algorithms are available, or when working in environments with efficient warp-level primitives [S1, unofficial].

## Background
This method is derived from a Vulkan-based implementation [S1, unofficial]. It represents a straightforward, iterative approach to parallel prefix sums, often used in educational or prototype contexts where simplicity and correctness are prioritized over optimal performance [S1, unofficial]. The ping-pong buffer strategy is a common technique in GPU programming to manage read/write dependencies in iterative algorithms [S1, unofficial].

## Sources
[S1] Show HN: 3DGS.cpp – performant, cross platform Gaussian Splatting with Vulkan, https://github.com/shg8/3DGS.cpp, fetched 2026-10-26 (unofficial)
