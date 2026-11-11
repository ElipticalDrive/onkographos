# Depth-Guided Supersampling

## Overview
Depth-Guided Supersampling generates sample positions along a ray by perturbing a depth estimate with Gaussian noise, encouraging the renderer to sample around the estimated surface location [S1, unofficial]. It differs from standard linear spacing by focusing computational effort on likely geometry, and from Inverse CDF sampling by relying on an external depth prior rather than a learned density distribution [S1, unofficial].

## How it works
**Inputs:**
- `rays`: Tensor containing ray origins, directions, and near/far bounds.
- `depth`: Tensor of estimated depth values for each ray, shape `[batch_size]` or `[batch_size, 1]`.
- `n_samples`: Integer, number of samples per ray.
- `start_sigma`: Float, initial standard deviation for noise (default 0.1) [S1, unofficial].

**Outputs:**
- `zvals`: Tensor of sample depths along the ray, shape `[batch_size, n_samples]`.

**Data Structures:**
- `zvals`: Float tensor.
- `mean`: Tensor of shape `[batch_size, n_samples]`, initialized with the depth estimate.
- `sigma`: Tensor of shape `[1, n_samples]`, initialized to `start_sigma`.
- `noise`: Tensor of shape `[batch_size, n_samples]`, drawn from $\mathcal{N}(0, 1)$.

**Steps:**
1. Extract `near`, `far`, and `depth` from the ray data.
2. Clone and detach the `depth` tensor to create `mean`. Expand `mean` to shape `[batch_size, n_samples]` by repeating along the sample dimension.
3. Create `sigma` as a tensor of ones with shape `[1, n_samples]`, multiplied by `start_sigma`.
4. Generate `noise` from a standard normal distribution $\mathcal{N}(0, 1)$ with shape `[batch_size, n_samples]` [S1, unofficial].
5. Compute initial sample depths:
   $$ zvals\_0\_level = noise \times sigma + mean $$
6. Identify rays where `depth == 0` (indicating no valid depth estimate). For these rays, replace `zvals_0_level` with random points uniformly distributed between `near` and `far` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def depth_guided_supersampling(rays, depth, n_samples, start_sigma=0.1):
    """
    Generate sample positions along a ray guided by a depth estimate.
    
    Args:
        rays: Tensor of shape [B, 3] or similar, containing near and far bounds.
              Assumes access to rays.near and rays.far.
        depth: Tensor of shape [B] or [B, 1] with depth estimates.
        n_samples: int, number of samples per ray.
        start_sigma: float, standard deviation of noise.
        
    Returns:
        zvals: Tensor of shape [B, n_samples] with sample depths.
    """
    # Extract bounds from rays (implementation detail depends on ray structure)
    # Assuming rays has attributes .near and .far of shape [B]
    near = rays.near
    far = rays.far
    
    # Ensure depth is 1D
    if depth.dim() == 2:
        depth = depth.squeeze(1)
        
    # Step 2: Set mean to depth, expanded to [B, n_samples]
    # Clone and detach to avoid gradient issues if not intended
    mean = depth.clone().detach()
    mean = mean.unsqueeze(1).expand(-1, n_samples)
    
    # Step 3: Set sigma to tensor of ones * start_sigma, shape [1, n_samples]
    sigma = torch.ones(1, n_samples, device=depth.device) * start_sigma
    
    # Step 4: Generate normal noise N(0, 1), shape [B, n_samples]
    noise = torch.randn(mean.shape, device=depth.device)
    
    # Step 5: Compute zvals_0_level
    zvals_0_level = noise * sigma + mean
    
    # Step 6: Handle rays with depth == 0
    zero_depth_mask = (depth == 0)
    
    if zero_depth_mask.any():
        # Generate uniform random points between near and far for these rays
        # Shape [B, n_samples]
        uniform_noise = torch.rand(mean.shape, device=depth.device)
        # Map [0, 1] to [near, far]
        # Note: near and far are [B], need broadcasting
        near_expanded = near.unsqueeze(1).expand(-1, n_samples)
        far_expanded = far.unsqueeze(1).expand(-1, n_samples)
        uniform_zvals = near_expanded + uniform_noise * (far_expanded - near_expanded)
        
        # Replace zvals for zero-depth rays
        zvals_0_level[zero_depth_mask] = uniform_zvals[zero_depth_mask]
        
    return zvals_0_level
```

**Parameters:**
- `start_sigma`: Float, default 0.1. Controls the spread of noise around the depth estimate [S1, unofficial].

**Edge Cases:**
- If `depth == 0`, the method falls back to uniform sampling between `near` and `far` [S1, unofficial].
- Precision issues may arise if `start_sigma` is large relative to the scene scale, causing samples to fall outside valid bounds. Clamping is not explicitly mentioned in the base method [S1, unofficial], but is required in variants.

## Subtopics

### Depth-Centric Sampling
Samples $K_{fd}$ positions around a specified depth value with Gaussian noise, clamped to ray bounds [S2, unofficial]. It repeats the input depth tensor of shape $(B,)$ along dimension 1 to shape $(B, K_{fd})$, adds Gaussian noise with standard deviation `depth_std`, and clamps the resulting values between `near` and `far` bounds from the rays [S2, unofficial]. This variant explicitly enforces bounds, unlike the base method which only handles zero-depth rays [S2, unofficial].

## Operating characteristics
**Time Complexity:** $O(N \cdot S)$, where $N$ is batch size and $S$ is number of samples. Generating noise and adding mean is linear in the number of samples.
**Memory Complexity:** $O(N \cdot S)$ to store the sample positions.
**Strengths:** Efficiently concentrates samples around estimated geometry, reducing noise in regions with accurate depth priors.
**Limitations:** Relies on the quality of the depth estimate; poor estimates lead to biased sampling. Does not inherently handle occlusion or complex geometry without additional logic.
**When to use:** When a reliable depth estimate (e.g., from a monocular depth network) is available.
**When not to use:** When no depth prior is available, or when the depth estimate is highly unreliable.

## Background
Originated in the context of neural radiance fields (NeRF) and related methods, specifically in implementations like `density_bounded_ray_weights` [S1, unofficial]. It is used to improve sampling efficiency by leveraging depth information, often in conjunction with multi-scale or multi-view setups [S1, unofficial].

## Sources
[S1] AlexSheldrick/density_bounded_ray_weights, https://github.com/AlexSheldrick/density_bounded_ray_weights, fetched 2026-10-27 (unofficial)
[S2] sxyu/pixel-nerf, https://github.com/sxyu/pixel-nerf, fetched 2026-10-26 (unofficial)
