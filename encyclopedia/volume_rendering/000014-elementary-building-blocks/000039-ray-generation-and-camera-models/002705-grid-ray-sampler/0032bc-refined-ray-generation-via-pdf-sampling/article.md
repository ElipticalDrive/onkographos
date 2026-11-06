# Refined Ray Generation via PDF Sampling

Also called: NeRF Ray Refinement Sampling, Refine Ray Generation, Refined Ray Generation

## Overview
Refined Ray Generation via PDF Sampling generates higher-fidelity ray samples for volume rendering by resampling depth positions based on the probability distribution defined by previous iteration's weights. It improves upon the coarse sampling provided by the [Grid Ray Sampler](3930-grid-ray-sampler.md) by concentrating samples in regions of high opacity or color change, thereby reducing noise and aliasing in the final render.

## How it works
The method takes previous depth samples and their associated importance weights, constructs a piecewise constant probability density function (PDF), and draws new samples from this distribution. It then computes the geometric properties (segment lengths, midpoints, and world positions) required for the volume rendering integral.

### Inputs and Outputs
**Inputs:**
*   `prev_ts`: Tensor of shape `(N, Rays, PrevSamples)` containing the depth distances of the previous iteration's samples.
*   `prev_weights`: Tensor of shape `(N, Rays, PrevSamples)` containing the importance weights (typically opacity $\times$ color contribution) corresponding to `prev_ts`.
*   `point_count`: Integer specifying the number of new samples to generate per ray.
*   `campos`: Camera position vector.
*   `raydir`: Ray direction vector.
*   `jitter`: Boolean flag indicating whether to add uniform noise to samples to prevent banding artifacts.

**Outputs:**
*   `raypos`: Tensor of shape `(N, Rays, point_count)` containing the 3D world positions of the new sample points.
*   `segment_length`: Tensor of shape `(N, Rays, point_count)` containing the length of each segment along the ray.
*   `validity_mask`: Boolean tensor indicating valid samples (initialized to all ones).

### Data Structures
*   `prev_ts`: Float tensor. Stores depth values $t$.
*   `prev_weights`: Float tensor. Stores weights $w$.
*   `end_point_ts`: Float tensor. Stores the sorted new depth samples.
*   `middle_point_ts`: Float tensor. Stores the midpoints of the segments defined by `end_point_ts`.
*   `segment_length`: Float tensor. Stores the distance between consecutive endpoints.

### Algorithm Steps

1.  **Sample New Depths:** Use a PDF sampling routine (`sample_pdf`) to draw `point_count + 1` new depth values from the distribution defined by `prev_ts` and `prev_weights`. This routine treats the weights as probabilities and samples from the cumulative distribution function (CDF). If `jitter` is true, uniform noise is added to the samples.
2.  **Reshape:** Reshape `end_point_ts` to remove any extra dimensions introduced by the sampling function, resulting in shape `(N, Rays, point_count + 1)`.
3.  **Compute Segment Lengths:** Calculate the difference between consecutive depth values:
    $$ \text{segment\_length}_i = \text{end\_point\_ts}_i - \text{end\_point\_ts}_{i-1} $$
    for $i = 1 \dots \text{point\_count}$.
4.  **Scale Segment Lengths:** Multiply the depth differences by the norm of the ray direction vector $\|\mathbf{d}\|$ to convert depth distance to Euclidean distance:
    $$ \text{segment\_length}_i^{\text{euclidean}} = \text{segment\_length}_i \times \|\mathbf{d}\| $$
5.  **Compute Midpoints:** Calculate the midpoint depth for each segment:
    $$ \text{middle\_point\_ts}_i = \frac{\text{end\_point\_ts}_i + \text{end\_point\_ts}_{i-1}}{2} $$
6.  **Compute Ray Positions:** Calculate the 3D world position for each midpoint:
    $$ \mathbf{p}_i = \mathbf{c} + \text{middle\_point\_ts}_i \times \mathbf{d} $$
    where $\mathbf{c}$ is `campos` and $\mathbf{d}$ is `raydir`.
7.  **Initialize Validity Mask:** Create a boolean mask of ones with the same shape and device as `middle_point_ts`.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def sample_pdf(bins, weights, n_samples, jitter=True):
    """
    PDF sampling from bins and weights.
    bins: (..., N) depth values
    weights: (..., N) importance weights
    n_samples: int, number of samples to draw
    jitter: bool, add uniform noise
    Returns: (..., n_samples) sampled depths
    """
    # Pad bins with zeros at start and end to define full range
    # Note: In NeRF context, bins usually represent intervals.
    # Weights are associated with intervals or points.
    # Standard implementation:
    # 1. Compute CDF from weights
    # 2. Sample uniform random numbers
    # 3. Invert CDF to get bins
    
    # Add zero at the beginning
    weights = F.pad(weights, (1, 0), 'constant', 0.0)
    
    # Compute CDF
    cdf = torch.cumsum(weights, dim=-1)
    cdf = cdf[..., :-1]  # Remove last element as it's sum of all weights
    cdf = cdf / torch.max(cdf[..., -1:], dim=-1, keepdim=True)[0]
    
    # Sample uniform random numbers
    u = torch.rand(list(weights.shape[:-1]) + [n_samples], device=weights.device)
    if jitter:
        u = u * (1.0 / n_samples) + torch.rand_like(u) / n_samples
        
    # Invert CDF
    # Find indices where cdf < u
    indices = torch.searchsorted(cdf, u, side='left')
    
    # Get lower and upper bins
    bins_g = bins.gather(-1, indices)
    bins_l = bins.gather(-1, indices - 1)
    
    # Interpolate
    # This is a simplified interpolation; exact implementation depends on source
    # For this method, we assume sample_pdf returns the new depth points
    return bins_g

def refined_ray_generation(prev_ts, prev_weights, point_count, campos, raydir, jitter=True):
    """
    prev_ts: (N, Rays, PrevSamples)
    prev_weights: (N, Rays, PrevSamples)
    """
    # Step 1: Sample new end-point ts
    # sample_pdf expects bins and weights. 
    # prev_ts are the bins (depths), prev_weights are the weights.
    # We sample point_count + 1 points.
    end_point_ts = sample_pdf(prev_ts, prev_weights, point_count + 1, jitter)
    
    # Step 2: Reshape end_point_ts
    # Ensure shape is (N, Rays, point_count + 1)
    # The sample_pdf output might have different shape depending on implementation
    # Assuming it returns the correct shape or needs reshaping
    if end_point_ts.dim() > 3:
        end_point_ts = end_point_ts.view(end_point_ts.shape[0], end_point_ts.shape[1], -1)
        
    # Step 3: Compute segment lengths
    # segment_length = end_point_ts[:, :, 1:] - end_point_ts[:, :, :-1]
    segment_length = end_point_ts[:, :, 1:] - end_point_ts[:, :, :-1]
    
    # Step 4: Compute middle-point ts
    middle_point_ts = (end_point_ts[:, :, :-1] + end_point_ts[:, :, 1:]) / 2.0
    
    # Step 5: Compute ray positions
    # raypos = campos + raydir * middle_point_ts
    # Note: raydir is typically normalized, but we scale segment_length by norm later
    # If raydir is not normalized, we must handle scaling carefully.
    # Standard NeRF assumes raydir is normalized.
    raypos = campos.unsqueeze(1).unsqueeze(1) + raydir.unsqueeze(1).unsqueeze(1) * middle_point_ts.unsqueeze(-1)
    
    # Step 6: Multiply segment_length by ray direction norm
    ray_dir_norm = torch.norm(raydir, dim=-1, keepdim=True)
    segment_length = segment_length * ray_dir_norm
    
    # Step 7: Initialize validity mask
    validity_mask = torch.ones_like(middle_point_ts, dtype=torch.bool)
    
    return raypos, segment_length, validity_mask
```

### Parameters
*   `point_count`: The number of refined samples per ray. Typical values range from 32 to 64 in NeRF implementations.
*   `jitter`: Boolean. When true, adds uniform noise to prevent banding. Default is often true.

### Edge Cases and Pitfalls
*   **Zero Weights:** If all `prev_weights` are zero, the CDF is undefined. A guard should ensure weights sum to a positive value or add a small epsilon.
*   **Unsorted Depths:** The algorithm assumes `prev_ts` are sorted. If not, sorting is required before computing CDF.
*   **Precision:** Floating point precision issues may arise when computing CDF if weights vary significantly in magnitude.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot R \cdot S \log S)$ or $O(N \cdot R \cdot S)$ depending on the sorting/searching implementation in `sample_pdf`, where $N$ is batch size, $R$ is number of rays, and $S$ is `PrevSamples`.
*   **Memory Complexity:** $O(N \cdot R \cdot S)$ for storing intermediate tensors.
*   **Strengths:** Significantly reduces noise compared to uniform sampling by concentrating samples in high-variance regions.
*   **Limitations:** Requires a previous pass of sampling and rendering to generate weights. Adds computational overhead per iteration.

## Background
This method is a core component of Neural Radiance Fields (NeRF) and its variants, such as PointNeRF. It was introduced to improve the efficiency of volume rendering by adaptively sampling along rays. The technique builds on the importance sampling principles used in computer graphics to reduce variance in Monte Carlo integration.

## Sources
[S1] Xharlie/pointnerf, https://github.com/Xharlie/pointnerf, fetched 2026-10-26 (unofficial)
