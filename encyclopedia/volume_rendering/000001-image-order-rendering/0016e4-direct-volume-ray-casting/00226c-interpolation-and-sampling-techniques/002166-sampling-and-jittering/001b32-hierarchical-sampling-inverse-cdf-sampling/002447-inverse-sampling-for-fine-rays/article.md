# Inverse Sampling for Fine Rays
Also called: FineSampler
## Overview
Inverse Sampling for Fine Rays is a hierarchical sampling technique in volume rendering that generates fine-grained sample positions along a ray by leveraging the probability distribution derived from a prior coarse rendering pass [S1, unofficial]. It concentrates samples in regions of high importance rather than using uniform spacing [G]. It differs from its parent, [Hierarchical Sampling (Inverse CDF Sampling)](973-hierarchical-sampling-inverse-cdf-sampling.md), by explicitly mapping discrete coarse weights to continuous fine depths via inverse CDF interpolation [G]. It differs from [Inverse Opacity Calculation](1099-inverse-opacity-calculation.md) by operating on pre-computed coarse weights rather than deriving probabilities from opacity accumulation during the fine pass [G].
## How it works
**Inputs and Outputs:**
- Input: `z_vals` (coarse sample depths, shape `[batch, num_coarse_samples]`), `weights` (coarse sample importance weights, shape `[batch, num_coarse_samples]`), `num_fine_samples` (integer, number of fine samples to generate per ray) [G].
- Output: `z_vals_fine` (fine sample depths, shape `[batch, num_fine_samples]`) [G].

**Data Structures:**
- `z_vals`: 1D array of depths along the ray [G].
- `weights`: 1D array of non-negative importance weights corresponding to `z_vals` [G].

**Steps:**
1. **Normalize Weights:** Compute the probability mass function (PMF) by normalizing the coarse weights.
   $$ p_i = \frac{w_i}{\sum_{j} w_j} $$
   where $w_i$ is the weight at index $i$. If $\sum w_j = 0$, use uniform distribution $p_i = 1/N$ [G].
2. **Compute CDF:** Compute the cumulative distribution function (CDF) from the PMF.
   $$ c_i = \sum_{k=0}^{i} p_k $$
   Ensure $c_{N-1} = 1.0$ to handle floating-point errors [G].
3. **Generate Uniform Random Numbers:** Generate $M$ uniform random numbers $u \sim \mathcal{U}(0, 1)$ for each ray in the batch [G].
4. **Inverse CDF Sampling:** For each random number $u$, find the index $i$ such that $c_{i-1} < u \le c_i$. Interpolate between $z_{i-1}$ and $z_i$ to find the continuous depth.
   $$ z_{fine} = z_{i-1} + \frac{u - c_{i-1}}{c_i - c_{i-1}} (z_i - z_{i-1}) $$
   If $u > c_{N-1}$, clamp to $z_{N-1}$ [G].
5. **Sort and Concatenate:** Sort the generated fine samples. Concatenate with original coarse samples if required by the parent class `RaySampler` logic, then sort the combined set [S1, unofficial].

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
    eps = 1e-10
    pmf = weights / (weights.sum(dim=1, keepdim=True) + eps)
    
    # Step 2: Compute CDF
    cdf = pmf.cumsum(dim=1)
    cdf[:, -1] = 1.0
    
    # Step 3: Generate uniform random numbers
    u = torch.rand(batch_size, num_fine_samples, device=z_vals.device)
    
    # Step 4: Inverse CDF Sampling
    # Find indices where cdf[i] >= u
    indices = torch.searchsorted(cdf, u, right=False)
    indices = torch.clamp(indices, 1, num_coarse - 1)
    
    # Gather bounds for interpolation
    indices_low = indices - 1
    indices_high = indices
    
    z_low = z_vals.gather(1, indices_low)
    z_high = z_vals.gather(1, indices_high)
    
    cdf_low = cdf.gather(1, indices_low)
    cdf_high = cdf.gather(1, indices_high)
    
    # Interpolate
    denom = cdf_high - cdf_low
    denom = torch.where(denom == 0, torch.ones_like(denom), denom)
    
    z_fine = z_low + (u - cdf_low) / denom * (z_high - z_low)
    
    return z_fine
```

**Edge Cases:**
- Zero weights: Handled by adding epsilon or falling back to uniform distribution [G].
- Flat CDF: Interpolation denominator becomes zero; handled by clamping or fallback [G].
- Boundary conditions: Clamping indices to valid range [G].

## Subtopics

### [Importance Sampling (Fine)](12868-importance-sampling-fine.md)
This method generates secondary sample positions along a ray by leveraging the probability distribution derived from a prior coarse rendering pass. It differs from its siblings by explicitly defining the construction of the probability density function (PDF) and cumulative distribution function (CDF) from coarse weights, and the specific noise-injection step required to map discrete indices back to continuous depth space [G]. It is chosen when hierarchical sampling strategies require coarse points to be uniformly sampled and fine points to be sampled based on the coarse probability estimation.

## Operating characteristics
- **Time Complexity:** $O(M)$ for inverse sampling where $M$ is the number of fine samples, or $O(N \log N)$ if sorting concatenated depths is required [G].
- **Memory Complexity:** $O(N + M)$ for storing coarse and fine samples [G].
- **Strengths:** Concentrates computation in high-variance regions [G].
- **Limitations:** Requires a prior coarse pass and depends on the quality of coarse weights [G].
- **When to use:** When coarse samples are available and high-frequency details are expected [G].
- **When not to use:** When memory is constrained or a coarse pass is not feasible [G].

## Background
The `FineSampler` inherits from `RaySampler` and uses coarse samples and weights to perform inverse sampling [S1, unofficial]. The base class `sample` method concatenates `additional_depths` to `z_vals` if provided, sorts them, and computes points via `points = (cam_loc.unsqueeze(1) + z_vals.unsqueeze(2) * ray_dirs.unsqueeze(1))` [S1, unofficial].

## Sources
[S1] albertgassol1/vf-nerf, https://github.com/albertgassol1/vf-nerf, fetched 2026-10-26 (unofficial)
