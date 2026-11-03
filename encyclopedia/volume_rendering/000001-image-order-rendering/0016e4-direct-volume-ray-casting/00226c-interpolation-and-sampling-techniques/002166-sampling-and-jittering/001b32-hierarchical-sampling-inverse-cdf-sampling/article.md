# Hierarchical Sampling (Inverse CDF Sampling)

Also called: Fine Sampling, Hierarchical Importance Sampling, Inverse CDF Sampling

## Overview
Hierarchical Sampling (Inverse CDF Sampling) generates importance samples along a ray in volume rendering by inverting the cumulative distribution function (CDF) of the rendering contribution [S1, S2]. It solves the problem of finding the ray parameter $t$ corresponding to a target cumulative density value $y$, allowing samples to be drawn directly from transmittance estimates rather than computing expensive integrals [S2]. Unlike standard NeRF hierarchical sampling, which interpolates exponentiated densities (probabilities) and requires auxiliary losses, this method interpolates the density field directly (piecewise linear or constant), offering better numerical stability for end-to-end optimization [S1]. It enables Monte Carlo estimates with fewer ray points and allows gradients to back-propagate through the sampling process to the density field parameters [S1].

## How it works
**Inputs:**
*   `coarse_z_vals`: 1D tensor of depth coordinates for coarse samples, shape `(N_c,)` or `(batch, N_c)` [S4].
*   `coarse_weights`: 1D tensor of unnormalized importance weights (typically derived from volume density or opacity) corresponding to the coarse samples, shape `(N_c,)` or `(batch, N_c)` [S4].
*   `N_samples`: Integer specifying the number of fine samples to generate per ray [S1, S5].
*   `deterministic`: Boolean flag. If `True`, uses deterministic uniform spacing; otherwise uses random jitter [S1].

**Outputs:**
*   `fine_z_vals`: 1D tensor of interpolated depth coordinates for the fine samples, shape `(N_samples,)` or `(batch, N_samples)` [S4].

**Data Structures:**
*   The method operates on 1D tensors representing the ray's depth interval [G].
*   For batched operations, dimensions are broadcasted or processed in parallel [G].
*   Intermediate structures include the Probability Density Function (PDF) and Cumulative Distribution Function (CDF) tensors [G].

**Steps:**

1.  **Compute PDF:** Normalize the coarse weights to form a probability density function. To prevent division by zero or NaNs, add a small epsilon ($\epsilon = 10^{-5}$) to all weights before normalization [S1, S2, S5, S7].
    $$ \text{pdf}_i = \frac{\text{weights}_i + \epsilon}{\sum_{j} (\text{weights}_j + \epsilon)} $$
    where $\text{weights}$ are the `coarse_weights` [S4].

2.  **Compute CDF:** Calculate the cumulative distribution function by taking the cumulative sum of the PDF. Prepend a zero tensor to the CDF to represent the start of the first bin (index 0) [S1, S2, S4, S5, S7].
    $$ \text{cdf}_0 = 0 $$
    $$ \text{cdf}_i = \text{cdf}_{i-1} + \text{pdf}_{i-1} \quad \text{for } i > 0 $$
    In code, this is often implemented as `cdf = torch.cat([torch.zeros_like(cdf[..., :1]), torch.cumsum(pdf, -1)], -1)` [S1, S4, S5].

3.  **Generate Query Values:** Generate uniform random numbers $u$ in the interval $[0, 1]$ with size `N_samples` [S1, S2, S4, S5, S7].
    *   If `deterministic` is `True`, use linearly spaced values: $u_i = \frac{i}{N_{\text{samples}}}$ for $i \in [0, N_{\text{samples}}-1]$ [S1, S4, S5, S7].
    *   If `deterministic` is `False`, use random uniform samples: $u_i \sim \mathcal{U}(0, 1)$ [S1, S2, S4, S5, S7].

4.  **Find Bin Indices:** Determine which bin in the CDF each query value $u$ falls into using binary search (`searchsorted`) [S1, S2, S4, S5, S7].
    $$ \text{inds} = \text{searchsorted}(\text{cdf}, u, \text{side}=\text{'right'}) $$
    This returns the index $k$ such that $\text{cdf}_{k-1} \le u < \text{cdf}_k$ [S1, S2, S4, S5, S7].

5.  **Determine Bin Boundaries:** Identify the lower (`below`) and upper (`above`) bin indices for interpolation. Clamp these indices to valid ranges $[0, \text{len}(\text{cdf})-1]$ [S1, S2, S4, S5, S7].
    $$ \text{below} = \max(0, \text{inds} - 1) $$
    $$ \text{above} = \min(\text{len}(\text{cdf}) - 1, \text{inds}) $$

6.  **Gather CDF and Depth Values:** Retrieve the CDF values and depth coordinates (`bins`, which are `coarse_z_vals`) for the lower and upper bins [S1, S4, S5, S7].
    $$ \text{cdf}_{\text{below}} = \text{cdf}[\text{below}] $$
    $$ \text{cdf}_{\text{above}} = \text{cdf}[\text{above}] $$
    $$ \text{z}_{\text{below}} = \text{bins}[\text{below}] $$
    $$ \text{z}_{\text{above}} = \text{bins}[\text{above}] $$

7.  **Interpolate Sample Positions:** Perform linear interpolation within the identified bin to find the final sample depth $z_{\text{sample}}$ [S1, S2, S4, S5, S7].
    $$ \text{denom} = \text{cdf}_{\text{above}} - \text{cdf}_{\text{below}} $$
    Handle division by zero by clamping `denom` to a small value (e.g., $10^{-5}$) if it is below the threshold [S1, S4, S5, S7].
    $$ t = \frac{u - \text{cdf}_{\text{below}}}{\max(\text{denom}, 10^{-5})} $$
    $$ z_{\text{sample}} = \text{z}_{\text{below}} + t \cdot (\text{z}_{\text{above}} - \text{z}_{\text{below}}) $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def hierarchical_sampling(coarse_z_vals, coarse_weights, N_samples, deterministic=True):
    """
    Perform hierarchical sampling using inverse CDF.
    
    Args:
        coarse_z_vals: Tensor of shape (N_c,) or (batch, N_c) containing coarse depth samples.
        coarse_weights: Tensor of shape (N_c,) or (batch, N_c) containing unnormalized weights.
        N_samples: int, number of fine samples to generate.
        deterministic: bool, if True uses linspace, else random uniform.
        
    Returns:
        fine_z_vals: Tensor of shape (N_samples,) or (batch, N_samples).
    """
    # Step 1: Compute PDF with epsilon to prevent NaNs
    eps = 1e-5
    weights = coarse_weights + eps
    pdf = weights / torch.sum(weights, dim=-1, keepdim=True)
    
    # Step 2: Compute CDF and prepend zero
    cdf = torch.cumsum(pdf, dim=-1)
    cdf = torch.cat([torch.zeros_like(cdf[..., :1]), cdf], dim=-1)
    
    # Step 3: Generate query values u
    if deterministic:
        u = torch.linspace(0., 1., steps=N_samples).to(cdf.device)
        # Handle batch dimension if present
        if cdf.dim() > 1:
            u = u.unsqueeze(0).expand(cdf.shape[0], -1)
    else:
        u = torch.rand_like(cdf[..., :1]).expand_as(cdf[..., :1]).reshape(cdf.shape[:-1] + (N_samples,))
        # Note: In batched case, u shape should be (batch, N_samples)
        # Simplified for single ray or assuming batch dim is handled by broadcasting in torch.rand
        if cdf.dim() > 1:
            u = torch.rand(cdf.shape[0], N_samples).to(cdf.device)
            
    # Step 4: Find bin indices
    inds = torch.searchsorted(cdf, u, right=True)
    
    # Step 5: Determine bin boundaries
    below = torch.clamp(inds - 1, min=0)
    above = torch.clamp(inds, max=cdf.shape[-1] - 1)
    
    # Step 6: Gather CDF and bin values
    # Expand indices for gathering
    inds_g = torch.stack([below, above], dim=-1)
    
    # Gather CDF values
    cdf_g = torch.gather(cdf.unsqueeze(1).expand_as(inds_g), 2, inds_g).squeeze(1)
    
    # Gather depth values (bins)
    # Assuming coarse_z_vals has shape (N_c,) or (batch, N_c)
    # We need to gather from coarse_z_vals using below and above indices
    bins_g = torch.gather(coarse_z_vals.unsqueeze(1).expand_as(inds_g), 2, inds_g).squeeze(1)
    
    # Step 7: Interpolate
    denom = cdf_g[..., 1] - cdf_g[..., 0]
    denom = torch.where(denom < 1e-5, torch.ones_like(denom), denom)
    t = (u - cdf_g[..., 0]) / denom
    fine_z_vals = bins_g[..., 0] + t * (bins_g[..., 1] - bins_g[..., 0])
    
    return fine_z_vals
```

**Parameters:**
*   `N_samples`: Integer specifying the number of fine samples to generate per ray. Typical values are 128 or 64 [S1, S4].
*   `deterministic`: Boolean flag. If `True`, uses deterministic linspace samples instead of random uniform samples for reproducibility [S1, S4].
*   `perturb`: Float controlling jittering. 0 disables jittering (deterministic midpoints), 1 enables full stratified jittering. This parameter is often used in the coarse pass, but the fine pass here is strictly inverse CDF [S1].

**Edge Cases and Pitfalls:**
*   **Zero Weights:** Zero weights cause division by zero during PDF normalization. Handled by adding $\epsilon = 10^{-5}$ to all weights before normalization [S1, S2, S5, S7].
*   **Zero-Width Bins:** Zero-width bins (where `cdf_above == cdf_below`) cause division by zero during interpolation. Handled by clamping the denominator `denom` to $10^{-5}$ if it falls below that threshold [S1, S4, S5, S7].
*   **Boundary Conditions:** Indices must be clamped to valid ranges $[0, \text{len}(\text{cdf})-1]$ to prevent out-of-bounds access [S1, S2, S4, S5, S7].

## Subtopics

### [Inverse Opacity Calculation](1099-inverse-opacity-calculation.md)
Inverse Opacity Calculation is a differentiable method for generating importance samples along a ray in volume rendering by inverting the cumulative distribution function (CDF) of the rendering contribution. It solves the problem of finding the ray parameter $t$ corresponding to a target cumulative density value $y$, allowing samples to be drawn directly from transmittance estimates rather than computing expensive integrals. Unlike standard NeRF hierarchical sampling, which interpolates exponentiated densities (probabilities) and requires auxiliary losses, this method interpolates the density field directly (piecewise linear or constant), offering better numerical stability for end-to-end optimization. It enables Monte Carlo estimates with fewer ray points and allows gradients to back-propagate through the sampling process to the density field parameters.

### [Proposal Network Sampling](1589-proposal-network-sampling.md)
Proposal Network Sampling is a method for importance sampling in volume rendering that directly estimates the Probability Density Function (PDF) along a ray using a coarse neural network, known as the proposal network. It solves the problem of inefficient uniform sampling by concentrating samples on high-contribution areas (such as surfaces) while spreading them in empty or occluded regions, providing more accurate transmittance estimation than spatial skipping methods. It builds on [Hierarchical Sampling (Inverse CDF Sampling)](973-hierarchical-sampling-inverse-cdf-sampling.md) by learning the sampling distribution dynamically rather than relying on static weights [G].

### [RangeFineSampler](3141-rangefinesampler.md)
RangeFineSampler is a hierarchical sampling technique for direct volume ray casting that refines coarse samples by generating a dense uniform grid around the coarse sample with the highest weight, supplemented by random samples across the full ray extent. It addresses the problem of missing high-frequency details between coarse samples by concentrating computational effort near the most significant signal while maintaining global coverage through random jittering. Unlike [Inverse Opacity Calculation](1099-inverse-opacity-calculation.md) which derives samples from opacity gradients, or [Proposal Network Sampling](1589-proposal-network-sampling.md) which uses learned neural networks to predict sample locations, RangeFineSampler relies on a deterministic geometric expansion around the peak weight index combined with stochastic uniform sampling.

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** The method involves computing the PDF and CDF ($O(N_c)$), generating random numbers ($O(N_f)$), binary search ($O(N_f \log N_c)$), and interpolation ($O(N_f)$). Overall complexity is dominated by $O(N_f \log N_c)$, where $N_c$ is the number of coarse samples and $N_f$ is the number of fine samples.
*   **Memory:** Requires storage for the CDF and intermediate tensors, proportional to $O(N_c + N_f)$.

**Measured Performance:**
*   Performance depends on the hardware and resolution. On a single 2080 Ti, training a NeRF on a low-res scene (e.g., `fern`) takes approximately 8 hours for 200k iterations [S7].
*   The method allows for fewer ray points while maintaining accuracy, reducing the computational load compared to uniform sampling [S1].

**Scaling:**
*   The method scales linearly with the number of rays and samples.
*   It is suitable for large-scale volume rendering due to its efficiency in concentrating samples in high-density regions.

**Strengths and Limitations:**
*   **Strengths:** Differentiable, allowing gradients to back-propagate through the sampling process [S1]. Numerically stable with proper epsilon handling [S1, S7]. Efficient importance sampling [S1].
*   **Limitations:** Requires coarse samples and weights as input [S4]. Performance may degrade if coarse weights are inaccurate [S1].

**When to Use:**
*   Use when importance sampling is needed to reduce variance in volume rendering [S1].
*   Use when differentiable sampling is required for end-to-end optimization [S1].

**When Not to Use:**
*   Avoid when computational resources are extremely limited and coarse weights are not available [S4].
*   Avoid when the scene has very low density variations, making importance sampling less beneficial [S1].

## Background
Hierarchical Sampling (Inverse CDF Sampling) originates from the NeRF (Neural Radiance Fields) paper by Mildenhall et al. [S2, S5]. It was introduced to improve the efficiency of volume rendering by concentrating samples in regions of high density. The method has been widely adopted in subsequent works, including [S1, S4, S7]. It is a key component in many modern neural rendering pipelines.

## Sources
[S1] GreatDrake/reparameterized-volume-sampling, https://github.com/GreatDrake/reparameterized-volume-sampling, fetched 2026-10-26 (unofficial)
[S2] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S3] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S4] albertgassol1/vf-nerf, https://github.com/albertgassol1/vf-nerf, fetched 2026-10-26 (unofficial)
[S5] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
[S6] InanXR/FLASH, https://github.com/InanXR/FLASH, fetched 2026-10-27 (unofficial)
[S7] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
