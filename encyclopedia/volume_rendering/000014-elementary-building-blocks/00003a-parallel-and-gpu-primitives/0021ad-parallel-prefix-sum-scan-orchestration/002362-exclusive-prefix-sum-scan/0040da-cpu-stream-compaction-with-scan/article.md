# CPU Stream Compaction with Scan

Also called: Prefix Sum based Compaction, StreamCompaction::CPU::compactWithScan

## Overview
CPU Stream Compaction with Scan removes zero-valued elements from an integer array on the CPU by mapping input data to a binary mask, computing an exclusive prefix sum on that mask, and scattering the retained elements to their computed output indices [S1, unofficial][S3]. It sets itself apart from [CPU Stream Compaction without Scan](10532-cpu-stream-compaction-without-scan.md) by using a scan operation to determine output indices rather than a simple sequential counter [S3]. This approach is suitable for scenarios where parallel reduction primitives are available or when preparing data structures for subsequent parallel processing [S3].

## How it works
**Inputs and Outputs:**
- `idata`: Pointer to the constant input array of $n$ integers [S1, unofficial].
- `odata`: Pointer to the output buffer for compacted elements [S1, unofficial].
- `n`: Number of elements in the input array [S1, unofficial].
- Return value: The count of non-zero elements (result count) [S1, unofficial].

**Data Structures:**
1. `temp`: An integer array of size $n$ that stores the binary mask where `temp[i] = 1` if `idata[i] != 0`, else `0` [S1, unofficial].
2. `scanArray`: An integer array of size $n$ that stores the exclusive prefix sum of `temp` [S1, unofficial].

**Steps:**
1. **Map**: Convert the input integer array into a boolean array (stored as integers 0/1) where non-zero elements map to 1 and zero elements map to 0 [S3].
   - Formula: $M[i] = (I[i] == 0 ? 0 : 1)$ [S1, unofficial].
2. **Scan**: Compute the exclusive prefix sum of the boolean array $M$ [S3].
   - Formula: $S[0] = 0$, $S[i] = S[i-1] + M[i-1]$ for $i > 0$ [S1, unofficial].
   - This step relies on the parent method [Exclusive Prefix Sum (Scan)](3007-exclusive-prefix-sum-scan.md) [S3].
3. **Scatter**: Iterate through the original input array and place each non-zero element into the output array at the index specified by the corresponding value in the scanned array [S3].
   - Output index for $I[i]$ is $S[i]$ if $M[i]=1$ [S1, unofficial].
4. **Result Count**: Compute `resultCount = scanArray[n-1] + temp[n-1]` [S1, unofficial].
5. **Cleanup**: Free `temp` and `scanArray` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compact_with_scan(idata, n, odata):
    if n == 0:
        return 0
    temp = [0] * n
    for i in range(n):
        temp[i] = 1 if idata[i] != 0 else 0
    scanArray = [0] * n
    for i in range(1, n):
        scanArray[i] = scanArray[i - 1] + temp[i - 1]
    for i in range(n):
        if temp[i] == 1:
            odata[scanArray[i]] = idata[i]
    result_count = scanArray[n - 1] + temp[n - 1]
    return result_count
```

**Parameters:**
- `n`: Number of elements. Must be $\ge 0$ [S1, unofficial].
- `odata`: Must have sufficient capacity to hold all non-zero elements (max $n$) [G].

**Edge Cases and Pitfalls:**
- **Empty Input**: If $n=0$, the algorithm returns 0 and performs no operations [G].
- **All Zeros**: The result count will be 0, and `odata` will remain unchanged [S1, unofficial].
- **No Zeros**: The result count will be $n$, and `odata` will be a copy of `idata` [S1, unofficial].
- **Precision**: Integer overflow in the scan step is possible if $n$ is very large and the number of non-zero elements exceeds the maximum value of the integer type used for `scanArray` [G]. Use 64-bit integers for `scanArray` if $n > 2^{31}$ [G].

## Subtopics

### [GPU Stream Compaction with Scan](10376-gpu-stream-compaction-with-scan.md)
GPU Stream Compaction with Scan filters an input array of integers based on a boolean flag array, returning only elements where the flag is true into an output array while preserving their relative order. It sets itself apart from CPU-based compaction by leveraging the GPU's parallel architecture and a work-efficient parallel scan algorithm, achieving performance improvements of over 4x for large arrays. It is chosen when parallel execution is available and high throughput is required.

### [CPU Stream Compaction without Scan](10532-cpu-stream-compaction-without-scan.md)
CPU Stream Compaction without Scan removes zero-valued elements from an integer array on the CPU by iterating through the input and writing non-zero elements sequentially to an output buffer. It solves the problem of filtering data in-place or to a new buffer without the overhead of computing an exclusive prefix sum. Unlike its sibling [CPU Stream Compaction with Scan](10375-cpu-stream-compaction-with-scan.md), which requires a scan operation to determine output indices, this method maintains a simple counter, making it suitable for sequential CPU execution where parallel reduction is unnecessary or unavailable.

## Operating characteristics
- **Time Complexity**: $O(n)$ [S1, unofficial]. The map, scan, and scatter steps each take linear time [S1, unofficial].
- **Memory Complexity**: $O(n)$ auxiliary space for `temp` and `scanArray` [S1, unofficial].
- **Measured Performance**: In tests on a CPU, compaction without scan took ~250ms for a power-of-two array size [S1, unofficial]. The scan-based approach serves as a foundational precursor for GPU stream compaction implementations [S3].
- **Strengths**: Provides a structured, parallelizable approach to compaction [S3]. The output indices are computed sequentially on CPU or in parallel on GPU, making it easy to extend to multi-threaded CPU implementations or GPU kernels [S3].
- **Limitations**: Requires $O(n)$ extra memory for the mask and scan arrays [S1, unofficial]. On a single CPU thread, it incurs additional overhead compared to the simple counter method [S3].
- **When to Use**: When the output indices are needed for other parallel operations, or when the compaction is part of a larger pipeline that already uses scan operations (e.g., on GPU) [S3]. It is also useful when the compaction criteria are complex and need to be computed in parallel [S3].
- **When Not to Use**: For simple filtering on a single CPU thread where memory overhead is a concern and performance is critical; use [CPU Stream Compaction without Scan](10532-cpu-stream-compaction-without-scan.md) instead [S3].

## Background
The method is part of the `StreamCompaction::CPU` namespace, as described in educational materials for GPU programming courses [S2, unofficial][S3]. It is a foundational technique for data parallelism, often taught as a precursor to GPU stream compaction [S3]. The approach is derived from standard parallel algorithms for stream compaction, such as those found in GPU Gems [S3].

## Sources
[S1] GitHub - jichumao/Stream-Compaction, https://github.com/jichumao/Stream-Compaction, fetched 2026-10-29 (unofficial)
[S2] GitHub - shineyruan/CUDA-Stream-Compaction, https://github.com/shineyruan/CUDA-Stream-Compaction, fetched 2026-10-29 (unofficial)
[S3] Project2-Stream-Compaction/INSTRUCTION.md at main, https://github.com/CIS565-Fall-2022/Project2-Stream-Compaction/blob/main/INSTRUCTION.md, fetched 2026-10-29
