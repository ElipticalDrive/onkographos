# Hierarchical Sampling (Inverse CDF Sampling)

Also called: Inverse CDF Sampling, Inverse CDF, PDF Sampling via Inverse CDF

## Overview
Hierarchical Sampling (Inverse CDF Sampling) is a method for generating fine-grained depth samples along a ray in volume rendering by drawing from a probability distribution derived from coarse rendering weights [S1, S2, S3, unofficial]. It solves the problem of inefficient uniform sampling by concentrating computational effort on high-density regions identified during a coarse pass, thereby improving density estimation and image quality with fewer total samples [S3, S4, S5, unofficial]. Unlike [Uniform Ray Sampling](1261-uniform-ray-sampling.md) which distributes points evenly, or [Stochastic Jittering](924-stochastic-jittering.md) which only adds noise to uniform grids, this method uses inverse transform sampling to adaptively place samples based on scene content [S1, S2, unofficial]. It builds on [Sampling and Jittering](2516-sampling-and-jittering.md) by providing a structured way to refine sample locations using importance sampling [G].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `bins`: A 1D tensor of shape `(N_bins,)` containing the boundaries of the coarse sample intervals (depth values $z$) [S1, S7, unofficial].
    *   `weights`: A 2D tensor of shape `(B, N_bins - 1)` containing the unnormalized importance weights (e.g., density or opacity contribution) for each bin along $B$ rays [S1, S7, unofficial]. Note: Some implementations exclude the first and last bins, resulting in shape `(B, N_bins - 2)` [S9, unofficial].
    *   `N_samples`: Integer specifying the number of fine samples to generate per ray [S1, S5, unofficial].
    *   `det`: Boolean flag. If `True`, uses deterministic uniform spacing; if `False`, uses random stratified sampling [S1, S5, unofficial].
*   **Outputs:**
    *   `z_samples`: A 2D tensor of shape `(B, N_samples)` containing the new depth values for the fine samples [S1, S9, unofficial].

**Data Structures:**
*   **Bins:** Sorted depth coordinates defining the intervals. Midpoints are computed as $m_i = 0.5 \times (bins_{i+1} + bins_i)$ [S5, unofficial].
*   **Weights:** Non-negative values representing the relative importance of each bin.
*   **CDF:** Cumulative Distribution Function tensor of shape `(B, N_bins)` (after prepending zeros) [S4, S7, unofficial].

**Algorithm Steps:**

1.  **Stabilize Weights:** Add a small epsilon $\epsilon = 10^{-5}$ to all weights to prevent division by zero during normalization [S2, S6, S9, unofficial].
    $$ w'_i = w_i + \epsilon $$

2.  **Compute PDF:** Normalize the stabilized weights along the last dimension (bin dimension) to obtain the Probability Density Function (PDF) [S3, S6, S8, unofficial].
    $$ \text{pdf}_i = \frac{w'_i}{\sum_{j} w'_j} $$
    Where the sum is over all bins for each ray [S1, S6, unofficial].

3.  **Compute CDF:** Compute the cumulative sum of the PDF and prepend a zero tensor to align indices with bin boundaries [S4, S7, S10, unofficial].
    $$ \text{cdf}_i = \sum_{k=0}^{i} \text{pdf}_k $$
    $$ \text{CDF} = [0, \text{cdf}_0, \text{cdf}_1, \dots, \text{cdf}_{N-1}] $$
    The resulting CDF has shape `(B, N_bins)` if `bins` has length `N_bins` and `weights` has length `N_bins - 1` [S7, unofficial].

4.  **Generate Query Values ($u$):** Generate $N_{samples}$ uniform random numbers $u \in [0, 1]$ for each ray [S3, S7, unofficial].
    *   If `det` is `False`: $u \sim \text{Uniform}(0, 1)$ [S1, S5, unofficial].
    *   If `det` is `True`: $u$ is linearly spaced between 0 and 1, e.g., $u_k = \frac{k}{N_{samples}-1}$ or stratified midpoints [S1, S5, unofficial].

5.  **Find Bin Indices:** Use binary search (`searchsorted`) to find the index in the CDF where each $u$ falls [S4, S7, S10, unofficial].
    $$ \text{inds} = \text{searchsorted}(\text{CDF}, u, \text{side}=\text{'right'}) $$
    This returns indices $i$ such that $\text{CDF}_{i-1} \le u < \text{CDF}_i$ [S4, unofficial].

6.  **Determine Interpolation Bounds:** Identify the lower (`below`) and upper (`above`) bin indices for linear interpolation, clamping to valid ranges [S4, S5, S8, unofficial].
    $$ \text{below} = \max(0, \text{inds} - 1) $$
    $$ \text{above} = \min(N_{bins} - 1, \text{inds}) $$
    Stack these into `inds_g` of shape `(B, N_samples, 2)` [S3, S9, unofficial].

7.  **Gather CDF and Bin Values:** Retrieve the CDF values and bin boundaries corresponding to the lower and upper indices [S4, S5, S8, unofficial].
    $$ \text{cdf}_g = \text{gather}(\text{CDF}, \text{inds}_g) $$
    $$ \text{bins}_g = \text{gather}(\text{bins}, \text{inds}_g) $$
    Here, $\text{bins}_g[..., 0]$ is the lower bound $z_{lower}$ and $\text{bins}_g[..., 1]$ is the upper bound $z_{upper}$ [S9, unofficial].

8.  **Interpolate Sample Positions:** Compute the interpolation factor $t$ and map it to the depth range [S4, S9, unofficial].
    $$ \text{denom} = \text{cdf}_g[..., 1] - \text{cdf}_g[..., 0] $$
    Handle zero denominators (flat CDF regions) by clamping:
    $$ \text{denom} = \begin{cases} 1 & \text{if } \text{denom} < \epsilon \\ \text{denom} & \text{otherwise} \end{cases} $$
    $$ t = \frac{u - \text{cdf}_g[..., 0]}{\text{denom}} $$
    $$ z_{\text{samples}} = \text{bins}_g[..., 0] + t \times (\text{bins}_g[..., 1] - \text{bins}_g[..., 0]) $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def hierarchical_sampling(bins, weights, N_samples, det=False, eps=1e-5):
    """
    Perform inverse CDF sampling to generate fine samples.
    
    Args:
        bins: Tensor of shape (N_bins,) containing bin boundaries.
        weights: Tensor of shape (B, N_bins - 1) containing bin weights.
        N_samples: int, number of fine samples to generate.
        det: bool, if True use deterministic sampling, else random.
        eps: float, epsilon for numerical stability.
        
    Returns:
        z_samples: Tensor of shape (B, N_samples) with new depth values.
    """
    # 1. Stabilize weights
    weights = weights + eps
    
    # 2. Compute PDF
    pdf = weights / torch.sum(weights, dim=-1, keepdim=True)
    
    # 3. Compute CDF and prepend zero
    cdf = torch.cumsum(pdf, dim=-1)
    cdf = torch.cat([torch.zeros_like(cdf[..., :1]), cdf], dim=-1)
    
    # 4. Generate query values u
    if det:
        # Deterministic: linearly spaced
        u = torch.linspace(0.0, 1.0, steps=N_samples, device=weights.device)
        u = u.expand(list(cdf.shape[:-1]) + [N_samples])
    else:
        # Random: uniform stratified
        u = torch.rand(list(cdf.shape[:-1]) + [N_samples], device=weights.device)
        
    # 5. Find bin indices
    inds = torch.searchsorted(cdf, u, right=True)
    
    # 6. Determine interpolation bounds
    below = torch.clamp(inds - 1, min=0)
    above = torch.clamp(inds, max=cdf.shape[-1] - 1)
    
    # Stack indices for gathering: shape (B, N_samples, 2)
    inds_g = torch.stack([below, above], dim=-1)
    
    # 7. Gather CDF and bin values
    # Expand inds_g to match gather dimensions if necessary
    # cdf shape: (B, N_bins), inds_g shape: (B, N_samples, 2)
    # We need to gather along the last dimension of cdf
    cdf_g = torch.gather(cdf.unsqueeze(1).expand(-1, N_samples, -1), 2, inds_g)
    
    # bins shape: (N_bins,), need to expand to (B, N_samples, N_bins) or gather carefully
    # Assuming bins is 1D, we gather from it directly after expanding or using advanced indexing
    # Standard approach: gather from bins expanded to match cdf_g structure
    bins_expanded = bins.unsqueeze(0).unsqueeze(0).expand(cdf_g.shape[0], cdf_g.shape[1], -1)
    bins_g = torch.gather(bins_expanded, 2, inds_g)
    
    # 8. Interpolate
    denom = cdf_g[..., 1] - cdf_g[..., 0]
    # Clamp denominator to avoid division by zero
    denom = torch.where(denom < eps, torch.ones_like(denom), denom)
    
    t = (u - cdf_g[..., 0]) / denom
    z_samples = bins_g[..., 0] + t * (bins_g[..., 1] - bins_g[..., 0])
    
    return z_samples
```

**Parameters:**
*   `eps`: $10^{-5}$ is typical to prevent NaNs in PDF normalization and interpolation [S2, S9, unofficial].
*   `N_samples`: Typically 128 for fine samples in NeRF, while coarse is 64 [S3, S4, unofficial].
*   `det`: `False` is standard for training to introduce jitter; `True` for testing or reproducibility [S1, S5, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Weights:** If all weights in a bin are zero, normalization fails. Adding $\epsilon$ prevents this [S1, S9, unofficial].
*   **Zero-Width Bins:** If `bins_g[..., 1] == bins_g[..., 0]`, the denominator in interpolation is zero. Clamping `denom` to 1 (or $\epsilon$) and returning the lower bound is a safe fallback [S1, S9, unofficial].
*   **Out-of-Bounds Indices:** `searchsorted` might return indices equal to the length of CDF if $u=1$. Clamping `above` to `N_bins - 1` prevents this [S4, S8, unofficial].

## Subtopics

### [Inverse Opacity Calculation](1099-inverse-opacity-calculation.md)
Inverse Opacity Calculation is a differentiable method for generating importance samples along a ray by inverting the cumulative distribution function (CDF) of the rendering contribution. It solves the problem of finding the ray parameter $t$ corresponding to a target cumulative density value $y$, allowing samples to be drawn directly from transmittance estimates rather than computing expensive integrals. Unlike standard NeRF hierarchical sampling, which interpolates exponentiated densities (probabilities) and requires auxiliary losses, this method interpolates the density field directly (piecewise linear or constant), offering better numerical stability for end-to-end optimization. It enables Monte Carlo estimates with fewer ray points and allows gradients to back-propagate through the sampling process to the density field parameters.

### [Proposal Network Sampling](1589-proposal-network-sampling.md)
Proposal Network Sampling is a method for importance sampling in volume rendering that directly estimates the Probability Density Function (PDF) along a ray using a coarse neural network, known as the proposal network. It solves the problem of inefficient uniform sampling by concentrating samples on high-contribution areas (such as surfaces) while spreading them in empty or occluded regions, providing more accurate transmittance estimation than spatial skipping methods. It builds on [Hierarchical Sampling (Inverse CDF Sampling)](973-hierarchical-sampling-inverse-cdf-sampling.md) by learning the sampling distribution dynamically rather than relying on static weights [G].

### [RangeFineSampler](3141-rangefinesampler.md)
RangeFineSampler is a hierarchical sampling technique for direct volume ray casting that refines coarse samples by generating a dense uniform grid around the coarse sample with the highest weight, supplemented by random samples across the full ray extent. It addresses missing high-frequency details between coarse samples by concentrating computational effort near the most significant signal while maintaining global coverage through random jittering. Unlike [Inverse Opacity Calculation](1099-inverse-opacity-calculation.md) which derives samples from opacity gradients, or [Proposal Network Sampling](1589-proposal-network-sampling.md) which uses learned neural networks to predict sample locations, RangeFineSampler relies on a deterministic geometric expansion around the peak weight index combined with stochastic uniform sampling.

### [Discrete Distribution Sampler](3906-discrete-distribution-sampler.md)
The Discrete Distribution Sampler selects indices from a finite set of choices based on associated probability weights, using a uniform random number to drive the selection via cumulative distribution function (CDF) traversal. It serves as the core sampling primitive for Hierarchical Sampling (Inverse CDF Sampling), enabling efficient point selection along rays in volume rendering by prioritizing bins with higher density or opacity. Unlike [Inverse Opacity Calculation](1099-inverse-opacity-calculation.md) which computes transmittance, or [Proposal Network Sampling](1589-proposal-network-sampling.md) which uses learned neural networks, this method relies on explicit weight arrays and supports both stochastic and deterministic top-k selection modes.

### Inverse Sampling for Fine Rays
Inverse Sampling for Fine Rays is a specific application of hierarchical sampling where the `FineSampler` inherits from a base `RaySampler` and uses coarse samples and weights to perform inverse sampling [S5, unofficial]. The base class `sample` method concatenates `additional_depths` to `z_vals` if provided, sorts them, and computes points via `points = (cam_loc.unsqueeze(1) + z_vals.unsqueeze(2) * ray_dirs.unsqueeze(1))` [S5, unofficial]. This variant focuses on the integration of the sampled depths into the final point cloud generation step within a rendering pipeline [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \log M)$ per ray, where $N$ is the number of fine samples and $M$ is the number of coarse bins, dominated by the `searchsorted` operation [S4, S7, unofficial]. The PDF/CDF computation is $O(M)$.
*   **Memory Complexity:** $O(M)$ per ray to store the CDF and intermediate tensors [S7, unofficial].
*   **Measured Performance:** In NeRF, using 64 coarse and 128 fine samples with hierarchical sampling significantly reduces PSNR degradation compared to uniform sampling with the same total sample count, effectively doubling the quality for the same computational budget [S3, S4, unofficial].
*   **Strengths:** Concentrates samples in high-density regions, improving reconstruction quality of thin structures and surfaces [S1, S3, unofficial]. Differentiable, allowing gradient flow to the coarse network [S1, unofficial].
*   **Limitations:** Requires a coarse pass first, adding latency [S4, unofficial]. Performance depends on the quality of the coarse weights; poor coarse estimates lead to poor fine sampling [S1, unofficial].
*   **When to Use:** When computational resources are limited but high-quality reconstruction of complex geometry is needed [S3, unofficial].
*   **When Not to Use:** When real-time performance is critical and a single pass is required, or when the scene is uniformly dense [G].

## Background
Hierarchical Sampling was introduced in the seminal NeRF paper "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" by Mildenhall et al. (2020) [S2, S9, unofficial]. It was developed to address the inefficiency of uniform sampling in neural radiance fields, where most samples fall in empty space. The method has since become a standard component in many NeRF variants and volume rendering pipelines [S1, S3, S4, unofficial]. It is widely used in academic research and open-source implementations such as the official NeRF repository and various "NeRF from Scratch" tutorials [S2, S3, S4, unofficial].

## Sources
[S1] GreatDrake/reparameterized-volume-sampling, https://github.com/GreatDrake/reparameterized-volume-sampling, fetched 2026-10-26 (unofficial)
[S2] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S3] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S4] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S5] albertgassol1/vf-nerf, https://github.com/albertgassol1/vf-nerf, fetched 2026-10-26 (unofficial)
[S6] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
[S7] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
[S8] InanXR/FLASH, https://github.com/InanXR/FLASH, fetched 2026-10-27 (unofficial)
[S9] NeRF: A Volume Rendering Perspective | Yue Yu - GitHub Pages, https://yconquesty.github.io/blog/ml/nerf/nerf_rendering.html, fetched 2026-10-29 (unofficial)
[S10] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
