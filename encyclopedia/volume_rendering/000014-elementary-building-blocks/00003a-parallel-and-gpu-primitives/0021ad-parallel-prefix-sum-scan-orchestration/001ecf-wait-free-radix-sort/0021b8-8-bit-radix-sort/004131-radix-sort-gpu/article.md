# Radix Sort (GPU)
Also called: RadixSortCountCS, RadixSortDigitPrefixSumCS, RadixSortPrefixSumCS, RadixSortScatterCS

## Overview
A GPU-based integer sorting algorithm implemented via four compute shader passes: histogram counting, prefix sum, digit prefix sum, and scatter [S1, unofficial]. It sorts keys by processing digits in a specific order, using parallel prefix sum (scan) as a subroutine to determine output positions [S2]. It is distinct from subgroup-based partitioning methods by explicitly separating the counting, offset calculation, and scattering phases into dedicated shader invocations [S1, unofficial][3368-subgroup-based-radix-sort-partition.md].

## How it works
The method processes a batch of elements through four sequential compute shader stages. Each stage operates on specific buffers and parameters.

### Inputs and Outputs
*   **Input Keys**: `SrcKeys` (StructuredBuffer<uint>)
*   **Input Values**: `SrcVals` (StructuredBuffer<uint>)
*   **Output Keys**: `DstKeys` (RWStructuredBuffer<uint>)
*   **Output Values**: `DstVals` (RWStructuredBuffer<uint>)
*   **Histogram**: `HistogramBuffer` (RWStructuredBuffer<uint>)
*   **Digit Offsets**: `DigitOffsetBuffer` (RWStructuredBuffer<uint>)
*   **Sort Parameters**: `SortParams` (StructuredBuffer<uint>) containing `RadixShift`, `Count`, `NumTiles`, `UseIndirectSort`.

### Data Structures
The parameters are packed into a single structured buffer `SortParams` with the following layout [S1, unofficial]:
*   `RadixShift`: `uint32` - The bit shift amount for the current radix digit being processed.
*   `Count`: `uint32` - The total number of elements to sort.
*   `NumTiles`: `uint32` - The number of tiles used for parallel histogramming.
*   `UseIndirectDispatch`: `uint32` - Flag indicating whether indirect dispatch is used.

The buffers are accessed as Unordered Access Views (UAVs) or Shader Resource Views (SRVs) [S1, unofficial].

### Steps

1.  **Histogram Counting (`RadixSortCountCS`)**
    *   **Input**: `SrcKeys`, `SortParams`.
    *   **Output**: `HistogramBuffer`.
    *   **Logic**: Each thread block processes a tile of elements. For each element, extract the current radix digit using `(key >> RadixShift) & 0xFF`. Increment the corresponding bin in the local or global histogram buffer.
    *   **Parameters**: `RadixShift`, `Count`, `NumTiles`, `UseIndirectSort` [S1, unofficial].

2.  **Prefix Sum (`RadixSortPrefixSumCS`)**
    *   **Input**: `HistogramBuffer`, `SortParams`.
    *   **Output**: `DigitOffsetBuffer`.
    *   **Logic**: Perform a parallel prefix sum (scan) on the `HistogramBuffer` to compute the starting offsets for each digit bin. The result is stored in `DigitOffsetBuffer`. This step determines the global starting position for each digit value in the output array [S1, unofficial][S2].

3.  **Digit Prefix Sum (`RadixSortDigitPrefixSumCS`)**
    *   **Input**: `DigitOffsetBuffer`.
    *   **Output**: `DigitOffsetBuffer` (updated).
    *   **Logic**: This pass appears to refine or finalize the offsets. In some implementations, this might involve a second scan or adjustment to handle specific memory layouts or indirect dispatch requirements. The source defines this as a distinct pass taking `DigitOffsetBuffer` as both input and output [S1, unofficial].

4.  **Scatter (`RadixSortScatterCS`)**
    *   **Input**: `SrcKeys`, `SrcVals`, `HistogramBuffer`, `DigitOffsetBuffer`, `SortParams`.
    *   **Output**: `DstKeys`, `DstVals`.
    *   **Logic**: For each element, extract the current radix digit. Use the `DigitOffsetBuffer` to find the starting position for that digit. Use an atomic add or a pre-computed position to scatter the key and value to the correct location in `DstKeys` and `DstVals`. Increment the offset for the next element with the same digit.
    *   **Parameters**: `RadixShift`, `Count`, `NumTiles`, `UseIndirectSort` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# FRadixSortCountCS
def RadixSortCountCS(SrcKeys, SortParams, HistogramBuffer):
    RadixShift = SortParams.RadixShift
    Count = SortParams.Count
    NumTiles = SortParams.NumTiles
    
    # Initialize histogram to zero
    for i in range(256):
        HistogramBuffer[i] = 0
        
    # Process elements
    for idx in range(Count):
        key = SrcKeys[idx]
        # Extract 8-bit digit
        digit = (key >> RadixShift) & 0xFF
        # Atomic increment
        HistogramBuffer[digit] += 1

# FRadixSortPrefixSumCS
def RadixSortPrefixSumCS(HistogramBuffer, DigitOffsetBuffer, SortParams):
    NumTiles = SortParams.NumTiles
    UseIndirectSort = SortParams.UseIndirectSort
    
    # Perform parallel prefix sum on HistogramBuffer
    # Result stored in DigitOffsetBuffer
    # Implementation depends on the specific scan algorithm used [S2]
    DigitOffsetBuffer = ParallelPrefixSum(HistogramBuffer)

# FRadixSortDigitPrefixSumCS
def RadixSortDigitPrefixSumCS(DigitOffsetBuffer):
    # Refine or finalize offsets
    # Specific logic not detailed in sources, but distinct pass [S1, unofficial]
    # Placeholder for specific digit offset adjustment
    DigitOffsetBuffer = AdjustDigitOffsets(DigitOffsetBuffer)

# FRadixSortScatterCS
def RadixSortScatterCS(SrcKeys, SrcVals, DstKeys, DstVals, HistogramBuffer, DigitOffsetBuffer, SortParams):
    RadixShift = SortParams.RadixShift
    Count = SortParams.Count
    
    for idx in range(Count):
        key = SrcKeys[idx]
        val = SrcVals[idx]
        
        # Extract 8-bit digit
        digit = (key >> RadixShift) & 0xFF
        
        # Get starting position for this digit
        # Note: In a full implementation, this might use atomic operations
        # or pre-computed positions from DigitOffsetBuffer
        pos = AtomicAdd(DigitOffsetBuffer[digit], 1)
        
        # Scatter to output
        DstKeys[pos] = key
        DstVals[pos] = val
```

### Edge Cases and Pitfalls
*   **DX11 UAV Limit**: DX11 has a limit on the number of UAVs, which may prevent running the full pipeline if not optimized [S1, unofficial].
*   **Indirect Dispatch**: The `UseIndirectSort` flag suggests that indirect dispatch may be required for certain configurations, likely to handle large counts or dynamic workloads [S1, unofficial].
*   **Precision**: Using `uint32` for keys and values ensures no precision loss for standard integer types.
*   **Boundaries**: The algorithm assumes `Count` is accurate and `SrcKeys`/`SrcVals` are valid.

## Operating characteristics
*   **Time Complexity**: $O(N \cdot D)$ where $N$ is the number of elements and $D$ is the number of digits (passes). For 32-bit integers with 8-bit radix, $D=4$.
*   **Memory Complexity**: $O(N + R)$ where $R$ is the radix size (256 for 8-bit).
*   **Strengths**: Efficient for large datasets on GPU; parallelizable histogram and scatter steps.
*   **Limitations**: Requires multiple passes; memory bandwidth intensive due to scatter step.
*   **When to use**: When sorting large arrays of integers on GPU where parallelism is beneficial.
*   **When not to use**: For small datasets where CPU sort might be faster due to overhead; when memory bandwidth is a critical bottleneck.

## Background
The method is described in the context of a GPU programming course project (Project2-Stream-Compaction) [S2] and implemented in the OpenSplat4D project [S1, unofficial]. The course project explicitly assigns radix sort as an extra credit task, requiring the use of a previously implemented scan subroutine [S2]. The OpenSplat4D implementation uses four distinct compute shader passes, reflecting a modular approach to GPU sorting [S1, unofficial].

## Sources
[S1] YuanBaoSMadLab/OpenSplat4D, https://github.com/YuanBaoSMadLab/OpenSplat4D, fetched 2026-10-27 (unofficial)
[S2] Project2-Stream-Compaction/INSTRUCTION.md at main, https://github.com/CIS565-Fall-2022/Project2-Stream-Compaction/blob/main/INSTRUCTION.md, fetched 2026-10-29
