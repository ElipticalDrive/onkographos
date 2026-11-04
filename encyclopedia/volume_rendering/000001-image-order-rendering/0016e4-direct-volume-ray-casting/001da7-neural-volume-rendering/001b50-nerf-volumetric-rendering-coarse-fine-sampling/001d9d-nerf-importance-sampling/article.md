# NeRF Importance Sampling

Also called: Importance Sampling for NeRF

## Overview
A technique to refine the sampling distribution along rays by using the weights from a coarse model to generate additional sample points, improving rendering quality with minimal overhead [S1, unofficial]. It concentrates computational effort on regions of high opacity by resampling based on coarse alpha compositing weights [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Coarse sample positions $z_{\text{vals}}$ (depths along the ray), corresponding alpha compositing weights $w$ from the coarse network evaluation, number of importance samples $N_{\text{importance}}$, and a boolean `perturb` flag [S1, unofficial].
- Output: Refined sample positions $z_{\text{samples}}$ concatenated with coarse positions, sorted, and their corresponding 3D coordinates for fine network evaluation [S1, unofficial].

**Data Structures:**
- $z_{\text{vals}}$: Tensor of shape $[N_{\text{coarse}}]$, sorted in descending order (far to near) [S1, unofficial].
- $w$: Tensor of shape $[N_{\text{coarse}}]$, representing the contribution weight of each coarse sample [S1, unofficial].
- $z_{\text{vals\_mid}}$: Tensor of shape $[N_{\text{coarse}} - 1]$, midpoints between adjacent coarse samples [S1, unofficial].

**Steps:**
1. **Calculate Midpoints:** Compute the midpoints between adjacent coarse samples to define integration intervals [S1, unofficial].
   $$ z_{\text{vals\_mid}} = 0.5 \times (z_{\text{vals}}[..., 1:] + z_{\text{vals}}[..., :-1]) $$
2. **Prepare Weights for Sampling:** Extract the weights corresponding to the intervals defined by the midpoints, excluding the first and last samples [S1, unofficial].
3. **Sample PDF:** Use inverse transform sampling on the normalized weights to select $N_{\text{importance}}$ sample positions $z_{\text{samples}}$ within the intervals, using a deterministic flag based on `perturb` [S1, unofficial].
4. **Stop Gradient:** Apply a stop-gradient operation on the sampled positions $z_{\text{samples}}$ to prevent gradients from flowing back through the sampling process to the coarse network during backpropagation [S1, unofficial].
5. **Concatenate and Sort:** Concatenate the original coarse $z_{\text{vals}}$ and the new $z_{\text{samples}}$. Sort the combined tensor in descending order to maintain the ray traversal order (far to near) [S1, unofficial].
6. **Evaluate Fine Network:** Compute 3D coordinates from the sorted depths and evaluate the fine neural network at these points to obtain refined color and opacity [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def importance_sampling(z_vals, weights, N_importance, perturb):
    # z_vals: [N_coarse] sorted descending
    # weights: [N_coarse] alpha compositing weights
    # N_importance: int
    # perturb: bool
    # 1. Calculate midpoints
    z_vals_mid = 0.5 * (z_vals[..., 1:] + z_vals[..., :-1])
    
    # 2. Prepare weights for sampling (exclude first and last)
    weights_mid = weights[..., 1:-1]
    
    # Normalize weights to form a PDF
    eps = 1e-10
    pdf = weights_mid / (weights_mid.sum(dim=-1, keepdim=True) + eps)
    
    # 3. Sample PDF using inverse transform sampling
    cdf = torch.cumsum(pdf, dim=-1)
    cdf = torch.cat([torch.zeros_like(cdf[..., :1]), cdf], dim=-1)
    
    if perturb:
        u = torch.rand(z_vals.shape[:-1] + (N_importance,), device=z_vals.device)
    else:
        u = torch.linspace(0, 1, N_importance, device=z_vals.device)
        u = u.expand_as(cdf[..., :-1])
        
    # Inverse transform sampling
    indices = torch.searchsorted(cdf[..., :-1], u, side='right')
    indices = torch.clamp(indices, 0, len(z_vals_mid) - 1)
    
    # Interpolate within bins
    lower_idx = indices
    upper_idx = torch.clamp(indices + 1, 0, len(z_vals_mid) - 1)
    
    z_lower = z_vals_mid[lower_idx]
    z_upper = z_vals_mid[upper_idx]
    
    # Linear interpolation within the bin
    z_samples = z_lower + (z_upper - z_lower) * u
    
    # 4. Stop gradient
    z_samples = z_samples.detach()
    
    # 5. Concatenate and sort
    z_vals_all = torch.cat([z_vals, z_samples], dim=-1)
    z_vals_all, _ = torch.sort(z_vals_all, dim=-1, descending=True)
    
    return z_vals_all
```

**Parameters:**
- $N_{\text{importance}}$: Number of fine samples. Typical value: 128 [S2].
- `perturb`: Boolean flag for training-time jittering [S1, unofficial].

**Edge Cases:**
- **Empty Regions:** If weights are zero everywhere, sampling fails. Mitigated by weight modification (see Subtopics) [S2].
- **Precision:** Floating point errors in CDF normalization can cause sampling bias. Use epsilon [G].
- **Boundaries:** Ensure indices are clamped to valid ranges when using `searchsorted` [G].

## Subtopics

### Hierarchical Sampling with Weight Modification
A variant used in mip-NeRF to improve sampling in empty regions [S2]. It modifies the coarse weights before sampling to create a smoother, wider envelope [S2].
- **Method:** Before sampling fine samples $t^f$, the weights $w$ are modified using a 2-tap max filter followed by a 2-tap blur filter (blurpool), then a hyperparameter $\alpha$ is added, and the result is re-normalized to sum to 1 [S2].
- **Formula:** The modified weight $w'_k$ is calculated as:
  $$ w'_k = \frac{1}{2}(\max(w_{k-1}, w_k) + \max(w_k, w_{k+1})) + \alpha $$
- **Parameters:**
  - Number of samples: 128 coarse and 128 fine [S2].
  - $\alpha$: Hyperparameter to ensure sampling in empty regions. Default value: 0.01 [S2].
- **When to choose:** Use when rendering scenes with large empty spaces or when standard importance sampling fails to capture details in low-density regions [S2].

## Operating characteristics
- **Time Complexity:** $O(N_{\text{coarse}} + N_{\text{importance}})$ for sampling and sorting [G]. The dominant cost is the fine network evaluation at $N_{\text{coarse}} + N_{\text{importance}}$ points [G].
- **Memory Complexity:** $O(N_{\text{coarse}} + N_{\text{importance}})$ to store sample positions and intermediate weights [G].
- **Measured Performance:** In mip-NeRF, using 128 coarse and 128 fine samples yields the same number of total MLP evaluations as NeRF's configuration for fair comparison [S2].
- **Strengths:** Reduces noise and aliasing by concentrating samples on opaque regions [S1, unofficial].
- **Limitations:** Requires two network evaluations (coarse and fine) or a hierarchical structure. Sensitive to the quality of coarse weights [S1, unofficial].
- **When to use:** High-quality rendering where noise reduction is critical [S1, unofficial].
- **When not:** Real-time applications with strict latency constraints [G].

## Background
Originated from the NeRF paper (Mildenhall et al., 2020) and refined in mip-NeRF (Barron et al., 2021) [S2]. The technique was introduced to address the issue of uniform sampling being inefficient for sparse scenes. It builds on [NeRF Volumetric Rendering (Coarse/Fine Sampling)](979-nerf-volumetric-rendering-coarse-fine-sampling.md).

## Sources
[S1] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
[S2] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
