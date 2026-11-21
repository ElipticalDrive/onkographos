# Importance Ray Sampling

Also called: Importance Sampling for NeRF

## Overview
Importance Ray Sampling is a technique used in neural volume rendering to refine the distribution of sample points along a ray. It addresses the inefficiency of uniform sampling by concentrating computational effort on regions with high expected density or opacity, thereby improving rendering quality with minimal overhead [S1, unofficial]. It builds on [NeRF Importance Sampling](1565-nerf-importance-sampling.md) by explicitly defining the procedural steps for binning, smoothing, and stratified sampling within a discrete partition of the ray.

## How it works
**Inputs:**
- `volume_range`: The spatial extent of the volume along the ray, partitioned into coarse bins.
- `self.weights`: Raw density or opacity estimates for each coarse bin, derived from a coarse model.
- `self.alpha`: A small scalar added to weights to prevent zero-probability bins.
- `blur_pool`: A boolean flag indicating whether to apply smoothing filters.
- `n_samples`: The number of fine sample points to generate.

**Outputs:**
- `ts`: A sorted array of normalized parameter values $t \in [0, 1]$ representing the positions of the fine sample points along the ray.

**Data Structures:**
- `weights`: A tensor of shape $(B, N_c)$, where $B$ is batch size and $N_c$ is the number of coarse bins.
- `pmf`: Probability Mass Function tensor of shape $(B, N_c)$, normalized such that $\sum_{i} \text{pmf}_i = 1$.
- `lower`, `upper`: Tensors of shape $(B, N_c)$ representing the start and end normalized parameters of each coarse bin.

**Steps:**
1. **Partition Volume:** Divide the volume range into coarse bins using timestamps $t_s$. Let $lower$ and $upper$ be the lower and upper bounds of these bins [S1, unofficial].
2. **Retrieve Weights:** Obtain the raw weights $w_i$ for each bin $i$ from the coarse model's output [S1, unofficial].
3. **Smooth Weights (Optional):** If `blur_pool` is true, apply a smoothing filter to $w$ to reduce noise and prevent aliasing artifacts [S1, unofficial]:
   - Pad $w$ with its first and last elements: $w_{padded} = \text{concat}(w[:, :1], w, w[:, -1:])$.
   - Compute adjacent maximums: $m_i = \max(w_{padded, i}, w_{padded, i+1})$.
   - Average adjacent maximums: $w'_i = 0.5 \times (m_i + m_{i+1})$.
   - Update weights: $w \leftarrow w'$ [S1, unofficial].
4. **Regularize Weights:** Add $\alpha$ to all weights to ensure non-zero probability: $w_i \leftarrow w_i + \alpha$. The default value for $\alpha$ is $10^{-5}$ [S1, unofficial].
5. **Compute PMF:** Normalize the weights along the bin dimension to form a probability mass function:
   $$ \text{pmf}_i = \frac{w_i}{\sum_{j=1}^{N_c} w_j} $$
   [S1, unofficial].
6. **Sample Indices:** Draw $n_{samples}$ indices $k$ from the PMF using categorical sampling: $k \sim \text{Categorical}(\text{pmf})$ [S1, unofficial].
7. **Generate Noise:** Generate random noise $t_{rand}$ with the same shape as the sampled indices $k$, uniformly distributed in $[0, 1]$ [S1, unofficial].
8. **Gather Bounds:** Retrieve the lower and upper bounds for the sampled bins:
   $$ l_k = \text{gather}(lower, k), \quad u_k = \text{gather}(upper, k) $$
   [S1, unofficial].
9. **Interpolate:** Compute the final sample positions by linearly interpolating within the sampled bins:
   $$ t = l_k + (u_k - l_k) \times t_{rand} $$
   [S1, unofficial].
10. **Sort:** Sort the resulting $t$ values along the sample dimension to maintain monotonicity along the ray [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def importance_ray_sampling(volume_range, weights, alpha=1e-5, blur_pool=False, n_samples=64):
    # 1. Partition volume range into coarse bins
    # Returns lower and upper bounds for each bin
    lower, upper = volume_range.partition()
    
    # 2. Retrieve weights from coarse model
    w = weights.clone()
    
    # 3. Optional smoothing (mip-NeRF style)
    if blur_pool:
        # Pad with first and last elements
        padded = torch.cat([w[..., :1, :], w, w[..., -1:, :]], dim=-2)
        # Max of adjacent pairs
        maxes = torch.maximum(padded[..., :-1, :], padded[..., 1:, :])
        # Average adjacent maxes
        w = 0.5 * (maxes[..., :-1, :] + maxes[..., 1:, :])
    
    # 4. Add alpha to prevent zero probabilities
    w = w + alpha
    
    # 5. Compute PMF
    pmf = w / w.sum(dim=-2, keepdim=True)
    
    # 6. Sample indices from PMF
    # sample_pmf draws n_samples indices for each ray in the batch
    inds = sample_pmf(pmf, n_samples)
    
    # 7. Generate random noise
    t_rand = torch.rand_like(inds, dtype=torch.float32)
    
    # 8. Gather bounds for sampled indices
    lower_ = gather(lower, -2, inds)
    upper_ = gather(upper, -2, inds)
    
    # 9. Interpolate within bins
    ts = lower_ + (upper_ - lower_) * t_rand
    
    # 10. Sort along sample dimension
    ts = torch.sort(ts, dim=-2).values
    
    return ts
```

**Parameters:**
- `alpha`: Default $10^{-5}$ [S1, unofficial].
- `blur_pool`: Boolean, default False [S1, unofficial].

**Edge Cases:**
- If all weights are zero, the addition of $\alpha$ ensures valid probabilities.
- If `volume_range` has only one bin, sampling degenerates to uniform distribution within that bin.

## Operating characteristics
- **Time Complexity:** Dominated by the categorical sampling and sorting steps, $O(N_c + N_r \cdot N_s \log N_s)$, where $N_c$ is coarse bins, $N_r$ is rays, and $N_s$ is samples per ray.
- **Memory Complexity:** $O(N_c)$ for storing weights and PMF per ray.
- **Strengths:** Reduces variance in regions of high density; computationally cheap compared to learning a separate importance network.
- **Limitations:** Relies on the accuracy of the coarse model's weights; may undersample if coarse weights are inaccurate.

## Background
This method is implemented in the `gsgen` repository, an unofficial source for 3D generation tools [S1, unofficial]. It adapts techniques from mip-NeRF's smoothing strategy for NeRF importance sampling.

## Sources
[S1] gsgen3d/gsgen, https://github.com/gsgen3d/gsgen, fetched 2026-10-27 (unofficial)
