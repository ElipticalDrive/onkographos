# Naive Parallel Prefix Sum (Scan)

Also called: Naive GPU Scan, Naive Parallel Inclusive Scan, Naive Parallel Scan, Naive Scan, Naïve algorithm, StreamCompaction::Naive::scan, kernNaiveScan

## Overview
The Naive Parallel Prefix Sum (Scan) is a GPU-based algorithm for computing the inclusive prefix sum of an array using global memory and multiple kernel invocations [S3]. It serves as a baseline implementation compatible with arbitrary input sizes, distinguishing itself from work-efficient variants by its iterative, level-by-level reduction approach which results in higher computational work ($O(n \log n)$) but simpler synchronization logic within each kernel launch [S2, unofficial][S3]. Unlike in-place tree-based methods, it avoids race conditions by double-buffering data between two device arrays across iterations [S3].

## How it works
**Inputs and Outputs:**
- Input: An array of $n$ integers `idata` on the host.
- Output: An array of $n$ integers `odata` on the host containing the exclusive prefix sum.

**Data Structures:**
- Two device arrays, `dev_data1` and `dev_data2`, of size sufficient to hold the input (padded to power-of-two if required by specific implementations, though the naive method described here handles arbitrary sizes via block-level processing) [S1, S2, unofficial].
- The algorithm processes the input in chunks fitting within a single thread block. If $n$ exceeds the block size, the array is split into multiple blocks, and the scan is performed recursively or iteratively across blocks using intermediate kernels to propagate block sums [S2, unofficial].

**Parameters:**
- `blockSize`: Number of threads per block. Typically 256 [S2, unofficial] or 512 [S1, unofficial].
- `ilog2ceil(n)`: The ceiling of the base-2 logarithm of $n$.

**Steps:**
1. **Allocation:** Allocate two device arrays `dev_data1` and `dev_data2` of size $n$ (or padded size) [S3]. Copy `idata` to `dev_data1` [S2, unofficial].
2. **Iteration Loop:** Calculate the number of iterations required: $L = \lceil \log_2(\min(n, \text{blockSize})) \rceil$ [S2, unofficial]. For each iteration $d$ from 1 to $L$:
   - Launch a kernel `kernNaiveScan` with `blocksPerGrid = \lceil n / blockSize \rceil` and `blockSize` threads [S1, unofficial].
   - The kernel parameter `offset` is $2^{d-1}$ [S1, unofficial].
   - Inside the kernel for each thread with global index `id`:
     - Read `data[id]` into a local buffer or shared memory (if used, but global memory double-buffering is the standard naive approach described) [S2, unofficial][S3].
     - Synchronize threads within the block (`__syncthreads()`) [S2, unofficial].
     - If `threadIdx.x >= offset`, update the value: `buffer[id] = data[id - offset] + data[id]` [S2, unofficial].
     - Synchronize threads (`__syncthreads()`) [S2, unofficial].
     - Write `buffer[id]` back to `data[id]` (or the other buffer) [S2, unofficial].
   - Swap `dev_data1` and `dev_data2` pointers after each kernel launch to double-buffer results [S1, unofficial].
3. **Post-Processing (Exclusive Scan):**
   - The result in `dev_data1` is an inclusive scan. Convert to exclusive scan:
     - Set `odata[0] = 0` [S1, unofficial].
     - Copy `dev_data1[1...n-1]` to `odata[1...n-1]` [S1, unofficial].
   - If the input size $n$ was larger than the block size, additional kernels (`kernExtractLastElementPerBlock`, `kernAddOffsetPerBlock`, `kernShiftToExclusive`) are used to combine block results and shift to exclusive [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def naive_scan(n, idata):
    # 1. Allocation and Initialization
    dev_data1 = cuda_malloc(n * sizeof(int))
    dev_data2 = cuda_malloc(n * sizeof(int))
    cuda_memcpy(dev_data1, idata, n * sizeof(int))
    
    block_size = 256  # Typical value [S2, unofficial]
    log2n = ilog2ceil(min(n, block_size))
    
    # 2. Iterative Kernel Launches
    for d in range(1, log2n + 1):
        offset = 1 << (d - 1)
        blocks_per_grid = (n + block_size - 1) // block_size
        
        # Launch kernel with current data source and target
        # Note: In a real implementation, this is a CUDA kernel launch.
        # Here we describe the logic performed by threads in kernNaiveScan.
        
        launch_kernel_kernNaiveScan(
            blocks=blocks_per_grid,
            threads=block_size,
            n=n,
            data_src=dev_data1,
            data_dst=dev_data2,
            offset=offset
        )
        
        # Swap buffers for next iteration
        dev_data1, dev_data2 = dev_data2, dev_data1

    # 3. Convert Inclusive to Exclusive
    odata = cuda_malloc(n * sizeof(int))
    
    # odata[0] = 0
    cuda_memset(odata, 0, sizeof(int))
    
    # Copy inclusive results from index 1 to n-1
    if n > 1:
        cuda_memcpy(odata + 1, dev_data1 + 1, (n - 1) * sizeof(int))
        
    return odata

def kernNaiveScan(n, data_src, data_dst, offset):
    id = blockIdx.x * blockDim.x + threadIdx.x
    tx = threadIdx.x
    
    if id < n:
        # Read from source
        val = data_src[id]
        
        # Synchronize to ensure all reads are complete before writes
        __syncthreads__()
        
        # Update logic
        if tx >= offset:
            neighbor_val = data_src[id - offset]
            data_dst[id] = neighbor_val + val
        else:
            data_dst[id] = val
            
        # Synchronize to ensure all writes are complete
        __syncthreads__()
```

## Subtopics

### [Work-Efficient Scan](10349-work-efficient-scan.md)
A parallel algorithm to compute the inclusive prefix sum of an array using a two-phase approach (upsweep/reduction and downsweep) to achieve work efficiency ($O(n)$ work, $O(\log n)$ depth). It builds on [Naive Parallel Prefix Sum (Scan)](10348-naive-parallel-prefix-sum-scan.md) by replacing the iterative $O(n \log n)$ work with a binary tree reduction, eliminating race conditions through in-place updates and synchronization. It is distinguished from the naive method by its tree-based structure and from hardware-optimized variants by its explicit multi-kernel orchestration for arbitrary input sizes.

### Naive GPU Parallel Scan
A naive GPU-based parallel scan implementation, implemented in the `StreamCompaction::Naive` namespace [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(\log n)$ per block, but requires multiple kernel launches for arrays larger than block size, leading to higher overhead [S2, unofficial].
- **Memory Complexity:** $O(n)$ for double-buffering (two device arrays) [S3].
- **Performance:** On an RTX 2060 Super, the naive scan took 31.3315ms for a power-of-two array size $2^{27}$ and 24.8398ms for a non-power-of-two array size $2^{27}$ [S2, unofficial]. Performance improves as block size increases from 32 to 128, reaching optimal performance at 64 threads per block, with no further gains at higher sizes [S1, unofficial].
- **Strengths:** Simple implementation, compatible with arbitrary input sizes, no shared memory management required (if using global memory double-buffering) [S2, unofficial][S3].
- **Limitations:** Not work-efficient ($O(n \log n)$ total work), suffers from warp divergence and uncoalesced global memory accesses due to interleaved thread utilization patterns [S2, unofficial]. Requires `ilog2ceil(n)` kernel launches, which adds overhead [S3].
- **When to use:** As a baseline for comparison or when implementation simplicity is prioritized over performance [S2, unofficial].
- **When not to use:** For large arrays where performance is critical, or when work efficiency is required [S2, unofficial][S3].

## Background
The Naive Parallel Prefix Sum algorithm is described in GPU Gems 3, Chapter 39, Section 39.2.1 [S3]. It is a foundational parallel primitive used in stream compaction and other data-parallel algorithms [S2, unofficial]. The method is notable for its simplicity but is known to have errors in the pseudocode and CUDA code provided in the original GPU Gems chapter (e.g., superscripting, missing braces, bad indentation) [S3].

## Sources
[S1] GitHub - jichumao/Stream-Compaction, https://github.com/jichumao/Stream-Compaction, fetched 2026-10-29 (unofficial)
[S2] GitHub - shineyruan/CUDA-Stream-Compaction, https://github.com/shineyruan/CUDA-Stream-Compaction, fetched 2026-10-29 (unofficial)
[S3] Project2-Stream-Compaction/INSTRUCTION.md at main, https://github.com/CIS565-Fall-2022/Project2-Stream-Compaction/blob/main/INSTRUCTION.md, fetched 2026-10-29
