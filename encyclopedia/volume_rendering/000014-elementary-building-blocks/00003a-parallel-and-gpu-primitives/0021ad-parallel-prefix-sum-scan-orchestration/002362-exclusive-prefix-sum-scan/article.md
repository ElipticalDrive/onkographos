# Exclusive Prefix Sum (Scan)

Also called: Exclusive Sum, Parallel Prefix Sum Scan for Voxel Compaction, exclusive scan, prefix sum scan

## Overview
The Exclusive Prefix Sum (Scan) is a parallel primitive that maps an input list of values to an output list where each element at index $i$ is the sum of all input elements at indices strictly less than $i$ [S2]. In volume rendering, it is primarily used for voxel compaction: given a list of boolean culling decisions, it computes the sequential indices for kept voxels, allowing culled voxels to be removed in parallel without race conditions [S2]. It differs from inclusive scans by excluding the current element from the sum, and from standard sequential scans by providing a parallel algorithmic structure suitable for GPU execution [S1, S2].

## How it works
**Inputs and Outputs:**
- Input: A list $D$ of $N$ elements (typically integers or booleans representing culling decisions or weights) [S2].
- Output: A list $S$ of $N$ elements, where $S_i$ is the exclusive prefix sum of $D$ [S1].

**Data Structures:**
- $D$: Array of size $N$. Elements $D_j$ are the input values [S2].
- $S$: Array of size $N$. Elements $S_i$ are the output values [S1].
- Temporary storage: Additional arrays of size $N$ are required for intermediate tree levels during the parallel reduction phases [G].

**Formula:**
The value at index $i$ in the output list $S$ is defined as [S1]:
$$S_i = \begin{cases} 0 & \text{if } i = 0 \\ \sum_{j=0}^{i-1} D_j & \text{otherwise} \end{cases}$$

**Algorithm Steps:**
The efficient parallel implementation treats the problem as a series of parallel reductions [S1]. It typically follows a two-phase tree-based approach [G]:
1. **Upsweep (Reduction):** Initialize leaves with input values $D$. Iteratively compute sums of pairs of elements, moving up the tree levels until a single root value representing the total sum is reached [G].
2. **Downsweep (Scan):** Initialize the root with $0$. Iteratively propagate values down the tree, where each node's left child receives the current node's value and the right child receives the current node's value plus the left child's original value [G]. This phase distributes the partial sums back to the leaves [G].
3. **Final Output:** The values at the leaves of the downsweep tree correspond to $S_i$ [G].

**Pseudocode:**
The following pseudocode implements the exclusive scan for an array of size $N$ using a standard parallel reduction approach [G]. It assumes a parallel environment where threads can synchronize, and handles arbitrary $N$ by padding to the next power of two internally [G].
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def exclusive_scan(D, N):
    # Pad to next power of two for standard tree reduction [G]
    M = 1
    while M < N:
        M *= 2
    T = [0] * (2 * M)
    T[M:2*M] = D + [0] * (M - N)
    
    # Upsweep phase: compute inclusive sums up the tree
    for stride in range(1, M):
        for i in range(0, 2 * M, 2 * stride):
            if i + stride < 2 * M:
                T[i + stride] += T[i]
                
    # Downsweep phase: compute exclusive sums down the tree
    T[2 * M - 1] = 0
    for stride in range(M, 0, -1):
        for i in range(0, 2 * M, 2 * stride):
            left = T[i]
            right = T[i + stride]
            T[i + stride] = left + right
            T[i] = 0
    return T[M:2*M][:N]
```

**Parameters:**
- $N$: Size of the input list [S2].
- Reference implementation details for GPU optimization are provided in [Harris et al. 2007] [S2].

**Edge Cases and Pitfalls:**
- **Precision:** Summing many small values can lead to floating-point precision loss [S1].
- **Empty or single-element lists:** The algorithm handles $N=0$ or $N=1$ trivially, returning zeros or the input respectively [G].
- **Synchronization:** Parallel implementations require careful synchronization between threads to ensure correct tree traversal [G].

## Subtopics

### [Log-space Exclusive Cumulative Sum for Transmittance](4819-log-space-exclusive-cumulative-sum-for-transmittance.md)
This method computes the exclusive cumulative sum of transmittance values in logarithmic space to prevent numerical underflow during alpha compositing and to enable efficient parallel scatter-add operations for weight calculation. It differs from standard linear-space exclusive scans by operating on $\log(1-\alpha)$, which preserves precision for small alpha values, and by explicitly handling ray-segment boundaries to reset cumulative state between independent rays. It builds on [Exclusive Prefix Sum (Scan)](3007-exclusive-prefix-sum-scan.md).

## Operating characteristics
- **Time Complexity:** $O(\log N)$ parallel time with $O(N)$ processors [S1, S2].
- **Memory Complexity:** $O(N)$ for input, output, and temporary tree storage [S2].
- **Strengths:** Enables parallel compaction of sparse data structures (e.g., voxels) without race conditions [S2].
- **Limitations:** Requires significant temporary memory for the tree structure. Precision issues can arise with large sums of small floating-point values [S1].

## Background
The theoretical foundation for efficient parallel prefix sums was laid by Blelloch [S2]. In volume rendering, it is used for voxel compaction, as described in [S2] for sparse voxelization and in [S1] for neural geometric level of detail rendering. The method is widely implemented on GPUs, with detailed descriptions available in [Harris et al. 2007] [S2].

## Sources
[S1] Neural Geometric Level of Detail: Real-time Rendering with Implicit 3D Shapes, https://doi.org/10.1109/cvpr46437.2021.01120, fetched 2026-10-26
[S2] Real-time high-resolution sparse voxelization with application to image-based modeling, https://doi.org/10.1145/2492045.2492053, fetched 2026-11-01
