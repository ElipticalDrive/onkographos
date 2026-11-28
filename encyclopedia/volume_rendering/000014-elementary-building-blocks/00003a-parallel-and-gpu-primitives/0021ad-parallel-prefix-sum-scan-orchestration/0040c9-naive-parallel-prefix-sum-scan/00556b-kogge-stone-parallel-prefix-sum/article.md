# Kogge-Stone Parallel Prefix Sum
Also called: Kogge-Stone's algorithm
## Overview
The Kogge-Stone algorithm is a parallel prefix sum (scan) method that computes inclusive prefix sums in a single phase using an in-place update strategy [S1]. Unlike Blelloch’s algorithm, which employs a two-phase (up-sweep/down-sweep) approach, Kogge-Stone’s single-phase structure simplifies functional correctness verification while maintaining data race freedom through explicit synchronization [S1]. It serves as a foundational primitive for volume rendering pipelines requiring cumulative distribution functions or histogram binning, building on the general principles of [Naive Parallel Prefix Sum (Scan)](10348-naive-parallel-prefix-sum-scan.md) [G].

## How it works
**Inputs and Outputs:**
- Input: An array `Input` of size $N$ [S1].
- Output: An array `Output` of size $N$, which initially holds the input values and is overwritten with the inclusive prefix sums [S1].
- Parameters: $N$ (array length), `offset` (distance between elements, starting at 1), `tid` (thread ID) [S1].

**Data Structures:**
- `Output`: A global memory array of size $N$. The algorithm operates in-place, meaning the output overwrites the input [S1].
- `temp`: A local register variable used to store the intermediate sum for a thread to prevent data races during in-place updates [S1].

**Algorithm Steps:**
1. Initialize `offset` to 1 [S1].
2. Each thread $i$ (where $0 \le i < N$) executes the following loop:
   a. Read the current value at `Output[i]` into `temp` [S1].
   b. If $i \ge \text{offset}$, add `Output[i - offset]` to `temp` [S1].
   c. Execute a thread block barrier to ensure all threads have read their required values before any writes occur [S1].
   d. If $i \ge \text{offset}$, write `temp` back to `Output[i]` [S1].
   e. Execute a second thread block barrier to ensure all writes are complete before updating the offset [S1].
   f. Double the offset: $\text{offset} = 2 \times \text{offset}$ [S1].
3. Repeat until $\text{offset} \ge N$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inclusive_prefix_sum_kernel(Output, N, tid):
    offset = 1
    while offset < N:
        temp = Output[tid]
        if tid >= offset:
            temp = Output[tid - offset] + temp
        # Barrier: ensure all threads finish reading previous values
        # __syncthreads()
        if tid >= offset:
            Output[tid] = temp
        # Barrier: ensure all threads finish writing before next iteration
        # __syncthreads()
        offset = 2 * offset
```

**Parameters:**
- `offset`: The distance between elements to be summed. It starts at 1 and doubles each iteration ($1, 2, 4, \dots$) until it exceeds $N$ [S1].
- `temp`: An auxiliary variable used to store the intermediate sum for a thread to avoid data races during in-place updates [S1].

**Edge Cases and Pitfalls:**
- **Data Races:** Since the algorithm is in-place, threads must read current values before updating. This is handled by using the auxiliary variable `temp` and barriers to synchronize reads and writes [S1]. Without barriers, a thread might read a value that has already been updated by another thread in the same iteration, leading to incorrect results [G].
- **Boundary Conditions:** The condition `tid >= offset` ensures that threads do not access memory locations outside the array bounds (specifically, `tid - offset` would be negative) [G].
- **Precision:** Not given in the sources.
- **Degenerate Input:** Not given in the sources.

## Subtopics

### Work-Efficient Scan
A variant that reduces the number of operations per element by using a different tree structure, trading off some simplicity for better performance on large datasets. Choose when memory bandwidth is a bottleneck. See [Work-Efficient Scan](10349-work-efficient-scan.md).

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** $O(N)$ for the output array. The algorithm is in-place, requiring no additional global memory beyond the input/output array [S1].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Single-phase structure makes functional correctness verification easier compared to two-phase algorithms like Blelloch’s [S1].
- **Limitations:** Requires explicit synchronization barriers, which can introduce latency [S1].
- **When to use:** Not given in the sources.
- **When not to use:** Not given in the sources.

## Background
The algorithm is analyzed in the context of formal verification of parallel prefix sums and stream processing [S1]. Its single-phase nature distinguishes it from later algorithms like Blelloch’s [S1]. The algorithm is frequently used in formal verification studies due to its straightforward structure [S1].

## Sources
[S1] Formal verification of parallel prefix sum and stream ..., https://research.utwente.nl/files/277832002/Safari_2022_Formal_verification_of_parallel_pre.pdf, fetched 2026-10-29
