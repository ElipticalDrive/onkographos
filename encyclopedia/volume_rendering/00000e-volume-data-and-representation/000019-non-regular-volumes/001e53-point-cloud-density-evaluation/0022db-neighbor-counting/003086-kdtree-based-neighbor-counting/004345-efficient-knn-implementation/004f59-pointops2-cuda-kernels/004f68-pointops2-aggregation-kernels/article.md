# PointOps2 Aggregation Kernels
## Overview
The PointOps2 Aggregation Kernels perform a weighted feature aggregation over a set of neighbors for each point in a point cloud, incorporating relative position offsets [S1, unofficial]. It computes a weighted sum of `(input_feature + relative_position)` for each neighbor [S1, unofficial]. It differs from sibling methods like [Point Grouping](13932-point-grouping.md) by explicitly handling the gradient flow for relative position inputs and performing the weighted sum in a single CUDA kernel pass [S1, unofficial].
## How it works
The method consists of two CUDA kernels: a forward pass that computes the aggregated features and a backward pass that computes gradients for the input features, relative positions, and aggregation weights [S1, unofficial].
### Inputs and Outputs
**Forward Pass:**
*   **Input:** `input` tensor of shape $(n, c)$, containing features for $n$ points with feature dimension $c$ [S1, unofficial].
*   **Input:** `position` tensor of shape $(n, \text{nsample}, c)$, containing relative position offsets for each of the $\text{nsample}$ neighbors [S1, unofficial].
*   **Input:** `weight` tensor of shape $(n, \text{nsample}, w\_c)$, containing learned or computed weights [S1, unofficial].
*   **Input:** `idx` tensor of shape $(n, \text{nsample})$, containing the indices of the neighbors in the `input` tensor [S1, unofficial].
*   **Output:** `output` tensor of shape $(n, c)$, containing the aggregated features [S1, unofficial].
**Backward Pass:**
*   **Input:** `grad_output` tensor of shape $(n, c)$, containing gradients with respect to the output [S1, unofficial].
*   **Input:** `input`, `position`, `weight`, `idx` (same as forward) [S1, unofficial].
*   **Output:** `grad_input` tensor of shape $(n, c)$ [S1, unofficial].
*   **Output:** `grad_position` tensor of shape $(n, \text{nsample}, c)$ [S1, unofficial].
*   **Output:** `grad_weight` tensor of shape $(n, \text{nsample}, w\_c)$ [S1, unofficial].
### Data Structures and Indices
The kernels operate on flat 1D arrays representing the tensors [S1, unofficial]. The indexing logic maps global thread IDs to specific elements in these tensors [S1, unofficial].
*   $n$: Number of points [S1, unofficial].
*   $\text{nsample}$: Number of neighbors per point [S1, unofficial].
*   $c$: Feature dimension of input [S1, unofficial].
*   $w\_c$: Feature dimension of weight (often equal to $c$ or $1$) [S1, unofficial].
For a global thread index `index`:
*   $c\_idx = \text{index} \pmod c$ (feature index within the output vector) [S1, unofficial].
*   $n\_idx = \lfloor \text{index} / c \rfloor$ (point index) [S1, unofficial].
For neighbor iteration $k$ from $0$ to $\text{nsample}-1$:
*   $idx\_idx = n\_idx \times \text{nsample} + k$ (index into `idx` tensor) [S1, unofficial].
*   $input\_idx = \text{idx}[idx\_idx] \times c + c\_idx$ (index into `input` tensor) [S1, unofficial].
*   $position\_idx = n\_idx \times \text{nsample} \times c + k \times c + c\_idx$ (index into `position` tensor) [S1, unofficial].
*   $weight\_idx = n\_idx \times \text{nsample} \times w\_c + k \times w\_c + (c\_idx \pmod {w\_c})$ (index into `weight` tensor) [S1, unofficial].
### Forward Algorithm
The forward kernel launches $n \times c$ threads [S1, unofficial]. Each thread computes one element of the output feature vector [S1, unofficial].
1.  Initialize `output[index]` to $0.0$ [S1, unofficial].
2.  Loop $k$ from $0$ to $\text{nsample}-1$:
    a. Calculate $input\_idx$, $position\_idx$, and $weight\_idx$ as defined above [S1, unofficial].
    b. Compute the term: $val = (\text{input}[input\_idx] + \text{position}[position\_idx]) \times \text{weight}[weight\_idx]$ [S1, unofficial].
    c. Accumulate: $\text{output}[index] \leftarrow \text{output}[index] + val$ [S1, unofficial].
### Backward Algorithm
The backward kernel also launches $n \times c$ threads [S1, unofficial]. It computes gradients for `input`, `position`, and `weight` [S1, unofficial]. Note that `grad_input` requires atomic addition because multiple neighbors may map to the same input point index [S1, unofficial].
1.  Loop $k$ from $0$ to $\text{nsample}-1$:
    a. Calculate $input\_idx$, $position\_idx$, and $weight\_idx$ as defined above [S1, unofficial].
    b. Compute the gradient value flowing from the output: $grad\_val = \text{grad\_output}[index] \times \text{weight}[weight\_idx]$ [S1, unofficial].
    c. Update gradient for input features: $\text{grad\_input}[input\_idx] \leftarrow \text{grad\_input}[input\_idx] + grad\_val$ (Use `atomicAdd` for thread safety) [S1, unofficial].
    d. Update gradient for positions: $\text{grad\_position}[position\_idx] \leftarrow grad\_val$ [S1, unofficial].
    e. Update gradient for weights: $\text{grad\_weight}[weight\_idx] \leftarrow \text{grad\_weight}[weight\_idx] + grad\_val \times (\text{input}[input\_idx] + \text{position}[position\_idx])$ (Use `atomicAdd` for thread safety) [S1, unofficial].
### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def aggregation_forward_cuda(input, position, weight, idx):
    n, c = input.shape
    nsample = position.shape[1]
    w_c = weight.shape[2]
    output = torch.zeros((n, c), device=input.device, dtype=input.dtype)
    
    # Simulates CUDA kernel launch with n * c threads
    for index in range(n * c):
        c_idx = index % c
        n_idx = index // c
        acc = 0.0
        for nsample_idx in range(nsample):
            idx_idx = n_idx * nsample + nsample_idx
            input_idx = idx[idx_idx].item() * c + c_idx
            position_idx = n_idx * nsample * c + nsample_idx * c + c_idx
            weight_idx = n_idx * nsample * w_c + nsample_idx * w_c + (c_idx % w_c)
            acc += (input[input_idx] + position[position_idx]) * weight[weight_idx]
        output[index] = acc
    return output

def aggregation_backward_cuda(grad_output, input, position, weight, idx):
    n, c = input.shape
    nsample = position.shape[1]
    w_c = weight.shape[2]
    grad_input = torch.zeros((n, c), device=input.device, dtype=input.dtype)
    grad_position = torch.zeros((n, nsample, c), device=input.device, dtype=input.dtype)
    grad_weight = torch.zeros((n, nsample, w_c), device=input.device, dtype=input.dtype)
    
    # Simulates CUDA kernel launch with n * c threads
    for index in range(n * c):
        c_idx = index % c
        n_idx = index // c
        for nsample_idx in range(nsample):
            idx_idx = n_idx * nsample + nsample_idx
            input_idx = idx[idx_idx].item() * c + c_idx
            position_idx = n_idx * nsample * c + nsample_idx * c + c_idx
            weight_idx = n_idx * nsample * w_c + nsample_idx * w_c + (c_idx % w_c)
            
            grad_val = grad_output[index] * weight[weight_idx]
            # In CUDA: atomicAdd(&grad_input[input_idx], grad_val)
            grad_input[input_idx] += grad_val
            grad_position[position_idx] = grad_val
            # In CUDA: atomicAdd(&grad_weight[weight_idx], grad_val * (input[input_idx] + position[position_idx]))
            grad_weight[weight_idx] += grad_val * (input[input_idx] + position[position_idx])
    return grad_input, grad_position, grad_weight
```
### Parameters
*   `n`: Number of points [S1, unofficial].
*   `nsample`: Number of neighbors per point [S1, unofficial].
*   `c`: Feature dimension of input [S1, unofficial].
*   `w_c`: Feature dimension of weight (often equal to $c$ or $1$) [S1, unofficial].
### Edge Cases and Pitfalls
Not given in the sources.
## Operating characteristics
Not given in the sources.
## Background
The PointOps2 Aggregation Kernels are part of the PointOps2 library, which provides efficient CUDA kernels for point cloud operations [S1, unofficial]. The library is used in various deep learning models for point cloud analysis, such as [4D Gaussian Splatting](https://github.com/LogosIntelligenceLab/4d-gaussian-splatting) [S1, unofficial]. The specific implementation of the aggregation kernel is designed to handle the gradients of relative positions, which is a common requirement in point-based neural networks [S1, unofficial].
## Sources
[S1] LogosIntelligenceLab/4d-gaussian-splatting, https://github.com/LogosIntelligenceLab/4d-gaussian-splatting, fetched 2026-10-27 (unofficial)
