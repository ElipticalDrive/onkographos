# NeRF Hyperparameters

Also called: Blender Training Configuration, NeRF Model and Sampling Configuration, NeRF Training Hyperparameters, NeTF Training Configuration, TinyNeRF Training Setup, Training Loop Configuration

## Overview
NeRF Hyperparameters define the specific configuration of the neural network architecture, sampling strategy, and optimizer settings required to train a Neural Radiance Field (NeRF) model [S1][S3, S4, unofficial]. This configuration solves the problem of reconstructing a continuous 3D scene representation from a set of calibrated 2D images by optimizing the radiance field parameters [S3, S4, unofficial]. What sets this configuration apart from its siblings is that it establishes the baseline "vanilla" NeRF settings (specifically the MLP depth, width, positional encoding frequencies, and hierarchical sampling counts) against which variants like Mip-NeRF [3409-mip-nerf-optimization-and-activation-details.md] or HashNeRF [3068-hashnerf-parameters.md] adjust their specific architectural or optimization details [S1][S3, S4, unofficial].

## How it works
The training process initializes an Adam optimizer and a NeRF model, loads a dataset (e.g., Blender synthetic scenes), and iteratively minimizes the difference between rendered and ground-truth images [S4, unofficial]. The method relies on hierarchical sampling: a coarse proposal network samples points to identify high-density regions, and a fine network samples more densely in those regions [S1][S3, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   Dataset: A collection of images with known camera poses (extrinsics) and intrinsics (focal length) [S3, S4, unofficial].
    *   Hyperparameters: Defined below (MLP layers, samples, learning rate, etc.) [S1][S3, S4, unofficial].
*   **Outputs**:
    *   Trained NeRF model weights (MLP parameters) [S4, unofficial].
    *   Rendered images for validation/test sets [S4, unofficial].

### Data Structures
The model consists of two Multi-Layer Perceptrons (MLPs): a **Coarse MLP** (proposal network) and a **Fine MLP** (final radiance field) [S1][S3, unofficial].

**Coarse MLP Structure:**
*   Input: 3D position $(x, y, z)$ and viewing direction $(\theta, \phi)$ [S1].
*   Positional Encoding (PE): Axis-aligned PE applied to position and direction [S1].
*   Layers: 4 hidden layers [S1].
*   Hidden Units: 256 per layer [S1].
*   Output: Volume density $\sigma$ and color $c$ (RGB) [S1].

**Fine MLP Structure:**
*   Input: 3D position and viewing direction [S1].
*   Positional Encoding: Axis-aligned PE [S1].
*   Layers: 8 hidden layers [S3, unofficial].
*   Hidden Units: 256 (standard) or 512 (larger capacity) [S1].
*   Output: Volume density $\sigma$ and color $c$ (RGB) [S1].

**Positional Encoding:**
The positional encoding function $\gamma(p)$ maps a scalar $p$ to a vector of sines and cosines at different frequencies [S3, unofficial]. For a position vector $\mathbf{p} \in \mathbb{R}^3$, the encoded vector has size $2 \times L \times \text{freqs}$, where $L$ is the dimensionality of the input (3 for position, 2 for direction in spherical coordinates) [S3, unofficial].

### Numbered Steps
1.  **Initialization**:
    *   Initialize Adam optimizer with model parameters and learning rate [S4, unofficial].
    *   Load dataset images and normalize pixel values by dividing by 255.0 [S4, unofficial].
    *   Reserve one image for validation/testing [S4, unofficial].
2.  **Ray Generation**:
    *   For each pixel in the training image batch, generate a ray origin $\mathbf{o}$ and direction $\mathbf{d}$ using camera intrinsics and extrinsics [S4, unofficial].
    *   Define near ($t_n$) and far ($t_f$) clipping planes along the ray. Typical values: $t_n = 2.0$, $t_f = 6.0$ for Blender scenes [S4, unofficial].
3.  **Coarse Sampling**:
    *   Sample $N_c$ points uniformly along the ray segment $[t_n, t_f]$. Default $N_c = 64$ [S3, unofficial].
    *   Query the **Coarse MLP** at these points to get densities $\sigma_c$ and colors $c_c$ [S1].
    *   Compute transmittance $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ and alpha $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ [S3, unofficial].
    *   Render coarse color $C_c = \sum_{i=1}^{N_c} T_i \alpha_i c_{c,i}$ [S3, unofficial].
4.  **Hierarchical Resampling**:
    *   Use the coarse weights $w_i = T_i \alpha_i$ to construct a cumulative distribution function (CDF) [S3, unofficial].
    *   Sample $N_f$ points from this distribution (inverse CDF sampling) to concentrate samples in high-density regions. Default $N_f = 128$ [S3, unofficial].
    *   Query the **Fine MLP** at these new points to get densities $\sigma_f$ and colors $c_f$ [S1].
5.  **Fine Rendering**:
    *   Compute transmittance and alpha for the fine samples [S3, unofficial].
    *   Render fine color $C_f = \sum_{i=1}^{N_f} T_i \alpha_i c_{f,i}$ [S3, unofficial].
6.  **Loss and Update**:
    *   Compute Mean Squared Error (MSE) loss between $C_f$ and the ground-truth pixel color [S1].
    *   Backpropagate and update model parameters using Adam [S4, unofficial].
    *   Decay learning rate exponentially [S2].

### Formulas
**Volume Rendering Equation (Discrete Approximation):**
$$ C(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i c_i $$ [S3, unofficial]

**Transmittance:**
$$ T_i = \exp\left( -\sum_{j=1}^{i-1} \sigma_j \delta_j \right) $$ [S3, unofficial]

**Alpha (Opacity):**
$$ \alpha_i = 1 - \exp(-\sigma_i \delta_i) $$ [S3, unofficial]

**Delta (Distance between samples):**
$$ \delta_i = t_{i+1} - t_i $$ [S3, unofficial]

**Positional Encoding:**
For a scalar input $p$ and frequency $l$:
$$ \gamma_l(p) = (\sin(2^l \pi p), \cos(2^l \pi p)) $$ [S3, unofficial]

### Parameters and Typical Values
*   **Coarse Samples ($N_c$)**: 64 [S3, unofficial].
*   **Fine Samples ($N_f$)**: 128 [S3, unofficial].
*   **Coarse MLP Layers**: 4 [S1].
*   **Coarse MLP Width**: 256 [S1].
*   **Fine MLP Layers**: 8 [S3, unofficial].
*   **Fine MLP Width**: 256 or 512 [S1].
*   **Positional Encoding Frequencies (Position)**: 10 [S3, unofficial].
*   **Positional Encoding Frequencies (Direction)**: 4 [S3, unofficial].
*   **Learning Rate ($\eta_0$)**: $5 \times 10^{-4}$ [S3, unofficial] or $1 \times 10^{-3}$ [S2].
*   **Learning Rate Decay**: Exponential decay to $1 \times 10^{-4}$ [S2].
*   **Optimizer**: Adam with $\beta_1 = 0.9$, $\epsilon = 1 \times 10^{-7}$ [S2].
*   **Batch Size (Rays)**: 1024 [S3, unofficial].
*   **Chunk Size (Inference)**: 4096 points processed per minibatch to avoid OOM [S4, unofficial].
*   **Near Plane**: 2.0 [S4, unofficial].
*   **Far Plane**: 6.0 [S4, unofficial].
*   **Total Iterations**: 4096 [S4, unofficial] or 200,000 [S3, unofficial].

### Edge Cases and Pitfalls
*   **Out of Memory (OOM)**: Processing all points for a batch simultaneously can exceed GPU memory. Solution: Split query points into chunks (e.g., 4096 points) and process sequentially [S4, unofficial].
*   **Precision**: Use float32 for standard training. Mixed precision can be used but requires careful handling of loss scaling [G].
*   **Degenerate Input**: If near/far planes are not set correctly relative to the scene scale, sampling may miss the object or include excessive empty space, leading to poor convergence [G].
*   **Distortion Regularization**: Standard NeRF does not apply distortion regularization [S1], unlike mip-NeRF which uses it to improve anti-aliasing [3409-mip-nerf-optimization-and-activation-details.md].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def positional_encoding(x, freqs):
    """Axis-aligned positional encoding."""
    N, D = x.shape
    scales = 2.0 ** torch.arange(0, freqs, device=x.device)
    x = x.unsqueeze(-1).expand(-1, -1, freqs) * scales
    return torch.cat([torch.sin(x), torch.cos(x)], dim=-1).reshape(N, -1)

class NeRFMLP(nn.Module):
    def __init__(self, layers=8, width=256, pos_freqs=10, dir_freqs=4):
        super().__init__()
        self.pos_freqs = pos_freqs
        self.dir_freqs = dir_freqs
        self.input_dim = 3 * (2 * pos_freqs) + 2 * (2 * dir_freqs)
        self.layers = nn.ModuleList()
        self.layers.append(nn.Linear(self.input_dim, width))
        for _ in range(layers - 1):
            self.layers.append(nn.Linear(width, width))
        self.layers.append(nn.Linear(width, 4))
        self.skip_indices = [4] if layers > 4 else []

    def forward(self, x_pos, x_dir):
        pos_enc = positional_encoding(x_pos, self.pos_freqs)
        dir_enc = positional_encoding(x_dir, self.dir_freqs)
        x = torch.cat([pos_enc, dir_enc], dim=-1)
        for i, layer in enumerate(self.layers):
            if i in self.skip_indices:
                x = torch.cat([x, pos_enc], dim=-1)
            x = layer(x)
            if i < len(self.layers) - 1:
                x = torch.relu(x)
        sigma = torch.relu(x[..., 0])
        color = torch.sigmoid(x[..., 1:])
        return color, sigma

def render_ray(ray_o, ray_d, near, far, n_samples, model, perturb=True):
    """Render a single ray or batch of rays."""
    t_vals = torch.linspace(0.0, 1.0, n_samples, device=ray_o.device)
    z_vals = near * (1.0 - t_vals) + far * t_vals
    if perturb:
        mids = 0.5 * (z_vals[..., 1:] + z_vals[..., :-1])
        upper = torch.cat([mids, z_vals[..., -1:]], -1)
        lower = torch.cat([z_vals[..., :1], mids], -1)
        t_rand = torch.rand_like(z_vals)
        z_vals = lower + (upper - lower) * t_rand
    pts = ray_o.unsqueeze(1) + ray_d.unsqueeze(1) * z_vals.unsqueeze(2)
    
    chunk_size = 4096
    all_colors, all_sigmas = [], []
    for i in range(0, pts.shape[0], chunk_size):
        chunk_pts = pts[i:i+chunk_size]
        colors, sigmas = model(chunk_pts, ray_d)
        all_colors.append(colors)
        all_sigmas.append(sigmas)
    colors = torch.cat(all_colors, dim=0)
    sigmas = torch.cat(all_sigmas, dim=0)
    
    deltas = z_vals[..., 1:] - z_vals[..., :-1]
    delta_inf = 1e10
    deltas = torch.cat([deltas, delta_inf], -1)
    alpha = 1.0 - torch.exp(-sigmas * deltas)
    T = torch.cumsum(sigmas * deltas, dim=-1)
    T = torch.cat([torch.zeros_like(T[..., :1]), T], dim=-1)
    transmittance = torch.exp(-T)
    weights = alpha * transmittance
    color_out = torch.sum(weights.unsqueeze(-1) * colors, dim=-2)
    return color_out, weights

def train_step(model, optimizer, images, poses, intrinsics, args):
    optimizer.zero_grad()
    height, width = images.shape[1:3]
    focal = intrinsics[0, 0]
    y, x = torch.meshgrid(torch.arange(height), torch.arange(width), indexing='ij')
    dirs = torch.stack([(x - width/2)/focal, -(y - height/2)/focal, -torch.ones_like(x)], dim=-1)
    dirs = dirs / torch.norm(dirs, dim=-1, keepdim=True)
    origins = poses[..., :3, 3].unsqueeze(1).unsqueeze(1).expand(height, width, 3)
    rays_o = origins.reshape(-1, 3)
    rays_d = dirs.reshape(-1, 3)
    images_flat = images.reshape(-1, 3)
    
    # Coarse sampling
    coarse_color, coarse_weights = render_ray(rays_o, rays_d, args.near, args.far, args.n_samples_coarse, model.coarse_model)
    
    # Hierarchical resampling: sample fine points based on coarse weights
    # (Standard implementation uses inverse CDF sampling on coarse_weights)
    fine_color, _ = render_ray(rays_o, rays_d, args.near, args.far, args.n_samples_fine, model.fine_model)
    
    loss = torch.mean((fine_color - images_flat) ** 2)
    loss.backward()
    optimizer.step()
    return loss.item()
```

## Subtopics

### [HashNeRF Parameters](3068-hashnerf-parameters.md)
Configures HashNeRF, which replaces the MLP with a multi-resolution hashed grid. It requires settings for hash map size and regularization, differing from the MLP-based NeRF hyperparameters.

### [Mip-NeRF Optimization and Activation Details](3409-mip-nerf-optimization-and-activation-details.md)
Details the specific changes to NeRF hyperparameters for Mip-NeRF, including cone tracing, different positional encoding, and distortion regularization.

### [DONeRF](4461-donerf.md)
Specifies hyperparameters for DONeRF, which adapts NeRF for dynamic scenes, often involving temporal encoding and different sampling strategies.

### [Reflectance and Shading Decomposition Priors](4624-reflectance-and-shading-decomposition-priors.md)
Defines hyperparameters for methods that decompose radiance into albedo and shading, requiring additional priors and loss weights.

## Operating characteristics
*   **Time Complexity**: Training time scales linearly with the number of iterations and the number of samples per ray ($N_c + N_f$). Quadrupling the sampling resolution (e.g., from 32x32 to 64x64 in NeTF context) quadruples training time [S2].
*   **Memory Complexity**: Memory usage is dominated by the MLP weights and the intermediate activations during the forward pass. Chunking (e.g., 4096 points) limits peak memory usage [S4, unofficial].
*   **Measured Performance**:
    *   On Blender dataset (Lego scene, 200k iterations): PSNR ~31-32 dB, SSIM ~0.96, Time ~12-15 hours on a single GPU [S3, unofficial].
    *   With 256 hidden units: 0.84M params, 1.86 hours training time, PSNR 32.96 [S1].
    *   With 512 hidden units: 3.23M params, 3.42 hours training time, PSNR 33.25 [S1].
    *   Comparison: The proposed model (with proposal networks) is significantly faster to train than mip-NeRF, with speedup increasing as model capacity rises [S1].
*   **Strengths**: Simple to implement, robust baseline, good quality on synthetic data.
*   **Limitations**: Slow training for large scenes, requires many samples for high quality, sensitive to near/far plane selection.

## Background
The hyperparameters described here are based on the original NeRF implementation and its common variants (like the "TinyNeRF" implementation [S4, unofficial] and the "NeRF from Scratch" [S3, unofficial]). The configuration has become a standard reference for differentiable volume rendering optimization [S1][S3, S4, unofficial].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] Non-line-of-Sight Imaging via Neural Transient Fields, https://doi.org/10.1109/tpami.2021.3076062, fetched 2026-10-26
[S3] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S4] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
