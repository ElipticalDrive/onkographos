# Work-Efficient Scan

Also called: Work-Efficient Parallel Scan, Work-Efficient Parallel Inclusive Scan, Work-Efficient Inclusive Scan

## Overview
A parallel algorithm to compute the inclusive prefix sum of an array using a two-phase approach (upsweep/reduction and downsweep) to achieve work efficiency ($O(n)$ work, $O(\log n)$ depth) [S2, unofficial][S3]. It builds on [Naive Parallel Prefix Sum (Scan)](10348-naive-parallel-prefix-sum-scan.md) by replacing the iterative $O(n \log n)$ work with a binary tree reduction, eliminating race conditions through in-place updates and synchronization [S3]. It is distinguished from the naive method by its tree-based structure and from hardware-optimized variants by its explicit multi-kernel orchestration for arbitrary input sizes [S2, unofficial][S3].

## How it works
**Inputs and Outputs:**
- Input: Array `idata` of size $n$ (arbitrary integer size).
- Output: Array `odata` of size $n$ containing the inclusive prefix sum.

**Data Structures:**
The algorithm requires managing multiple levels of reduction for arbitrary $n$.
- `dev_idata`: Array of device pointers, size `num_scans`. `dev_idata[i]` points to the input buffer for level $i$.
- `dev_odata`: Array of device pointers, size `num_scans`. `dev_odata[i]` points to the output buffer for level $i$.
- `dev_buffer`: Array of device pointers, size `num_scans`. Temporary buffer for intermediate results.
- `array_sizes`: Array of integers, size `num_scans`. `array_sizes[i]` is the logical size of the data at level $i$.
- `grid_sizes`: Array of integers, size `num_scans`. `grid_sizes[i]` is the number of blocks launched at level $i$.

**Algorithm Steps:**

1.  **Preparation:**
    - Calculate padded size: $n_{pad} = 2^{\lceil \log_2 n \rceil}$.
    - Determine number of scan levels (`num_scans`) by iteratively dividing the length by `block_size` until it fits in one block:
      ```python
      len = n_pad
      num_scans = 0
      while (len + block_size - 1) // block_size > 1:
          num_scans += 1
          len = (len + block_size - 1) // block_size
      ```
    - Allocate device memory for `dev_idata`, `dev_odata`, and `dev_buffer` for each level $i \in [0, num\_scans - 1]$ with size `array_sizes[i]` (padded to power of two).
    - Copy input data `idata` to `dev_idata[0]` for $n$ elements.

2.  **Forward Pass (Inclusive Scan):**
    - Loop $i$ from $0$ to `num_scans - 1`:
        - Launch `kernScanInclusive` on `dev_idata[i]` into `dev_buffer[i]`.
        - Copy `dev_buffer[i]` back to `dev_idata[i]`.
        - If $i < num\_scans - 1$:
            - Extract the last element of each block from `dev_idata[i]` into `dev_idata[i+1]` using `kernExtractLastElementPerBlock` to prepare the next level's input.

3.  **Backward Pass (Exclusive Scan/Shift):**
    - Loop $i$ from `num_scans - 1` down to $0$:
        - Launch `kernShiftToExclusive` on `dev_buffer[i]` (which holds the inclusive result) to produce exclusive indices in `dev_odata[i]`.
        - If $i \ge 1$:
            - Launch `kernAddOffsetPerBlock` to add the offset from the previous level's exclusive result (`dev_odata[i]`) to the current level's buffer (`dev_buffer[i-1]`) to produce the final exclusive indices for the next lower level.

4.  **Finalization:**
    - Copy the final exclusive scan results from `dev_odata[0]` to host output `odata` for $n$ elements.

**Kernel: `kernScanInclusive` (Upsweep and Downsweep)**
This kernel operates in-place on a block of data. It assumes the input size is a power of two or padded appropriately.

-   **Inputs:** `idata` (device array), `n` (logical size), `blockDim.x` (threads per block).
-   **Local Variables:** `reduction_sum` (scalar), `temp` (scalar).

**Steps:**
1.  Calculate `log2n = ilog2ceil(min(n, blockDim.x))`.
2.  **Upsweep Phase:**
    -   For each thread with global index `id < n`:
        -   Iterate `d` from $0$ to `log2n - 1`:
            -   If `(id % (1 << (d + 1))) == 0`:
                -   `idata[id + (1 << (d + 1)) - 1] += idata[id + (1 << d) - 1]`
            -   `__syncthreads()`
3.  **Store Total Sum:**
    -   The last thread (`tid == blockDim.x - 1`) or the thread processing the last element (`id == n - 1`) performs:
        -   `reduction_sum = idata[n - 1]`
        -   `idata[n - 1] = 0`
    -   `__syncthreads()`
4.  **Downsweep Phase:**
    -   Iterate `d` from `log2n - 1` down to $0$:
        -   For each thread with global index `id < n`:
            -   If `(id % (1 << (d + 1))) == 0`:
                -   `temp = idata[id + (1 << d) - 1]`
                -   `idata[id + (1 << d) - 1] = idata[id + (1 << (d + 1)) - 1]`
                -   `idata[id + (1 << (d + 1)) - 1] += temp`
            -   `__syncthreads()`
5.  **Convert to Inclusive Output:**
    -   For each thread with global index `id < n`:
        -   If `id` is the last thread in the block or processes the last element:
            -   `odata[id] = reduction_sum`
        -   Else:
            -   `odata[id] = idata[id + 1]`

**Parameters:**
-   `block_size`: The CUDA block size used for kernel launches. Typical values tested: 1024, 512, 256, 128, 64, 32. Optimal value identified in profiling: 128 [S2, unofficial]. Note: Block size of 1024 does not yield ideal results, likely due to excessive block size reducing GPU parallelism [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Non-Power-of-Two:** For non-power-of-two arrays, intermediate array sizes must be rounded up to the next power of two, requiring extra memory [S3].
-   **SM Workload Imbalance:** The upsweep phase is the bottleneck, with some Streaming Multiprocessors (SMs) underutilized by as much as 43.66% due to workload imbalance [S1, unofficial].
-   **Memory Access:** High memory throughput indicates non-optimal memory access patterns [S1, unofficial].
-   **Reference Errors:** The pseudocode in GPU Gems 3 Chapter 39 Example 39-2 contains known errors [S3].

## Subtopics

### GPU Scan with Shared Memory and Hardware Optimization
An optimized GPU scan implementation using shared memory and bank conflict avoidance, based on GPU Gems 3 Chapter 39 Examples 39.1 and 39.2 [S3]. It optimizes memory access patterns to avoid shared memory bank conflicts [S3]. The size of shared memory is dynamic and related to the block size; occupancy is affected by the amount of shared memory used per block relative to the SM's limit [S3]. Using maximum shared memory per block reduces occupancy (e.g., max 1 block per SM), which may not yield best performance [S3].

## Operating characteristics
-   **Complexity:** Work-efficient: $O(n)$ total work and $O(\log n)$ parallel depth per block, compared to naive $O(n \log n)$ work [S2, unofficial].
-   **Performance:** The upsweep phase consumes the majority of execution time (averaging 28.38 ms), while the downsweep phase is shorter (averaging 20.26 ms) [S1, unofficial].
-   **Hardware:** Tested on Ubuntu 20.04 LTS, Ryzen 3700X @ 2.22GHz 48GB, RTX 2060 Super @ 7976MB [S2, unofficial].
-   **Strengths:** Works in-place on a single device array, avoiding race conditions of the naive method [S3]. Compatible with arbitrary sized input arrays [S2, unofficial].
-   **Limitations:** SM workload imbalance is a main bottleneck in the upsweep phase [S1, unofficial]. Requires extra memory for non-power-of-two arrays [S3].

## Background
The work-efficient scan algorithm is based on the "Work-Efficient" algorithm from GPU Gems 3, Section 39.2.2 [S3]. It is a standard parallel primitive used in stream compaction and other data-parallel operations [S2, unofficial].

## Sources
[S1] GitHub - jichumao/Stream-Compaction, https://github.com/jichumao/Stream-Compaction, fetched 2026-10-29 (unofficial)
[S2] GitHub - shineyruan/CUDA-Stream-Compaction, https://github.com/shineyruan/CUDA-Stream-Compaction, fetched 2026-10-29 (unofficial)
[S3] Project2-Stream-Compaction/INSTRUCTION.md at main, https://github.com/CIS565-Fall-2022/Project2-Stream-Compaction/blob/main/INSTRUCTION.md, fetched 2026-10-29
