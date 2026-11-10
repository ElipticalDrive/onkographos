# NeRF Hyperparameters

Also called: Blender Training Configuration, FastNeRF Training Configuration, NeRF Model and Sampling Configuration, NeRF Training Hyperparameters, NeTF Training Configuration, TinyNeRF Setup, Training Loop Configuration

## Overview
NeRF Hyperparameters define the configuration for training Neural Radiance Fields, specifically optimizing an MLP that maps 3D coordinates and view directions to density ($\sigma$) and color ($c$) [S1][S3, S4, unofficial]. The method minimizes the Mean Squared Error (MSE) between rendered images and ground truth using the differentiable volume rendering equation [S3, unofficial]. It differs from standard implementations by specifying precise architectural dimensions (layer counts, hidden sizes), sampling strategies (coarse/fine stages, hierarchical sampling), and optimizer settings (Adam variants, learning rate schedules) required to converge on synthetic datasets like Blender or LLFF [S1][S3, unofficial][S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A dataset of images with known camera poses (intrinsics $K$, extrinsics $T$), near/far clipping planes, and ground truth pixel values normalized to $[0, 1]$ [S3, unofficial][S5, unofficial].
*   **Outputs:** Optimized MLP weights for the NeRF model, and optionally a cached density field or mesh for inference acceleration [S4, unofficial].

**Data Structures:**
*   **Ray Bundle:** A tensor of shape $[N_{rays}, 3]$ for origins and $[N_{rays}, 3]$ for directions [S5, unofficial].
*   **Query Points:** A tensor of shape $[N_{rays}, N_{samples}, 3]$ representing 3D coordinates along rays [S5, unofficial].
*   **Model Architecture:**
    *   **Position MLP:** 8 layers, 256 (or 512) hidden units [S1][S3, unofficial]. Uses axis-aligned Implicit Positional Encoding (IPE) or Fourier features [S1][S4, unofficial].
    *   **Direction MLP:** 4 layers, 128 hidden units (for view-dependent color) [S4, unofficial].
    *   **Proposal MLP:** 4 layers, 256 hidden units (used for importance sampling) [S1].

**Numbered Steps:**
1.  **Data Loading:** Load images, poses, and focal length. Normalize pixel values by dividing by 255.0 [S5, unofficial].
2.  **Ray Sampling:** For each training step, sample $N_{rays}$ rays from the camera poses.
    *   **Coarse Sampling:** Stratified sampling along the ray interval $[near, far]$ with $N_{coarse}$ samples (typically 64) [S3, unofficial][S4, unofficial].
    *   **Fine Sampling:** Hierarchical sampling. Compute weights $w_i = T_i \alpha_i$ from the coarse pass. Resample $N_{fine}$ points (typically 128) around high-weight regions using inverse CDF sampling [S3, unofficial].
3.  **Chunked Forward Pass:** To manage GPU memory, split query points into chunks of size `chunksize` (e.g., 4096) [S5, unofficial].
    *   For each chunk, query the Position MLP to get density $\sigma$ and feature vector $f$.
    *   Query the Direction MLP with $f$ and view direction to get color $c$ [S5, unofficial].
4.  **Volume Rendering:** Compute opacity $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ and transmittance $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ [S3, unofficial]. Render color $C(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i c_i$ [S3, unofficial].
5.  **Loss Computation:** Calculate MSE loss between rendered color and ground truth [S3, unofficial].
6.  **Optimization:** Update weights using the Adam optimizer [S1][S2][S3, S4, unofficial].

**Formulas:**
*   **Opacity:** $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ [S3, unofficial].
*   **Transmittance:** $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ [S3, unofficial].
*   **Rendering Equation:** $C(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i c_i$ [S3, unofficial].
*   **Loss (Standard):** $L = \frac{1}{M} \sum_{i=1}^{M} \| C_i(\mathbf{r}_i) - C_i^*(\mathbf{r}_i) \|^2$ [S3, unofficial].

**Parameters:**
*   **Learning Rate:** Initial $5 \times 10^{-4}$ [S3, unofficial][S4, unofficial]. Decays exponentially to $5 \times 10^{-6}$ or $1 \times 10^{-4}$ [S2][S6].
*   **Optimizer:** Adam with $\beta_1 = 0.9, \beta_2 = 0.999, \epsilon = 1 \times 10^{-7}$ [S2][S4, unofficial].
*   **Batch Size:** 1024 rays per batch [S3, unofficial].
*   **Samples per Ray:** Coarse: 64, Fine: 128 (Blender/360); Coarse: 64, Fine: 64 (LLFF) [S4, unofficial].
*   **Training Iterations:** 300,000 to 4096 (depending on dataset/scale) [S4, unofficial][S5, unofficial].
*   **Positional Encoding Frequencies:** 10 for position, 4 for direction [S3, unofficial].

**Pitfalls:**
*   **Overfitting:** High-frequency view-dependent MLPs can cause flickering artifacts in motion [S4, unofficial]. Mitigate by reducing direction encoding dimensionality ($L=1$) or using a smaller direction cache [S4, unofficial].
*   **Memory:** Large chunk sizes cause OOM; small chunk sizes reduce throughput. Balance with `chunksize` [S5, unofficial].
*   **Precision:** Floaters may appear in sparse views; requires regularization (e.g., Simple-RF loss) [S6].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

class NeRF:
    def __init__(self, pos_layers=8, pos_width=256, dir_layers=4, dir_width=128, 
                 pos_freqs=10, dir_freqs=4, near=2.0, far=6.0):
        self.near = near
        self.far = far
        self.pos_freqs = pos_freqs
        self.dir_freqs = dir_freqs
        # Position MLP
        self.pos_mlp = nn.Sequential()
        for i in range(pos_layers):
            self.pos_mlp.append(nn.Linear(3 + 2*pos_freqs*3, pos_width))
            if i < pos_layers - 1:
                self.pos_mlp.append(nn.ReLU())
        self.pos_mlp.append(nn.Linear(pos_width, 1)) # Density
        
        # Direction MLP
        self.dir_mlp = nn.Sequential()
        self.dir_mlp.append(nn.Linear(pos_width + 2*dir_freqs*3, dir_width))
        self.dir_mlp.append(nn.ReLU())
        for i in range(dir_layers - 2):
            self.dir_mlp.append(nn.Linear(dir_width, dir_width))
            self.dir_mlp.append(nn.ReLU())
        self.dir_mlp.append(nn.Linear(dir_width, 3)) # Color

    def positional_encoding(self, x, freqs):
        # Standard NeRF PE: sin(2^i * pi * x)
        x = x.unsqueeze(-1)
        scales = torch.tensor([2**i for i in range(freqs)], dtype=x.dtype, device=x.device)
        shaped_freqs = scales.view(1, 1, -1)
        x = x * shaped_freqs * torch.pi
        return torch.cat([torch.sin(x), torch.cos(x)], dim=-1).view(x.shape[0], -1)

    def forward(self, rays_o, rays_d, n_samples, perturb=True):
        # Stratified sampling
        t_vals = torch.linspace(0.0, 1.0, steps=n_samples, device=rays_o.device)
        z_vals = self.near * (1.0 - t_vals) + self.far * t_vals
        if perturb:
            mids = 0.5 * (z_vals[..., 1:] + z_vals[..., :-1])
            upper = torch.cat([mids, z_vals[..., -1:]], -1)
            lower = torch.cat([z_vals[..., :1], mids], -1)
            t_rand = torch.rand(z_vals.shape, device=rays_o.device)
            z_vals = lower + (upper - lower) * t_rand
        
        pts = rays_o.unsqueeze(1) + rays_d.unsqueeze(1) * z_vals.unsqueeze(2)
        
        # Chunked processing
        chunk_size = 4096
        rgb_list = []
        sigma_list = []
        
        for i in range(0, pts.shape[0], chunk_size):
            chunk_pts = pts[i:i+chunk_size]
            chunk_dirs = rays_d[i:i+chunk_size].expand_as(chunk_pts)
            
            # PE
            pts_pe = self.positional_encoding(chunk_pts, self.pos_freqs)
            dirs_pe = self.positional_encoding(chunk_dirs, self.dir_freqs)
            
            # Pos MLP
            pos_out = self.pos_mlp(pts_pe)
            sigma = pos_out[..., 0]
            feature = pos_out[..., 1:] # Assuming last dim is width, but standard NeRF outputs sigma and feature separately or uses skip connections. 
            # Simplified: Assume pos_mlp outputs sigma and feature concatenated or separate heads. 
            # Standard NeRF [S3, unofficial] uses skip connections. For this pseudocode, we assume a simplified linear projection for feature.
            # Correcting for standard NeRF structure:
            # pos_mlp outputs sigma. Feature is intermediate layer.
            # Here we approximate:
            sigma = torch.relu(pos_out[..., 0]) + 1e-6 # Ensure positive density
            
            # Dir MLP
            dir_input = torch.cat([feature, dirs_pe], dim=-1)
            color = self.dir_mlp(dir_input)
            
            rgb_list.append(color)
            sigma_list.append(sigma)
            
        rgb = torch.cat(rgb_list, dim=0)
        sigma = torch.cat(sigma_list, dim=0)
        
        # Volume Rendering
        deltas = z_vals[..., 1:] - z_vals[..., :-1]
        deltas = torch.cat([deltas, torch.tensor([1e10], device=rays_o.device).expand_as(deltas[..., :1])], dim=-1)
        
        alpha = 1.0 - torch.exp(-sigma * deltas)
        transmittance = torch.cumprod(torch.cat([torch.ones_like(alpha[..., :1]), 1.0 - alpha[..., :-1]], dim=-1), dim=-1)
        
        weights = alpha * transmittance
        rgb_rendered = torch.sum(weights.unsqueeze(-1) * rgb, dim=1)
        
        return rgb_rendered, sigma, z_vals

def train_step(model, rays_o, rays_d, gt_rgb, optimizer):
    optimizer.zero_grad()
    rgb_pred, sigma, z_vals = model(rays_o, rays_d, n_samples=128)
    loss = torch.mean((rgb_pred - gt_rgb) ** 2)
    loss.backward()
    optimizer.step()
    return loss.item()
```

## Subtopics

### [HashNeRF Parameters](3068-hashnerf-parameters.md)
HashNeRF parameters configure the training loop and model architecture for the HashNeRF method, which builds on [NeRF Hyperparameters](3065-nerf-hyperparameters.md). Unlike standard NeRF which relies on positional encoding or fixed-resolution grids, HashNeRF uses a multi-resolution hashed embedding table indexed by voxel coordinates, requiring specific settings for the hash map size, loss regularization, and dataset-specific loading behaviors.

### [NeRF Initialization Strategies](8674-nerf-initialization-strategies.md)
This method initializes the weights of a classical image-based Neural Radiance Field (NeRF) using parameters generated by a separate model trained on 3D point clouds, specifically the Points2NeRF hypernetwork. It solves the problem of slow convergence and high computational cost associated with training NeRFs from random initialization by providing a structured starting point that approximates the target scene geometry and appearance. Unlike standard random initialization, this approach trades a significant upfront generation time for a drastic reduction in the number of epochs required for the final NeRF to reach comparable quality.

### [Mega-NeRF Training Configuration](8766-mega-nerf-training-configuration.md)
Mega-NeRF is a training configuration for Neural Radiance Fields (NeRF) designed to accelerate training and reduce model query costs for large-scale scenes. It differs from standard NeRF by using a single shared Multi-Layer Perceptron (MLP) for both coarse and fine sampling stages, which reduces model size and saves 25% of model queries per ray by reusing coarse outputs during the fine rendering stage.

### [Alpha Invariance / Inverse Scaling](10127-alpha-invariance-inverse-scaling.md)
Alpha invariance is a property in neural volume rendering where the opacity (alpha value) of a discretized segment remains constant despite changes in the scene's physical scale. It solves the problem that the magnitude of the optimized volume density $\sigma(\mathbf{x})$ is intrinsically tied to the domain of integration (distance along the ray), meaning that scaling the scene requires inversely scaling the density to preserve the visual appearance of local geometry. Unlike Multi-Plane Images (MPIs) which fix discretization in advance, NeRF's continuous density parameterization allows flexible resampling but requires this specific handling to maintain robust view synthesis across arbitrary scene scales.

### [Batch size scaling for computational parity](10942-batch-size-scaling-for-computational-parity.md)
Not given in the sources.

### [Model Architecture Modifications](11313-model-architecture-modifications.md)
Model architecture modifications in NeRF training refer to structural changes in the neural network components or the introduction of auxiliary networks to address specific rendering artifacts, such as overfitting in sparse views or the lack of depth-of-field (DoF) modeling. These modifications differ from hyperparameter tuning or regularization by altering the computational graph or the set of learnable parameters to better represent the underlying scene physics or geometry.

### [Regularization Techniques](11314-regularization-techniques.md)
Not given in the sources.

### Simple-RF Loss Function
The total loss function used to train the Simple-RF models, combining multiple regularization terms to mitigate floaters and improve geometry in sparse input scenarios [S6]. During the first 10k iterations of training, only impose $L_m$, $L_a$, and $L_{sd}$. Do not impose $L_{aug}$ and $L_{cfc}$ [S6]. After 10k iterations, impose all loss terms including $L_{aug}$ and $L_{cfc}$ [S6]. The formula is $L = \lambda_m L_m + \lambda_a L_a + \lambda_{sd} L_{sd} + \lambda_{aug} L_{aug} + \lambda_{cfc} L_{cfc} + \lambda_{mc} L_{mc}$ [S6]. Parameters: $\lambda_{sd} = 0.1$, $\lambda_{aug} = 0.1$, $\lambda_{cfc} = 0.1$ [S6]. Pitfall: $L_{cfc}$ and $L_{mc}$ are imposed only for the main NeRF and augmented TensoRF models respectively, not for all variants simultaneously in the same way [S6].

### NeRF Training Configuration for View-Dependent Artifacts
Mitigates direction-dependent flickering artifacts on synthetic datasets caused by overfitting of the view-dependent MLP [S4, unofficial]. Choose a direction-dependent cache resolution between $16^3$ and $64^3$, which uses interpolation at lookup by default [S4, unofficial]. Alternatively, adjust the Fourier feature encoding of view directions at training time by setting $L = 1$ (instead of $L = 4$ used in the baseline) [S4, unofficial]. Parameter: Fourier encoding dimensionality for directions ($L$): 1 (best for NeRF 360 Synthetic dataset); baseline uses $L = 4$ [S4, unofficial]. Number: PSNR with $L=1$: 29.748dB, $L=2$: 29.646dB, $L=3$: 29.449dB (using 8-layer 256 hidden MLP for position, 4-layer 128 MLP for direction) [S4, unofficial].

### Asymmetric Predicted Normal Loss
Apply an asymmetric predicted normal loss to tie predicted normals $\tilde{\mathbf{n}}$ to geometry normals $\mathbf{n}$ and rendering weights $w$, allowing separate multipliers for gradients flowing from geometry normals and weights to predicted normals, and vice versa [S7]. Formula: Compute asymmetric loss $\mathcal{L}_{\text{pred}} = \lambda_n \mathcal{L}_p(w, \mathbf{n}, \text{sg}(\tilde{\mathbf{n}})) + \lambda_{\tilde{n}} \mathcal{L}_p(\text{sg}(w), \text{sg}(\mathbf{n}), \tilde{\mathbf{n}})$, where $\mathcal{L}_p(w, \mathbf{n}, \tilde{\mathbf{n}}) = \sum_{i=1}^N w^{(i)} \|\mathbf{n}^{(i)} - \tilde{\mathbf{n}}^{(i)}\|^2$ and $\text{sg}(\cdot)$ is stop-gradient [S7]. Parameter: $\lambda_n$: Multiplier for the term tying predicted normals to geometry normals. Typical value: $10^{-3}$ [S7]. Pitfall: Using standard symmetric predicted normal loss tends to oversmooth or underconstrain geometry, preventing normal vectors from converging to the correct solution [S7].

### RoGUENeRF Training and Loss
Pre-train on LLFF dataset using NeRF renders for 3000 epochs (~5 days) [S8]. Fine-tune on new scenes/models for 1 hour per scene [S8]. Formula: $L = |\hat{H}_i - H_i|_1 + 10^{-3}|\omega(\hat{H}_i) - \omega(H_i)|_1$, where $\hat{H}_i$ is predicted enhanced image, $H_i$ is GT, and $\omega()$ is pre-trained VGG-19 [S8]. Parameters: Crop size: 512x512, Batch size: 4, Learning rate: $1 \times 10^{-4}$, Optimizer: Adam, Hardware: 4x Nvidia V100 GPUs [S8].

### Depth-Guided NeRF Training Configuration
Training configuration for a NeRF model using depth priors, including architecture modifications, loss weights, and optimizer settings [S9]. Add dropout layers with probability 0.1 after each of the first eight fully connected ReLU-activated layers that encode the positional encoding [S9]. Parameters: Depth loss weight ($\lambda$): 0.007 for all scenes except scene 710, where it is 0.02. Uncertainty weight ($\gamma$): 1. Optimizer: Adam. Initial learning rate: 5e-4. Learning rate schedule: Reduce to 5e-5 after 80% of steps. Weight decay: 1e-6. Total training steps: 500,000. Depth scale learning rate: 1e-7 [S9].

## Operating characteristics
*   **Time Complexity:** Training time scales with the number of iterations, samples per ray, and model capacity. Quadruples when increasing sampling resolution from 32x32 to 64x64 [S2].
*   **Memory Complexity:** Scales with batch size, chunk size, and model parameters. Chunked processing is required to fit large scenes into GPU memory [S5, unofficial].
*   **Measured Performance:** On the Blender dataset with 256 hidden units, mip-NeRF achieves PSNR 33.09, SSIM 0.961, LPIPS 0.043, Time 2.89 hrs, Params 0.61M [S1]. Our Model (with proposal) achieves PSNR 32.96, SSIM 0.960, LPIPS 0.043, Time 1.86 hrs, Params 0.84M [S1]. On the Lego scene (Blender synthetic) after 200k iterations, PSNR is ~31–32 dB, SSIM is ~0.96, training time is ~12–15 hours on a single GPU [S3, unofficial].
*   **Strengths:** Hierarchical sampling improves convergence speed and image quality [S3, unofficial]. Proposal networks significantly reduce training time compared to mip-NeRF [S1].
*   **Limitations:** High computational cost for large scenes. View-dependent artifacts can occur without specific regularization [S4, unofficial].
*   **When to Use:** Standard NeRF training for synthetic datasets (Blender, LLFF) where high-quality novel view synthesis is required [S3, unofficial][S4, unofficial].
*   **When Not to Use:** Real-time applications without caching/acceleration structures [S4, unofficial]. Large-scale unbounded scenes without modifications like Mega-NeRF or HashNeRF [S1].

## Background
Originated from the NeRF paper [S3, unofficial] and extended by mip-NeRF [S1] and FastNeRF [S4, unofficial]. The standard configuration uses an 8-layer MLP for position and 4-layer MLP for direction, with Adam optimizer and hierarchical sampling [S3, unofficial][S4, unofficial]. It is widely used for synthetic dataset evaluation (Blender, LLFF) [S1][S3, unofficial][S4, unofficial].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] Non-line-of-Sight Imaging via Neural Transient Fields, https://doi.org/10.1109/tpami.2021.3076062, fetched 2026-10-26
[S3] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S4] FastNeRF by Garbin et al., https://arxiv.org/abs/2103.10380, fetched 2026-10-26 (unofficial)
[S5] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
[S6] Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions, https://arxiv.org/abs/2404.19015v5, fetched 2026-10-27
[S7] NeRF-Casting: Improved View-Dependent Appearance with Consistent Reflections, https://arxiv.org/abs/2405.14871v1, fetched 2026-10-26
[S8] RoGUENeRF: A Robust Geometry-Consistent Universal Enhancer for NeRF, https://doi.org/10.1007/978-3-031-73254-6_4, fetched 2026-10-29
[S9] Depth-Guided NeRF Training via Earth Mover’s Distance, https://link.springer.com/chapter/10.1007/978-3-031-73039-9_1, fetched 2026-10-27
