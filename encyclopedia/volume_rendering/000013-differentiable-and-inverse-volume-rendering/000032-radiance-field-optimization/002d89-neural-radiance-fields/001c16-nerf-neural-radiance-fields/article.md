# NeRF (Neural Radiance Fields)

Also called: NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis, Neural Radiance Field, Neural Radiation Field, Original NeRF

## Overview
NeRF represents a 3D scene as a continuous volumetric function that maps 3D spatial coordinates and 2D viewing directions to emitted RGB color and volume density [S1][S5][S10, unofficial]. It solves the problem of novel view synthesis by replacing explicit geometric primitives (meshes or point clouds) with an implicit neural representation encoded in network weights, enabling photorealistic rendering of complex scenes with view-dependent effects like specularities and transparency [S1][S2, unofficial][S14, unofficial]. It distinguishes itself from its parent method [Neural Radiance Fields](4972-neural-radiance-fields.md) by introducing a specific asymmetric MLP architecture that injects viewing direction near the output layer, and a two-stage hierarchical sampling strategy (coarse and fine networks) to efficiently allocate samples to opaque regions [S2, unofficial][S4, unofficial][S7, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ calibrated images $\{I_i\}$ with known camera poses (extrinsics $R_i, t_i$ and intrinsics $K_i$) [S15, unofficial].
*   **Outputs:** A rendered image $\hat{C}(\mathbf{r})$ for a new camera pose, and a trained neural network parameterizing the radiance field [S1][S3, unofficial].

**Data Structures:**
The scene is represented by two Multi-Layer Perceptrons (MLPs), often sharing weights or architecture:
1.  **Coarse Network ($f_\theta$):** Maps encoded position to intermediate features and density.
2.  **Fine Network ($f'_\theta$):** Maps encoded position, intermediate features, and encoded direction to color.
*   **Positional Encoding ($\gamma$):** A fixed Fourier feature mapping applied to inputs [S9, unofficial].
*   **Ray Parameters:** For each pixel, a ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ is defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ [S5].

**Positional Encoding:**
Inputs are mapped to high-frequency features to enable the MLP to represent high-frequency details [S1][S9, unofficial]. For an input component $p$, the encoding $\gamma(p)$ is:
$$
\gamma(p) = (\sin(2^0 \pi p), \cos(2^0 \pi p), \sin(2^1 \pi p), \cos(2^1 \pi p), \dots, \sin(2^{L-1} \pi p), \cos(2^{L-1} \pi p))
$$
where $L$ is the number of frequency bands [S9, unofficial][S19, unofficial].
*   Position $\mathbf{x} \in \mathbb{R}^3$: $L_x = 10$, resulting in $3 + 3 \times 2 \times 10 = 63$ dimensions [S6, unofficial][S12, unofficial].
*   Direction $\mathbf{d} \in \mathbb{R}^3$: $L_d = 4$, resulting in $3 + 3 \times 2 \times 4 = 27$ dimensions [S6, unofficial][S12, unofficial].

**Network Architecture:**
The standard architecture consists of two networks [S6, unofficial][S20]:
1.  **Density/Feature Network:**
    *   Input: $\gamma(\mathbf{x})$ (63-dim).
    *   Layers: 8 linear layers with 256 hidden units and ReLU activation.
    *   Skip Connections: The original input $\gamma(\mathbf{x})$ is concatenated to the output of layers 4, 5, 6, and 7 (indices 0-based: 3, 4, 5, 6) [S2, unofficial][S6, unofficial][S12, unofficial].
    *   Output: Intermediate feature vector $\mathbf{h} \in \mathbb{R}^{256}$ and raw density $\sigma' \in \mathbb{R}$.
2.  **Color Network:**
    *   Input: $\mathbf{h}$, $\gamma(\mathbf{d})$ (27-dim).
    *   Layers: 1 linear layer (or 2 in some implementations) with ReLU.
    *   Output: Raw color $\mathbf{c}' \in \mathbb{R}^3$.

**Sampling Strategy:**
1.  **Ray Generation:** For each pixel $(u,v)$, compute direction $\mathbf{d}_c = K^{-1}[u, v, 1]^T$, normalize, rotate to world space $\mathbf{d}_w = R \mathbf{d}_c$, and set origin $\mathbf{o} = t$ [S7, unofficial][S12, unofficial].
2.  **Coarse Sampling:** Divide the interval $[t_n, t_f]$ into $N_c$ bins (typically 64). Sample one point uniformly at random from each bin (stratified sampling) [S7, unofficial][S10, unofficial]. If `perturb=True`, add uniform noise within each bin [S4, unofficial].
3.  **Fine Sampling (Importance Sampling):** Compute weights from coarse samples to form a PDF. Sample $N_f$ additional points (typically 128) using inverse-CDF sampling, concentrating samples where coarse density is high [S7, unofficial][S10, unofficial].

**Volume Rendering:**
For a ray with samples $t_1, \dots, t_N$ (sorted), let $\delta_i = t_{i+1} - t_i$ (with $\delta_N$ large or handled via transmittance) [S4, unofficial][S7, unofficial].
1.  **Density:** $\sigma_i = \text{Softplus}(\sigma'_i)$ or $\text{ReLU}(\sigma'_i)$ [S6, unofficial][S12, unofficial]. (Note: Original paper uses Softplus for smoother gradients, but ReLU is common in implementations) [S6, unofficial][S12, unofficial].
2.  **Alpha (Opacity):** $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ [S7, unofficial][S12, unofficial].
3.  **Transmittance:** $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) = \prod_{j=1}^{i-1} (1 - \alpha_j)$ [S7, unofficial][S19, unofficial].
4.  **Weight:** $w_i = T_i \alpha_i$ [S7, unofficial].
5.  **Color:** $\hat{C}(\mathbf{r}) = \sum_{i=1}^N w_i \mathbf{c}_i$, where $\mathbf{c}_i = \text{Sigmoid}(\mathbf{c}'_i)$ [S7, unofficial][S12, unofficial].
6.  **Background:** If `white_bkgd=True`, add $(1 - \sum w_i) \times 1.0$ to the RGB result [S4, unofficial][S7, unofficial].

**Training:**
Minimize the Mean Squared Error (MSE) between rendered color $\hat{C}(\mathbf{r})$ and ground truth $C(\mathbf{r})$ [S3, unofficial][S19, unofficial]:
$$
\mathcal{L} = \sum_{\mathbf{r}} \left[ \|\hat{C}_{\text{coarse}}(\mathbf{r}) - C(\mathbf{r})\|_2^2 + \|\hat{C}_{\text{fine}}(\mathbf{r}) - C(\mathbf{r})\|_2^2 \right]
$$
The coarse loss is often weighted by 0.5 [S6, unofficial]. Optimization uses Adam ($\beta_1=0.9, \beta_2=0.999$) with learning rate decaying from $5 \times 10^{-4}$ to $5 \times 10^{-5}$ [S6, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def positional_encoding(x, num_freq_bands):
    """
    Applies positional encoding to input tensor x.
    x: (..., D)
    Returns: (..., D + D * 2 * num_freq_bands)
    """
    freq_bands = 2.0 ** torch.arange(num_freq_bands, device=x.device).float() * torch.pi
    # Expand x for broadcasting: (..., D) -> (..., 1, D)
    # Expand freq_bands: (B,) -> (1, B, 1)
    scaled_x = x.unsqueeze(-2) * freq_bands.unsqueeze(-1) # (..., B, D)
    encoded = torch.cat([torch.sin(scaled_x), torch.cos(scaled_x)], dim=-2) # (..., 2*B, D)
    return torch.cat([x, encoded.flatten(start_dim=-2)], dim=-1)

class NeRFMLP(torch.nn.Module):
    def __init__(self, in_dim_xyz, in_dim_dir, hidden_dim=256, skips=[4]):
        super().__init__()
        self.skips = skips
        # Density/Feature Network
        self.linears_xyz = torch.nn.ModuleList([torch.nn.Linear(in_dim_xyz, hidden_dim)])
        for _ in range(7): # 8 layers total
            self.linears_xyz.append(torch.nn.Linear(hidden_dim, hidden_dim))
        
        # Color Network
        self.linear_feat = torch.nn.Linear(hidden_dim, hidden_dim)
        self.linear_color = torch.nn.Linear(hidden_dim + in_dim_dir, 3)

    def forward(self, xyz_encoded, dir_encoded):
        # Process position
        h = xyz_encoded
        for i, linear in enumerate(self.linears_xyz):
            h = linear(h)
            if i < 7: # ReLU for all except last if needed, but standard is ReLU then concat
                h = F.relu(h)
            if i in self.skips:
                h = torch.cat([h, xyz_encoded], dim=-1)
        
        # Intermediate feature h is output of last linear (index 7)
        # Note: In standard NeRF, h is passed to color network. 
        # The density is predicted from h before direction injection.
        raw_density = self.linears_xyz[-1](h) # This is actually the output of the 8th layer
        
        # Actually, standard implementation predicts density from the feature vector 
        # before the final color layer. Let's follow the specific structure:
        # Layer 0-7 produce h. Density is predicted from h.
        # Then h is passed to color network.
        
        # Re-implementation of standard 8-layer MLP for density/feature:
        # Input -> L0 -> ReLU -> L1 -> ReLU -> ... -> L3 -> ReLU -> Concat(x) -> L4 -> ... -> L7 -> h
        # Density = Linear(h)
        # Color Input = Concat(h, dir)
        
        # Let's stick to the provided code structure [S12, unofficial] which is slightly different 
        # but functionally equivalent for the purpose of this article's "standard" definition.
        # We will use the explicit structure from [S6, unofficial][S20].
        
        return h, raw_density

def render_ray(rays_o, rays_d, near, far, n_coarse, n_fine, network_coarse, network_fine, perturb=True):
    """
    Renders a single ray using coarse and fine sampling.
    """
    # 1. Coarse Sampling
    if perturb:
        t_vals = torch.linspace(0., 1., n_coarse + 1)
        t_vals = t_vals[:-1]
        t_rand = torch.rand_like(t_vals)
        t_samples = near * (1.0 - t_vals) + far * t_vals + (t_rand - 0.5) * (far - near) / n_coarse
    else:
        t_vals = torch.linspace(0., 1., n_coarse + 1)[1:] # Stratified without jitter logic simplified
        t_samples = near * (1.0 - t_vals) + far * t_vals
        
    # Query Coarse Network
    # xyz_encoded = positional_encoding(t_samples, 10) # Shape: (n_coarse, 63)
    # dir_encoded = positional_encoding(rays_d, 4) # Shape: (3, 27) - broadcasted
    # h_coarse, raw_sigma_coarse = network_coarse(xyz_encoded, dir_encoded) # dir not used in coarse for sigma
    
    # For simplicity, assume network takes (xyz_enc, dir_enc) but coarse ignores dir for sigma
    # In practice, coarse network outputs sigma and feature h.
    
    # 2. Compute Coarse Weights for Importance Sampling
    # sigma_coarse = softplus(raw_sigma_coarse)
    # dists = t_samples[1:] - t_samples[:-1]
    # alpha_coarse = 1.0 - torch.exp(-sigma_coarse * dists)
    # T_coarse = torch.cumprod(1.0 - alpha_coarse + 1e-10, dim=0)
    # weights_coarse = alpha_coarse * T_coarse
    
    # 3. Fine Sampling (Importance)
    # Use weights_coarse to sample n_fine points
    # t_fine = inverse_cdf_sampling(weights_coarse, t_samples, n_fine)
    
    # 4. Query Fine Network
    # h_fine, raw_sigma_fine = network_fine(xyz_fine_enc, dir_fine_enc)
    # c_fine = sigmoid(network_color(h_fine, dir_fine_enc))
    
    # 5. Volume Rendering
    # sigma_fine = softplus(raw_sigma_fine)
    # dists_fine = t_fine[1:] - t_fine[:-1]
    # alpha_fine = 1.0 - torch.exp(-sigma_fine * dists_fine)
    # T_fine = torch.cumprod(1.0 - alpha_fine + 1e-10, dim=0)
    # weights_fine = alpha_fine * T_fine
    # color = torch.sum(weights_fine[..., None] * c_fine, dim=0)
    
    # Handle background
    # acc = torch.sum(weights_fine, dim=0)
    # color = color + (1.0 - acc[..., None]) * 1.0 # White background
    
    return color
```

## Subtopics

### [NeRF Training Configuration](1203-nerf-training-configuration.md)
Defines the optimization loop, data handling, and regularization strategies. It focuses on the standard MLP architecture and training dynamics, distinguishing itself from [NeuS Training Loss](2535-neus-training-loss.md) by relying on volumetric opacity rather than explicit surface constraints.

### [NeRF-SH](1348-nerf-sh.md)
Replaces direct RGB output with spherical harmonic (SH) coefficients, eliminating viewing direction as a network input. This enables efficient storage in a sparse octree (PlenOctree) and faster rendering.

### [Recursive-NeRF](3029-recursive-nerf.md)
Routes query points through a hierarchy of branch networks based on predicted uncertainty, allowing dynamic network growth and early termination for simple regions.

### [Mip-NeRF](3381-mip-nerf.md)
Renders anti-aliased conical frustums instead of infinitesimal rays using Integrated Positional Encoding (IPE), reducing aliasing artifacts.

### [NeMF (Neural Microflake Field)](3387-nemf-neural-microflake-field.md)
Models light transportation via microflake normals and roughness, handling complex geometry and translucency better than standard NeRF.

### [EgoNeRF](4096-egonerf.md)
Adapts NeRF for large-scale unbounded scenes using a balanced spherical feature grid, sharing a single density grid for coarse and fine sampling.

### [Intrinsic NeRF](4623-intrinsic-nerf.md)
Decouples geometry, illumination, and material properties by outputting separate shading and reflectance components.

### [Sparse Neural Volume Representation](7055-sparse-neural-volume-representation.md)
Uses a sparse 3D grid of per-voxel neural features to improve memory efficiency and scalability for large scenes.

## Operating characteristics
*   **Time Complexity:** Training is slow, typically requiring 10–20 hours on a single NVIDIA A100 GPU for synthetic scenes [S1][S6, unofficial]. Rendering a single novel view takes approximately 30 seconds [S1].
*   **Memory Complexity:** The MLP weights are small (~5 MB) [S4, unofficial], but rendering requires storing intermediate transmittance and alpha values for all samples along all rays, leading to high GPU memory usage during training (mitigated by chunking rays into batches of 4096) [S6, unofficial][S12, unofficial].
*   **Performance:** On the NeRF Synthetic (Blender) dataset, NeRF achieves a PSNR of ~31.0 dB [S1]. It is slower than [Instant-NGP](8539-graf.md) (which trains in minutes) and [3D Gaussian Splatting](6569-point-nerf.md) (real-time), but provides higher quality than early hash-grid methods [S1].
*   **Strengths:** High-quality novel view synthesis with accurate view-dependent effects (specularities, transparency) and robust handling of complex geometry without explicit mesh extraction [S1][S2, unofficial].
*   **Limitations:** Slow training and rendering speeds; prone to aliasing artifacts at varying resolutions (addressed by [Mip-NeRF](3381-mip-nerf.md)); density field does not explicitly represent surfaces, making mesh extraction difficult [S1].

## Background
NeRF was introduced by Mildenhall et al. in 2020 [S1][S3, unofficial]. It established the field of neural radiance fields by demonstrating that a simple MLP, optimized via differentiable volume rendering, could synthesize photorealistic novel views from sparse input images. The method builds on earlier neural implicit representations [S5] and positional encoding techniques [S9, unofficial]. It is widely used in virtual reality, film production, and 3D asset creation [S1].

## Sources
[S1] Neural Rendering Techniques, https://doi.org/10.5281/zenodo.19614729, fetched 2026-10-26
[S2] Fourier Feature Networks and Neural Volume Rendering, https://github.com/matajoh/fourier_feature_nets, fetched 2026-10-26 (unofficial)
[S3] NeRF++: Analyzing and Improving Neural Radiance Fields, https://doi.org/10.48550/arxiv.2010.07492, fetched 2026-10-26 (unofficial)
[S4] yenchenlin/nerf-pytorch, https://github.com/yenchenlin/nerf-pytorch, fetched 2026-10-26 (unofficial)
[S5] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S6] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S7] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S8] PDF Volume Rendering - faculty.cc.gatech.edu, https://faculty.cc.gatech.edu/~hays/compvision2025spring/lectures/26.pdf, fetched 2026-10-26 (unofficial)
[S9] GRAF: Generative Radiance Fields for 3D-Aware Image Synthesis, https://doi.org/10.48550/arxiv.2007.02442, fetched 2026-10-26 (unofficial)
[S10] NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis | Takuma Yoneda, https://takuma.yoneda.xyz/posts/nerf, fetched 2026-10-27 (unofficial)
[S11] charan2598/NeRF-PyTorch, https://github.com/charan2598/NeRF-PyTorch, fetched 2026-10-27 (unofficial)
[S12] sr-bang/NeRF, https://github.com/sr-bang/NeRF, fetched 2026-10-27 (unofficial)
[S13] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S14] Generalizing Neural Radiance Fields - BAIR Commons, https://bcommons.berkeley.edu/generalizing-neural-radiance-fields, fetched 2026-10-28 (unofficial)
[S15] Taichi NeRF (Part 1): Develop and Deploy Instant NGP without writing CUDA | Taichi Docs, https://docs.taichi-lang.org/blog/taichi-instant-ngp, fetched 2026-10-29 (unofficial)
[S16] A Visual Exploration of Neural Radiance Fields, https://djrhails.github.io/post--radiance-fields/, fetched 2026-10-29 (unofficial)
[S17] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
[S18] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S19] ニューラル放射輝度場 NeRF 入門, https://zenn.dev/abist_inno/articles/353f36119d6b39, fetched 2026-10-30 (unofficial)
[S20] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
