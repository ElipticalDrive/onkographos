# Flat Rank

Also called: Flat Rank for Compact Child Indexing

## Overview
Flat Rank is a space-efficient approach for answering access and rank queries on bit vectors in constant time [S1]. It counts the number of set bits (1-bits) preceding a given index in a bitmask [S1]. It achieves this with a minimal additional space overhead of 3.51% compared to other constant-time rank approaches [S1].

## How it works
**Inputs and Outputs:**
- Input: A bit vector $W$ of length $n$, and an index $i$ ($0 \le i \le n$).
- Output: The rank value $r = \text{rank}_1(i)$, defined as the number of 1-bits in $W$ at indices $j < i$ [S1].

**Data Structures:**
The structure consists of the original bit vector $W$ and an auxiliary array storing cumulative 1-bit counts up to each block [S1].
- $W$: The bit vector itself.
- $A$: An auxiliary array where $A[k]$ stores the cumulative count of 1-bits in $W$ from index $0$ up to the end of block $k-1$ [G].

**Parameters:**
- Block size $b$: Not given in the sources. [G] Typical values in literature are powers of 2 (e.g., 64 or 128) to align with word sizes.
- Space overhead: 3.51% of the original bit vector size [S1].

**Steps:**
1. Partition the bit vector $W$ into contiguous blocks of size $b$ [S1].
2. Compute the cumulative count of 1-bits up to the end of each block and store them in the auxiliary array [S1].
3. To query $\text{rank}_1(i)$, determine the block index $k = \lfloor i / b \rfloor$ [G].
4. Retrieve the base count from the auxiliary array at index $k$ [S1].
5. Count the number of 1-bits in the partial block from the start of block $k$ up to index $i$ [G].
6. Return the sum of the base count and the partial block count [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def flat_rank(W, A, b, i, n):
    """
    W: bit vector (list of integers 0/1)
    A: auxiliary array of cumulative block counts
    b: block size
    i: query index
    n: length of W
    Returns: number of 1-bits in W[0...i-1]
    """
    if i <= 0:
        return 0
    if i >= n:
        return A[-1]
    k = i // b
    base_count = A[k]
    partial_count = sum(1 for j in range(k * b, i) if W[j])
    return base_count + partial_count
```

**Edge Cases and Pitfalls:**
- If $i=0$, the rank is 0 [G].
- If $i=n$, the rank is the total number of 1-bits in $W$, stored in the last element of the auxiliary array [G].
- Boundary conditions for block alignment are handled by integer division and slice bounds [G].

## Subtopics

### [Sequential Access via Bit Mask Iteration](22343-sequential-access-via-bit-mask-iteration.md)
Sequential Access via Bit Mask Iteration is an access pattern that iterates over active voxels or nodes by scanning the compact `mChildMask` and `mValueMask` bitmasks embedded in each node, outperforming random access for algorithms invariant to voxel sequence.

## Operating characteristics
- **Time Complexity:** $O(1)$ for rank queries [S1]. The block lookup is $O(1)$, and the partial block count is bounded by $b$, which is constant [G].
- **Space Complexity:** $O(n)$ for the bit vector plus $O(n/b)$ for the auxiliary array. The total space overhead is 3.51% [S1].
- **Measured Performance:** The source cites it as the "currently most space-efficient approach maintaining reasonable query performance" [S1].
- **Strengths:** Minimal space overhead, constant-time queries.
- **Limitations:** Requires precomputation of the auxiliary array. Performance depends on block size choice for the partial block scan [G].
- **When to use:** When space is constrained and constant-time rank queries are needed on bit vectors [S1].
- **When not to use:** When dynamic updates to the bit vector are frequent, as the auxiliary array would need updates [G].

## Background
Flat Rank was introduced by Kurth in 2022 [S1]. It is based on the general idea of partitioning bit vectors into blocks and storing cumulative counts [S1]. This technique traces back to constant-time rank queries on bit vectors [Jac89] [S1]. It is used in the context of wavelet trees and compressible index data structures for applications like text indexing and computational geometry [S1].

## Sources
[S1] Random Access Segmentation Volume Compression for Interactive Volume Rendering, https://doi.org/10.1111/cgf.70116, fetched 2026-10-28
