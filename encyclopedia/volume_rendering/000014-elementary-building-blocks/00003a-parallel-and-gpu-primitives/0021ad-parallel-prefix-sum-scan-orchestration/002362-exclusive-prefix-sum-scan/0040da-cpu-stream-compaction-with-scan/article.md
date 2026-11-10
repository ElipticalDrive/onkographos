# CPU Stream Compaction with Scan

Also called: Prefix Sum based Compaction, StreamCompaction::CPU::compactWithScan

## Overview
CPU Stream Compaction with Scan removes zero-valued elements from an integer array on the CPU by mapping input data to a binary mask, computing an exclusive prefix sum on that mask, and scattering the retained elements to their computed output indices [S1, unofficial][S3]. It sets itself apart from [CPU Stream Compaction without Scan](10532-cpu-stream-compaction-without-scan.md) by using a parallel prefix sum (scan) to determine output positions, which allows for potential parallelization of the scatter phase, whereas the without-scan variant uses a sequential counter [S3]. It differs from [GPU Stream Compaction with Scan](10376-gpu-stream-compaction-with-scan.md) by executing these steps on the CPU, typically resulting in lower performance for large arrays due to the lack of massive parallelism [S2, unofficial].

## How it works
**Inputs and Outputs:**
- `idata`: Pointer to the constant input array of $n$ integers.
- `n`: Number of elements in the input array.
- `odata`: Pointer to the output buffer for compacted elements. The buffer must be large enough to hold all non-zero elements (size $\ge n$).
- **Output:** The number of compacted elements (count of non-zero values).

**Data Structures:**
1. `temp`: An integer array of size $n$. Stores the binary mask where `temp[i] = 1` if `idata[i] != 0`, else `0`.
2. `scanArray`: An integer array of size $n$. Stores the exclusive prefix sum of `temp`.

**Steps:**
1. **Map:** Convert `idata` into a binary mask `temp`. For each index $i$ from $0$ to $n-1$:
   $$M[i] = (I[i] == 0 ? 0 : 1)$$
   Set `temp[i] = 1` if `idata[i]` is non-zero, otherwise `0` [S1, unofficial][S3].

2. **Scan:** Compute the exclusive prefix sum of `temp` into `scanArray`.
   - Initialize `scanArray[0] = 0` [S1, unofficial].
   - For $i$ from $1$ to $n-1$:
     $$S[i] = S[i-1] + M[i-1]$$
     Set `scanArray[i] = scanArray[i-1] + temp[i-1]` [S1, unofficial][S3].

3. **Scatter:** Iterate through the original input array. For each index $i$ from $0$ to $n-1$:
   - If `temp[i] == 1` (the element is non-zero):
     - Place `idata[i]` into `odata` at index `scanArray[i]`.
     - $$O[S[i]] = I[i] \text{ if } M[i]=1$$ [S1, unofficial].

4. **Count:** Compute the total number of compacted elements.
   - If $n > 0$: `resultCount = scanArray[n-1] + temp[n-1]`.
   - If $n = 0$: `resultCount = 0`.
   - This value represents the exclusive prefix sum of the last element plus its mask value, which equals the total count of ones in `temp` [S1, unofficial].

5. **Cleanup:** Free `temp` and `scanArray` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compact_with_scan(idata, n, odata):
    if n == 0:
        return 0

    # Step 1: Map
    temp = [0] * n
    for i in range(n):
        temp[i] = 1 if idata[i] != 0 else 0

    # Step 2: Scan (Exclusive Prefix Sum)
    scanArray = [0] * n
    scanArray[0] = 0
    for i in range(1, n):
        scanArray[i] = scanArray[i - 1] + temp[i - 1]

    # Step 3: Scatter
    for i in range(n):
        if temp[i] == 1:
            odata[scanArray[i]] = idata[i]

    # Step 4: Count
    resultCount = scanArray[n - 1] + temp[n - 1]

    # Step 5: Cleanup (conceptual, as Python handles memory)
    # del temp
    # del scanArray

    return resultCount
```

**Parameters:**
- `n`: Number of elements. Must be $\ge 0$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Empty Input:** If $n=0$, the algorithm should return 0 immediately to avoid index errors [S1, unofficial].
- **All Zeros:** If all elements are zero, `temp` is all zeros, `scanArray` is all zeros, and `odata` remains unchanged (or undefined depending on initialization). `resultCount` will be 0 [S3].
- **No Zeros:** If no elements are zero, `temp` is all ones, `scanArray` is $[0, 1, 2, ..., n-1]$, and `odata` is a copy of `idata`. `resultCount` is $n$ [S3].
- **Precision:** The algorithm uses integer arithmetic. For very large $n$, the sum in the scan step might overflow if the data type is not sufficiently large (e.g., 32-bit int for $n > 2^{31}$) [G].

## Subtopics

### [GPU Stream Compaction with Scan](10376-gpu-stream-compaction-with-scan.md)
GPU Stream Compaction with Scan efficiently removes elements from a stream that do not satisfy a specific criteria (typically zero values) by mapping input data to a binary mask, computing an exclusive prefix sum on that mask, and scattering the retained elements to their computed output indices. It sets itself apart from CPU-based compaction by leveraging the GPU's parallel architecture and a work-efficient parallel scan algorithm, achieving performance improvements of over 4x for large arrays.

### [CPU Stream Compaction without Scan](10532-cpu-stream-compaction-without-scan.md)
Removes zero-valued elements from an array on CPU by iterating through the input and writing non-zero elements sequentially to an output buffer.

## Operating characteristics
- **Time Complexity:** $O(n)$ for mapping, $O(n)$ for scan, and $O(n)$ for scattering. Total time is $O(n)$ [S1, unofficial].
- **Memory Complexity:** $O(n)$ auxiliary space for `temp` and `scanArray` [S1, unofficial].
- **Measured Performance:** In tests on a CPU, stream compaction without scan took approximately 251ms for a power-of-two array size. The CPU scan-based method was not directly timed in the provided snippets, but the CPU scan itself took ~190ms for a similar sized array, suggesting the total time for compaction with scan would be significantly higher than without scan for CPU execution due to the overhead of the scan and scatter phases compared to a single pass [S1, unofficial].
- **Strengths:** The scan-based approach is a fundamental building block for parallel algorithms and can be easily adapted to GPU implementations where the scatter phase can be fully parallelized [S3]. It provides a deterministic mapping for each element.
- **Limitations:** On CPU, the sequential nature of the scan (if implemented naively as in the notes) and the scatter phase can be slower than a simple single-pass compaction without scan, especially for small to medium arrays [S1, unofficial][S3]. It requires extra memory for the mask and scan arrays [S1, unofficial].
- **When to use:** Use when the downstream processing requires the indices provided by the scan or when porting the algorithm to a GPU where the scan and scatter can be parallelized efficiently [S3]. Use when the input size is large enough that the overhead of extra memory and passes is justified by the potential for parallelization (on GPU) or specific algorithmic requirements.
- **When not to use:** For simple CPU-based compaction of small to medium arrays, [CPU Stream Compaction without Scan](10532-cpu-stream-compaction-without-scan.md) is likely faster due to lower overhead and better cache locality [S1, unofficial][S3].

## Background
The method is part of the `StreamCompaction::CPU` namespace, often used in educational contexts to teach parallel programming concepts before moving to GPU implementations [S3]. It is based on the standard stream compaction algorithm described in resources like GPU Gems [S3]. The technique relies on the exclusive prefix sum (scan) operation, which is a fundamental primitive in parallel computing [S1, unofficial][S3]. The specific implementation details provided in the notes are from student projects and unofficial repositories [S1, S2, unofficial][S3].

## Sources
[S1] GitHub - jichumao/Stream-Compaction, https://github.com/jichumao/Stream-Compaction, fetched 2026-10-29 (unofficial)
[S2] GitHub - shineyruan/CUDA-Stream-Compaction, https://github.com/shineyruan/CUDA-Stream-Compaction, fetched 2026-10-29 (unofficial)
[S3] Project2-Stream-Compaction/INSTRUCTION.md at main, https://github.com/CIS565-Fall-2022/Project2-Stream-Compaction/blob/main/INSTRUCTION.md, fetched 2026-10-29
