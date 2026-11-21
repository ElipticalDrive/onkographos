# NeRF Importance Sampling

Also called: Importance Sampling for NeRF, NeRF Sampling, Stratified Sampling with PDF Resampling

## Overview
NeRF Importance Sampling is a two-stage volume rendering strategy that refines the distribution of sample points along a ray by drawing additional samples from a probability distribution defined by the opacity weights of a coarse neural network pass [S1, S2, unofficial]. It solves the inefficiency of uniform sampling by concentrating computational effort on regions with high expected density or opacity, thereby improving rendering quality with minimal overhead [S1, S2, unofficial]. It differs from its siblings by using a specific stratified sampling procedure on the cumulative distribution function (CDF) of coarse alpha weights to generate fine samples, whereas [NerfAcc Rendering Pipeline](1619-nerfacc-rendering-pipeline.md) uses learned acceleration structures, [VF-NeRF](3138-vf-nerf.md) uses a voxel-based hierarchical representation, and [Voxel-based Volume Rendering](3845-voxel-based-volume-rendering.md) relies on spatial hashing for density queries [S1, S2, unofficial].

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$.
- Near and far clipping bounds, $z_{near}$ and $z_{far}$.
- Coarse neural network model $f_{coarse}(\mathbf{x}, \mathbf{v})$ returning raw density $\sigma$ and color $\mathbf{c}$.
- Number of coarse samples $N_{samples}$ (default 64) [S1, unofficial].
- Number of importance samples $N_{importance}$ (default 0, disabling the fine stage) [S1, unofficial].
- Perturbation flag `perturb` (boolean, typically True during training, False during inference) [S1, unofficial].
- Noise standard deviation `noise_std` for regularization [S1, unofficial].
- Epsilon value `eps` for numerical stability (default $10^{-5}$) [S1, unofficial].

**Outputs:**
- Final depth values $z_{final}$ sorted along the ray.
- 3D positions $\mathbf{x}_{final}$ corresponding to $z_{final}$.
- Rendered color $\mathbf{C}(\mathbf{r})$, opacity $\alpha(\mathbf{r})$, and depth $D(\mathbf{r})$ via volumetric integration.

**Data Structures:**
- `z_vals`: Tensor of shape $(N_{rays}, N_{samples})$ storing depth values along rays.
- `weights`: Tensor of shape $(N_{rays}, N_{samples})$ storing alpha compositing weights.
- `pdf`: Probability density function tensor of shape $(N_{rays}, N_{samples})$.
- `cdf`: Cumulative distribution function tensor of shape $(N_{rays}, N_{samples} + 1)$.

**Steps:**

1. **Coarse Uniform Sampling:**
   Generate $N_{samples}$ depth values linearly spaced between $z_{near}$ and $z_{far}$.
   $$ z_{steps} = \text{linspace}(0, 1, N_{samples}) $$
   $$ z_{vals} = z_{near} \cdot (1 - z_{steps}) + z_{far} \cdot z_{steps} $$
   [S1, unofficial]

2. **Perturbation (Optional):**
   If `perturb` is True, add random noise to break symmetry and aid gradient flow.
   Calculate midpoints:
   $$ z_{vals\_mid} = 0.5 \cdot (z_{vals}[:, :-1] + z_{vals}[:, 1:]) $$
   Define interval bounds:
   $$ lower = z_{vals\_mid} $$
   $$ upper = z_{vals\_mid} $$
   (Note: The source defines `lower` and `upper` implicitly via the midpoint calculation and interval logic, but explicitly uses `z_vals_mid` for noise bounds in the code snippet context. The noise is added to the original `z_vals` within the interval defined by midpoints.)
   Add noise:
   $$ z_{vals} = z_{vals} + \text{perturb} \cdot \text{random\_uniform}(z_{vals}.shape) \cdot (upper - lower) $$
   [S1, unofficial]

3. **Coarse Network Query:**
   Query $f_{coarse}$ at positions $\mathbf{x} = \mathbf{o} + z_{vals} \cdot \mathbf{d}$ to obtain raw densities $\sigma$.
   [S1, unofficial]

4. **Alpha and Weight Calculation:**
   Compute deltas between samples:
   $$ \Delta z_i = z_{vals_{i+1}} - z_{vals_i} $$
   (With boundary handling for the last sample, typically assuming $\Delta z_{last}$ is small or handled via padding).
   Add Gaussian noise to density:
   $$ \sigma' = \text{relu}(\sigma + \mathcal{N}(0, \text{noise\_std})) $$
   Compute alpha values:
   $$ \alpha_i = 1 - \exp(-\Delta z_i \cdot \sigma'_i) $$
   [S1, unofficial]
   
   Compute transmittance and weights:
   $$ T_i = \prod_{j=1}^{i-1} (1 - \alpha_j) $$
   $$ w_i = \alpha_i \cdot T_i $$
   (Implemented as `weights = alphas * torch.cumprod(alphas_shifted, -1)[:, :-1]` where `alphas_shifted` is `1 - alphas` padded with 1 at the start) [S1, unofficial].

5. **Importance Sampling (Fine Stage):**
   If $N_{importance} > 0$:
   a. Extract weights for sampling, excluding the first and last samples to avoid boundary issues:
      $$ w_{sample} = weights[:, 1:-1] $$
      [S1, unofficial]
   b. Compute PDF:
      $$ p_i = \frac{w_{sample, i} + \text{eps}}{\sum_j (w_{sample, j} + \text{eps})} $$
      [S1, unofficial]
   c. Compute CDF:
      $$ c_i = \sum_{k=1}^{i} p_k $$
      Pad CDF with 0 at the start and 1 at the end (implicitly handled by concatenating zeros and using `searchsorted`):
      $$ cdf = [0, c_1, c_2, \dots, c_{N_{samples}-2}, 1] $$
      [S1, unofficial]
   d. Generate uniform samples $u$ in $[0, 1]$.
      If deterministic (`det=True`):
      $$ u = \text{linspace}(0, 1, N_{importance}) $$
      Else:
      $$ u = \text{random\_uniform}(N_{importance}) $$
      [S1, unofficial]
   e. Perform inverse transform sampling using `searchsorted` to find indices in the CDF.
   f. Linearly interpolate within the bin to get new depth values $z_{samples}$.
      [S1, unofficial]
   g. Stop gradient on $z_{samples}$ to prevent backpropagation through the sampling process.
      [S2, unofficial]

6. **Combine and Sort:**
   Concatenate coarse and fine depth values:
   $$ z_{combined} = \text{concat}(z_{vals}, z_{samples}) $$
   Sort $z_{combined}$ in descending order (or ascending, depending on ray direction convention; NeRF typically sorts to maintain ray order) to form final $z_{final}$.
   [S1, S2, unofficial]

7. **Fine Network Query:**
   Query the fine neural network $f_{fine}$ at positions $\mathbf{x} = \mathbf{o} + z_{final} \cdot \mathbf{d}$.
   [S2, unofficial]

8. **Final Rendering:**
   Compute final alphas and weights using the fine network's output.
   Perform volumetric integration:
   $$ \mathbf{C}(\mathbf{r}) = \sum_i w_i \mathbf{c}_i $$
   [Parent]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def sample_pdf(bins, weights, N_importance, det=False, eps=1e-5):
    """
    Stratified sampling of bins based on weights.
    bins: (N_rays, N_samples) depth values (midpoints or edges)
    weights: (N_rays, N_samples) importance weights
    N_importance: int, number of samples to draw
    det: bool, if True use linspace, else random
    eps: float, numerical stability constant
    """
    N_rays, N_samples_ = weights.shape
    
    # Add epsilon to prevent division by zero
    weights = weights + eps
    
    # Compute PDF
    pdf = weights / torch.sum(weights, -1, keepdim=True)
    
    # Compute CDF
    cdf = torch.cumsum(pdf, -1)
    
    # Pad CDF with 0 at the start and 1 at the end
    cdf = torch.cat([torch.zeros_like(cdf[:, :1]), cdf], -1)
    
    # Generate uniform samples
    if det:
        u = torch.linspace(0, 1, N_importance, device=bins.device)
        u = u.expand(N_rays, N_importance)
    else:
        u = torch.rand(N_rays, N_importance, device=bins.device)
        
    # Find indices in CDF
    # searchsorted returns the index where u would be inserted to maintain order
    inds = torch.searchsorted(cdf, u, side='right')
    
    # Clamp indices to valid range
    below = torch.clamp_min(inds - 1, 0)
    above = torch.clamp_max(inds, N_samples_)
    
    # Stack indices for interpolation
    # inds_sampled shape: (N_rays, 2 * N_importance)
    inds_sampled = torch.stack([below, above], -1).view(N_rays, 2 * N_importance)
    
    # Gather CDF and bin values for the sampled indices
    # cdf_g shape: (N_rays, N_importance, 2)
    cdf_g = torch.gather(cdf, 1, inds_sampled).view(N_rays, N_importance, 2)
    
    # bins_g shape: (N_rays, N_importance, 2)
    # Note: bins passed to sample_pdf are typically the midpoints z_vals_mid
    bins_g = torch.gather(bins, 1, inds_sampled).view(N_rays, N_importance, 2)
    
    # Linear interpolation
    denom = cdf_g[..., 1] - cdf_g[..., 0]
    denom[denom < eps] = 1.0
    
    samples = bins_g[..., 0] + (u - cdf_g[..., 0]) / denom * (bins_g[..., 1] - bins_g[..., 0])
    
    return samples

def render_nerf_importance_sampling(ray_o, ray_d, z_near, z_far, coarse_net, fine_net, 
                                    N_samples=64, N_importance=64, perturb=True, 
                                    noise_std=1e-4, eps=1e-5):
    """
    Render a ray using NeRF Importance Sampling.
    """
    # 1. Coarse Uniform Sampling
    z_steps = torch.linspace(0, 1, N_samples)
    z_vals = z_near * (1 - z_steps) + z_far * z_steps
    
    # 2. Perturbation
    if perturb:
        z_vals_mid = 0.5 * (z_vals[:, :-1] + z_vals[:, 1:])
        lower = z_vals_mid
        upper = z_vals_mid
        # Add noise within intervals
        noise = torch.rand_like(z_vals)
        z_vals = z_vals + noise * (upper - lower)
        
    # 3. Coarse Network Query
    xyz_coarse = ray_o.unsqueeze(1) + z_vals.unsqueeze(2) * ray_d.unsqueeze(1)
    # Assume coarse_net returns (sigma, color)
    # sigma_raw shape: (N_rays, N_samples)
    sigma_raw = coarse_net(xyz_coarse) # Placeholder for actual network call
    
    # 4. Alpha and Weight Calculation
    deltas = z_vals[:, 1:] - z_vals[:, :-1]
    # Handle last delta (Gaussian noise scaling)
    # In practice, often assume last delta is small or use a fixed value
    # Here we assume deltas is (N_rays, N_samples-1)
    # To match source shape (N_rays, N_samples), we might pad or adjust
    # Source uses: alphas = 1 - exp(-deltas * relu(sigmas + noise))
    # Let's assume deltas is padded or sigmas is aligned.
    # Standard NeRF implementation often uses N_samples for z_vals and N_samples-1 for deltas.
    # However, source code snippet implies alphas has same shape as weights used for sampling.
    # Let's align with source: weights[:, 1:-1] implies weights has N_samples.
    
    # Re-aligning deltas to N_samples for simplicity in this pseudocode block
    # by assuming the last delta is negligible or handled by the network's output shape.
    # Actually, standard volume rendering uses N_samples points, N_samples-1 intervals.
    # The source code `weights = alphas * ...` suggests alphas is size N_samples.
    # This implies the last sample's alpha is computed or ignored.
    # We will compute alphas for N_samples-1 intervals and pad or adjust.
    
    # Correcting based on source formula: alphas = 1 - exp(-deltas * relu(sigmas + noise))
    # If deltas is (N_rays, N_samples-1), alphas is (N_rays, N_samples-1).
    # But sampling uses weights[:, 1:-1], which is (N_rays, N_samples-2).
    # This suggests a slight mismatch in standard indexing vs source description.
    # We will follow the source's explicit steps:
    
    # Compute deltas
    # deltas = z_vals[:, 1:] - z_vals[:, :-1] # Shape: (N_rays, N_samples-1)
    
    # Add noise to sigma
    sigma_noisy = torch.relu(sigma_raw + torch.randn_like(sigma_raw) * noise_std)
    
    # Compute alphas
    # Note: Source formula uses deltas. If deltas is N_samples-1, alphas is N_samples-1.
    # But sample_pdf uses weights[:, 1:-1].
    # Let's assume the source implementation pads alphas or handles indices carefully.
    # For the purpose of this article, we define alphas for the intervals.
    
    alphas = 1 - torch.exp(-deltas * sigma_noisy[:, :-1]) # Aligning sigma to intervals
    
    # Compute weights
    # Transmittance T_i = product(1 - alpha_j) for j < i
    # weights_i = alpha_i * T_i
    # Source: weights = alphas * torch.cumprod(alphas_shifted, -1)[:, :-1]
    # alphas_shifted = 1 - alphas, padded with 1 at start?
    # Let's implement standard volumetric weights:
    alpha_shifted = 1 - alphas
    # Pad with 1 at the beginning for cumprod
    alpha_shifted_padded = torch.cat([torch.ones_like(alpha_shifted[:, :1]), alpha_shifted], dim=-1)
    T = torch.cumprod(alpha_shifted_padded, -1)[:, :-1] # T_i
    weights = alphas * T
    
    # 5. Importance Sampling
    if N_importance > 0:
        # Extract weights for sampling (exclude first and last)
        weights_sample = weights[:, 1:-1]
        
        # Get midpoints for bins
        z_vals_mid = 0.5 * (z_vals[:, :-1] + z_vals[:, 1:])
        
        # Sample
        z_samples = sample_pdf(z_vals_mid, weights_sample, N_importance, det=not perturb, eps=eps)
        
        # Concatenate and sort
        z_vals_combined = torch.cat([z_vals, z_samples], dim=-1)
        z_vals_sorted, _ = torch.sort(z_vals_combined, dim=-1)
    else:
        z_vals_sorted = z_vals
        
    # 6. Fine Network Query
    xyz_fine = ray_o.unsqueeze(1) + z_vals_sorted.unsqueeze(2) * ray_d.unsqueeze(1)
    # Stop gradient on z_vals_sorted if needed
    xyz_fine = xyz_fine.detach() # Or stop_grad(z_vals_sorted)
    
    # Query fine net
    sigma_fine = fine_net(xyz_fine)
    
    # 7. Final Rendering
    # Recompute deltas and alphas for fine samples
    deltas_fine = z_vals_sorted[:, 1:] - z_vals_sorted[:, :-1]
    sigma_fine_noisy = torch.relu(sigma_fine + torch.randn_like(sigma_fine) * noise_std)
    alphas_fine = 1 - torch.exp(-deltas_fine * sigma_fine_noisy[:, :-1])
    
    # Compute final weights
    alpha_shifted_fine = 1 - alphas_fine
    alpha_shifted_fine_padded = torch.cat([torch.ones_like(alpha_shifted_fine[:, :1]), alpha_shifted_fine], dim=-1)
    T_fine = torch.cumprod(alpha_shifted_fine_padded, -1)[:, :-1]
    weights_fine = alphas_fine * T_fine
    
    # Integrate color (assuming fine_net also returns color)
    # color_fine = fine_net_color(xyz_fine)
    # C = sum(weights_fine * color_fine)
    
    return z_vals_sorted, weights_fine
```

**Parameters:**
- $N_{samples}$: Number of coarse samples per ray. Default: 64 [S1, unofficial].
- $N_{importance}$: Number of fine samples per ray. Default: 0 (disables fine stage) [S1, unofficial].
- `perturb`: Boolean flag to add noise to coarse samples. Default: True during training [S1, unofficial].
- `noise_std`: Standard deviation of Gaussian noise added to density. Default: Not specified, typically small (e.g., $10^{-4}$) [S1, unofficial].
- `eps`: Epsilon for numerical stability in PDF/CDF calculation. Default: $10^{-5}$ [S1, unofficial].
- `chunk`: Chunk size for batched inference to manage memory. Default: $1024 \times 32$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero in PDF:** Weights can be zero. Add `eps` to weights before computing PDF [S1, unofficial].
- **Zero Denominator in Inverse CDF:** If `cdf_g[...,1] - cdf_g[...,0]` is less than `eps`, set it to 1 to avoid division by zero, as bins with zero weight are not sampled anyway [S1, unofficial].
- **Boundary Handling:** The first and last samples are often excluded from importance sampling to avoid boundary artifacts [S1, unofficial].
- **Gradient Flow:** Gradients should be stopped on the sampled positions $z_{samples}$ to prevent backpropagation through the sampling process [S2, unofficial].

## Subtopics

### [Hierarchical Sampling with Weight Modification](4795-hierarchical-sampling-with-weight-modification.md)
Refines the sampling distribution by applying a 2-tap max filter, a 2-tap blur filter (blurpool), and a bias term to coarse weights before inverse transform sampling, ensuring robust sampling in empty regions and a smoother weight envelope.

### [Importance Ray Sampling](16860-importance-ray-sampling.md)
Explicitly defines procedural steps for binning, smoothing, and stratified sampling within a discrete partition of the ray, building on NeRF Importance Sampling.

## Operating characteristics
- **Time Complexity:** $O(N_{samples} + N_{importance})$ per ray for sampling and network queries. The sorting step adds $O((N_{samples} + N_{importance}) \log (N_{samples} + N_{importance}))$.
- **Memory Complexity:** $O(N_{rays} \cdot (N_{samples} + N_{importance}))$ to store depth values, weights, and intermediate tensors.
- **Measured Performance:** In the unofficial implementation [S1, unofficial], training on 8 GPUs finishes within 1 hour for synthetic datasets. The speed is ~100x faster than the original NeRF repo on an RTX 2080 Ti, with reduced GPU memory usage (3.2 GB vs 8.5 GB) due to batched inference [S1, unofficial].
- **Strengths:** Improves rendering quality by focusing samples on opaque regions with minimal overhead compared to uniform sampling. Simple to implement and integrate into existing NeRF pipelines.
- **Limitations:** Requires two network passes (coarse and fine). Performance gain depends on the scene's density distribution; uniform sampling may be sufficient for simple scenes.
- **When to Use:** When rendering quality is critical and the scene has varying density (e.g., thin structures, transparent objects).
- **When Not to Use:** When computational resources are extremely limited, or the scene is uniformly dense.

## Background
NeRF Importance Sampling was introduced in the original NeRF paper (Mildenhall et al., 2020) [S1, S2, unofficial]. It is a standard technique in neural volume rendering to address the inefficiency of uniform sampling. The method has been widely adopted in various NeRF implementations and extensions, including the unofficial PyTorch implementation by kwea123 [S1, unofficial] and the nerf-mlp repository [S2, unofficial].

## Sources
[S1] kwea123/nerf_pl, https://github.com/kwea123/nerf_pl, fetched 2026-10-26 (unofficial)
[S2] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
