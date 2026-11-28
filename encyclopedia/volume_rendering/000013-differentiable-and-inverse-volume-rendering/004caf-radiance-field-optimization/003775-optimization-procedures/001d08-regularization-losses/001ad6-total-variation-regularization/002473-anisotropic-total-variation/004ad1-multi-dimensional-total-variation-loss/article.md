# Multi-dimensional Total Variation Loss


## Overview
Multi-dimensional Total Variation Loss computes the total variation regularization on multi-dimensional tensors by summing absolute differences along all spatial and channel dimensions [S1, unofficial]. It generalizes the difference operation to arbitrary tensor shapes [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A tensor $\mathbf{x}$ of shape $(B, C, \dots)$, where $B$ is batch size, $C$ is channel size, and subsequent dimensions are spatial or feature dimensions [S1, unofficial].
- Output: A scalar loss value representing the sum of absolute differences across the specified dimensions [S1, unofficial].

**Data Structures:**
- The input $\mathbf{x}$ is a dense tensor [S1, unofficial]. No auxiliary structures are required beyond standard tensor indexing [S1, unofficial].

**Algorithm:**
1. Initialize `tv = 0`.
2. Loop over dimensions $i$ from $2$ to $\text{len}(\mathbf{x}.shape) - 1$.
3. Let $n_{res} = \mathbf{x}.shape[i]$.
4. Create index sets $idx_1 = [1, 2, \dots, n_{res}-1]$ and $idx_2 = [0, 1, \dots, n_{res}-2]$.
5. Select slices $\mathbf{x}_1 = \mathbf{x}.index\_select(i, idx_1)$ and $\mathbf{x}_2 = \mathbf{x}.index\_select(i, idx_2)$.
6. Compute the absolute difference and add its sum to `tv`.
7. Return `tv`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_dim_tv_loss(x):
    """
    Computes the multi-dimensional total variation loss.
    x: Tensor of shape (B, C, D1, D2, ...)
    """
    tv = 0.0
    for i in range(2, len(x.shape)):
        n_res = x.shape[i]
        if n_res < 2:
            continue
        idx1 = list(range(1, n_res))
        idx2 = list(range(0, n_res - 1))
        x1 = index_select(x, i, idx1)
        x2 = index_select(x, i, idx2)
        tv += sum(abs(v1 - v2) for v1, v2 in zip(flatten(x1), flatten(x2)))
    return tv

def index_select(tensor, dim, indices):
    """Selects elements along dimension dim at given indices."""
    # Standard tensor indexing operation
    return tensor[..., indices, ...]

def flatten(tensor):
    """Returns a flat list of elements."""
    return [v for row in tensor for v in row]
```

**Parameters:**
- None explicitly defined in the core algorithm [S1, unofficial].

**Edge Cases and Pitfalls:**
- If any dimension has size 1, the difference along that dimension is zero or undefined. The loop bounds naturally handle this by producing empty index ranges, resulting in a zero contribution [S1, unofficial].
- Precision issues with large tensors are not specified in the sources.

## Subtopics

### Decomposition Loss (Ld)
A variant used in inverse rendering to regularize material and lighting fields separately [S2]. It computes TV on a mask field $\hat{M}$ and a light field $\hat{E}$ using exponential weighting based on image intensity differences to preserve edges [S2].
- **Formula for Mask:** $TV_{mat} = \frac{1}{|\hat{M}|} \sum_{i,j} \Delta \hat{M}_{ij}$, where $\Delta \hat{M}_{ij} = \exp(-|I_{i,j} - I_{i-1,j}|) (\hat{M}_{i,j} - \hat{M}_{i-1,j})^2 + \exp(-|I_{i,j} - I_{i,j-1}|) (\hat{M}_{i,j} - \hat{M}_{i,j-1})^2$ [S2].
- **Formula for Light:** $TV_{light} = \frac{1}{|\hat{E}|} \sum_{i,j} \Delta \hat{E}_{ij}$, where $\Delta \hat{E}_{ij} = (\hat{E}_{i,j} - \hat{E}_{i-1,j})^2 + (\hat{E}_{i,j} - \hat{E}_{i,j-1})^2$ [S2].

### [PlaneTV Regularization](16189-planetv-regularization.md)
PlaneTV Regularization computes the total variation of spatial planes within a feature tensor to encourage smoothness in the learned feature fields. It differs from standard isotropic TV by explicitly separating horizontal and vertical gradient computations across 2D spatial slices, making it suitable for multi-dimensional tensors where specific dimensions represent spatial planes.
- **Formula:** $compute\_plane\_tv(t) = 2 \times \left( \frac{\sum (t_{i+1} - t_i)^2}{count_h} + \frac{\sum (t_{j+1} - t_j)^2}{count_w} \right)$, where $count_h = B \times C \times (H-1) \times W$ and $count_w = B \times C \times H \times (W-1)$.

## Operating characteristics
- Time and memory complexity: Not given in the sources.
- Measured performance: Not given in the sources.
- Strengths and limitations: Not given in the sources.
- When to use and when not to use: Not given in the sources.

## Background
The method is adapted for multi-dimensional tensors in recent implementations [S1, unofficial]. It builds on the anisotropic total variation prior by extending the difference operation to arbitrary tensor shapes [S1, unofficial].

## Sources
[S1] InternRobotics/AnySplat, https://github.com/InternRobotics/AnySplat, fetched 2026-10-27 (unofficial)
[S2] GS-IR: 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.1109/cvpr52733.2024.02045, fetched 2026-10-26
