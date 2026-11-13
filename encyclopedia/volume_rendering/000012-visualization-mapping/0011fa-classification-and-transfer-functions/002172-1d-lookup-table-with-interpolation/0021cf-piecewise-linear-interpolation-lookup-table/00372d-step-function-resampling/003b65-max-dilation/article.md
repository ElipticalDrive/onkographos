# Max Dilation

Also called: Max-Dilation of Step Functions

## Overview
Max Dilation is a resampling technique that expands the support of a non-negative step function or weight distribution via max-pooling, effectively smoothing or widening the distribution [S1, S2, unofficial]. It sets itself apart from [CDF Integration from Weights](12191-cdf-integration-from-weights.md) by preserving the peak values of the original distribution rather than integrating mass, which makes it suitable for maintaining sharp features while adapting sample density [S2, unofficial].

## How it works
**Inputs:**
- Original interval boundaries $t$ (sorted, shape $[N]$).
- Weights $w$ associated with intervals $(t_i, t_{i+1})$ (shape $[N-1]$).
- Dilation amount $d \ge 0$.
- Domain bounds $[t_{min}, t_{max}]$ (optional, for clipping).

**Outputs:**
- Dilated interval boundaries $t_{dilate}$ (sorted, shape $[M]$).
- Dilated weights $w_{dilate}$ (shape $[M-1]$).

**Data Structures:**
- $t$: 1D tensor of boundaries.
- $w$: 1D tensor of weights, where $w_i$ corresponds to interval $[t_i, t_{i+1}]$.
- $t_{dilate}$: 1D tensor of new boundaries.
- $w_{dilate}$: 1D tensor of new weights.

**Steps:**

1. **Compute Expanded Boundaries:**
   Calculate the expanded interval boundaries by shifting the original boundaries outward by the dilation amount:
   $$ t_0 = t[:-1] - d $$
   $$ t_1 = t[1:] + d $$
   [S1, S2, unofficial]

2. **Sort Combined Endpoints:**
   Concatenate the original boundaries $t$, the left-shifted boundaries $t_0$, and the right-shifted boundaries $t_1$. Sort the combined array to form the new grid $t_{dilate}$:
   $$ t_{combined} = \text{sort}(\text{concat}(t, t_0, t_1)) $$
   $$ t_{dilate} = t_{combined} $$
   [S1, S2, unofficial]

3. **Clip to Domain:**
   Clip the dilated boundaries to the valid domain $[t_{min}, t_{max}]$ to prevent out-of-bounds values:
   $$ t_{dilate} = \text{clip}(t_{dilate}, t_{min}, t_{max}) $$
   [S1, S2, unofficial]

4. **Compute Dilated Weights:**
   For each interval in $t_{dilate}$, determine the weight by finding the maximum weight from the original intervals that cover any point within the dilated interval. Specifically, for each new boundary $t_{dilate, j}$, identify original intervals $[t_i, t_{i+1}]$ such that $t_0[i] \le t_{dilate, j} < t_1[i]$. The weight for the new interval is the maximum of these covering weights.
   
   Implementation detail:
   $$ w_{dilate, j} = \max_{i} \left( w_i \cdot \mathbb{I}(t_0[i] \le t_{dilate, j} < t_1[i]) \right) $$
   where $\mathbb{I}$ is the indicator function. The final weight vector $w_{dilate}$ corresponds to the intervals defined by $t_{dilate}$.
   [S1, S2, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def max_dilate(t: torch.Tensor, w: torch.Tensor, dilation: float, domain: tuple = None) -> tuple:
    """
    Dilates a step function defined by boundaries t and weights w.
    
    Args:
        t: Sorted boundaries of shape (N,)
        w: Weights of shape (N-1,), corresponding to intervals [t[i], t[i+1]]
        dilation: Non-negative dilation amount
        domain: Optional tuple (min, max) for clipping
        
    Returns:
        t_dilate: Sorted dilated boundaries
        w_dilate: Dilated weights
    """
    # Step 1: Compute expanded boundaries
    # t0 corresponds to left expansion of intervals starting at t[:-1]
    # t1 corresponds to right expansion of intervals ending at t[1:]
    t0 = t[:-1] - dilation
    t1 = t[1:] + dilation
    
    # Step 2: Sort combined endpoints
    # Concatenate original, left-expanded, and right-expanded boundaries
    t_combined = torch.cat([t, t0, t1], dim=-1)
    t_dilate, _ = torch.sort(t_combined, dim=-1)
    
    # Step 3: Clip to domain
    if domain is not None:
        t_dilate = torch.clip(t_dilate, min=domain[0], max=domain[1])
        
    # Step 4: Compute dilated weights
    # For each point in t_dilate, find the max weight from original intervals that cover it
    # We create a mask for each original interval covering each dilated point
    # t0[i] <= t_dilate[j] < t1[i]
    
    # Expand dimensions for broadcasting
    # t0, t1: (N-1, 1)
    # t_dilate: (1, M)
    t0_exp = t0.unsqueeze(-1)
    t1_exp = t1.unsqueeze(-1)
    t_dilate_exp = t_dilate.unsqueeze(0)
    
    # Boolean mask: True if dilated point j is covered by original interval i
    # Note: t1[i] is the right edge of interval i. We use strict inequality on the right
    # to handle boundary cases consistently with the definition.
    mask = (t0_exp <= t_dilate_exp) & (t1_exp > t_dilate_exp)
    
    # Replace non-covering weights with -inf so max ignores them
    # w has shape (N-1, 1)
    w_exp = w.unsqueeze(-1)
    masked_w = torch.where(mask, w_exp, torch.tensor(float('-inf'), device=w.device))
    
    # Take max over original intervals
    w_dilate_raw = torch.max(masked_w, dim=0)[0]
    
    # Handle cases where no interval covers a point (should not happen if domain is correct)
    # Replace -inf with 0
    w_dilate = torch.where(torch.isinf(w_dilate_raw), torch.zeros_like(w_dilate_raw), w_dilate_raw)
    
    return t_dilate, w_dilate

def max_dilate_weights(t: torch.Tensor, w: torch.Tensor, dilation: float, 
                       renormalize: bool = False, eps: float = None) -> torch.Tensor:
    """
    Variant: Convert weights to PDF, dilate, convert back to weights.
    
    Args:
        t: Boundaries
        w: Weights
        dilation: Dilation amount
        renormalize: Whether to renormalize weights to sum to 1
        eps: Numerical stability constant
        
    Returns:
        w_dilate: Dilated weights
    """
    if eps is None:
        eps = torch.finfo(w.dtype).eps ** 2
        
    # Convert weights to PDF
    # p[i] = w[i] / sum(w)
    total_w = torch.sum(w)
    p = w / torch.maximum(eps, total_w)
    
    # Dilate the PDF
    t_dilate, p_dilate = max_dilate(t, p, dilation)
    
    # Convert back to weights
    # w_dilate[i] = p_dilate[i] * sum(w)
    # However, the sum of p_dilate may not equal 1 due to dilation effects
    # The source suggests converting back using the original total weight sum
    w_dilate = p_dilate * torch.maximum(eps, total_w)
    
    # Optional renormalization
    if renormalize:
        w_dilate = w_dilate / torch.maximum(eps, torch.sum(w_dilate))
        
    return w_dilate
```

**Parameters:**
- `dilation`: Non-negative float. Controls the width of the expansion. [S1, S2, unofficial]
- `renormalize`: Boolean flag. If True, renormalizes dilated weights to sum to 1 after dilation. Default: False. [S2, unofficial]
- `eps`: Numerical stability constant for renormalization. Default: $\text{torch.finfo}(\text{torch.float16}).\text{eps}^2$. [S2, unofficial]

**Edge Cases and Pitfalls:**
- **Precision:** When using floating-point arithmetic, boundary comparisons ($t_0 \le t_{dilate} < t_1$) may suffer from precision issues. Using a small epsilon for comparisons is recommended [G].
- **Degenerate Input:** If $t$ has fewer than 2 points, the method is undefined. [G]
- **Empty Intervals:** If dilation is 0, the output should match the input. [G]

## Subtopics

### Max Dilation (Morton)
Performs max pooling with Morton coordinates on a 3D density grid. It is used for efficient spatial dilation in voxel-based representations. It sets itself apart from the 1D step function dilation by operating on 3D grids using Morton ordering for cache efficiency. Choose it when working with 3D density grids in neural rendering. [S3, unofficial]
- **Definition:** Dilates a 3D density grid using Morton coordinates. [S3, unofficial]
- **Steps:**
  1. Ensure the grid is a contiguous CUDA tensor. [S3, unofficial]
  2. Calculate $H = \sqrt[3]{H^3}$ (where $H^3$ is the total number of voxels). [S3, unofficial]
  3. Call the backend CUDA kernel `_backend.morton3D_dilation(grid, C, H, grid_dilation)`. [S3, unofficial]
- **Parameters:**
  - `grid`: Density values, float tensor of shape $[C, H \times H \times H]$, assuming $H \% 2 == 0$. [S3, unofficial]
- **Layout:**
  - `grid_dilation`: Float tensor of shape $[C, H \times H \times H]$. [S3, unofficial]

## Operating characteristics
- **Time Complexity:** $O(N \log N)$ due to sorting the combined boundaries, where $N$ is the number of original intervals. The weight computation is $O(N \cdot M)$ where $M$ is the number of dilated intervals, but can be optimized to $O(N + M)$ with two-pointer techniques. [G]
- **Memory Complexity:** $O(N + M)$ for storing the combined boundaries and masks. [G]
- **Strengths:** Simple to implement, differentiable (if using soft max or appropriate smoothing), preserves peak values. [S2, unofficial]
- **Limitations:** Can introduce artifacts if dilation is too large relative to interval size. Not suitable for preserving mass conservation without renormalization. [S2, unofficial]
- **When to use:** When you need to smooth a weight distribution while preserving its peaks, such as in adaptive sampling for volume rendering. [S2, unofficial]
- **When not to use:** When mass conservation is critical and renormalization is not desired, or when sharp boundaries must be preserved exactly. [S2, unofficial]

## Background
Max Dilation was introduced in the context of neural volume rendering to adapt sample density to scene complexity. It is used in methods like InstantSplat and ER-NeRF for efficient resampling of weights. [S1, S3, unofficial]

## Sources
[S1] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S2] AlexSheldrick/density_bounded_ray_weights, https://github.com/AlexSheldrick/density_bounded_ray_weights, fetched 2026-10-27 (unofficial)
[S3] Fictionarry/ER-NeRF, https://github.com/Fictionarry/ER-NeRF, fetched 2026-10-26 (unofficial)
