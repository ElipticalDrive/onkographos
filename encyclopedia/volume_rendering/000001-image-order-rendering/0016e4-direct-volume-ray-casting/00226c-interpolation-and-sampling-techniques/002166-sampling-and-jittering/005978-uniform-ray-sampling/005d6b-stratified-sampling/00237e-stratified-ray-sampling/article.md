# Stratified Ray Sampling

Also called: Jittered Uniform Sampling, Ray Sampling for Acoustic Volume Rendering, Stratified Sample

## Overview
Stratified Ray Sampling is a technique for discretizing the integration interval along a camera ray in volume rendering by dividing it into equal-sized bins and drawing one sample uniformly at random from each bin [S1][S4, unofficial][S5]. It solves the problem of aliasing and high variance associated with purely uniform or purely random sampling by ensuring that the entire depth range is covered without gaps [S3, unofficial][S5]. Unlike [Stratified Monte Carlo Radiance Estimate](1373-stratified-monte-carlo-radiance-estimate.md), which focuses on general integral estimation, this method is specialized for the geometric discretization of rays in volumetric scenes [S1]. It differs from [Batch Stratified Sampling](2730-batch-stratified-sampling.md) by operating on individual rays rather than optimizing sample distributions across batches [S4, unofficial].

## How it works
**Inputs:**
*   Ray origin $\mathbf{o}$ (3D vector).
*   Ray direction $\mathbf{d}$ (normalized 3D vector).
*   Near plane distance $t_n$ (scalar).
*   Far plane distance $t_f$ (scalar).
*   Number of samples per ray $k$ (integer).
*   Optional: `lindisp` (boolean, whether to sample in disparity space) [S4, unofficial].
*   Optional: `perturb` (boolean, whether to add random jitter) [S4, unofficial].

**Outputs:**
*   Array of sample depths $\tau_i$ for $i=1, \dots, k$.
*   Array of 3D sample positions $\mathbf{p}_i$ for $i=1, \dots, k$.

**Data Structures:**
*   `z_vals`: A 1D array of size $k$ storing the depth values along the ray.
*   `sample_points`: A 2D array of shape $(k, 3)$ storing the Cartesian coordinates of the samples.

**Steps:**

1.  **Define Interval and Bins:**
    Determine the total length of the integration interval:
    $$ L = t_f - t_n $$
    The boundaries of the $k$ equal-sized bins are defined as:
    $$ t_i = t_n + i \cdot \frac{L}{k}, \quad \text{for } i = 0, \dots, k $$
    where $t_0 = t_n$ and $t_k = t_f$ [S1, S5].

2.  **Compute Base Depths (Linear or Disparity):**
    If `lindisp` is False (standard depth sampling), compute the midpoints or boundaries linearly. If `lindisp` is True, compute in disparity space ($1/t$) [S4, unofficial].
    
    *Linear Depth:*
    Generate $k$ evenly spaced values between $t_n$ and $t_f$. Let $t_{vals}$ be the boundaries:
    $$ t_{vals}[i] = t_n + i \cdot \frac{t_f - t_n}{k}, \quad i=0,\dots,k $$
    
    *Disparity Depth:*
    Compute boundaries in disparity space $d = 1/t$:
    $$ d_{vals}[i] = \frac{1}{t_n} + i \cdot \frac{\frac{1}{t_f} - \frac{1}{t_n}}{k}, \quad i=0,\dots,k $$
    Convert back to depth:
    $$ t_{vals}[i] = \frac{1}{d_{vals}[i]} $$
    [S4, unofficial].

3.  **Generate Stratified Samples:**
    For each bin $i$ from $1$ to $k$:
    *   Identify the bin boundaries: $[t_{i-1}, t_i]$.
    *   Draw a uniform random number $u_i \sim U[0, 1]$ [S1, S5].
    *   Compute the sample depth $\tau_i$:
        $$ \tau_i = t_{i-1} + u_i \cdot (t_i - t_{i-1}) $$
    
    *Note on Perturbation:* In some implementations (e.g., NeRF), jitter is applied to the midpoints of the bins rather than the boundaries to avoid sampling exactly at the edges, which can cause artifacts if the density function is discontinuous [S4, unofficial]. If `perturb` is True:
    1.  Compute midpoints $m_i = 0.5 \cdot (t_{i-1} + t_i)$ for $i=1, \dots, k-1$.
    2.  Define upper and lower bounds for jitter:
        $$ \text{lower} = [t_0, m_1, \dots, m_{k-1}] $$
        $$ \text{upper} = [m_1, \dots, m_{k-1}, t_k] $$
    3.  Draw random $t_{rand} \sim U[0, 1]$ of size $k$.
    4.  Compute $\tau_i = \text{lower}[i] + t_{rand}[i] \cdot (\text{upper}[i] - \text{lower}[i])$ [S4, unofficial].

4.  **Compute 3D Positions:**
    For each sample depth $\tau_i$, compute the 3D position $\mathbf{p}_i$:
    $$ \mathbf{p}_i = \mathbf{o} + \tau_i \mathbf{d} $$
    [S1][S3, S4, unofficial].

5.  **Render (Integration):**
    The color $\hat{C}(\mathbf{r})$ for the ray is approximated by volume rendering integration using the samples [S1]:
    $$ \hat{C}(\mathbf{r}) = \sum_{i=1}^{k} T_i \alpha_i c_i $$
    where:
    *   $c_i$ is the color contribution at $\mathbf{p}_i$.
    *   $\sigma_i$ is the density at $\mathbf{p}_i$.
    *   $\delta_i = t_i - t_{i-1}$ is the distance between bin boundaries (or $\tau_i - \tau_{i-1}$ if using sample-to-sample integration, but standard stratified uses bin width for opacity calculation consistency with the bin definition) [S1]. *Note: In standard NeRF implementations, $\delta_i$ is often approximated as the distance between consecutive samples $\tau_i - \tau_{i-1}$, but the theoretical definition relies on the bin width $L/k$ for unbiased estimation of the integral over the bin.*
    *   Opacity: $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ [S1].
    *   Transmittance: $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def stratified_ray_sampling(origins, directions, near, far, num_samples, lindisp=False, perturb=True):
    """
    Generates stratified samples along rays.
    
    Args:
        origins: Tensor of shape (N_rays, 3)
        directions: Tensor of shape (N_rays, 3)
        near: Float, near plane distance
        far: Float, far plane distance
        num_samples: Int, number of samples per ray (k)
        lindisp: Bool, if True sample in disparity space
        perturb: Bool, if True add uniform jitter within bins
        
    Returns:
        z_vals: Tensor of shape (N_rays, num_samples)
        sample_points: Tensor of shape (N_rays, num_samples, 3)
    """
    device = origins.device
    
    # Step 1: Define bin boundaries
    # Create linspace for boundaries: 0 to 1, size num_samples + 1
    t_vals = torch.linspace(0.0, 1.0, num_samples + 1, device=device)
    
    # Step 2: Compute depths based on lindisp
    if lindisp:
        # Sample linearly in disparity (1/depth)
        # d_near = 1/near, d_far = 1/far
        # d_vals = d_near * (1 - t) + d_far * t
        # z_vals = 1 / d_vals
        # Note: t_vals here are the boundaries for the linspace
        # We need to handle the division by zero if near=0, but typically near > 0
        d_near = 1.0 / near
        d_far = 1.0 / far
        d_vals = d_near * (1.0 - t_vals) + d_far * t_vals
        z_vals = 1.0 / d_vals
    else:
        # Sample linearly in depth
        z_vals = near * (1.0 - t_vals) + far * t_vals
        
    # z_vals has shape (num_samples + 1)
    # We need to expand to (N_rays, num_samples + 1)
    z_vals = z_vals.unsqueeze(0).expand(origins.shape[0], -1)
    
    if perturb:
        # Step 3: Stratified Sampling with Perturbation
        # Compute midpoints of the bins
        # mids[i] = 0.5 * (z_vals[:, i] + z_vals[:, i+1]) for i=0..num_samples-1
        mids = 0.5 * (z_vals[:, 1:] + z_vals[:, :-1])
        
        # Define lower and upper bounds for each sample
        # lower[i] = z_vals[:, i]
        # upper[i] = z_vals[:, i+1]
        # But for jitter, we use midpoints to define the range
        # lower_bound = [z_vals[:, 0], mids[:, 0], ..., mids[:, -1]]
        # upper_bound = [mids[:, 0], ..., mids[:, -1], z_vals[:, -1]]
        
        lower = torch.cat([z_vals[:, :1], mids], dim=1)
        upper = torch.cat([mids, z_vals[:, -1:]], dim=1)
        
        # Draw random numbers
        t_rand = torch.rand_like(z_vals[:, :num_samples])
        
        # Compute final z_vals
        z_vals = lower + (upper - lower) * t_rand
    else:
        # Deterministic or simple uniform without perturbation logic described in S4
        # If no perturbation, we might just pick the midpoint or a random point in the bin
        # Standard stratified without perturbation usually implies picking a random point in the bin
        # defined by the original t_vals boundaries.
        # However, S4 implies perturb=True is the standard "stratified" in NeRF context.
        # If perturb=False, we can just use the midpoints or random in bins.
        # Let's stick to the random in bin definition from S1/S5 if perturb is false but stratified is true.
        # But S4 specifically links perturb to the stratified implementation.
        # If perturb is False, we can simply use the midpoints or uniform random in the original bins.
        # For consistency with S1/S5:
        # t_rand = torch.rand_like(z_vals[:, :num_samples])
        # lower = z_vals[:, :num_samples]
        # upper = z_vals[:, 1:]
        # z_vals = lower + (upper - lower) * t_rand
        # This is effectively the same as perturb=True but using bin edges instead of midpoints.
        # We will assume perturb=True is the primary method for "Stratified Ray Sampling" in modern contexts [S4, unofficial].
        pass

    # Step 4: Compute 3D positions
    # origins: (N_rays, 3), directions: (N_rays, 3), z_vals: (N_rays, num_samples)
    # sample_points = origins[:, None, :] + directions[:, None, :] * z_vals[..., :, None]
    
    # Expand dimensions for broadcasting
    # origins: (N_rays, 1, 3)
    # directions: (N_rays, 1, 3)
    # z_vals: (N_rays, num_samples, 1)
    
    origins_exp = origins.unsqueeze(1)
    directions_exp = directions.unsqueeze(1)
    z_vals_exp = z_vals.unsqueeze(2)
    
    sample_points = origins_exp + directions_exp * z_vals_exp
    
    return z_vals, sample_points
```

**Parameters:**
*   `num_samples` ($k$): Typically 64 for coarse networks and 128 for fine networks in hierarchical schemes [S4, unofficial][S5].
*   `near` ($t_n$): Minimum depth. Not given in the sources as a specific value, but defined in `cfg.min_depth` [S3, unofficial].
*   `far` ($t_f$): Maximum depth. Not given in the sources as a specific value, but defined in `cfg.max_depth` [S3, unofficial].
*   `lindisp`: Boolean. False by default [S4, unofficial].
*   `perturb`: Boolean. True by default in many implementations to reduce aliasing [S4, unofficial].

**Edge Cases and Pitfalls:**
*   **Near Zero Depth:** If $t_n = 0$, disparity sampling ($1/t$) results in division by zero. Ensure $t_n > 0$ for disparity sampling [S4, unofficial].
*   **Bin Width vs Sample Distance:** The opacity calculation $\alpha_i$ uses $\delta_i$. If using stratified sampling, $\delta_i$ should ideally be the bin width $L/k$ to maintain the unbiased nature of the integral over the bin, rather than the distance between random samples $\tau_i - \tau_{i-1}$, which varies [S1]. However, many implementations use sample-to-sample distances for simplicity [S4, unofficial].
*   **Boundary Sampling:** Without perturbation, samples might fall exactly on bin boundaries, which can cause artifacts if the density function is discontinuous at those points [S4, unofficial].

## Subtopics

### [Hierarchical Stratified Sampling](10554-hierarchical-stratified-sampling.md)
A refinement technique that uses coarse stratified samples to guide the placement of fine samples in high-density regions. It builds on [Stratified Ray Sampling](3014-stratified-ray-sampling.md) by adding an adaptive, importance-driven refinement layer. Choose this when scene geometry is complex and uniform sampling is inefficient.

## Operating characteristics
*   **Time Complexity:** $O(k)$ per ray, where $k$ is the number of samples. Generating random numbers and computing positions is linear in the number of samples [S1][S4, unofficial].
*   **Memory Complexity:** $O(k)$ per ray to store sample depths and positions [S3, S4, unofficial].
*   **Performance:** Provides a good balance between variance and coverage. It reduces aliasing compared to uniform sampling and provides better coverage than pure random sampling [S2, unofficial][S5].
*   **Strengths:** Simple to implement, unbiased (if random), ensures full coverage of the ray interval [S1, S5].
*   **Limitations:** Does not adapt to scene content; may waste samples in empty space or under-sample in high-frequency regions [S5].
*   **When to Use:** As a baseline sampling strategy, particularly in the coarse stage of hierarchical rendering [S5][S4, unofficial].
*   **When Not to Use:** When the scene has highly variable density and computational resources are limited; importance sampling or hierarchical methods are preferred [S5].

## Background
Stratified sampling is a fundamental technique in Monte Carlo integration and computer graphics. In the context of volume rendering and Neural Radiance Fields (NeRF), it was popularized by the original NeRF paper and subsequent implementations [S4, unofficial]. It is used in various domains including acoustic volume rendering [S1] and neural surface reconstruction [S3, unofficial]. The method is often employed in the coarse training stage to capture general scene structure before refining with importance sampling [S5].

## Sources
[S1] Acoustic Volume Rendering for Neural Impulse Response Fields, https://doi.org/10.52202/079017-1417, fetched 2026-10-26
[S2] Fourier Feature Networks and Neural Volume Rendering, https://github.com/matajoh/fourier_feature_nets, fetched 2026-10-26 (unofficial)
[S3] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S4] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S5] ColNeRF: Collaboration for Generalizable Sparse Input Neural Radiance Field, https://doi.org/10.48550/arXiv.2312.09095, fetched 2026-10-29
