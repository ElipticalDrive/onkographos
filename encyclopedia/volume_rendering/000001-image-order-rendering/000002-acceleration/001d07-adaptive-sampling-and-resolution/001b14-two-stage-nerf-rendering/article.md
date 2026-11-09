# Two-stage NeRF Rendering

Also called: Coarse and Fine Sampling, Proposal Network Sampler, Rendering Sampling Strategy

## Overview
Two-stage NeRF Rendering is a volume rendering acceleration technique that decouples the estimation of scene density from the final color and density evaluation by splitting ray sampling into a coarse pass and a fine pass [S1]. It solves the inefficiency of uniform sampling, which wastes queries on empty space or redundant high-density regions, by using the coarse pass to learn a density distribution that guides the adaptive sampling of the fine pass [S1][S2, unofficial]. It sets itself apart from single-stage methods by explicitly separating the rough density estimation from the detailed evaluation, and from uniform sampling by concentrating computational effort on regions identified as likely to contain geometry [S2, unofficial][S3].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray bundle (origin $o$, direction $d$), near/far bounds ($t_n, t_f$), coarse network $N_c$, fine network $N_f$, and optional proposal networks $N_p$.
*   **Outputs:** Rendered pixel color $C$ and accumulated transmittance.

**Data Structures:**
*   **Ray Samples:** A list of 1D positions $t_i$ along the ray, sorted in ascending order.
*   **Network Inputs:** For each sample $t_i$, the 3D position $x_i = o + t_i d$ is encoded.
    *   *Positional Encoding:* Standard positional encoding $\gamma(x)$ is applied to $x_i$ [S1][S3].
    *   *Directional Encoding:* For color prediction, viewing direction $d$ is encoded via spherical harmonics or standard positional encoding [S1][S3].
*   **Network Outputs:**
    *   Coarse/Fine/Proposal Network: Outputs density $\sigma$ (often pre-activation or post-ReLU) and color $c$ (RGB) [S1][S3].

**Numbered Steps:**

1.  **Initialization:**
    *   If using a proposal network sampler, initialize the initial sampler. If `config.proposal_initial_sampler` is 'uniform', create a `UniformSampler` with single jittering; otherwise, initialize to `None` (defaulting to piecewise sampling) [S2, unofficial].
    *   Create the `ProposalNetworkSampler` instance with parameters: `num_proposal_network_iterations` (number of proposal networks), `num_proposal_samples_per_ray` (samples per proposal network), `num_nerf_samples_per_ray` (samples for main NeRF field), and `update_sched` (function determining update frequency based on training step) [S2, unofficial].

2.  **Coarse Sampling (Pass 1):**
    *   Sample $N_c$ points uniformly or via a simple distribution along the ray segment $[t_n, t_f]$ [S1].
    *   Standard NeRF uses $N_c = 64$ points per ray [S1].
    *   Query the coarse network $N_c$ at these positions to obtain density $\sigma_c$ and color $c_c$ [S1].
    *   Compute the cumulative transmittance $T_i$ and weights $w_i$ for importance sampling:
        $$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_c(x_j) \Delta t_j\right) $$
        $$ w_i = T_i (1 - \exp(-\sigma_c(x_i) \Delta t_i)) $$
        where $\Delta t_j$ is the distance between adjacent samples [S1].

3.  **Fine Sampling (Pass 2):**
    *   Use the weights $w_i$ from the coarse pass to define a probability density function (PDF) for sampling [S1].
    *   Sample $N_f$ points from this PDF. The points are clustered around high-density regions identified in the coarse pass [S1].
    *   Standard NeRF uses $N_f = 128$ additional points (total 192 per ray including coarse) [S1].
    *   Query the fine network $N_f$ at these new positions to obtain density $\sigma_f$ and color $c_f$ [S1].
    *   *Note:* In some variants (e.g., Base-NeRF), the final NeRF model queries 48 samples during training, increased to 192 at test time for higher quality [S3].

4.  **Rendering:**
    *   Compute the final pixel color $C$ using the fine pass samples:
        $$ C = \sum_{i=1}^{N_f} w_i c_f(x_i) $$
    *   The total number of network queries per ray is $N_c + N_f$. Standard NeRF uses $64 + 128 = 192$ fine samples, totaling 256 queries per ray (64 coarse + 192 fine) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_two_stage(ray_origins, ray_directions, t_near, t_far, coarse_net, fine_net):
    """
    Renders a batch of rays using two-stage NeRF sampling.
    
    Args:
        ray_origins: Tensor of shape (B, 3)
        ray_directions: Tensor of shape (B, 3)
        t_near: Float, near plane distance
        t_far: Float, far plane distance
        coarse_net: Function/Model for coarse density/color
        fine_net: Function/Model for fine density/color
        
    Returns:
        colors: Tensor of shape (B, 3)
    """
    # Step 1: Coarse Sampling
    # Sample 64 points uniformly along the ray
    num_coarse = 64
    t_coarse = torch.linspace(t_near, t_far, num_coarse).expand_as(ray_origins[:, :1]).transpose(0, 1)
    # Add jitter for anti-aliasing
    t_coarse = t_coarse + torch.rand_like(t_coarse) * (t_far - t_near) / num_coarse
    
    # Compute 3D positions
    x_coarse = ray_origins[:, None, :] + t_coarse[:, :, None] * ray_directions[:, None, :]
    
    # Query coarse network
    # Assume coarse_net returns (sigma, color)
    sigma_coarse, color_coarse = coarse_net(x_coarse, ray_directions)
    
    # Compute weights for importance sampling
    # Delta t is constant for uniform sampling
    delta_t = (t_far - t_near) / num_coarse
    # Alpha = 1 - exp(-sigma * delta_t)
    alpha_coarse = 1.0 - torch.exp(-sigma_coarse * delta_t)
    
    # Transmittance T_i = product(1 - alpha_j) for j < i
    # T_0 = 1
    ones = torch.ones_like(alpha_coarse[:, :1])
    alpha_coarse = torch.cat([ones, alpha_coarse[:, :-1]], dim=1)
    T_coarse = torch.cumprod(1.0 - alpha_coarse, dim=1)
    
    # Weights w_i = T_i * alpha_i
    weights_coarse = T_coarse * alpha_coarse
    
    # Step 2: Fine Sampling
    # Sample 128 points based on weights_coarse
    num_fine = 128
    # Resample t_fine based on weights_coarse PDF
    # This requires a custom resampling function (e.g., inverse CDF sampling)
    t_fine = resample_from_pdf(t_coarse, weights_coarse, num_fine)
    
    # Compute 3D positions for fine samples
    x_fine = ray_origins[:, None, :] + t_fine[:, :, None] * ray_directions[:, None, :]
    
    # Query fine network
    sigma_fine, color_fine = fine_net(x_fine, ray_directions)
    
    # Compute weights for fine pass
    # Note: In standard NeRF, the fine pass replaces the coarse pass for the final color
    # but uses the same transmittance logic relative to the start of the ray
    # However, often the fine samples are inserted into the ray structure.
    # Simplified: Just use fine samples for final color if they cover the volume well.
    # More accurate: Combine coarse and fine or just use fine if coarse was only for guidance.
    # Standard NeRF implementation: Use fine samples for final color.
    
    delta_t_fine = torch.diff(t_fine, dim=1)
    # Handle edge case for last sample
    delta_t_fine = torch.cat([delta_t_fine, delta_t_fine[:, -1:]], dim=1)
    
    alpha_fine = 1.0 - torch.exp(-sigma_fine * delta_t_fine)
    
    # Transmittance for fine pass
    ones_fine = torch.ones_like(alpha_fine[:, :1])
    alpha_fine = torch.cat([ones_fine, alpha_fine[:, :-1]], dim=1)
    T_fine = torch.cumprod(1.0 - alpha_fine, dim=1)
    
    weights_fine = T_fine * alpha_fine
    
    # Step 3: Render
    colors = torch.sum(weights_fine[:, :, None] * color_fine, dim=1)
    
    return colors
```

**Parameters:**
*   `num_coarse_samples`: 64 [S1].
*   `num_fine_samples`: 128 (additional) [S1].
*   `total_samples_per_ray`: 256 (64 coarse + 192 fine) [S1].
*   `initial_sampler`: 'uniform' or 'piecewise' [S2, unofficial].
*   `update_sched`: Function for proposal network update frequency [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision in transmittance calculation can lead to underflow in deep scenes. Use log-space computations if necessary [G].
*   **Boundaries:** Ensure $t_n$ and $t_f$ are correctly set to avoid sampling outside the scene or missing geometry [G].
*   **Degenerate Input:** If density is zero everywhere, weights will be zero, resulting in black pixels. Handle background color appropriately [G].

## Subtopics

### [Combined Sampling (Occupancy Grid + Proposal Network)](1626-combined-sampling-occupancy-grid-proposal-network.md)
Combined sampling stacks an occupancy grid on top of a proposal network to reduce ray queries and shrink the near-far plane. It decouples spatial sparsity estimation (grid) from density prediction (network), pruning rays that pass through empty voxels identified by the grid while using the proposal network for likely regions. Choose this when scenes have significant empty space and you need to prune rays entirely. Link: [Combined Sampling (Occupancy Grid + Proposal Network)](1626-combined-sampling-occupancy-grid-proposal-network.md).

### [VF-NeRF Rendering Pipeline](3302-vf-nerf-rendering-pipeline.md)
VF-NeRF uses a vector field network to predict surface normals and Jacobians for density estimation, deriving density from the directional derivative of the vector field. It enables explicit surface representation within the implicit volume and builds on two-stage rendering by integrating vector field-based density computation into the coarse and fine passes. Choose this when explicit surface geometry is required alongside view synthesis. Link: [VF-NeRF Rendering Pipeline](3302-vf-nerf-rendering-pipeline.md).

### [Coarse-Fine Sampling](8172-coarse-fine-sampling.md)
Coarse-Fine Sampling is the foundational two-pass technique where the coarse pass estimates density distribution and the fine pass importance-samples based on that distribution. It concentrates effort on high-density regions and decouples rough estimation from detailed evaluation. Choose this as the standard baseline for NeRF rendering. Link: [Coarse-Fine Sampling](8172-coarse-fine-sampling.md).

### Base-NeRF Sampling & Architecture
Base-NeRF uses a hierarchical proposal network structure: Proposal 1 queries 128 linearly spaced samples, Proposal 2 queries 96 samples via PDF, and the Final NeRF queries 48 samples via PDF (increased to 192 at test time) [S3]. It employs Hash Grids for positional encoding (levels/log2 size/base/max resolution) and MLPs for hidden dimensions [S3]. Proposal 1: Hash Grid 5/17/16/64, MLP 1/16. Proposal 2: Hash Grid 5/17/16/256, MLP 1/16. NeRF: Hash Grid 5/19/16/2048, MLP 1/64 [S3]. Choose this for large-scale urban scene reconstruction with memory-efficient hashing [S3].

### Speedup via importance thresholding
This variant prunes rays corresponding to background regions based on an importance threshold, yielding significant speedups on datasets like Blender where background dominates [S4, unofficial]. NeRF-ID renders views up to 50% faster while achieving better results than NeRF† [S4, unofficial]. Choose this when background pruning is effective for the scene type [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot (N_c + N_f))$ per image, where $N$ is the number of rays, $N_c$ is coarse samples, and $N_f$ is fine samples [S1].
*   **Memory Complexity:** $O(N \cdot (N_c + N_f))$ to store intermediate densities and colors during rendering [G].
*   **Measured Performance:** On an NVIDIA V100, rendering a realistic synthetic dataset (640k rays/image) or real scenes (762k rays/image) takes approximately 30 seconds per frame, resulting in 150-200 million network queries per image [S1].
*   **Scaling:** Performance scales linearly with the number of rays and samples. Proposal networks can reduce the effective number of queries by focusing on dense regions [S2, unofficial].
*   **Strengths:** Significantly reduces noise and improves quality compared to uniform sampling with the same number of queries; efficient use of network capacity [S1][S2, unofficial].
*   **Limitations:** Requires training two networks (or a network with two heads); increased training complexity; potential for artifacts if coarse sampling is insufficient [S1][S2, unofficial].
*   **When to Use:** High-quality novel view synthesis where rendering time is acceptable but quality is paramount; scenes with complex geometry [S1].
*   **When Not to Use:** Real-time applications requiring extremely low latency without hardware acceleration; scenes with very sparse geometry where uniform sampling might be sufficient [S1].

## Background
Two-stage NeRF Rendering was introduced by Mildenhall et al. in the original NeRF paper (2020) [S1]. It was developed to address the high computational cost of volume rendering in neural radiance fields by improving sampling efficiency. The method has since become a standard component in many NeRF variants and is foundational for more advanced sampling strategies like proposal networks [S1][S2, unofficial]. It is widely used in academic research and commercial applications for high-fidelity 3D reconstruction and view synthesis [S1][S3].

## Sources
[S1] NeRF: Representing scenes as neural radiance fields for view synthesis, http://www.matthewtancik.com/nerf, fetched 2026-10-27
[S2] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
[S3] PlaNeRF: SVD Unsupervised 3D Plane Regularization for NeRF Large-Scale Urban Scene Reconstruction, https://doi.org/10.1109/3dv62453.2024.00114, fetched 2026-11-04
[S4] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
