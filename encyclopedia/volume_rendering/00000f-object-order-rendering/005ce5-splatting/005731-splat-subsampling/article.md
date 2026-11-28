# Splat Subsampling

Also called: Strided Splat Subsampling

## Overview
Splat Subsampling reduces the number of Gaussian splats to fit within a fixed training or rendering budget by selecting a regular subset of the primitive set [S1, unofficial]. It differs from hierarchical methods like [Hierarchical and Multi-Resolution Splatting](5618-hierarchical-and-multi-resolution-splatting.md) by using a deterministic, uniform stride rather than error-driven or distance-driven adaptive traversal [S1, unofficial]. This approach provides a predictable computational cost but lacks the spatial adaptivity of sibling methods like [Multi Resolution Alpha Blending](12118-multi-resolution-alpha-blending.md).

## How it works
**Inputs:**
- $V$: The full set of $N$ splats (Gaussian primitives).
- $M$: The target maximum number of splats to retain (`max_splats`) [S1, unofficial].

**Outputs:**
- $V' \subseteq V$: A subsampled set of splats with $|V'| \le M$ [S1, unofficial].

**Data Structures:**
- The splats are stored in a contiguous array or list, indexed from $0$ to $N-1$ [S1, unofficial]. No hierarchical tree structure is built for this specific subsampling step [S1, unofficial].

**Steps:**
1. Calculate the stride $S$ required to select approximately $M$ elements from $N$:
   $$ S = \lceil N / M \rceil $$
   where $\lceil \cdot \rceil$ denotes the ceiling function [S1, unofficial].
2. Iterate through the array $V$ with a step size of $S$, starting from index $0$.
3. Select elements at indices $0, S, 2S, \dots$ until the end of the array is reached [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def splat_subsampling(splats: list, max_splats: int) -> list:
    """
    Subsample a list of splats using a strided approach.
    
    Args:
        splats: List of all Gaussian splats.
        max_splats: Maximum number of splats to keep.
        
    Returns:
        A new list containing the subsampled splats.
    """
    n = len(splats)
    if max_splats <= 0:
        return []
    
    # Calculate stride: ceiling of n / max_splats
    # Equivalent to (n + max_splats - 1) // max_splats for integers
    stride = (n + max_splats - 1) // max_splats
    
    # Select elements using stride
    # This is equivalent to splats[0::stride] in Python slicing
    subsampled = []
    for i in range(0, n, stride):
        subsampled.append(splats[i])
        
    return subsampled
```

**Parameters:**
- `max_splats` ($M$): The target number of splats. Typical values depend on the hardware memory budget and desired frame rate [S1, unofficial].

**Edge Cases and Pitfalls:**
- If $M \ge N$, the original set is returned (stride becomes 1) [S1, unofficial].
- The selection is deterministic and uniform. It does not account for spatial distribution, potentially removing dense clusters or retaining sparse regions arbitrarily [S1, unofficial].
- Precision issues are minimal as the operation relies on integer arithmetic for indexing [S1, unofficial].

## Subtopics

### [Stochastic Splat Dropout](19695-stochastic-splat-dropout.md)
A regularization technique that randomly drops splats during training loss computation to prevent over-committing to specific views, unlike the deterministic strided selection of Splat Subsampling.

## Operating characteristics
- **Time Complexity:** $O(N)$ to calculate the stride and iterate through the list, where $N$ is the total number of splats [S1, unofficial].
- **Memory Complexity:** $O(M)$ to store the resulting subsampled list [S1, unofficial].
- **Performance:** Provides a fast, predictable reduction in computational load for training or rendering [S1, unofficial].
- **Strengths:** Simple to implement; deterministic; easy to parallelize [S1, unofficial].
- **Limitations:** Lacks spatial adaptivity; may remove important geometric details if they fall outside the stride indices [S1, unofficial].
- **When to use:** When a strict, uniform reduction in primitive count is required for budget constraints, and spatial adaptivity is not critical [S1, unofficial].
- **When not to use:** When preserving visual fidelity in complex or dense regions is paramount; hierarchical methods are preferred [S1, unofficial].

## Background
Splat Subsampling is described in the context of the "Brush" Gaussian splatting engine, which aims for compatibility and performance on various platforms including Android [S1, unofficial]. It is presented as a method to reduce the number of splats to fit within a training budget [S1, unofficial]. The technique is attributed to the authors of the Brush project [S1, unofficial].

## Sources
[S1] Brush – A new compatible Gaussian splatting engine, https://github.com/ArthurBrussee/brush, fetched 2026-10-26 (unofficial)
