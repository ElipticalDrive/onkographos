# NeRF (Neural Radiance Fields)

Also called: NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis, Neural Radiance Field, Original NeRF

## Overview
NeRF represents a 3D scene as a continuous volumetric function that maps 3D spatial coordinates and 2D viewing directions to emitted color and volume density [S1][S3, S4, unofficial][S5][S6, S7, S8, S9, S11, S13, S16, unofficial]. It solves the problem of novel view synthesis by replacing explicit geometric primitives with an implicit neural representation encoded in network weights, trained to reproduce input views via differentiable volume rendering [S1][S3, S4, S8, S10, S13, S16, unofficial]. It distinguishes itself from its parent [Neural Radiance Fields](4972-neural-radiance-fields.md) by introducing a specific two-network architecture (coarse and fine) with hierarchical importance sampling and positional encoding to handle high-frequency details and view-dependent effects [S2, S4, S6, S7, S14, unofficial]. Unlike [NeRF with Softplus Activation and Importance Sampling](3524-nerf-with-softplus-activation-and-importance-sampling.md) which modifies the activation, or [Direct Voxel Grid Optimization](4694-direct-voxel-grid-optimization.md) which uses explicit grids, NeRF relies on a dense MLP [S1][S4, S6, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ calibrated images $\{I_i\}$ with known camera poses (extrinsics $R_i, t_i$ and intrinsics $K_i$) [S8, S10, unofficial].
*   **Outputs:** A trained MLP parameterized by $\theta$ that predicts volume density $\sigma$ and RGB color $\mathbf{c}$ for any 3D point $\mathbf{x}$ and viewing direction $\mathbf{d}$ [S1][S3, unofficial][S5][S7, S16, unofficial].

**Data Structures:**
The scene is represented by two Multi-Layer Perceptrons (MLPs): a coarse network $F_{\theta}^{coarse}$ and a fine network $F_{\theta}^{fine}$ [S2, S4, S6, S7, S14, unofficial].
*   **Network Architecture:** Both networks are 8-layer fully connected networks with 256 hidden units per layer [S6, S8, S14, unofficial].
*   **Skip Connections:** A skip connection concatenates the original positional encoding to the hidden state at layer 4 to address signal attenuation [S2, S6, S14, unofficial].
*   **Asymmetric Input Handling:**
    *   Position $\mathbf{x}$ is processed through the first 4 layers.
    *   Viewing direction $\mathbf{d}$ is concatenated to the output of layer 4 (or the intermediate feature vector) and processed only by the final layers [S2, S3, S16, unofficial]. This ensures $\sigma$ depends only on $\mathbf{x}$ (structural consistency) while $\mathbf{c}$ depends on $\mathbf{x}$ and $\mathbf{d}$ (view-dependent appearance) [S2, S3, unofficial].
*   **Output Layer:**
    *   Density $\sigma$: Predicted by a linear layer followed by a ReLU activation (or Softplus in some implementations, but ReLU is standard in the original codebase [S4, S11, unofficial]).
    *   Color $\mathbf{c}$: Predicted by a linear layer followed by a Sigmoid activation to bound values to $[0, 1]$ [S4, S11, unofficial].

**Positional Encoding:**
To enable the MLP to represent high-frequency details, inputs are mapped to Fourier features [S1][S3, S6, S14, S15, unofficial].
*   For an input vector $\mathbf{v}$ and $L$ frequency bands, the encoding $\gamma(\mathbf{v})$ is:
    $$ \gamma(\mathbf{v}) = [\sin(2^0 \pi \mathbf{v}), \cos(2^0 \pi \mathbf{v}), \dots, \sin(2^{L-1} \pi \mathbf{v}), \cos(2^{L-1} \pi \mathbf{v})] $$
*   **Parameters:**
    *   Position $\mathbf{x} \in \mathbb{R}^3$: $L=10$, resulting in $3 + 3 \times 2 \times 10 = 63$ dimensions [S6, S11, unofficial].
    *   Direction $\mathbf{d} \in \mathbb{R}^3$: $L=4$, resulting in $3 + 3 \times 2 \times 4 = 27$ dimensions [S6, unofficial].

**Numbered Steps:**

1.  **Ray Generation:** For each pixel $(u, v)$ in the target image, generate a ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ [S5][S7, S16, unofficial].
    *   Normalize pixel coordinates: $u' = u - W/2, v' = v - H/2$ [S10, unofficial].
    *   Compute camera-space direction: $\mathbf{d}_c = [u'/f, -v'/f, -1]^T$ (assuming $f_x=f_y=f$) [S7, S10, unofficial].
    *   Transform to world space: $\mathbf{d} = R \mathbf{d}_c$, normalized to unit length [S7, S10, unofficial].
    *   Origin $\mathbf{o}$ is the camera center from the pose matrix [S7, S10, unofficial].

2.  **Stratified Sampling (Coarse):**
    *   Define near and far bounds $t_n, t_f$ (e.g., 2.0 and 6.0 for Blender datasets) [S7, S10, S11, unofficial].
    *   Divide $[t_n, t_f]$ into $N_c$ equal bins (default $N_c=64$) [S6, S7, S10, unofficial].
    *   Sample one point $t_i$ uniformly from each bin: $t_i \sim \mathcal{U}[t_n + \frac{i-1}{N_c}(t_f-t_n), t_n + \frac{i}{N_c}(t_f-t_n)]$ [S7, S9, unofficial].
    *   If `perturb=True` (training), add uniform noise within each bin [S7, S10, unofficial].
    *   Compute 3D points: $\mathbf{x}_i = \mathbf{o} + t_i \mathbf{d}$ [S7, unofficial].

3.  **Coarse Network Query:**
    *   Apply positional encoding $\gamma(\mathbf{x}_i)$ and $\gamma(\mathbf{d})$ [S4, S11, S12, unofficial].
    *   Pass through $F_{\theta}^{coarse}$ to get raw density $\hat{\sigma}_i$ and color $\hat{\mathbf{c}}_i$ [S4, S11, S12, unofficial].
    *   Apply ReLU to $\hat{\sigma}_i$ and Sigmoid to $\hat{\mathbf{c}}_i$ [S4, S11, unofficial].

4.  **Coarse Rendering:**
    *   Compute segment lengths $\delta_i = t_{i+1} - t_i$ (with $t_{N_c+1} = t_f + \text{large value}$ or handled via transmittance) [S4, S7, S11, unofficial].
    *   Compute opacity $\alpha_i = 1 - \exp(-\hat{\sigma}_i \delta_i)$ [S4, S7, S11, unofficial].
    *   Compute transmittance $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ [S7, S11, unofficial].
    *   Compute weights $w_i = T_i \alpha_i$ [S7, S11, unofficial].
    *   Render coarse color $\hat{C}_{coarse} = \sum w_i \hat{\mathbf{c}}_i$ [S7, S11, unofficial].

5.  **Importance Sampling (Fine):**
    *   Use coarse weights $w_i$ to form a probability density function (PDF) [S7, S9, S14, unofficial].
    *   Normalize weights to sum to 1: $p_i = w_i / \sum w_j$ [S7, unofficial].
    *   Compute Cumulative Distribution Function (CDF) [S7, unofficial].
    *   Draw $N_f$ uniform random samples $u \sim \mathcal{U}[0,1]$ [S7, unofficial].
    *   Invert CDF to find corresponding $t$ values for fine samples (inverse transform sampling) [S7, unofficial].
    *   Default $N_f=128$ [S6, S7, unofficial].

6.  **Fine Network Query:**
    *   Query $F_{\theta}^{fine}$ at the fine sample locations $\mathbf{x}'_j$ and directions $\mathbf{d}$ [S4, S7, S14, unofficial].
    *   Get raw density $\hat{\sigma}'_j$ and color $\hat{\mathbf{c}}'_j$ [S4, S11, unofficial].

7.  **Fine Rendering:**
    *   Compute $\alpha'_j, T'_j, w'_j$ for fine samples [S7, S11, unofficial].
    *   Render fine color $\hat{C}_{fine} = \sum w'_j \hat{\mathbf{c}}'_j$ [S7, S9, S11, unofficial].

8.  **Loss Computation:**
    *   Calculate Mean Squared Error (MSE) between rendered colors and ground truth images $I$ [S2, S4, S6, S16, unofficial].
    *   Total Loss $\mathcal{L} = \lambda_{coarse} \|\hat{C}_{coarse} - I\|^2 + \lambda_{fine} \|\hat{C}_{fine} - I\|^2$ [S6, S9, unofficial].
    *   Typical weights: $\lambda_{coarse}=0.5, \lambda_{fine}=1.0$ [S6, unofficial].

9.  **Optimization:**
    *   Update network weights using Adam optimizer [S6, S11, unofficial].
    *   Learning rate schedule: Start at $5 \times 10^{-4}$, decay exponentially to $5 \times 10^{-5}$ over 200,000 steps [S6, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def positional_encoding(v, num_freq_bands):
    """
    Applies positional encoding to input vector v.
    v: (..., 3)
    Returns: (..., 3 + 3 * 2 * num_freq_bands)
    """
    freq_bands = 2.0 ** torch.arange(num_freq_bands, device=v.device).float()
    scaled_v = v.unsqueeze(-1) * freq_bands * torch.pi
    s = torch.sin(scaled_v)
    c = torch.cos(scaled_v)
    return torch.cat([v, s.flatten(-2), c.flatten(-2)], dim=-1)

def nerf_forward(x, d, coarse_mlp, fine_mlp, near, far, N_coarse, N_fine, perturb=True):
    """
    x: (..., 3) 3D points
    d: (..., 3) viewing directions
    """
    # 1. Positional Encoding
    emb_x = positional_encoding(x, num_freq_bands=10) # 63 dims
    emb_d = positional_encoding(d, num_freq_bands=4)  # 27 dims
    
    # 2. Coarse Network
    # Split MLP into position part and direction part as per NeRF architecture
    # Position part: Linear(63, 256) -> ReLU -> ... -> Linear(256, 256) -> ReLU -> Linear(256, 128) -> ReLU
    # Skip connection at layer 4 (index 3 if 0-indexed layers before skip)
    # For simplicity, assume coarse_mlp handles the full forward pass including skip and direction concat
    raw_coarse = coarse_mlp(emb_x, emb_d) # Output: (..., 4) [RGB, Sigma]
    
    sigma_coarse = F.relu(raw_coarse[..., 3])
    rgb_coarse = torch.sigmoid(raw_coarse[..., :3])
    
    # 3. Stratified Sampling for Coarse
    # In a real implementation, t_vals are generated before querying the network.
    # Here we assume x and d are already sampled points.
    # We need t_vals corresponding to x.
    # t_vals = near + (far - near) * torch.rand(...) # Simplified
    # For the sake of the algorithm description, we assume t_vals are available.
    
    # 4. Coarse Rendering
    # Compute deltas, alpha, weights
    # ... (see raw2outputs logic below)
    
    # 5. Importance Sampling for Fine
    # Use coarse weights to sample new t_vals
    # ... (see inverse CDF logic below)
    
    # 6. Fine Network
    raw_fine = fine_mlp(emb_x_fine, emb_d)
    sigma_fine = F.relu(raw_fine[..., 3])
    rgb_fine = torch.sigmoid(raw_fine[..., :3])
    
    # 7. Fine Rendering
    # ...
    
    return rgb_fine, sigma_fine

def raw2outputs(raw, z_vals, rays_d, raw_noise_std=0.0, white_bkgd=False):
    """
    raw: (..., 4) [RGB, Sigma]
    z_vals: (..., N) sample depths
    rays_d: (..., 3) ray directions
    """
    # Distances between samples
    dists = z_vals[..., 1:] - z_vals[..., :-1]
    # Append a large value for the last segment to ensure full transmittance calculation
    dists = torch.cat([dists, torch.Tensor([1e10]).expand(dists[..., :1].shape).to(dists.device)], dim=-1)
    # Scale by ray direction length
    dists = dists * torch.norm(rays_d[..., None, :], dim=-1)
    
    # Apply noise to density
    noise = torch.randn(raw[..., 3].shape, device=raw.device) * raw_noise_std
    sigma = F.relu(raw[..., 3] + noise)
    
    # Compute alpha
    alpha = 1.0 - torch.exp(-sigma * dists)
    
    # Compute transmittance
    # T_i = product_{j<i} (1 - alpha_j)
    # Use cumprod with epsilon for stability
    transmittance = torch.cumprod(torch.cat([
        torch.ones((*alpha.shape[:-1], 1), device=alpha.device),
        1.0 - alpha + 1e-10
    ], dim=-1), dim=-1)[..., :-1]
    
    # Compute weights
    weights = transmittance * alpha
    
    # Composite color
    rgb = torch.sigmoid(raw[..., :3])
    rgb_map = torch.sum(weights[..., None] * rgb, dim=-2)
    
    # Composite depth
    depth_map = torch.sum(weights * z_vals, dim=-1)
    
    # Accumulated opacity
    acc_map = torch.sum(weights, dim=-1)
    
    if white_bkgd:
        rgb_map = rgb_map + (1.0 - acc_map[..., None])
        
    return rgb_map, depth_map, acc_map, weights, sigma

def importance_sampling(z_coarse, weights_coarse, N_fine):
    """
    z_coarse: (..., N_coarse)
    weights_coarse: (..., N_coarse)
    Returns: z_fine (..., N_fine)
    """
    # Midpoints of coarse segments
    z_mid = 0.5 * (z_coarse[..., 1:] + z_coarse[..., :-1])
    # Weights for midpoints (exclude first and last if necessary, or adjust)
    # Standard NeRF uses weights[1:-1] for midpoints
    weights_mid = weights_coarse[..., 1:-1] + 1e-5
    
    # Normalize to PDF
    pdf = weights_mid / weights_mid.sum(dim=-1, keepdim=True)
    cdf = torch.cumsum(pdf, dim=-1)
    cdf = torch.cat([torch.zeros_like(cdf[..., :1]), cdf], dim=-1)
    
    # Sample uniform u
    u = torch.rand(*cdf.shape[:-1], N_fine, device=cdf.device)
    
    # Inverse CDF via searchsorted
    inds = torch.searchsorted(cdf, u, right=True)
    below = torch.clamp(inds - 1, min=0)
    above = torch.clamp(inds, max=cdf.shape[-1] - 1)
    
    # Gather cdf and z values
    cdf_g = torch.gather(cdf, -1, inds.view(*inds.shape[:-2], -1)).view(*inds.shape)
    z_g = torch.gather(z_mid, -1, inds.view(*inds.shape[:-2], -1)).view(*inds.shape)
    
    # Linear interpolation
    denom = cdf_g[..., 1] - cdf_g[..., 0]
    denom = torch.where(denom < 1e-5, torch.ones_like(denom), denom)
    z_fine = z_g[..., 0] + (u - cdf_g[..., 0]) / denom * (z_g[..., 1] - z_g[..., 0])
    
    return z_fine
```

**Parameters with Typical Values:**
*   $N_c$ (Coarse samples): 64 [S6, S10, unofficial].
*   $N_f$ (Fine samples): 128 [S6, S10, unofficial].
*   Positional Encoding Bands (Position): 10 [S6, unofficial].
*   Positional Encoding Bands (Direction): 4 [S6, unofficial].
*   MLP Hidden Units: 256 [S6, S8, S14, unofficial].
*   MLP Layers: 8 [S6, S8, S14, unofficial].
*   Near Plane: 2.0 (Blender) [S10, S11, unofficial].
*   Far Plane: 6.0 (Blender) [S10, S11, unofficial].
*   Learning Rate: $5 \times 10^{-4}$ decaying to $5 \times 10^{-5}$ [S6, unofficial].
*   Batch Size: 4096 rays [S11, unofficial].

**Edge Cases and Pitfalls:**
*   **Division by Zero in Cumulative Product:** Add epsilon ($10^{-10}$) to $(1-\alpha)$ before `cumprod` to prevent numerical instability [S11, unofficial].
*   **Last Segment Distance:** The last segment distance is artificially set to a large value ($10^{10}$) to ensure the last sample contributes correctly to the volume integral, effectively treating the space beyond the far plane as empty but allowing the final transmittance to decay properly [S4, S7, unofficial].
*   **Aliasing:** Standard frequency encoding treats rays as infinitely thin lines, causing aliasing at varying resolutions [S5]. (Addressed by Mip-NeRF).
*   **Camera Calibration:** Requires accurate extrinsics and intrinsics, typically estimated via SfM [S8, unofficial].

## Subtopics

### [NeRF Training Configuration](1203-nerf-training-configuration.md)
Defines the optimization loop, data handling, and regularization strategies.

### [NeRF-SH](1348-nerf-sh.md)
Replaces RGB output with spherical harmonic coefficients for efficient view-dependent appearance.

### [Recursive-NeRF](3029-recursive-nerf.md)
Uses a hierarchy of branch networks based on uncertainty to adapt to scene complexity.

### [Mip-NeRF](3381-mip-nerf.md)
Renders anti-aliased conical frustums instead of infinitesimal rays.

### [NeMF (Neural Microflake Field)](3387-nemf-neural-microflake-field.md)
Models light transportation via microflake normals and roughness.

### [EgoNeRF](4096-egonerf.md)
Uses a balanced spherical feature grid for large-scale egocentric scenes.

### [Intrinsic NeRF](4623-intrinsic-nerf.md)
Decouples geometry, illumination, and material properties.

### [Sparse Neural Volume Representation](7055-sparse-neural-volume-representation.md)
Uses a sparse 3D grid of per-voxel neural features.

## Operating characteristics
*   **Time Complexity:** Training is slow, typically taking 10-20 hours on a single NVIDIA A100 GPU for a single scene [S1][S6, unofficial]. Rendering a single novel view takes approximately 30 seconds [S1][S6, unofficial]. This is due to the need to query the MLP at hundreds of points per ray for hundreds of thousands of rays [S1].
*   **Memory Complexity:** The model weights are small (~5MB) [S4, unofficial], but the training process requires storing intermediate gradients and activations for many rays, managed via chunking (e.g., 4096 rays per batch) [S6, S11, unofficial].
*   **Performance:** On the NeRF Synthetic (Blender) dataset, NeRF achieves a PSNR of ~31.0 dB [S1]. On unbounded scenes (Mip-NeRF 360), it drops to ~24.2 dB [S1].
*   **Strengths:** High-quality novel view synthesis with realistic view-dependent effects (specularity, transparency) [S2, S4, S13, unofficial]. Implicit representation handles complex topology without explicit meshing [S1].
*   **Limitations:** Slow training and rendering speeds compared to explicit methods like 3D Gaussian Splatting [S1]. Prone to aliasing artifacts [S5]. Requires calibrated camera poses [S8, unofficial]. Poor surface accuracy compared to SDF-based methods like NeuS [S1].
*   **When to Use:** When high photorealism is required and computational resources/time are available. Suitable for static scenes with sparse input views.
*   **When Not to Use:** For real-time applications (use 3DGS or Instant-NGP), for large-scale unbounded scenes without extensions (use Mip-NeRF 360 or EgoNeRF), or when explicit surface geometry is needed (use NeuS).

## Background
NeRF was introduced by Mildenhall et al. in 2020 [S1][S3, unofficial][S5]. It established the field of neural rendering by demonstrating that a simple MLP could represent a 5D radiance field and be optimized via differentiable volume rendering to synthesize photorealistic novel views from sparse 2D images [S1][S3, unofficial][S5]. The method builds on earlier work in neural implicit representations and volume rendering [S5]. It has since become a foundational technique, inspiring numerous variants and acceleration methods [S1].

## Sources
[S1] Neural Rendering Techniques, https://doi.org/10.5281/zenodo.19614729, fetched 2026-10-26
[S2] Fourier Feature Networks and Neural Volume Rendering, https://github.com/matajoh/fourier_feature_nets, fetched 2026-10-26 (unofficial)
[S3] NeRF++: Analyzing and Improving Neural Radiance Fields, https://doi.org/10.48550/arxiv.2010.07492, fetched 2026-10-26 (unofficial)
[S4] yenchenlin/nerf-pytorch, https://github.com/yenchenlin/nerf-pytorch, fetched 2026-10-26 (unofficial)
[S5] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S6] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S7] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S8] PDF Volume Rendering - faculty.cc.gatech.edu, https://faculty.cc.gatech.edu/~hays/compvision2025spring/lectures/26.pdf, fetched 2026-10-26 (unofficial)
[S9] NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis | Takuma Yoneda, https://takuma.yoneda.xyz/posts/nerf, fetched 2026-10-27 (unofficial)
[S10] charan2598/NeRF-PyTorch, https://github.com/charan2598/NeRF-PyTorch, fetched 2026-10-27 (unofficial)
[S11] sr-bang/NeRF, https://github.com/sr-bang/NeRF, fetched 2026-10-27 (unofficial)
[S12] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S13] Generalizing Neural Radiance Fields - BAIR Commons, https://bcommons.berkeley.edu/generalizing-neural-radiance-fields, fetched 2026-10-28 (unofficial)
[S14] A Visual Exploration of Neural Radiance Fields, https://djrhails.github.io/post--radiance-fields/, fetched 2026-10-29 (unofficial)
[S15] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S16] ニューラル放射輝度場 NeRF 入門, https://zenn.dev/abist_inno/articles/353f36119d6b39, fetched 2026-10-30 (unofficial)
