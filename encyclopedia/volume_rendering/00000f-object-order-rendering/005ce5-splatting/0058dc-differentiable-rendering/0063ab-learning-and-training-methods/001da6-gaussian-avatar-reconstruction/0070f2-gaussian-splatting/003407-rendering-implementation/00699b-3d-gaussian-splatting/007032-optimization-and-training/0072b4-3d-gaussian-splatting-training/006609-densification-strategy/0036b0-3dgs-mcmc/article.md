# 3DGS-MCMC

Also called: 3D Gaussian Splatting as Markov Chain Monte Carlo, MCMc Densification Strategy, MCMC-based Gaussian Lifecycle, Markov Chain Monte Carlo Strategy

## Overview
3DGS-MCMC replaces the standard heuristic-based densification (gradient magnitude thresholds) of 3D Gaussian Splatting with a probabilistic framework based on Markov Chain Monte Carlo (MCMC) sampling [S1, unofficial][S4]. It treats the set of 3D Gaussians as samples from an underlying probability distribution, using Metropolis–Hastings steps to guide the spawning and pruning of primitives [S1, unofficial]. This approach allows for precise control over the total number of Gaussians, reduces reliance on high-quality initialization, and yields sharper details and fewer artifacts compared to the original cloning/splitting heuristics [S2, unofficial][S4].

## How it works
**Inputs and Outputs:**
*   **Input:** Current set of $N$ Gaussians $\mathcal{G} = \{g_1, \dots, g_N\}$, where each $g_k$ has parameters $\mu_k, \Sigma_k, \alpha_k, \mathbf{c}_k$; current training iteration $t$; rendering loss $\mathcal{L}$; and configuration hyperparameters [S4].
*   **Output:** Updated set of Gaussians $\mathcal{G}'$ with modified parameters and potentially different cardinality $N'$ [S4].

**Data Structures:**
*   **Gaussian Parameters:** Stored in contiguous tensors or arrays [G].
    *   `means`: $[N, 3]$ float32.
    *   `scales`: $[N, 3]$ float32 (log-space storage is common in implementations) [G].
    *   `quats`: $[N, 4]$ float32 (normalized quaternions) [G].
    *   `opacities`: $[N]$ float32 (logit-space storage is common) [G].
    *   `sh_coeffs`: $[N, K, 3]$ or similar, depending on SH degree [G].
*   **Optimizer States:** Separate buffers for gradients and optimizer momentum for each parameter group [G].
*   **Binomial Table:** A precomputed table `binoms` of size $51 \times 51$ where `binoms[n, k] = \binom{n}{k}` [S3, unofficial].

**Parameters:**
*   `cap_max`: Maximum number of Gaussians allowed. Default: $1,000,000$ [S3, unofficial].
*   `refine_start_iter`: Iteration number after which Gaussian refinement begins. Default: $500$ [S3, unofficial].
*   `refine_stop_iter`: Iteration number after which Gaussian refinement stops. Default: $25,000$ [S3, unofficial].
*   `refine_every`: Frequency of refinement steps in iterations. Default: $100$ [S3, unofficial].
*   `min_opacity`: Threshold below which Gaussians are considered for pruning or relocation. Default: $0.005$ [S3, unofficial].
*   `noise_lr`: MCMC sampling noise learning rate. Default: $5 \times 10^{-5}$ [S3, unofficial].

**Algorithm Steps:**
The method is executed periodically every `refine_every` iterations, starting at `refine_start_iter` and stopping at `refine_stop_iter` [S3, unofficial].

1.  **Initialization:** Ensure optimizer states are synchronized with parameters. Sanity check that each optimizer has one param_group and params contain keys 'means', 'scales', 'quats', 'opacities' [S3, unofficial].
2.  **Sample Add (Spawn):**
    *   Calculate current opacities $\alpha = \text{sigmoid}(\text{params['opacities']})$ [S3, unofficial].
    *   Sample $n$ new source indices from the existing population using a multinomial distribution weighted by $\alpha$ [S3, unofficial].
    *   Compute new opacities and scales for these samples using a `compute_relocation` function (see below) [S3, unofficial].
    *   Clamp new opacities to $[\text{min\_opacity}, 1-\epsilon]$ [S3, unofficial].
    *   Append copies of the sampled Gaussians (with updated opacities/scales) to the parameter arrays [S3, unofficial].
    *   Initialize new optimizer states to zero [S3, unofficial].
3.  **Relocation (Prune/Move):**
    *   Identify "dead" Gaussians (mask=True) and "alive" Gaussians (mask=False) [S3, unofficial].
    *   Sample new source indices from alive Gaussians using a multinomial distribution weighted by $\text{sigmoid}(\text{opacities})$ [S3, unofficial].
    *   Compute new opacities and scales for the relocated Gaussians using `compute_relocation` with the sampled opacities, $\exp(\text{scales})$, ratios (bincount of sampled indices + 1), and the `binoms` table [S3, unofficial].
    *   Clamp new opacities to $[\text{min\_opacity}, 1-\epsilon]$ [S3, unofficial].
    *   Update parameters: Set opacities at sampled indices to $\text{logit}(\text{new\_opacities})$, scales to $\log(\text{new\_scales})$, and copy these values to the dead indices [S3, unofficial].
    *   Reset optimizer states at the sampled indices [S3, unofficial].
4.  **Inject Noise (Perturbation):**
    *   Calculate current opacities $\alpha$ and scales $s = \exp(\text{params['scales']})$ [S3, unofficial].
    *   Compute covariance matrices $\Sigma$ from quats and scales [S3, unofficial].
    *   Define a sigmoid function for noise scaling: $\text{op\_sigmoid}(x, k=100, x_0=0.995) = \frac{1}{1 + \exp(-k(x - x_0))}$ [S3, unofficial].
    *   Generate noise: $\text{noise} = \mathcal{N}(0, I) \cdot \text{op\_sigmoid}(1 - \alpha) \cdot \text{scaler}$ [S3, unofficial].
    *   Transform noise by covariance: $\text{noise}_{\text{cov}} = \Sigma \cdot \text{noise}$ [S3, unofficial].
    *   Update means: $\mu \leftarrow \mu + \text{noise}_{\text{cov}}$ [S3, unofficial].

**Formulas:**

*   **Metropolis–Hastings Spawn Probability:**
    The probability of spawning a new Gaussian at $\mu_{\text{new}}$ is determined by the ratio of rendering errors and proposal distributions [S1, unofficial]:
    $$
    \alpha_{\text{spawn}} = \min \left( 1, \frac{\mathcal{L}(\boldsymbol{\mu}_{\text{new}})}{\mathcal{L}(\boldsymbol{\mu}_{\text{old}})} \cdot \frac{q(\boldsymbol{\mu}_{\text{old}}|\boldsymbol{\mu}_{\text{new}})}{q(\boldsymbol{\mu}_{\text{new}}|\boldsymbol{\mu}_{\text{old}})} \right)
    $$
    where $\mathcal{L}$ is the rendering loss and $q$ is the proposal distribution [S1, unofficial].

*   **Relocation Computation:**
    The specific formula for `compute_relocation` is not explicitly detailed in the provided sources, but it utilizes the binomial coefficient table `binoms` and ratios derived from the multinomial sampling counts to adjust opacities and scales [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def compute_binoms_table(max_n=50):
    """Precompute binomial coefficients."""
    binoms = torch.zeros(max_n + 1, max_n + 1, dtype=torch.float32)
    for n in range(max_n + 1):
        for k in range(n + 1):
            binoms[n, k] = math.comb(n, k)
    return binoms

def op_sigmoid(x, k=100, x0=0.995):
    """Sigmoid function for noise scaling."""
    return 1.0 / (1.0 + torch.exp(-k * (x - x0)))

def compute_covariance(quats, scales):
    """Compute 3x3 covariance matrices from quaternions and scales [G]."""
    qw, qx, qy, qz = quats[:, 0], quats[:, 1], quats[:, 2], quats[:, 3]
    r11 = 1 - 2*(qy**2 + qz**2); r12 = 2*(qx*qy - qw*qz); r13 = 2*(qx*qz + qw*qy)
    r21 = 2*(qx*qy + qw*qz); r22 = 1 - 2*(qx**2 + qz**2); r23 = 2*(qy*qz - qw*qx)
    r31 = 2*(qx*qz - qw*qy); r32 = 2*(qy*qz + qw*qx); r33 = 1 - 2*(qx**2 + qy**2)
    R = torch.stack([r11, r12, r13, r21, r22, r23, r31, r32, r33], dim=-1).reshape(-1, 3, 3)
    S = torch.diag_embed(scales**2)
    return torch.bmm(R, torch.bmm(S, R.transpose(1, 2)))

def compute_relocation(sampled_opacities, sampled_scales, binoms, ratios):
    """
    Compute new opacities and scales for relocated/sampled Gaussians.
    Sources do not give the exact mathematical formula for compute_relocation.
    It uses the binomial table and sampling ratios to adjust opacities and scales [S3, unofficial].
    """
    scale_factor = ratios.float() / ratios.max()
    new_opacities = sampled_opacities * scale_factor.unsqueeze(-1)
    new_scales = sampled_scales * scale_factor.unsqueeze(-1)
    return new_opacities, new_scales

def mcmc_densification_step(
    params, 
    optimizers, 
    binoms, 
    cap_max, 
    min_opacity, 
    refine_every, 
    current_iter,
    refine_start_iter,
    refine_stop_iter,
    n_add
):
    """
    Perform one MCMC densification step.
    params: dict with keys 'means', 'scales', 'quats', 'opacities' (tensors of shape [N, ...])
    optimizers: dict of optimizer states
    n_add: Number of Gaussians to spawn per step [G]
    """
    # 1. Check if refinement should occur
    if current_iter < refine_start_iter or current_iter > refine_stop_iter:
        return
    if current_iter % refine_every != 0:
        return

    # 2. Extract current parameters
    means = params['means']
    scales_log = params['scales']
    quats = params['quats']
    opacities_log = params['opacities']
    
    N = means.shape[0]
    
    # Convert to physical space
    opacities = torch.sigmoid(opacities_log)
    scales = torch.exp(scales_log)
    
    # 3. Sample Add (Spawn new Gaussians)
    # Sample indices weighted by opacity
    weights = opacities / opacities.sum()
    source_indices = torch.multinomial(weights, n_add, replacement=True)
    
    # Compute new opacities and scales for new Gaussians
    ratios = torch.bincount(source_indices, minlength=N) + 1
    new_opacities, new_scales = compute_relocation(
        opacities[source_indices], 
        scales[source_indices], 
        binoms,
        ratios
    )
    
    # Clamp opacities
    new_opacities = torch.clamp(new_opacities, min=min_opacity, max=1.0 - 1e-6)
    new_opacities_log = torch.logit(new_opacities)
    new_scales_log = torch.log(new_scales)
    
    # Append to params
    new_means = means[source_indices].clone()
    new_quats = quats[source_indices].clone()
    
    params['means'] = torch.cat([means, new_means], dim=0)
    params['scales'] = torch.cat([scales_log, new_scales_log], dim=0)
    params['quats'] = torch.cat([quats, new_quats], dim=0)
    params['opacities'] = torch.cat([opacities_log, new_opacities_log], dim=0)
    
    # Initialize new optimizer states to zero [S3, unofficial]
    # (Implementation depends on specific optimizer structure)
    
    # 4. Relocation (Prune/Move low-opacity Gaussians)
    # Identify dead Gaussians (mask=True) and alive Gaussians (mask=False) [S3, unofficial]
    dead_mask = opacities < min_opacity
    alive_mask = ~dead_mask
    
    if dead_mask.any():
        # Sample from alive Gaussians to replace dead ones
        alive_weights = opacities[alive_mask] / opacities[alive_mask].sum()
        n_relocate = dead_mask.sum().item()
        alive_indices = torch.where(alive_mask)[0]
        sampled_alive_indices = alive_indices[torch.multinomial(alive_weights, n_relocate, replacement=True)]
        
        # Compute relocation parameters
        relocate_ratios = torch.bincount(sampled_alive_indices, minlength=N) + 1
        relocated_opacities, relocated_scales = compute_relocation(
            opacities[sampled_alive_indices],
            scales[sampled_alive_indices],
            binoms,
            relocate_ratios
        )
        
        # Clamp
        relocated_opacities = torch.clamp(relocated_opacities, min=min_opacity, max=1.0 - 1e-6)
        relocated_opacities_log = torch.logit(relocated_opacities)
        relocated_scales_log = torch.log(relocated_scales)
        
        # Update dead Gaussians with relocated parameters
        dead_indices = torch.where(dead_mask)[0]
        params['opacities'][dead_indices] = relocated_opacities_log
        params['scales'][dead_indices] = relocated_scales_log
        # Copy means, quats from sampled alive to dead [S3, unofficial]
        params['means'][dead_indices] = means[sampled_alive_indices]
        params['quats'][dead_indices] = quats[sampled_alive_indices]
        
        # Reset optimizer states for relocated indices [S3, unofficial]
        # (Implementation depends on specific optimizer structure)

    # 5. Inject Noise
    # Compute covariance matrices
    covars = compute_covariance(quats, scales)
    
    # Generate noise
    noise_scale = op_sigmoid(1.0 - opacities).unsqueeze(-1) * 5e-5 # noise_lr [S3, unofficial]
    noise = torch.randn_like(means) * noise_scale
    
    # Transform noise by covariance
    # einsum('bij,bj->bi', covars, noise)
    noise_transformed = torch.einsum('bij,bj->bi', covars, noise)
    
    # Update means
    params['means'].add_(noise_transformed)
```

## Subtopics

### [Optimization and Training Strategy](9157-optimization-and-training-strategy.md)
This strategy defines the optimization schedule, loss formulation, and regularization techniques for training radiance fields using 3D Gaussian Splatting with MCMC-based densification. It distinguishes itself from standard 3DGS by incorporating specific MCMC-derived regularization terms on opacity and scale, using cosine annealing for feature and MLP learning rates, applying Exponential Moving Average (EMA) to MLP weights, and executing a final refinement phase that freezes geometry. Choose it when precise control over Gaussian population and artifact reduction during iterative refinement is required.

## Operating characteristics
*   **Time Complexity:** The MCMC step involves sampling and tensor operations on $N$ Gaussians [S3, unofficial]. Sampling is $O(N)$ or $O(k \log N)$ depending on implementation [G]. The noise injection and covariance computation are $O(N)$ [S3, unofficial]. The overall overhead per refinement step is low compared to the rasterization cost, but it adds a periodic $O(N)$ pass [S3, unofficial].
*   **Memory Complexity:** Requires storage for the Gaussian parameters and optimizer states [G]. The binomial table is negligible ($51 \times 51$) [S3, unofficial]. No significant additional memory overhead compared to standard 3DGS [S3, unofficial].
*   **Performance:** Yields sharper details and better performance compared to the original 3DGS densification process, particularly in scenes with high-frequency details or underwater environments [S4]. It produces more efficient and artifact-free clouds by exploring regions with unknown gradients [S1, unofficial].
*   **Strengths:** Precise control over the number of Gaussians; less reliance on good initialization; reduces artifacts from noisy gradient heuristics [S2, unofficial][S4].
*   **Limitations:** Requires careful tuning of hyperparameters (`refine_start_iter`, `refine_stop_iter`, `min_opacity`); the exact formula for `compute_relocation` is not fully specified in the primary sources, requiring implementation inference [S3, unofficial].

## Background
3DGS-MCMC was introduced as an improvement to the original 3D Gaussian Splatting densification heuristic, which relied on noisy positional gradient magnitudes [S1, unofficial][S4]. It reinterprets the Gaussian cloud as samples from an underlying probability distribution and uses Markov Chain Monte Carlo sampling to guide the life cycle of primitives [S1, unofficial]. The strategy allows for precise control of the Gaussian count and serves as an alternative to default cloning and splitting [S2, unofficial]. It has been adopted in various implementations, including those for underwater rendering, where it helps capture high-frequency details [S4].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Better Gaussian Splatting in Julia, https://pxl-th.github.io/blog/better-gs-julia/, fetched 2026-10-26 (unofficial)
[S3] carlinds/splatad, https://github.com/carlinds/splatad, fetched 2026-10-27 (unofficial)
[S4] Gaussian splashing enables direct volumetric rendering underwater, https://doi.org/10.1038/s41598-026-56917-4, fetched 2026-10-27
