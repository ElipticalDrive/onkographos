# GPU Stream Compaction with Scan

Also called: Map-Scan-Scatter, Stream Compaction via Scan, GPU Stream Compaction (Work-Efficient)

## Overview
GPU Stream Compaction with Scan efficiently removes elements from a stream that do not satisfy a specific criteria (typically zero values) by mapping input data to a binary mask, computing an exclusive prefix sum on that mask, and scattering the retained elements to their computed output indices [S1, S2, unofficial][S3]. It sets itself apart from CPU-based compaction by leveraging the GPU's parallel architecture and a work-efficient parallel scan algorithm, achieving performance improvements of over 4x for large arrays [S2, unofficial].

## How it works
**Inputs and Outputs:**
- `idata`: Pointer to the constant input array of integers (size $n$).
- `odata`: Pointer to the output buffer (size at least $n$).
- `n`: Number of elements in the input array.

**Data Structures:**
The algorithm requires four device arrays, all allocated with size $N = 2^{\lceil \log_2 n \rceil}$ (the next power of two greater than or equal to $n$) [S1, unofficial]:
1. `dev_idata`: `int` array of size $N$. Stores the input data, zero-padded from index $n$ to $N-1$.
2. `dev_temp`: `int` array of size $N$. Stores the binary mask (1 for keep, 0 for discard).
3. `dev_scanArr`: `int` array of size $N$. Stores the exclusive prefix sum of the mask, representing output indices.
4. `dev_odata`: `int` array of size $N$. Stores the compacted output.

**Steps:**
1. **Preprocessing (Zero-Padding):**
   - Compute $N = 2^{\lceil \log_2 n \rceil}$ [S1, unofficial].
   - Allocate `dev_idata`, `dev_odata`, `dev_temp`, and `dev_scanArr` of size $N$ [S1, unofficial].
   - Copy `idata` to `dev_idata` for the first $n$ elements [S1, unofficial].
   - Zero-pad `dev_idata` from index $n$ to $N-1$ [S1, unofficial].

2. **Map (Kernel `kernMapToBoolean`):**
   - For each thread $i$ in $[0, N)$:
     - If `dev_idata[i] != 0`, set `dev_temp[i] = 1`.
     - Else, set `dev_temp[i] = 0` [S1, unofficial][S3].

3. **Scan (Exclusive Prefix Sum):**
   - Perform a work-efficient parallel exclusive scan on `dev_temp` to produce `dev_scanArr` [S1, unofficial][S3].
   - The work-efficient scan typically involves:
     - **Up-Sweep:** A parallel reduction to compute inclusive sums in a tree structure [S1, unofficial].
     - **Root Reset:** Set the root of the tree (index $N-1$) to 0 [S1, unofficial].
     - **Down-Sweep:** Propagate sums down the tree to compute exclusive sums [S1, unofficial].
   - Note: The scan must handle the padded zeros correctly; since they are 0, they do not affect the sum but ensure the tree structure remains complete [S1, unofficial].

4. **Scatter (Kernel `kernScatter`):**
   - For each thread $i$ in $[0, N)$:
     - If `dev_temp[i] == 1`:
       - Let `idx = dev_scanArr[i]`.
       - Set `dev_odata[idx] = dev_idata[i]` [S1, unofficial][S3].
     - Else:
       - Do nothing (or leave `dev_odata[idx]` unchanged, though only indices from `dev_scanArr` where `dev_temp[i]==1` are valid output positions) [S1, unofficial].

5. **Result Count:**
   - The number of remaining elements is the last element of the exclusive scan array (`dev_scanArr[N-1]`) plus the value of the last element of the mask if the mask was inclusive, but for exclusive scan, the count is simply the value at `dev_scanArr[N-1]` if the last element was kept, or more generally, the sum of the mask. Since `dev_scanArr` is exclusive, the index for the last kept element is `dev_scanArr[last_kept_index]`. The total count of kept elements is `dev_scanArr[N-1]` if the last element of the mask was 0, or `dev_scanArr[N-1] + 1` if the last element was 1? No, the exclusive scan value at index $i$ is the sum of elements $0$ to $i-1$. The total number of 1s is the sum of the mask. The last element of the exclusive scan array (`dev_scanArr[N-1]`) is the sum of `dev_temp[0]` to `dev_temp[N-2]`. The total count is `dev_scanArr[N-1] + dev_temp[N-1]`. However, a simpler way is to note that the maximum index written is `dev_scanArr[i]` for the last kept element. The count is the sum of the mask. In the provided code, the count is derived from the scan result. Specifically, `dev_scanArr[N-1]` is the number of elements before the last element. If the last element is kept, the count is `dev_scanArr[N-1] + 1`. If not, it is `dev_scanArr[N-1]`. A robust way is to compute the sum of the mask, which is the last element of an *inclusive* scan. Since we use exclusive, the count is `dev_scanArr[N-1] + dev_temp[N-1]` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def gpu_stream_compaction_with_scan(idata, n):
    # Step 1: Preprocessing
    # Compute next power of two
    if n == 0:
        return []
    
    # ilog2ceil(n)
    if n == 1:
        N = 1
    else:
        N = 1 << (math.ceil(math.log2(n)))
    
    # Allocate device arrays (simulated)
    dev_idata = [0] * N
    dev_odata = [0] * N
    dev_temp = [0] * N
    dev_scanArr = [0] * N
    
    # Copy and pad input
    for i in range(n):
        dev_idata[i] = idata[i]
    # dev_idata[n:] is already 0
    
    # Step 2: Map to Boolean
    for i in range(N):
        if dev_idata[i] != 0:
            dev_temp[i] = 1
        else:
            dev_temp[i] = 0
            
    # Step 3: Exclusive Scan (Work-Efficient)
    # This is a simplified simulation of the work-efficient scan
    # In reality, this is done via parallel kernels (Up-Sweep, Root Reset, Down-Sweep)
    # Here we compute it sequentially for correctness of the algorithm description
    # The result is an exclusive prefix sum
    
    # Inclusive scan first
    inclusive = [0] * N
    inclusive[0] = dev_temp[0]
    for i in range(1, N):
        inclusive[i] = inclusive[i-1] + dev_temp[i]
        
    # Exclusive scan: shift right, set first to 0
    dev_scanArr[0] = 0
    for i in range(1, N):
        dev_scanArr[i] = inclusive[i-1]
        
    # Step 4: Scatter
    for i in range(N):
        if dev_temp[i] == 1:
            idx = dev_scanArr[i]
            dev_odata[idx] = dev_idata[i]
            
    # Step 5: Count
    # Total count is the sum of dev_temp
    count = sum(dev_temp)
    
    # Return compacted data and count
    return dev_odata[:count], count
```

**Parameters:**
- `n`: Number of elements in the input array [S1, unofficial].
- `blockSize`: Thread block size for CUDA kernels (e.g., 1024) [S2, unofficial].
- `gridSize`: Calculated as `(N + blockSize - 1) / blockSize` [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Non-power-of-two sizes:** Input arrays must be zero-padded to the next power of two to ensure the work-efficient scan tree is complete [S1, unofficial][S3].
- **Precision:** Integer types are used for indices and counts, avoiding floating-point precision issues [S1, unofficial].
- **Boundaries:** The scan must handle the padded zeros correctly. Since they are 0, they do not contribute to the sum, but they ensure the tree structure is balanced [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ work and $O(\log N)$ depth for the scan, where $N$ is the next power of two greater than or equal to $n$ [S1, unofficial].
- **Memory Complexity:** $O(N)$ additional memory for `dev_temp` and `dev_scanArr` [S1, unofficial].
- **Measured Performance:** On an RTX 2060 Super, for an array size of $2^{27}$, the work-efficient compact took 58.4025ms for power-of-two and 48.5331ms for non-power-of-two, over 4x faster than the CPU version (886.643ms) [S2, unofficial].
- **Strengths:** High performance on GPU, handles arbitrary input sizes via padding, work-efficient scan minimizes redundant operations [S1, S2, unofficial].
- **Limitations:** Requires additional memory for intermediate arrays, padding overhead for non-power-of-two sizes [S1, unofficial].
- **When to use:** When compacting large arrays on GPU and performance is critical [S2, unofficial].
- **When not to use:** For small arrays where CPU overhead is negligible, or when memory is constrained [S1, unofficial].

## Background
The method is part of the broader field of parallel prefix sum (scan) algorithms and stream compaction, widely used in graphics and scientific computing. The work-efficient scan algorithm was popularized by Hillis and Steele, and later optimized for GPUs. The specific implementation details, including the `kernMapToBoolean` and `kernScatter` kernels, are derived from educational projects and GitHub repositories focusing on CUDA stream compaction [S1, S2, unofficial][S3].

## Sources
[S1] GitHub - jichumao/Stream-Compaction, https://github.com/jichumao/Stream-Compaction, fetched 2026-10-29 (unofficial)
[S2] GitHub - shineyruan/CUDA-Stream-Compaction, https://github.com/shineyruan/CUDA-Stream-Compaction, fetched 2026-10-29 (unofficial)
[S3] Project2-Stream-Compaction/INSTRUCTION.md at main, https://github.com/CIS565-Fall-2022/Project2-Stream-Compaction/blob/main/INSTRUCTION.md, fetched 2026-10-29
