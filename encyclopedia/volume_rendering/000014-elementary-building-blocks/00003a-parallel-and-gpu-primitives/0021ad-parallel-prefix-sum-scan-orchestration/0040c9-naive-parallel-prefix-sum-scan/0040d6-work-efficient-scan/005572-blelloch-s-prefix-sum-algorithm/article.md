# Blelloch’s Prefix Sum Algorithm

Also called: Blelch's algorithm, Blelloch Scan, Blelloch's parallel prefix sum, Up-sweep/Down-sweep Scan, up-sweep/down-sweep prefix sum

## Overview
Blelloch’s algorithm is an exclusive parallel in-place algorithm for computing the prefix sum of an array of size $N$ [S1]. It solves the problem by decomposing the computation into two distinct phases: an up-sweep (reduce) phase that builds a binary tree of partial sums, and a down-sweep (scan) phase that distributes these sums to compute the final exclusive prefix values [S1]. Unlike Kogge-Stone, which is an inclusive algorithm, Blelloch’s method explicitly excludes the current element from its own sum, making it suitable for operations where the current value must not influence its own prefix result [S1]. It is a work-efficient alternative to naive parallel scan, building on the principles of [Work-Efficient Scan](10349-work-efficient-scan.md).

## How it works
**Inputs and Outputs:**
*   **Input:** An array `Input` of size $N$ (assumed to be a power of 2 for the standard implementation described in [S1]).
*   **Output:** An array `Output` of size $N$ such that $Output[i] = \sum_{t=0}^{i-1} Input[t]$ for $0 \le i < N$ [S1].
*   **Constraint:** The algorithm is in-place; `Input` and `Output` initially share the same values, and `Output` is overwritten during execution [S1].

**Data Structures:**
The algorithm operates on a single array `Output` of size $N$. Threads are indexed by `tid` (thread ID). The implementation assumes a block size of $N/2$ threads, where each thread handles two elements in the initial step, or generally maps to the indices defined by the `indicator` variable [S1].

**Parameters:**
*   `stride`: The distance between elements being summed. Starts at 1 in up-sweep, $N/2$ in down-sweep.
*   `indicator`: The index of the element currently being processed by the thread.

**Steps:**

1.  **Initialization:**
    *   Let `tid` be the thread ID.
    *   Initialize `stride = 1`.
    *   Initialize `indicator = 2 * tid + 1`.

2.  **Phase 1: Up-sweep (Reduce):**
    *   Loop while `stride < N`:
        *   If `indicator < N` and `indicator >= stride`:
            *   Update `Output[indicator] = Output[indicator] + Output[indicator - stride]`.
        *   Synchronize all threads (`__syncthreads()`).
        *   Update `indicator = 2 * indicator + 1`.
        *   Update `stride = 2 * stride`.
    *   Synchronize all threads (`__syncthreads()`).
    *   *Result:* The last element `Output[N-1]` holds the total sum of all input values [S1].

3.  **Phase 2: Down-sweep (Scan):**
    *   Initialize `indicator = N * tid + N - 1`.
    *   Initialize `stride = N / 2`.
    *   If `indicator < N`:
        *   Set `Output[indicator] = 0`.
    *   Loop while `stride >= 1`:
        *   If `indicator < N` and `indicator >= stride`:
            *   Store `temporary = Output[indicator]`.
            *   Update `Output[indicator] = Output[indicator] + Output[indicator - stride]`.
            *   Set `Output[indicator - stride] = temporary`.
        *   Synchronize all threads (`__syncthreads()`).
        *   Update `indicator = (indicator - 1) / 2`.
        *   Update `stride = stride / 2`.
    *   *Result:* The array `Output` contains the exclusive prefix sums [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assumes N is a power of 2.
# Thread block size is N/2.
# Input and Output arrays are of size N.
# Input values are copied to Output before kernel launch.

def blelloch_scan_kernel(Output, N):
    tid = get_thread_id()  # threadIdx.x
    indicator = 2 * tid + 1
    stride = 1

    # Phase 1: Up-sweep
    while stride < N:
        if indicator < N and indicator >= stride:
            Output[indicator] = Output[indicator] + Output[indicator - stride]
        sync_threads()  # __syncthreads()
        indicator = 2 * indicator + 1
        stride = 2 * stride
    
    sync_threads()  # Barrier between phases

    # Phase 2: Down-sweep
    indicator = N * tid + N - 1
    stride = N // 2
    
    if indicator < N:
        Output[indicator] = 0
        
    while stride >= 1:
        if indicator < N and indicator >= stride:
            temporary = Output[indicator]
            Output[indicator] = Output[indicator] + Output[indicator - stride]
            Output[indicator - stride] = temporary
        sync_threads()  # __syncthreads()
        indicator = (indicator - 1) // 2
        stride = stride // 2
```

**Edge Cases and Pitfalls:**
*   **Data Race Freedom:** The algorithm requires careful permission management. In each iteration, some threads lose write permissions because their `indicator` exceeds the array bounds $N$. To maintain correctness, thread 0 collects these lost permissions in each iteration and redistributes them at the barrier [S1].
*   **In-place Verification:** Because the algorithm is in-place, elements change in every iteration. Verifying functional correctness requires tracking the history of values using ghost variables, as intermediate steps are non-trivial [S1].
*   **Synchronization:** Barriers are strictly required between iterations in both phases and between the up-sweep and down-sweep phases to ensure that partial sums from previous levels are available for the next level [S1].

## Operating characteristics
*   **Time Complexity:** $O(\log N)$ parallel steps, where each step involves a constant number of operations per active thread.
*   **Memory Complexity:** $O(N)$ for the input/output array. The algorithm is in-place, requiring no additional auxiliary storage proportional to $N$ beyond the input array itself [S1].
*   **Strengths:** It is work-efficient (linear total work $O(N)$) and avoids the high memory bandwidth usage of naive approaches. It is widely used in libraries such as AMD APP SDK and NVIDIA CUDA SDK [S1].
*   **Limitations:** It requires synchronization barriers at every level, which can introduce latency on hardware with poor barrier performance. It assumes $N$ is a power of 2 for the standard index calculations described; non-power-of-2 sizes require padding or modified boundary checks [S1].

## Background
Blelloch’s algorithm was introduced by Guy E. Blelloch [S1]. It is frequently cited alongside Kogge-Stone as one of the two primary parallel in-place prefix sum algorithms used in practice [S1]. The algorithm is implemented in major parallel computing SDKs, including the AMD Accelerated Parallel Processing (APP) SDK and the NVIDIA CUDA SDK [S1]. Formal verification of its correctness, particularly regarding data race freedom and in-place state management, has been studied in recent literature [S1].

## Sources
[S1] Formal verification of parallel prefix sum and stream ..., https://research.utwente.nl/files/277832002/Safari_2022_Formal_verification_of_parallel_pre.pdf, fetched 2026-10-29
