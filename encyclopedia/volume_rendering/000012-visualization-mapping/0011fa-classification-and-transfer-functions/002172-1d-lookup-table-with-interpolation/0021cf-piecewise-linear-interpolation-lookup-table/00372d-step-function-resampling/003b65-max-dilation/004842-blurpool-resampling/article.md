# Blurpool Resampling

Also called: Resampling Along Rays

## Overview
Blurpool Resampling generates new sample positions ($t$ values) along rays from a weight distribution [S1, unofficial][S2, unofficial]. It differs from [Max Dilation](8984-max-dilation.md) by applying average pooling to max-pooled weights before sampling [S1, unofficial][S2, unofficial].

## How it works
The method transforms input weights associated with initial sample positions into a new set of sample positions [S1, unofficial][S2, unofficial]. It operates on a piecewise constant probability density function (PDF) derived from the weights [G].

### Inputs and Outputs
*   **Input Weights**: Tensor of shape `[batch_size, num_samples]` representing the importance or probability mass of each initial sample [S1, unofficial].
*   **Input Parameters**:
    *   `resample_padding`: Float value added to the weights before normalization [S1, unofficial][S2, unofficial].
    *   `stop_grad`: Boolean flag to prevent backpropagation through the sampling process [S1, unofficial].
*   **Output t_samples**: Tensor of shape `[batch_size, num_samples + 1]` containing the new sampled $t$ values [S1, unofficial].
*   **Output Points**: Tensor of shape `[batch_size, num_samples, 3]` containing the 3D coordinates corresponding to the new samples, derived via ray casting [S1, unofficial].

### Data Structures and Layout
The core data structure is the weight tensor [S1, unofficial][S2, unofficial]. The algorithm assumes the weights define a discrete distribution over the initial sample indices [G]. The output `t_samples` defines the boundaries of the new intervals, implying `num_samples + 1` boundaries for `num_samples` intervals [G].

### Steps
1.  **Padding**: Pad the input weights tensor `weights` along the last dimension by concatenating the first element, the original weights, and the last element: `weights_pad = cat([weights[..., :1], weights, weights[..., -1:]], dim=-1)` [S1, unofficial][S2, unofficial].
2.  **Max Pooling**: Compute the element-wise maximum of adjacent elements in `weights_pad`: `weights_max = maximum(weights_pad[..., :-1], weights_pad[..., 1:])` [S1, unofficial][S2, unofficial].
3.  **Blurpool**: Compute the element-wise average of adjacent elements in `weights_max`: `weights_blur = 0.5 * (weights_max[..., :-1] + weights_max[..., 1:])` [S1, unofficial][S2, unofficial].
4.  **Add Padding**: Add the `resample_padding` scalar to `weights_blur`: `weights_final = weights_blur + resample_padding` [S1, unofficial][S2, unofficial].
5.  **Sampling**: Sample new $t$ values from the normalized distribution defined by `weights_final` using a sorted piecewise constant PDF sampler: `new_t_vals = sorted_piecewise_constant_pdf(t_vals, weights_final, t_vals.shape[-1], randomized)` [S1, unofficial][S2, unofficial].
6.  **Gradient Control**: If `stop_grad` is true, wrap steps 1–5 in a no-gradient context (e.g., `torch.no_grad()` or `jax.lax.stop_gradient`) [S1, unofficial][S2, unofficial].
7.  **Ray Casting**: Cast rays using the new $t$ values, ray origins, directions, radii, and `ray_shape` to compute the mean positions and covariances: `means, covs = cast_rays(new_t_vals, origins, directions, radii, ray_shape, a, f, l)` [S1, unofficial][S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def blurpool_resampling(weights, t_vals, origins, directions, radii, ray_shape, resample_padding=1e-6, stop_grad=False, randomized=True, a=None, f=None, l=None):
    """
    Resamples t-values along rays based on weights using blurpool technique.
    Args:
        weights: Tensor [batch_size, num_samples]
        t_vals: Tensor [batch_size, num_samples + 1]
        origins: Tensor [batch_size, 3]
        directions: Tensor [batch_size, 3]
        radii: Tensor [batch_size] or scalar
        ray_shape: String defining ray geometry
        resample_padding: Float added to weights
        stop_grad: Boolean, if True, disable gradients
        randomized: Boolean, if True, use randomized sampling
        a, f, l: Additional ray parameters (exact usage not specified in sources [G])
    Returns:
        new_t_vals: Tensor [batch_size, num_samples + 1]
        means: Tensor [batch_size, num_samples, 3]
        covs: Tensor [batch_size, num_samples, 3, 3]
    """
    if stop_grad:
        with torch.no_grad():
            return _blurpool_core(weights, t_vals, origins, directions, radii, ray_shape, resample_padding, randomized, a, f, l)
    return _blurpool_core(weights, t_vals, origins, directions, radii, ray_shape, resample_padding, randomized, a, f, l)

def _blurpool_core(weights, t_vals, origins, directions, radii, ray_shape, resample_padding, randomized, a, f, l):
    # Step 1: Pad weights with first and last elements along the last dimension
    weights_pad = torch.cat([weights[..., :1], weights, weights[..., -1:]], dim=-1)
    # Step 2: Max pooling of adjacent elements
    weights_max = torch.maximum(weights_pad[..., :-1], weights_pad[..., 1:])
    # Step 3: Blurpool (average pooling) of adjacent elements
    weights_blur = 0.5 * (weights_max[..., :-1] + weights_max[..., 1:])
    # Step 4: Add padding
    weights_final = weights_blur + resample_padding
    # Step 5: Sample new t values
    new_t_vals = _sample_piecewise_constant(t_vals, weights_final, t_vals.shape[-1], randomized)
    # Step 6: Cast rays
    means, covs = _cast_rays(new_t_vals, origins, directions, radii, ray_shape, a, f, l)
    return new_t_vals, (means, covs)

def _sample_piecewise_constant(t_vals, weights, num_samples, randomized):
    # Normalize weights to form a probability distribution
    probs = weights / weights.sum(dim=-1, keepdim=True)
    # Compute cumulative distribution function
    cdf = torch.cumsum(probs, dim=-1)
    # Generate uniform samples
    u = torch.rand_like(probs) if randomized else torch.linspace(0.0, 1.0, probs.shape[-1], device=probs.device)
    # Inverse CDF sampling
    indices = torch.searchsorted(cdf, u, right=True) - 1
    indices = torch.clamp(indices, 0, probs.shape[-1] - 1)
    # Linear interpolation within intervals
    batch_idx = torch.arange(t_vals.shape[0])[:, None]
    t_low = t_vals[batch_idx, indices]
    t_high = t_vals[batch_idx, indices + 1]
    p_low = cdf[batch_idx, indices]
    p_high = cdf[batch_idx, indices + 1]
    denom = p_high - p_low
    denom = torch.where(denom == 0, 1.0, denom)
    t_new = t_low + (u - p_low) / denom * (t_high - t_low)
    # Return boundaries: prepend first t, append last t to match [B, N+1] shape
    return torch.cat([t_vals[..., :1], t_new, t_vals[..., -1:]], dim=-1)

def _cast_rays(t_vals, origins, directions, radii, ray_shape, a, f, l):
    # Compute mean positions along rays
    means = origins[..., None, :] + t_vals[..., :, None] * directions[..., None, :]
    # Compute covariance based on ray shape and radii [G]
    # Exact covariance formula for parameters a, f, l is not specified in sources [G]
    if ray_shape == "gaussian":
        r2 = radii[..., None, None] ** 2
        covs = torch.zeros(means.shape[0], means.shape[1], 3, 3, device=means.device)
        covs[:, :, 0, 0] = r2
        covs[:, :, 1, 1] = r2
    else:
        covs = torch.zeros(means.shape[0], means.shape[1], 3, 3, device=means.device)
    return means, covs
```

### Edge Cases and Pitfalls
*   **Zero Weights**: If all weights are zero, normalization fails. The `resample_padding` parameter is added to weights before normalization [S1, unofficial][S2, unofficial].
*   **Precision**: Floating-point precision issues in cumulative sum and inverse CDF sampling can lead to incorrect sample positions. Use stable CDF computation [G].
*   **Boundary Conditions**: The padding step handles boundaries by repeating the first and last weights, which affects the max and blurpool operations at the edges [S1, unofficial][S2, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$ per ray, where $N$ is the number of samples. The operations are element-wise tensor operations [G].
*   **Memory Complexity**: $O(N)$ for storing intermediate tensors (`weights_pad`, `weights_max`, `weights_blur`) [G].
*   **Strengths**: Differentiable, allowing gradient-based optimization of sample positions [G]. Smooths the weight distribution, potentially leading to more stable sampling [G].
*   **Limitations**: The blurpool operation may oversmooth sharp peaks in the weight distribution, leading to less accurate sampling in high-detail regions compared to non-smoothed methods [G].
*   **When to Use**: When differentiable sampling is required and a smooth weight distribution is beneficial [G].
*   **When Not to Use**: When sharp, precise sampling is needed and differentiability is not required [G].

## Background
Blurpool Resampling is used in neural volume rendering frameworks, specifically in implementations like [AlexSheldrick/density_bounded_ray_weights](https://github.com/AlexSheldrick/density_bounded_ray_weights) and [CPREgroup/CaSE-NeRF](https://github.com/CPREgroup/CaSE-NeRF) [S1, unofficial][S2, unofficial]. It adapts the "blurpool" concept from image processing to ray sampling [G]. The method was introduced in unofficial repositories as part of efforts to improve the stability and quality of NeRF training by adapting sample densities [S1, unofficial][S2, unofficial].

## Sources
[S1] AlexSheldrick/density_bounded_ray_weights, https://github.com/AlexSheldrick/density_bounded_ray_weights, fetched 2026-10-27 (unofficial)
[S2] https://github.com/CPREgroup/CaSE-NeRF, https://github.com/CPREgroup/CaSE-NeRF, fetched 2026-10-30 (unofficial)
