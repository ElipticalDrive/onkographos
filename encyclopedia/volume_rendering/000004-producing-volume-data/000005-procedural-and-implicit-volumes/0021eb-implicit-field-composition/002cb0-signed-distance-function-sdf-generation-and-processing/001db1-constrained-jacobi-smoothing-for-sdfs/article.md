# Constrained Jacobi Smoothing for SDFs

Also called: Constrained Jacobi Smoothing for Signed Distance Functions, constrained smoothing, energy minimization via Jacobi iteration

## Overview
Constrained Jacobi Smoothing converts a binary volume into a smooth signed distance function (SDF) by minimizing an energy functional through Jacobi iteration, subject to constraints that preserve the binary topology [S1, unofficial]. It differs from sibling methods like Gaussian smoothing by guaranteeing the preservation of the exact binary topology (sign of the level set) and producing a valid SDF, whereas Gaussian smoothing may alter topology and does not guarantee an SDF [S1, unofficial].

## How it works
**Inputs and Outputs**
- Input: A binary volume array $V_{\text{binary}}$ of dimension $D \in \{2, 3\}$.
- Output: A signed distance function array $V_{\text{sdf}}$ of the same dimensions and resolution.

**Data Structures**
- **Stencil ($J$)**: A convolution kernel. For 3D, $J$ is a $5 \times 5 \times 5$ `float64` array [S1, unofficial].
  - Center element $(2,2,2)$: $-4$.
  - Neighbors at Chebyshev distance 1 along axes (e.g., $(1,2,2)$): $-4$.
  - Neighbors at Chebyshev distance 1 along two axes (face diagonals, e.g., $(1,1,2)$): $1$.
  - All other elements: $0$.
  - For 2D, a corresponding 2D stencil is used [S1, unofficial].
- **Parameters**:
  - `jacobi_d`: Scaling factor. $1/12$ for 2D, $1/18$ for 3D [S1, unofficial].
  - `jacobi_weight`: Relaxation weight. Default $0.5$ [S1, unofficial].
  - `max_iters`: Maximum iterations. Default $500$ [S1, unofficial].

**Algorithm Steps**

1. **Initial SDF Construction**:
   Convert the binary input $V_{\text{binary}}$ into an initial signed distance field $V_{\text{init}}$ [S1, unofficial].
   - Set positive values to $1.0$ and others to $0.0$.
   - Compute the Euclidean Distance Transform (EDT) for positive regions, subtract $0.5$.
   - Negate the EDT for negative regions, add $0.5$ [S1, unofficial].

2. **Bound Initialization**:
   For each voxel $v$, determine constraints $L[v]$ (lower bound) and $U[v]$ (upper bound) based on $V_{\text{init}}[v]$ [S1, unofficial].
   - If $V_{\text{init}}[v] < 0$: $U[v] = V_{\text{init}}[v]$, $L[v] = -\infty$.
   - If $V_{\text{init}}[v] \ge 0$: $L[v] = V_{\text{init}}[v]$, $U[v] = \infty$.

3. **Bound Clamping**:
   Refine bounds near the zero-level set [S1, unofficial].
   - If $|U[v]| < 1$, set $U[v] = 0$.
   - If $|L[v]| < 1$, set $L[v] = 0$.

4. **Jacobi Iteration**:
   Repeat for $k = 0$ to `max_iters` - 1 [S1, unofficial]:
   - Compute residual $R$ by convolving the current array with the stencil $J$ using nearest-neighbor padding.
   - Compute update $\Delta V = -\text{jacobi\_d} \times R$.
   - Update array: $V_{\text{new}} = \text{jacobi\_weight} \times \Delta V + (1 - \text{jacobi\_weight}) \times V_{\text{curr}}$.
   - Apply constraints: clamp $V_{\text{new}}$ to be at least $L$ and at most $U$.
   - Set $V_{\text{curr}} = V_{\text{new}}$.

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.ndimage import convolve, distance_transform_edt

def constrained_jacobi_smoothing(binary_volume: np.ndarray, max_iters: int = 500) -> np.ndarray:
    if binary_volume.ndim not in (2, 3):
        raise ValueError("Input must be 2D or 3D array")

    # 1. Initial SDF Construction
    # Normalize binary input: positive -> 1.0, others -> 0.0
    norm = np.where(binary_volume > 0, 1.0, 0.0)
    
    # Compute EDT for positive and negative regions
    edt_pos = distance_transform_edt(norm == 1.0)
    edt_neg = distance_transform_edt(norm == 0.0)
    
    # Initialize SDF
    sdf = np.zeros_like(norm, dtype=np.float64)
    sdf[norm == 1.0] = edt_pos[norm == 1.0] - 0.5
    sdf[norm == 0.0] = -(edt_neg[norm == 0.0] - 0.5)
    
    # 2. Bound Initialization
    lower_bound = np.full_like(sdf, -np.inf)
    upper_bound = np.full_like(sdf, np.inf)
    
    neg_mask = sdf < 0
    pos_mask = ~neg_mask
    
    upper_bound[neg_mask] = sdf[neg_mask]
    lower_bound[pos_mask] = sdf[pos_mask]
    
    # 3. Bound Clamping
    upper_close = np.abs(upper_bound) < 1.0
    lower_close = np.abs(lower_bound) < 1.0
    upper_bound[upper_close] = 0.0
    lower_bound[lower_close] = 0.0
    
    # 4. Jacobi Iteration
    jacobi_weight = 0.5
    jacobi_d = 1.0 / 18.0 if binary_volume.ndim == 3 else 1.0 / 12.0
    
    # Construct 3D Stencil
    if binary_volume.ndim == 3:
        stencil = np.zeros((5, 5, 5), dtype=np.float64)
        stencil[2, 2, 2] = -4.0
        # Axis neighbors
        for axis in range(3):
            for offset in [-1, 1]:
                idx = [2, 2, 2]
                idx[axis] += offset
                stencil[tuple(idx)] = -4.0
        # Face diagonals
        for i in range(3):
            for j in range(i + 1, 3):
                for si in [-1, 1]:
                    for sj in [-1, 1]:
                        idx = [2, 2, 2]
                        idx[i] += si
                        idx[j] += sj
                        stencil[tuple(idx)] = 1.0
    else:
        # 2D stencil values not provided in the sources [S1, unofficial]
        # Using a standard 5-point Laplacian scaled by jacobi_d for completeness
        stencil = np.zeros((3, 3), dtype=np.float64)
        stencil[1, 1] = -4.0
        stencil[0, 1] = -1.0
        stencil[2, 1] = -1.0
        stencil[1, 0] = -1.0
        stencil[1, 2] = -1.0

    current_sdf = sdf.copy()
    
    for _ in range(max_iters):
        residual = convolve(current_sdf, stencil, mode='nearest')
        update = -jacobi_d * residual
        new_sdf = jacobi_weight * update + (1.0 - jacobi_weight) * current_sdf
        current_sdf = np.clip(new_sdf, lower_bound, upper_bound)
        
    return current_sdf
```

**Edge Cases and Pitfalls**
- **Dimensionality**: The method requires 2D or 3D input arrays. 1D or 4D+ arrays raise a `ValueError` [S1, unofficial].
- **Precision**: The stencil and intermediate calculations use `float64` to maintain precision during the iterative process [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(I \cdot N)$, where $I$ is the number of iterations (default 500) and $N$ is the number of voxels [G]. Each iteration involves a convolution and element-wise operations [S1, unofficial].
- **Memory Complexity**: $O(N)$ for storing the volume arrays and bounds. The stencil size is constant ($5^3$ for 3D) [G].
- **Strengths**: Guarantees topological preservation of the binary input. Produces a valid SDF [S1, unofficial].
- **Limitations**: Computationally expensive compared to single-pass methods. Requires an initial EDT, which can also be costly. The choice of `max_iters` affects the trade-off between accuracy and speed [S1, unofficial].
- **When to use**: When topological correctness is critical and the input is a binary volume [S1, unofficial].
- **When not to use**: When real-time performance is required or when the input is already a smooth SDF [S1, unofficial].

## Background
The technique is used to preprocess binary volumes for surface extraction, ensuring that the resulting representation preserves the original topology [S1, unofficial]. Implementation details, including the stencil values and iteration logic, are derived from the source code of the `PyMCubes` library [S1, unofficial].

## Sources
[S1] pmneila/PyMCubes, https://github.com/pmneila/PyMCubes, fetched 2026-10-26 (unofficial)
