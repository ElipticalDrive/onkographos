# CPU Stream Compaction without Scan

Also called: CPU Stream Compaction, CPU compact without scan, Linear Scan Compaction, Naive Stream Compaction (without Scan), Sequential Stream Compaction, StreamCompaction::CPU::compactWithoutScan

## Overview
CPU Stream Compaction without Scan removes zero-valued elements from an integer array on the CPU by iterating through the input and writing non-zero elements sequentially to an output buffer [S1, unofficial][S3]. It solves the problem of filtering data without the overhead of computing an exclusive prefix sum [S3]. Unlike its sibling [CPU Stream Compaction with Scan](10375-cpu-stream-compaction-with-scan.md), which requires a scan operation to determine output indices, this method maintains a simple counter, making it suitable for sequential CPU execution [S3].

## How it works
**Inputs and Outputs:**
- `idata`: Pointer to the constant input array of integers [S1, unofficial].
- `n`: Number of elements in the input array [S1, unofficial].
- `odata`: Pointer to the output buffer for compacted elements [S1, unofficial].

**Data Structures:**
- The method uses two contiguous memory blocks: the input array `idata` of size $n$ and the output array `odata` of size at least $n$ [S1, unofficial]. No auxiliary data structures are required [S1, unofficial].

**Algorithm Steps:**
1. Initialize a counter variable `cnt` to 0 [S1, unofficial]. This counter tracks the number of non-zero elements found and serves as the write index for `odata` [S3].
2. Iterate through the input array `idata` with an index `i` from 0 to $n-1$ [S1, unofficial].
3. For each element `idata[i]`, check if it is non-zero [S1, unofficial].
4. If `idata[i]` is not equal to 0, write `idata[i]` to `odata[cnt]` and increment `cnt` by 1 [S1, unofficial].
5. After the loop completes, `cnt` holds the total number of non-zero elements, which is the new size of the compacted array [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compact_without_scan(idata, n, odata):
    cnt = 0
    for i in range(n):
        if idata[i] != 0:
            odata[cnt] = idata[i]
            cnt += 1
    return cnt
```

**Parameters:**
- `n`: The length of the input array [S1, unofficial].
- `idata`: Input array [S1, unofficial].
- `odata`: Output array [S1, unofficial].

**Edge Cases and Pitfalls:**
- If the input array contains no non-zero elements, `cnt` remains 0, and `odata` is not written to [S1, unofficial].
- If all elements are non-zero, `odata` becomes a copy of `idata` [S3].
- The output buffer `odata` must be allocated with a size of at least $n$ to prevent buffer overflows, as the worst-case scenario requires full capacity [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time complexity is $O(n)$, as the algorithm performs a single pass through the input array [S1, unofficial].
- Auxiliary space complexity is $O(1)$, as only a single integer counter is used [S1, unofficial].

**Measured Performance:**
- Not given in the sources.

**Scaling:**
- The method scales linearly with input size [S1, unofficial].

**Strengths and Limitations:**
- Strengths: Simple implementation, low overhead, and no need for prefix sum logic [S3].
- Limitations: Inherently sequential and not easily parallelized on GPU architectures compared to scan-based approaches [S3].

**When to use it:**
- Use when implementing on CPU where simplicity is preferred [S3].
- Use when the input size is small or when scan primitives are not available [S3].

**When not to use it:**
- Not given in the sources.

## Subtopics

### CPU Stream Compaction with Scan
Computes an exclusive prefix sum on a binary mask to determine output indices, enabling parallel scattering [G]. Choose when integrating with GPU pipelines or requiring parallel execution [G]. [10375-cpu-stream-compaction-with-scan.md]

### GPU Stream Compaction with Scan
Optimized for GPU architectures, leveraging shared memory and warp-level primitives for high throughput [G]. Choose when maximum parallel throughput is required [G]. [10376-gpu-stream-compaction-with-scan.md]

## Background
The method is part of the `StreamCompaction::CPU` namespace in educational and open-source stream compaction projects [S3]. It is designed to provide a baseline sequential implementation for comparison against parallel GPU algorithms [S3]. The approach is described in academic projects such as those from CIS565, where it serves as a reference for understanding the fundamental logic of stream compaction before moving to complex GPU implementations [S3].

## Sources
[S1] GitHub - jichumao/Stream-Compaction, https://github.com/jichumao/Stream-Compaction, fetched 2026-10-29 (unofficial)
[S2] GitHub - shineyruan/CUDA-Stream-Compaction, https://github.com/shineyruan/CUDA-Stream-Compaction, fetched 2026-10-29 (unofficial)
[S3] Project2-Stream-Compaction/INSTRUCTION.md at main, https://github.com/CIS565-Fall-2022/Project2-Stream-Compaction/blob/main/INSTRUCTION.md, fetched 2026-10-29
