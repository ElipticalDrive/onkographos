# Max Dilation

Also called: Max-Dilation of Step Functions

## Overview
Max Dilation resamples a non-negative step function or weight distribution by expanding its support intervals via max-pooling, effectively smoothing or widening the distribution [S1, S2, unofficial]. It builds on [Step Function Resampling](7924-step-function-resampling.md) by providing a specific weight transformation logic that prioritizes peak preservation over mass integration [S1, S2, unofficial].

## How it works
**Inputs:**
- Original interval boundaries $t$ (sorted, shape $[N]$) [S1, S2, unofficial].
- Weights $w$ associated with intervals $(t_i, t_{i+1})$ (shape $[N-1]$) [S1, S2, unofficial].
- Dilation amount $d \ge 0$ [S1, S2, unofficial].
- Domain bounds $[t_{min}, t_{max}]$ (optional, for clipping) [S1, unofficial].

**Outputs:**
- Dilated interval boundaries $t_{dilate}$ (sorted, shape $[M]$) [S1, S2, unofficial].
- Dilated weights $w_{dilate}$ (shape $[M-1]$) [S1, S2, unofficial].

**Data Structures:**
- $t$: 1D tensor of boundaries.
- $w$: 1D tensor of weights, where $w_i$ corresponds to interval $[t_i, t_{i+1}]$.
- $t_{dilate}$: 1D tensor of new boundaries.
- $w_{dilate}$: 1D tensor of new weights.

**Steps:**

1. **Compute Expanded Boundaries:** Calculate the expanded interval boundaries by shifting the original boundaries outward by the dilation amount: $t_0 = t[:-1] - d$ and $t_1 = t[1:] + d$ [S1, S2, unofficial].
2. **Sort Combined Endpoints:** Concatenate the original boundaries $t$, the left-shifted boundaries $t_0$, and the right-shifted boundaries $t_1$, then sort them along the last axis to form $t_{dilate}$ [S1, S2, unofficial].
3. **Clip to Domain:** Clip the dilated boundaries to the valid domain $[t_{min}, t_{max}]$ to prevent out-of-bounds values [S1, unofficial].
4. **Compute Dilated Weights:** For each new boundary in $t_{dilate}$, find the maximum weight from original intervals that cover it using a boolean mask: $w_{dilate, j} = \max_i (w_i \cdot \mathbb{I}(t_0[i] \le t_{dilate, j} < t_1[i]))$, then take the interval weights as $w_{dilate} = w_{points}[:-1]$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def max_dilate(t: torch.Tensor, w: torch.Tensor, dilation: float, domain: tuple = None) -> tuple:
    """
    Dilates a step function defined by boundaries t and weights w.
    
    Args:
        t: Sorted boundaries of shape (N,) or (B, N)
        w: Weights of shape (N-1,) or (B, N-1), corresponding to intervals [t[i], t[i+1]]
        dilation: Non-negative dilation amount
        domain: Optional tuple (min, max) for clipping
        
    Returns:
        t_dilate: Sorted dilated boundaries
        w_dilate: Dilated weights
    """
    # Step 1: Compute expanded boundaries
    t0 = t[..., :-1] - dilation
    t1 = t[..., 1:] + dilation
    
    # Step 2: Sort combined endpoints
    t_combined = torch.cat([t, t0, t1], dim=-1)
    t_dilate, _ = torch.sort(t_combined, dim=-1)
    
    # Step 3: Clip to domain
    if domain is not None:
        t_dilate = torch.clip(t_dilate, min=domain[0], max=domain[1])
        
    # Step 4: Compute dilated weights
    # Create boolean mask: True if dilated point j is covered by original interval i
    # t0[..., None, :] <= t_dilate[..., None] compares (..., N-1, 1) with (..., 1, M)
    mask = (t0[..., None, :] <= t_dilate[..., None]) & (t1[..., None, :] > t_dilate[..., None])
    
    # Replace non-covering weights with 0 so max ignores them
    masked_w = torch.where(mask, w[..., None, :], torch.zeros_like(w[..., None, :]))
    
    # Take max over original intervals for each dilated point
    w_points = torch.max(masked_w, dim=-2)[0]
    
    # Interval weights correspond to points[..., :-1]
    w_dilate = w_points[..., :-1]
    
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
    total_w = torch.sum(w, dim=-1, keepdim=True)
    p = w / torch.maximum(eps, total_w)
    
    # Dilate the PDF
    t_dilate, p_dilate = max_dilate(t, p, dilation)
    
    # Convert back to weights
    w_dilate = p_dilate * torch.maximum(eps, total_w)
    
    # Optional renormalization
    if renormalize:
        w_dilate = w_dilate / torch.maximum(eps, torch.sum(w_dilate, dim=-1, keepdim=True))
        
    return w_dilate
```

**Parameters:**
- `dilation`: Non-negative float controlling the width of the expansion [S1, S2, unofficial].
- `domain`: Optional tuple `(min, max)` for clipping dilated boundaries [S1, unofficial].
- `renormalize`: Boolean flag to renormalize dilated weights to sum to 1 after dilation [S2, unofficial].
- `eps`: Numerical stability constant for renormalization, default $\text{torch.finfo}(\text{torch.float16}).\text{eps}^2$ [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** When using floating-point arithmetic, boundary comparisons ($t_0 \le t_{dilate} < t_1$) may suffer from precision issues [G].
- **Degenerate Input:** If $t$ has fewer than 2 points, the method is undefined [G].
- **Empty Intervals:** If dilation is 0, the output should match the input [G].

## Subtopics

### Max Dilation (Morton)
Performs max pooling with Morton coordinates on a 3D density grid [S3, unofficial]. It sets itself apart from the 1D step function dilation by operating on 3D grids using Morton ordering for cache efficiency [S3, unofficial]. Choose it when working with 3D density grids in neural rendering [S3, unofficial].
- **Definition:** Dilates a 3D density grid using Morton coordinates [S3, unofficial].
- **Steps:**
  1. Ensure the grid is a contiguous CUDA tensor [S3, unofficial].
  2. Calculate $H = \sqrt[3]{H^3}$ (where $H^3$ is the total number of voxels) [S3, unofficial].
  3. Call the backend CUDA kernel `_backend.morton3D_dilation(grid, C, H, grid_dilation)` [S3, unofficial].
- **Parameters:**
  - `grid`: Density values, float tensor of shape $[C, H \times H \times H]$, assuming $H \% 2 == 0$ [S3, unofficial].
- **Layout:**
  - `grid_dilation`: Float tensor of shape $[C, H \times H \times H]$ [S3, unofficial].

### [Blurpool Resampling](12212-blurpool-resampling.md)
Blurpool Resampling is a differentiable resampling method used in neural volume rendering to generate new sample positions ($t$ values) along rays based on a learned or estimated weight distribution [Brief]. It solves the problem of adapting sample density to scene complexity by treating the sampling process as a differentiable operation on a discrete probability distribution [Brief]. It sets itself apart from [Max Dilation](8984-max-dilation.md) by using average pooling on max-pooled weights to create a smoother, blurrier distribution, whereas Max Dilation uses only max-pooling [Brief]. It builds on [Step Function Resampling](8985-step-function-resampling.md) by providing the specific weight transformation logic required before sampling [Brief].

## Operating characteristics
- **Strengths:** Preserves peak values of the original distribution [S1, S2, unofficial].
- **Limitations:** Can introduce artifacts if dilation is too large relative to interval size [G]. Not suitable for preserving mass conservation without renormalization [S2, unofficial].
- **When to use:** When you need to smooth a weight distribution while preserving its peaks, such as in adaptive sampling for volume rendering [S2, unofficial].
- **When not to use:** When mass conservation is critical and renormalization is not desired, or when sharp boundaries must be preserved exactly [S2, unofficial].

## Background
Max Dilation was introduced in the context of neural volume rendering to adapt sample density to scene complexity [S1, S3, unofficial]. It is used in methods like InstantSplat and ER-NeRF for efficient resampling of weights [S1, S3, unofficial].

## Sources
[S1] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S2] AlexSheldrick/density_bounded_ray_weights, https://github.com/AlexSheldrick/density_bounded_ray_weights, fetched 2026-10-27 (unofficial)
[S3] Fictionarry/ER-NeRF, https://github.com/Fictionarry/ER-NeRF, fetched 2026-10-26 (unofficial)
