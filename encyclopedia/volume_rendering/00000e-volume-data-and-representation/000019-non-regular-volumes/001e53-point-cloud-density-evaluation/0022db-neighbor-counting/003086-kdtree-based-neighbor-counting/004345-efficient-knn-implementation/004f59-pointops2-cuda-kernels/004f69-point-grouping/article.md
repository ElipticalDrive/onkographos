# Point Grouping


## Overview
Point Grouping gathers feature vectors for neighboring points based on pre-computed neighbor indices [S1, unofficial]. It is implemented as a CUDA kernel and performs only the gathering step without applying reduction or transformation functions to the features [S1, unofficial]. It is distinct from [PointOps2 Aggregation Kernels](13922-pointops2-aggregation-kernels.md) in that it does not apply any reduction or transformation functions to the gathered features [S1, unofficial].

## How it works
The method consists of two primary operations: a forward pass to gather features and a backward pass to scatter gradients [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** `input` tensor of shape $(n, c)$, where $n$ is the number of points and $c$ is the feature dimension [S1, unofficial]. `idx` tensor of shape $(m, \text{nsample})$, containing the indices of the neighbors for $m$ query points [S1, unofficial].
*   **Output:** `output` tensor of shape $(m, \text{nsample}, c)$ [S1, unofficial].

**Data Structures:**
The `idx` tensor is a 2D integer array where each row $i$ contains $\text{nsample}$ indices pointing into the first dimension of the `input` tensor [S1, unofficial].

**Steps:**
1.  **Forward Pass:** Call the CUDA kernel `grouping_forward_cuda` to gather features [S1, unofficial]. This kernel iterates over each query point $i$ from $0$ to $m-1$ and each neighbor index $j$ from $0$ to $\text{nsample}-1$ [G]. It reads the index $k = \text{idx}[i, j]$ and copies the feature vector $\text{input}[k]$ to $\text{output}[i, j]$ [G].
2.  **Backward Pass:** Call the CUDA kernel `grouping_backward_cuda` to scatter gradients [S1, unofficial]. This kernel iterates over the output tensor [G]. For each element $\text{output}[i, j, d]$, it adds the gradient value to $\text{grad\_input}[\text{idx}[i, j], d]$ [G]. This effectively scatters the gradients back to the original points [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def grouping_forward(input, idx):
    """
    Gathers features for neighbors based on indices.
    input: list of lists of shape (n, c)
    idx: list of lists of shape (m, nsample)
    returns: list of lists of shape (m, nsample, c)
    """
    n, c = len(input), len(input[0])
    m, nsample = len(idx), len(idx[0])
    output = [[[0.0] * c for _ in range(nsample)] for _ in range(m)]
    for i in range(m):
        for j in range(nsample):
            src_idx = idx[i][j]
            for k in range(c):
                output[i][j][k] = input[src_idx][k]
    return output

def grouping_backward(grad_output, idx, n):
    """
    Scatters gradients from grouped features back to original points.
    grad_output: list of lists of shape (m, nsample, c)
    idx: list of lists of shape (m, nsample)
    n: int, number of original points
    returns: grad_input of shape (n, c)
    """
    c = len(grad_output[0][0])
    grad_input = [[0.0] * c for _ in range(n)]
    m, nsample = len(idx), len(idx[0])
    for i in range(m):
        for j in range(nsample):
            src_idx = idx[i][j]
            for k in range(c):
                grad_input[src_idx][k] += grad_output[i][j][k]
    return grad_input
```

**Parameters:**
*   `nsample`: The number of neighbors to group, determined by the second dimension of `idx` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Index Out of Bounds:** If `idx` contains values $\ge n$, the kernel will read invalid memory [S1, unofficial]. The caller must ensure `idx` is valid [G].
*   **Gradient Accumulation:** The backward pass uses atomic addition or simple addition assuming no concurrent writes to the same `grad_input` element from different threads in a way that causes race conditions without atomics [G]. In standard PointOps2 implementations, the kernel is designed such that each `src_idx` is accessed by multiple threads, requiring atomic addition or a reduction step if multiple neighbors map to the same source point [G].

## Subtopics

### Query and Group
A combined operation that finds k-nearest neighbors and groups their features and coordinates, optionally computing relative coordinates [S1, unofficial]. It is chosen when neighbor indices are not pre-computed [G]. It differs from basic Point Grouping by including the KNN search step and coordinate handling [G].
*   **Definition:** Combines `knnquery` and `grouping` [S1, unofficial].
*   **Steps:**
    1.  If `idx` is None, compute it using `knnquery` on `xyz` and `new_xyz` [S1, unofficial].
    2.  Gather source coordinates: `grouped_xyz = xyz[idx.view(-1).long(), :].view(m, nsample, 3)` [S1, unofficial].
    3.  If `relative` is True, subtract query point coordinates: `grouped_xyz -= new_xyz.unsqueeze(1)` [S1, unofficial].
    4.  Gather source features: `grouped_feat = feat[idx.view(-1).long(), :].view(m, nsample, c)` [S1, unofficial].
    5.  Concatenate `grouped_xyz` and `grouped_feat` along the feature dimension if `use_xyz` is True [S1, unofficial].
*   **Parameters:**
    *   `use_xyz`: Boolean flag to include coordinates in the output. Default is True [S1, unofficial].
    *   `relative`: Boolean flag to compute relative coordinates (neighbor - query). Default is True [S1, unofficial].
*   **Layout:**
    *   **Input:** `xyz` $(n, 3)$, `new_xyz` $(m, 3)$, `feat` $(n, c)$, `idx` $(m, \text{nsample})$, `offset` $(b)$, `new_offset` $(b)$ [S1, unofficial].
    *   **Output:** `new_feat` $(m, c+3, \text{nsample})$ if `use_xyz` is True, else $(m, c, \text{nsample})$ [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Point Grouping is part of the PointOps2 CUDA kernels collection [S1, unofficial]. It is used in deep learning models for point cloud analysis [G]. The method is implemented in the `pointops_cuda` module [S1, unofficial].

## Sources
[S1] LogosIntelligenceLab/4d-gaussian-splatting, https://github.com/LogosIntelligenceLab/4d-gaussian-splatting, fetched 2026-10-27 (unofficial)
