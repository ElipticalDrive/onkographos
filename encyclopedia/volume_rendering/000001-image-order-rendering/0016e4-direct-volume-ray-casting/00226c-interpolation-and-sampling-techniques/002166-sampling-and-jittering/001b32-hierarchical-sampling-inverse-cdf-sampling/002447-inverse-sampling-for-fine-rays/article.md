# Inverse Sampling for Fine Rays
Also called: FineSampler
## Overview
Inverse Sampling for Fine Rays is a hierarchical sampling technique used in volume rendering to generate high-frequency samples along a ray based on the probability distribution derived from coarse rendering weights [S2, unofficial]. It solves the problem of missing fine geometric details when relying solely on coarse sampling grids by concentrating samples in regions of high opacity or gradient [S2, unofficial]. It differs from [Inverse Opacity Calculation](1099-inverse-opacity-calculation.md) by explicitly constructing a discrete probability density function (PDF) from coarse weights rather than inverting the accumulated opacity function, and from [Proposal Network Sampling](1589-proposal-network-sampling.md) by using a deterministic inverse CDF lookup instead of a learned neural network to predict sample locations [S2, unofficial].
## How it works
**Inputs and Outputs:**
- Input: `z_vals` (coarse sample depths, shape `[B, Kc]`), `weights` (coarse rendering weights, shape `[B, Kc]`), `Kf` (total fine samples), `Kfd` (fine samples dedicated to depth/disparity, often equal to `Kf` or a subset), `n_coarse` (number of coarse samples, typically `Kc`).
- Output: `z_fine` (fine sample depths, shape `[B, Kf - Kfd]` or similar depending on implementation).

**Data Structures:**
- `z_vals`: Tensor of depth values along the ray.
- `weights`: Tensor of unnormalized importance weights derived from the coarse pass.
- `cdf`: Cumulative distribution function tensor, shape `[B, Kc + 1]` after prepending zero.

**Steps:**
1. **Regularize Weights:** Add a small epsilon ($\epsilon = 10^{-5}$) to `weights` to prevent division by zero or zero-probability bins [S2, unofficial].
   $$ w'_i = w_i + \epsilon $$
2. **Compute PDF:** Normalize the regularized weights along the ray dimension (last dimension) to obtain the probability density function [S2, unofficial].
   $$ \text{pdf}_i = \frac{w'_i}{\sum_{j=0}^{Kc-1} w'_j} $$
3. **Compute CDF:** Compute the cumulative sum of the PDF [S2, unofficial].
   $$ \text{cdf}_i = \sum_{j=0}^{i} \text{pdf}_j $$
4. **Prepend Zero:** Prepend a zero to the CDF to handle the first interval correctly, resulting in a CDF of length $Kc + 1$ [S2, unofficial].
   $$ \text{cdf}_{\text{prepended}} = [0, \text{cdf}_0, \text{cdf}_1, \dots, \text{cdf}_{Kc-1}] $$
5. **Generate Uniform Samples:** Generate $B \times (K_f - K_{fd})$ uniform random numbers $u \sim \mathcal{U}(0, 1)$ [S2, unofficial].
6. **Inverse Transform (Search):** Use `searchsorted` to find the indices in the CDF corresponding to $u$. This identifies which coarse bin each fine sample falls into [S2, unofficial].
   $$ \text{indices} = \text{searchsorted}(\text{cdf}_{\text{prepended}}, u) $$
7. **Clamp Indices:** Clamp indices to a minimum of 0 to handle edge cases where $u$ might be slightly negative due to precision or boundary conditions [S2, unofficial].
8. **Normalize Indices:** Divide the indices by $n_{\text{coarse}}$ to get normalized positions within the coarse grid [S2, unofficial].
   $$ \text{norm\_idx} = \frac{\text{indices}}{n_{\text{coarse}}} $$
9. **Add Noise:** Add uniform noise to the normalized indices to jitter samples within the bin, preventing aliasing [S2, unofficial].
   $$ \text{noise} \sim \mathcal{U}(0, 1) $$
   $$ \text{z\_steps} = \text{norm\_idx} + \text{noise} $$
10. **Map to Depth/Disparity:** Map `z_steps` back to the actual depth or disparity space using the same transformation logic applied during coarse sampling (e.g., linear mapping from $[0, 1]$ to $[z_{near}, z_{far}]$ or inverse disparity mapping) [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def inverse_sampling_for_fine_rays(z_vals, weights, Kf, Kfd, n_coarse, eps=1e-5):
    """
    z_vals: [B, Kc] coarse depths
    weights: [B, Kc] coarse weights
    Kf: total fine samples
    Kfd: fine samples for depth/disparity (often Kf)
    n_coarse: number of coarse samples (len(z_vals[-1]))
    """
    # 1. Regularize weights
    weights = weights + eps
    
    # 2. Compute PDF
    pdf = weights / weights.sum(dim=-1, keepdim=True)
    
    # 3. Compute CDF
    cdf = pdf.cumsum(dim=-1)
    
    # 4. Prepend zeros
    # cdf shape: [B, Kc] -> [B, Kc+1]
    cdf = F.pad(cdf, (1, 0), value=0.)
    
    # 5. Generate uniform random numbers
    # Number of fine samples to generate
    num_fine_samples = Kf - Kfd
    u = torch.rand(z_vals.shape[0], num_fine_samples, device=z_vals.device)
    
    # 6. Find indices in CDF
    # searchsorted returns indices where u should be inserted to maintain order
    indices = torch.searchsorted(cdf, u)
    
    # 7. Clamp indices
    indices = torch.clamp(indices, min=0)
    
    # 8. Normalize indices
    z_steps = indices.float() / n_coarse
    
    # 9. Add noise
    noise = torch.rand_like(z_steps)
    z_steps = z_steps + noise
    
    # 10. Map to depth space
    # Assuming linear mapping from [0, 1] to [z_vals[..., 0], z_vals[..., -1]]
    z_near = z_vals[..., 0:1]
    z_far = z_vals[..., -1:]
    z_fine = z_near + z_steps * (z_far - z_near)
    
    return z_fine
```

**Parameters:**
- `eps`: $10^{-5}$ [S2, unofficial].
- `Kf`, `Kfd`: Determined by the rendering pipeline configuration.

**Edge Cases:**
- If all weights are zero, the PDF becomes uniform after adding epsilon, leading to uniform sampling [S2, unofficial].
- Precision issues in `searchsorted` near 0 or 1 are handled by clamping and noise addition [S2, unofficial].

## Subtopics

### Importance Sampling (Fine)
This is the specific implementation of Inverse Sampling for Fine Rays as described in the PixelNeRF codebase [S2, unofficial]. It is distinguished by its explicit handling of `Kfd` (fine samples dedicated to depth/disparity) and the specific normalization step dividing by `n_coarse` before adding noise [S2, unofficial]. It is chosen when using the PixelNeRF architecture for few-shot novel view synthesis, particularly when fine-grained depth estimation is required alongside color rendering [S2, unofficial]. It relies on the `searchsorted` operation for efficient inverse CDF lookup [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(K_c + K_f)$ for computing PDF/CDF and $O(K_f)$ for generating samples via `searchsorted` [S2, unofficial].
- Memory: $O(K_c + K_f)$ to store CDF and random samples [S2, unofficial].

**Measured Performance:**
- Performance depends on the GPU implementation of `searchsorted` and tensor operations [S2, unofficial].
- No specific benchmark numbers are provided in the sources [S2, unofficial].

**Scaling:**
- Scales linearly with the number of coarse samples $K_c$ and fine samples $K_f$ [S2, unofficial].

**Strengths and Limitations:**
- Strengths: Efficiently concentrates samples in high-weight regions without requiring a neural network [S2, unofficial].
- Limitations: Relies on the quality of coarse weights; if coarse weights are inaccurate, fine samples will be misplaced [S2, unofficial].

**When to Use:**
- Use when hierarchical sampling is needed and coarse weights are available [S2, unofficial].
- Not suitable when the probability distribution is continuous and not derived from discrete coarse bins.

## Background
Inverse Sampling for Fine Rays is part of the PixelNeRF architecture, introduced by Alex Yu, Vickie Ye, Matthew Tancik, and Angjoo Kanazawa in 2021 [S2, unofficial]. It builds on the concept of importance sampling in volume rendering, adapting it for the hierarchical sampling scheme used in NeRF-like models [S2, unofficial]. The implementation is based on the PixelNeRF codebase [S2, unofficial].

## Sources
[S1] albertgassol1/vf-nerf, https://github.com/albertgassol1/vf-nerf, fetched 2026-10-26 (unofficial)
[S2] sxyu/pixel-nerf, https://github.com/sxyu/pixel-nerf, fetched 2026-10-26 (unofficial)
