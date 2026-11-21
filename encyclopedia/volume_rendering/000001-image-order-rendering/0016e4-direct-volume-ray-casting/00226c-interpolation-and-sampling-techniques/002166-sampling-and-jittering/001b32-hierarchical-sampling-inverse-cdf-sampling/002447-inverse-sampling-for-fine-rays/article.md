# Inverse Sampling for Fine Rays
Also called: FineSampler
## Overview
Inverse Sampling for Fine Rays is a hierarchical sampling technique in volume rendering that generates fine-grained sample positions along a ray by leveraging the probability distribution derived from a prior coarse rendering pass. It solves the problem of inefficient uniform sampling in regions of high radiance or opacity by concentrating samples where they contribute most to the final image. It differs from its parent, [Hierarchical Sampling (Inverse CDF Sampling)](973-hierarchical-sampling-inverse-cdf-sampling.md), by explicitly defining the construction of the probability density function (PDF) and cumulative distribution function (CDF) from coarse weights, and the specific noise-injection step required to map discrete indices back to continuous depth space. It differs from [Inverse Opacity Calculation](1099-inverse-opacity-calculation.md) by operating on pre-computed coarse weights rather than deriving probabilities from opacity accumulation during the fine pass.
## How it works
**Inputs and Outputs:**
- Input: `z_vals` (coarse sample depths, shape `[batch, num_coarse_samples]`), `weights` (coarse sample importance weights, shape `[batch, num_coarse_samples]`), `num_fine_samples` (integer, number of fine samples to generate per ray).
- Output: `z_vals_fine` (fine sample depths, shape `[batch, num_fine_samples]`).

**Data Structures:**
- `z_vals`: 1D array of depths along the ray.
- `weights`: 1D array of non-negative importance weights corresponding to `z_vals`.

**Steps:**
1. **Normalize Weights:** Compute the probability mass function (PMF) by normalizing the coarse weights.
   $$ p_i = \frac{w_i}{\sum_{j} w_j} $$
   where $w_i$ is the weight at index $i$. If $\sum w_j = 0$, use uniform distribution $p_i = 1/N$. [G]

2. **Compute CDF:** Compute the cumulative distribution function (CDF) from the PMF.
   $$ c_i = \sum_{k=0}^{i} p_k $$
   Ensure $c_{N-1} = 1.0$ to handle floating-point errors. [G]

3. **Generate Uniform Random Numbers:** Generate $M$ uniform random numbers $u \sim \mathcal{U}(0, 1)$ for each ray in the batch.

4. **Inverse CDF Sampling:** For each random number $u$, find the index $i$ such that $c_{i-1} < u \le c_i$. Interpolate between $z_i$ and $z_{i+1}$ to find the continuous depth.
   $$ z_{fine} = z_i + \frac{u - c_i}{c_{i+1} - c_i} (z_{i+1} - z_i) $$
   If $u > c_{N-1}$, clamp to $z_{N-1}$. [G]

5. **Sort and Concatenate:** Sort the generated fine samples. Concatenate with original coarse samples if required by the parent class `RaySampler` logic, then sort the combined set. [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def inverse_sampling_fine_rays(z_vals, weights, num_fine_samples):
    """
    z_vals: [batch, num_coarse]
    weights: [batch, num_coarse]
    num_fine_samples: int
    """
    batch_size, num_coarse = z_vals.shape
    
    # Step 1: Normalize weights to get PMF
    # Add small epsilon to avoid division by zero
    eps = 1e-10
    pmf = weights / (weights.sum(dim=1, keepdim=True) + eps)
    
    # Step 2: Compute CDF
    cdf = pmf.cumsum(dim=1)
    # Ensure last element is 1.0
    cdf[:, -1] = 1.0
    
    # Step 3: Generate uniform random numbers
    # Shape: [batch, num_fine_samples]
    u = torch.rand(batch_size, num_fine_samples, device=z_vals.device)
    
    # Step 4: Inverse CDF Sampling
    # Find indices where cdf < u
    # We need to find the bin index for each u
    # Using searchsorted to find the right index
    indices = torch.searchsorted(cdf, u, right=True) - 1
    indices = torch.clamp(indices, 0, num_coarse - 2)
    
    # Get lower and upper bounds for interpolation
    z_low = z_vals.gather(1, indices)
    z_high = z_low + 1e-6 # Placeholder for z[i+1], need to gather properly
    
    # Correct gathering for z_high
    indices_high = indices + 1
    z_high = z_vals.gather(1, indices_high)
    
    # Get CDF values for interpolation
    cdf_low = cdf.gather(1, indices)
    cdf_high = cdf.gather(1, indices_high)
    
    # Interpolate
    # Avoid division by zero if cdf_low == cdf_high
    denom = cdf_high - cdf_low
    denom = torch.where(denom == 0, torch.ones_like(denom), denom)
    
    z_fine = z_low + (u - cdf_low) / denom * (z_high - z_low)
    
    return z_fine
```

**Edge Cases:**
- Zero weights: Handled by adding epsilon or falling back to uniform distribution.
- Flat CDF: Interpolation denominator becomes zero; handled by clamping or fallback.
- Boundary conditions: Clamping indices to valid range.

## Subtopics

### [Importance Sampling (Fine)](12868-importance-sampling-fine.md)
This method generates secondary sample positions along a ray by leveraging the probability distribution derived from a prior coarse rendering pass. It solves the problem of inefficient uniform sampling in regions of high radiance or opacity by concentrating samples where they contribute most to the final image. It differs from its parent, [Inverse Sampling for Fine Rays](3235-inverse-sampling-for-fine-rays.md), by explicitly defining the construction of the probability density function (PDF) and cumulative distribution function (CDF) from coarse weights, and the specific noise-injection step required to map discrete indices back to continuous depth space.

## Operating characteristics
- **Time Complexity:** $O(N \log N)$ for sorting if concatenation is used, or $O(M)$ for inverse sampling where $M$ is the number of fine samples. [G]
- **Memory Complexity:** $O(N + M)$ for storing coarse and fine samples. [G]
- **Strengths:** Reduces noise in high-variance regions by concentrating samples where they matter. [S1, unofficial]
- **Limitations:** Requires a prior coarse pass. Performance depends on the quality of coarse weights. [G]
- **When to use:** When coarse samples are available and high-frequency details are expected. [G]
- **When not to use:** When memory is constrained or coarse pass is not feasible. [G]

## Background
The `FineSampler` inherits from `RaySampler` and uses coarse samples and weights to perform inverse sampling. The base class `sample` method concatenates `additional_depths` to `z_vals` if provided, sorts them, and computes points via `points = (cam_loc.unsqueeze(1) + z_vals.unsqueeze(2) * ray_dirs.unsqueeze(1))`. [S1, unofficial]

## Sources
[S1] albertgassol1/vf-nerf, https://github.com/albertgassol1/vf-nerf, fetched 2026-10-26 (unofficial)
