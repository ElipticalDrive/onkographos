# Parallel Prefix Sum (Scan) Orchestration

Also called: Blelloch Scan, Exclusive/Inclusive Scan via Ping-Pong, Parallel Prefix Sum, Prefix Sum (Scan), Up-Sweep Down-Sweep Scan, Work-Efficient GPU Scan

## Overview
Parallel Prefix Sum (Scan) Orchestration is a method for computing prefix sums (scans) of an input array on the GPU, serving as a foundational primitive for stream compaction, histogram generation, and indirect dispatch in volume rendering and geometry processing [S1, S2, unofficial]. It solves the problem of mapping parallel, non-contiguous data decisions (e.g., voxel culling or vertex visibility) to sequential output indices without race conditions [S2, unofficial]. Unlike naive iterative approaches that may require multiple kernel launches or lack work-efficiency, this orchestration typically employs a work-efficient balanced binary tree approach (up-sweep and down-sweep phases) combined with a ping-pong buffer strategy to handle arbitrary input sizes and ensure memory consistency [S1, S2, unofficial]. It differs from [Naive Parallel Prefix Sum (Scan)](10348-naive-parallel-prefix-sum-scan.md) by explicitly optimizing for work-efficiency ($O(n)$ work) and handling non-power-of-two inputs via padding, while providing a structured multi-timestep execution model [S3, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** An array $I$ of $N$ elements (typically integers or floats) representing values to be summed.
- **Output:** An array $O$ of size $N$ (or $N+1$ for exclusive scan) containing the prefix sums. For an exclusive scan, $O[i] = \sum_{j=0}^{i-1} I[j]$ with $O[0]=0$ [S3, unofficial][S4]. For an inclusive scan, $O[i] = \sum_{j=0}^{i} I[j]$.
- **Auxiliary:** A host-accessible buffer to retrieve the total sum (the last element of the inclusive scan) [S1, unofficial].

**Data Structures:**
1.  **Ping-Pong Buffers:** Two storage buffers (`prefixSumPingBuffer`, `prefixSumPongBuffer`) of size $M$, where $M$ is the smallest power of two greater than or equal to $N$ ($M = 2^{\lceil \log_2 N \rceil}$) [S1, S2, unofficial]. These buffers store intermediate results during the up-sweep and down-sweep phases.
2.  **Push Constant:** A 32-bit integer `timestep` indicating the current iteration step of the scan algorithm [S1, unofficial].
3.  **Total Sum Buffer:** A host-accessible buffer (`totalSumBufferHost`) to receive the final total sum from the active ping-pong buffer after computation completes [S1, unofficial].

**Algorithm Steps:**
The method consists of two main phases: Up-Sweep (Reduction) and Down-Sweep (Scan), executed over multiple timesteps (iterations) to handle the tree levels.

1.  **Padding:** Pad the input array $I$ of size $N$ to size $M = 2^{\lceil \log_2 N \rceil}$ by appending zeros [S2, unofficial].
2.  **Initialization:** Copy the padded input array into the `prefixSumPingBuffer` [S1, S2, unofficial].
3.  **Up-Sweep Phase (Reduction):**
    -   Iterate $d$ from $0$ to $\lceil \log_2 M \rceil - 1$ [S2, unofficial].
    -   In each iteration (timestep), threads compute partial sums of pairs of elements.
    -   For even timesteps $t$: If `index` $< 2^t$, copy `src[index]` to `dst[index]`. Otherwise, compute `index2` $= \text{index} - 2^t$ and set `dst[index]` $= \text{src[index]} + \text{src[index2]}$ [S1, unofficial].
    -   For odd timesteps $t$: If `index` $< 2^t$, copy `dst[index]` to `src[index]`. Otherwise, compute `index2` $= \text{index} - 2^t$ and set `src[index]` $= \text{dst[index]} + \text{dst[index2]}$ [S1, unofficial].
    -   Swap ping-pong buffers and apply synchronization barriers based on timestep parity to prevent read/write conflicts [S1, unofficial].
    -   After the final up-sweep iteration, the root of the tree (last element of the active buffer) contains the total sum of the input array [S2, unofficial].
4.  **Root Zeroing:** Set the root value (last element of the active buffer) to 0 to prepare for the down-sweep phase [S2, unofficial].
5.  **Down-Sweep Phase (Scan):**
    -   Iterate $d$ from $\lceil \log_2 M \rceil - 1$ down to $0$ [S2, unofficial].
    -   Propagate partial sums back down the tree. The root's 0 is passed to the left child. The right child receives the sum of the parent's original value and the left child's value [S2, unofficial].
    -   The same ping-pong buffer swapping and barrier logic as the up-sweep phase is applied [S1, unofficial].
6.  **Finalization:**
    -   After the down-sweep, the active buffer contains the inclusive prefix sums [S2, unofficial].
    -   If an exclusive scan is required, shift the results or adjust the initial root zeroing logic accordingly [S3, unofficial].
    -   Copy the final total sum from the last element of the active buffer to the host-accessible `totalSumBufferHost` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parallel_prefix_sum_orchestration(input_array, N):
    # 1. Padding
    M = 1
    while M < N:
        M *= 2
    padded_input = input_array + [0] * (M - N)
    
    # 2. Initialization
    ping_buffer = padded_input.copy()
    pong_buffer = [0] * M
    src = ping_buffer
    dst = pong_buffer
    
    iters = ceil(log2(M))
    
    # 3. Up-Sweep Phase
    for t in range(iters):
        # Dispatch kernel with timestep t
        # Kernel logic:
        # index = gl_GlobalInvocationID.x
        # if index < M:
        #     if t % 2 == 0: # Even timestep
        #         if index < 2**t:
        #             dst[index] = src[index]
        #         else:
        #             index2 = index - 2**t
        #             dst[index] = src[index] + src[index2]
        #     else: # Odd timestep
        #         if index < 2**t:
        #             src[index] = dst[index]
        #         else:
        #             index2 = index - 2**t
        #             src[index] = dst[index] + dst[index2]
        
        # Synchronization and Buffer Swap
        if t % 2 == 0:
            src, dst = pong_buffer, ping_buffer
        else:
            src, dst = ping_buffer, pong_buffer
            
    # 4. Root Zeroing
    # The root is at index M-1
    if iters % 2 == 0:
        ping_buffer[M-1] = 0
    else:
        pong_buffer[M-1] = 0
        
    # 5. Down-Sweep Phase
    for t in range(iters - 1, -1, -1):
        # Dispatch kernel with timestep t
        # Kernel logic for down-sweep is more complex, involving parent-child relationships
        # Simplified representation based on source notes:
        # index = gl_GlobalInvocationID.x
        # if index < M:
        #     if t % 2 == 0: # Even timestep
        #         if index < 2**t:
        #             dst[index] = src[index]
        #         else:
        #             index2 = index - 2**t
        #             dst[index] = src[index] + src[index2]
        #     else: # Odd timestep
        #         if index < 2**t:
        #             src[index] = dst[index]
        #         else:
        #             index2 = index - 2**t
        #             src[index] = dst[index] + dst[index2]
            
        # Synchronization and Buffer Swap
        if t % 2 == 0:
            src, dst = pong_buffer, ping_buffer
        else:
            src, dst = ping_buffer, pong_buffer
            
    # 6. Finalization
    # Determine final active buffer
    if iters % 2 == 0:
        final_buffer = ping_buffer
    else:
        final_buffer = pong_buffer
        
    total_sum = final_buffer[M-1]
    return final_buffer[:N], total_sum
```

**Parameters:**
-   `timestep`: Integer push constant indicating the current iteration step [S1, unofficial].
-   `numGroups`: Number of compute shader workgroups, calculated as `(N + 255) / 256` for a local group size of 256 [S1, unofficial].
-   `iters`: Number of prefix sum iterations, calculated as `ceil(log2(M))` where $M$ is the padded size [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Non-Power-of-Two Inputs:** Must be padded to the next power of two to ensure the tree is balanced and complete [S2, unofficial].
-   **Barrier Ordering:** Barrier ordering must strictly match the ping-pong swap direction to avoid GPU read/write conflicts [S1, unofficial].
-   **Inefficient Power Calculation:** Using a naive loop for power calculation (`power(base, exponent)`) instead of bit shifting is inefficient for large exponents [S1, unofficial].
-   **Precision:** Floating-point inputs may suffer from precision loss in large sums; use appropriate data types (e.g., `float` vs `double`) based on requirements [G].

## Subtopics

### [CUDA-based Exclusive Prefix Sum for Vertex Indexing](1112-cuda-based-exclusive-prefix-sum-for-vertex-indexing.md)
Solves duplicate vertex calculation in GPU-based parallel isosurface extraction by using an exclusive prefix sum to compute correct global vertex indices. It applies scan-based indexing to both vertices and triangles, ensuring unique, non-overlapping global indices without expensive post-processing merging.

### [Wait-Free Radix Sort](1879-wait-free-radix-sort.md)
A parallel sorting algorithm for heterogeneous GPU environments (WebGPU) that avoids inter-workgroup spin-waiting by using a hierarchical Blelloch scan to compute element positions. It decouples workgroup synchronization via a multi-level scan dispatch, ensuring consistent behavior without busy-waiting.

### [Group Prefix Sum for Shell Lists](2746-group-prefix-sum-for-shell-lists.md)
A parallel shell list compaction method used in GPU-based isosurface extraction, replacing sequential processing by thread 0 with a parallel prefix sum approach. It coordinates with the octree to derive block offsets from the first active octree nodes in each shell.

### [Exclusive Prefix Sum (Scan)](3007-exclusive-prefix-sum-scan.md)
Computes an output array where each element at index $i$ is the sum of all input elements preceding it. It maps boolean voxel culling decisions to sequential indices for kept voxels, enabling parallel stream compaction. It handles non-power-of-two inputs via padding and offers both naive iterative and work-efficient tree-based implementations.

### [Naive Parallel Prefix Sum (Scan)](10348-naive-parallel-prefix-sum-scan.md)
A GPU-based algorithm for computing the inclusive prefix sum using global memory and multiple kernel invocations. It uses an iterative, level-by-level reduction approach with double-buffering to avoid race conditions, resulting in higher computational work ($O(n \log n)$) but simpler synchronization logic.

### [Global Accumulator + Compaction](10410-global-accumulator-compaction.md)
A two-pass system to gather visible splat counts per proxy and compute prefix sums to generate indirect dispatch/draw arguments for global Phase-3 passes.

## Operating characteristics
-   **Time Complexity:** $O(\log N)$ parallel time with $O(N)$ work for the work-efficient Blelloch scan [S2, unofficial]. The naive iterative approach has $O(N \log N)$ work [S3].
-   **Memory Complexity:** $O(N)$ for the input and output arrays, plus $O(N)$ for the ping-pong buffers [S1, S2, unofficial].
-   **Measured Performance:** Performance depends on the GPU architecture and input size. For power-of-two inputs, the work-efficient scan is significantly faster than the naive approach [S2, unofficial].
-   **Scaling:** Scales well with input size due to the logarithmic depth of the tree. However, memory bandwidth can become a bottleneck for very large arrays [G].
-   **Strengths:** Work-efficient, handles arbitrary input sizes via padding, provides a clear mechanism for stream compaction and indirect dispatch [S1, S2, unofficial].
-   **Limitations:** Requires multiple kernel launches or a complex single-kernel implementation with synchronization. The ping-pong buffer strategy adds memory overhead and requires careful synchronization [S1, unofficial].
-   **When to Use:** When stream compaction, histogram generation, or indirect dispatch is required in volume rendering or geometry processing pipelines [S2, unofficial].
-   **When Not to Use:** For small arrays where the overhead of parallelization outweighs the benefits, or when a simple sequential scan is sufficient [G].

## Background
The parallel prefix sum (scan) is a fundamental primitive in parallel computing, with origins in the 1970s and 1980s [G]. The work-efficient Blelloch scan, introduced by Guy E. Blelloch in 1990, optimized the algorithm to perform $O(N)$ work in $O(\log N)$ time, making it suitable for GPU implementation [G]. It has since become a standard component in GPU programming libraries (e.g., CUB, Thrust) and is widely used in volume rendering for tasks such as stream compaction, histogram generation, and indirect dispatch [S1, S2, unofficial].

## Sources
[S1] Show HN: 3DGS.cpp – performant, cross platform Gaussian Splatting with Vulkan, https://github.com/shg8/3DGS.cpp, fetched 2026-10-26 (unofficial)
[S2] GitHub - jichumao/Stream-Compaction, https://github.com/jichumao/Stream-Compaction, fetched 2026-10-29 (unofficial)
