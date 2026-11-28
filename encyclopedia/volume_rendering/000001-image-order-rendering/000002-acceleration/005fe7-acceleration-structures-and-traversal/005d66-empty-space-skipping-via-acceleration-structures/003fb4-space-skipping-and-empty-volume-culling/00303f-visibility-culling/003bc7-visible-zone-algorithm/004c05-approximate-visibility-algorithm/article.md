# Approximate Visibility Algorithm
Also called: Stride-based Visibility Sampling
## Overview
A method to compute approximate visibility for line drawings from volume data by reducing the number of vertices checked for occlusion, trading accuracy for performance [S1]. It sets itself apart from the exact visibility algorithm [Visible Zone Algorithm](9086-visible-zone-algorithm.md) by sacrificing worst-case correctness for a predictable speedup factor, making it suitable for large datasets where interactive frame rates are prioritized over perfect occlusion handling [S1].
## How it works
### Inputs and outputs
- **Input**: A loop of vertices $V = \{v_0, v_1, \dots, v_{N-1}\}$ representing a contour or cutting plane intersection, and a stride parameter $k$ [S1].
- **Output**: A visibility mask $M$ for each vertex in the loop, where $M[i]$ indicates whether vertex $v_i$ is visible or occluded [S1].

### Data structures
- **Visibility Array**: An array of boolean values or flags, size $N$, corresponding to the vertices in the loop [G].
- **Stride Parameter**: Integer $k \ge 1$ [S1].

### Steps
1. Initialize the visibility status for all vertices in the loop as "unknown" [G].
2. Iterate through the loop indices $i$ with a step size of $k$ (i.e., $i = 0, k, 2k, \dots$) [S1].
3. For each sampled vertex $v_i$: perform the exact visibility test to determine if $v_i$ is visible, and store the result in $M[i]$ [S1].
4. For each interval between two consecutive sampled vertices $v_i$ and $v_{i+k}$: if $M[i] \neq M[i+k]$ (a transition is detected), iterate through all intervening vertices $j$ where $i < j < i+k$ and perform the exact visibility test for each $v_j$ to determine the exact transition point [S1]. If $M[i] == M[i+k]$ (no transition detected), assume all intervening vertices $j$ where $i < j < i+k$ have the same visibility as $M[i]$ and set $M[j] = M[i]$ for all such $j$ [S1].
5. Return the visibility mask $M$ [G].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def approximate_visibility(vertices, k, visibility_test_func):
    """
    Compute approximate visibility for a loop of vertices.
    
    Args:
        vertices: List of vertex objects or indices.
        k: Stride length for sampling.
        visibility_test_func: Function that returns True if vertex is visible, False otherwise [G].
        
    Returns:
        List of booleans representing visibility status for each vertex.
    """
    n = len(vertices)
    visibility = [None] * n

    # Step 1: Sample every k-th vertex
    for i in range(0, n, k):
        visibility[i] = visibility_test_func(vertices[i])

    # Step 2: Check intervals between consecutive sampled vertices
    for i in range(0, n - k, k):
        start = i
        end = i + k
        if visibility[start] != visibility[end]:
            for j in range(start + 1, end):
                visibility[j] = visibility_test_func(vertices[j])
        else:
            for j in range(start + 1, end):
                visibility[j] = visibility[start]

    # Handle remaining vertices if n is not a multiple of k (wrap-around for closed loops)
    last_sampled = (n - 1) // k * k
    if last_sampled < n - 1:
        if visibility[last_sampled] != visibility[0]:
            for j in range(last_sampled + 1, n):
                visibility[j] = visibility_test_func(vertices[j])
        else:
            for j in range(last_sampled + 1, n):
                visibility[j] = visibility[last_sampled]

    return visibility
```

### Parameters
- **$k$**: The stride length for sampling vertices. Typical values tested: 1, 2, 4, 8 [S1]. $k=1$ corresponds to the exact visibility algorithm [S1].

### Edge cases and pitfalls
- **Narrow occluders**: If the stride $k$ is too large, narrow occluders may be missed, causing visible artifacts (mislabeled vertices) [S1].
- **Guarantee**: The method guarantees that no details larger than $k$ voxels are missed [S1].
- **Precision**: The method relies on the visibility test function being consistent. Inconsistent tests may lead to false transitions [G].

## Operating characteristics
- **Time Complexity**: Reduces the algorithm's running time by a factor of at most $k$ compared to checking every vertex [S1]. The exact visibility algorithm is $O(n^2)$ for a volume of size $n^3$ [S1]. The approximate method is therefore roughly $O(n^2/k)$ in the best case (no transitions), but can degrade to $O(n^2)$ if every interval requires full checking [S1].
- **Memory Complexity**: Not given in the sources.
- **Measured Performance**: On the aneurysm dataset: Stride 1: 590 msec/frame, Stride 2: 370 msec/frame, Stride 4: 280 msec/frame, Stride 8: 244 msec/frame [S1].
- **Strengths**: Significant speedup for large datasets where many vertices share the same visibility status [S1].
- **Limitations**: May mislabel vertices if narrow occluders are present and $k$ is large [S1]. Not suitable for real-time rendering if high accuracy is required for narrow features [S1].
- **When to use**: Large datasets where interactive frame rates are needed and narrow occluders are not critical [S1].
- **When not to use**: When exact visibility is required, or when the dataset contains many narrow occluders and $k$ must be small [S1].

## Background
- **Origin**: Developed as part of a system for line drawings from volume data [S1].
- **History**: Introduced in the context of line drawings from volume data, specifically for handling visibility of contours and cutting planes [S1].
- **Usage**: Used in the aneurysm dataset rendering to handle narrow occluders, though video sequences were created without it [S1]. It is part of a system that provides user control over line styles based on visibility [S1].

## Sources
[S1] Line drawings from volume data, https://doi.org/10.1145/1073204.1073222, fetched 2026-10-28
