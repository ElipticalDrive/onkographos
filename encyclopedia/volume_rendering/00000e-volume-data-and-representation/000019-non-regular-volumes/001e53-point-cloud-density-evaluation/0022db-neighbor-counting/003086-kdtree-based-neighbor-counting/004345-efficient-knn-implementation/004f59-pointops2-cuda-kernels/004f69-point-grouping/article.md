# Point Grouping

Also called: feature grouping, neighbor feature gathering

## Overview
Point Grouping is a CUDA kernel operation that gathers feature vectors for neighboring points based on pre-computed neighbor indices. It solves the problem of efficiently accessing local neighborhood data in point cloud processing pipelines, specifically within the PointOps2 framework. It is distinct from [PointOps2 Aggregation Kernels](13922-pointops2-aggregation-kernels.md) in that it performs only the gathering (scattering indices to a new tensor) without applying any reduction or transformation functions to the gathered features.

## How it works
The method consists of two primary operations: a forward pass to gather features and a backward pass to scatter gradients.

**Inputs and Outputs:**
*   **Input:** `input` tensor of shape $(n, c)$, where $n$ is the number of points and $c$ is the feature dimension. `idx` tensor of shape $(m, \text{nsample})$, containing the indices of the neighbors for $m$ query points.
*   **Output:** `output` tensor of shape $(m, \text{nsample}, c)$.

**Data Structures:**
The `idx` tensor is a 2D integer array. Each row $i$ contains $\text{nsample}$ indices pointing into the first dimension of the `input` tensor.

**Steps:**
1.  **Forward Pass:** Call the CUDA kernel `pointops_cuda.grouping_forward_cuda`. This kernel iterates over each query point $i$ from $0$ to $m-1$ and each neighbor index $j$ from $0$ to $\text{nsample}-1$. It reads the index $k = \text{idx}[i, j]$ and copies the feature vector $\text{input}[k]$ to $\text{output}[i, j]$.
2.  **Backward Pass:** Call the CUDA kernel `pointops_cuda.grouping_backward_cuda`. This kernel iterates over the output tensor. For each element $\text{output}[i, j, d]$, it adds the gradient value to $\text{grad\_input}[\text{idx}[i, j], d]$. This effectively scatters the gradients back to the original points.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def grouping_forward_cuda(input, idx):
    """
    Gathers features for neighbors based on indices.
    input: (n, c)
    idx: (m, nsample)
    returns: (m, nsample, c)
    """
    n, c = input.shape
    m, nsample = idx.shape
    output = torch.zeros(m, nsample, c, device=input.device)
    
    # CUDA kernel launch configuration
    # Threads per block: 256
    # Grid size: ceil(m * nsample / 256)
    
    # Kernel logic (conceptual):
    # global_idx = blockIdx.x * blockDim.x + threadIdx.x
    # if global_idx < m * nsample:
    #     i = global_idx // nsample
    #     j = global_idx % nsample
    #     src_idx = idx[i, j]
    #     for k in range(c):
    #         output[i, j, k] = input[src_idx, k]
        
    return output

def grouping_backward_cuda(grad_output, idx):
    """
    Scatters gradients from grouped features back to original points.
    grad_output: (m, nsample, c)
    idx: (m, nsample)
    returns: grad_input (n, c)
    """
    n, c = idx.shape[0], grad_output.shape[2] # Note: n is derived from input size in actual impl, here assumed known or passed
    # In actual implementation, n is passed as argument or inferred from grad_input size
    grad_input = torch.zeros(n, c, device=grad_output.device)
    
    # CUDA kernel launch configuration
    # Threads per block: 256
    # Grid size: ceil(m * nsample / 256)
    
    # Kernel logic (conceptual):
    # global_idx = blockIdx.x * blockDim.x + threadIdx.x
    # if global_idx < m * nsample:
    #     i = global_idx // nsample
    #     j = global_idx % nsample
    #     src_idx = idx[i, j]
    #     for k in range(c):
    #         grad_input[src_idx, k] += grad_output[i, j, k]
        
    return grad_input
```

**Parameters:**
*   `nsample`: The number of neighbors to group. Determined by the second dimension of `idx`.

**Edge Cases and Pitfalls:**
*   **Index Out of Bounds:** If `idx` contains values $\ge n$, the kernel will read invalid memory. The caller must ensure `idx` is valid.
*   **Gradient Accumulation:** The backward pass uses atomic addition or simple addition assuming no concurrent writes to the same `grad_input` element from different threads in a way that causes race conditions without atomics. In standard PointOps2 implementations, the kernel is designed such that each `src_idx` is accessed by multiple threads, requiring atomic addition or a reduction step if multiple neighbors map to the same source point. However, the provided notes describe a direct scatter. [G] In practice, `grad_input` accumulation often requires atomic operations or a two-pass approach if multiple query points share the same neighbor index, but the basic kernel described here assumes a direct mapping or relies on the specific launch configuration to handle conflicts.

## Subtopics

### Query and Group
A combined operation that finds k-nearest neighbors and groups their features and coordinates, optionally computing relative coordinates. It is chosen when neighbor indices are not pre-computed. It differs from basic Point Grouping by including the KNN search step and coordinate handling.
*   **Definition:** Combines `knnquery` and `grouping`.
*   **Steps:**
    1.  If `idx` is None, compute it using `knnquery` on `xyz` and `new_xyz`.
    2.  Gather source coordinates: `grouped_xyz = xyz[idx.view(-1).long(), :].view(m, nsample, 3)`.
    3.  If `relative` is True, subtract query point coordinates: `grouped_xyz -= new_xyz.unsqueeze(1)`.
    4.  Gather source features: `grouped_feat = feat[idx.view(-1).long(), :].view(m, nsample, c)`.
    5.  Concatenate `grouped_xyz` and `grouped_feat` along the feature dimension if `use_xyz` is True.
*   **Parameters:**
    *   `use_xyz`: Boolean flag to include coordinates in the output. Default is True.
    *   `relative`: Boolean flag to compute relative coordinates (neighbor - query). Default is True.
*   **Layout:**
    *   **Input:** `xyz` $(n, 3)$, `new_xyz` $(m, 3)$, `feat` $(n, c)$, `idx` $(m, \text{nsample})$, `offset` $(b)$, `new_offset` $(b)$.
    *   **Output:** `new_feat` $(m, c+3, \text{nsample})$ if `use_xyz` is True, else $(m, c, \text{nsample})$.

## Operating characteristics
*   **Time Complexity:** $O(m \cdot \text{nsample} \cdot c)$ for the forward pass, where $m$ is the number of query points, $\text{nsample}$ is the number of neighbors, and $c$ is the feature dimension.
*   **Memory Complexity:** $O(m \cdot \text{nsample} \cdot c)$ for the output tensor.
*   **Performance:** Highly optimized via CUDA kernels. Performance scales with the number of threads and memory bandwidth.
*   **Strengths:** Efficiently handles large point clouds by leveraging GPU parallelism.
*   **Limitations:** Requires pre-computed indices. Memory usage grows linearly with the number of neighbors and features.

## Background
Point Grouping is part of the PointOps2 library, which provides efficient CUDA kernels for point cloud operations. It is used in deep learning models for point cloud analysis, such as 4D Gaussian Splatting, where efficient neighborhood feature gathering is crucial. The method is implemented in the `pointops_cuda` module.

## Sources
[S1] LogosIntelligenceLab/4d-gaussian-splatting, https://github.com/LogosIntelligenceLab/4d-gaussian-splatting, fetched 2026-10-27 (unofficial)
