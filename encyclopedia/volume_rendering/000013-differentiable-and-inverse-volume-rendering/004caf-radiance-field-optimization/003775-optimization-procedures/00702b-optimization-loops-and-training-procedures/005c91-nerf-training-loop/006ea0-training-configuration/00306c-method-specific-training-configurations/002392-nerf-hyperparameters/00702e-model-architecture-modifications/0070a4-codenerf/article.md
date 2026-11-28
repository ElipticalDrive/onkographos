# CodeNeRF

Also called: Disentangled Neural Radiance Fields for Object Categories

## Overview
CodeNeRF is an implicit 3D neural representation that learns the variation of object shapes and textures across a semantic category, enabling the synthesis of novel views for unseen objects from a single unposed input image [S2]. Unlike standard NeRF which is scene-specific, CodeNeRF disentangles shape and texture by learning separate latent embeddings, allowing for explicit editing of object geometry and appearance by modifying these codes [S2]. It differs from sibling methods like [PixelNeRF](13911-transformer-and-vision-model-integration.md) by using an auto-decoder architecture with decoupled embedding spaces rather than conditioning on image features, and from [NeRF-VAE](11991-nerf-vae.md) by optimizing latent codes at inference time rather than encoding them via an encoder network [S2].

## How it works
**Inputs and Outputs:**
*   **Training Inputs:** A set of $N$ images $\mathcal{V} = \{I_i, K_i, T_i\}_{i=1}^N$ depicting $M$ objects, where $K_i$ and $T_i$ are the intrinsic and extrinsic camera parameters for each image [S2].
*   **Inference Input:** A single unposed reference image $I_{test}$ of an unseen object [S2].
*   **Outputs:** Rendered novel views, estimated camera pose (azimuth $\phi$, elevation $\theta$, distance $\rho$), and disentangled shape $\mathbf{z}_s$ and texture $\mathbf{z}_t$ latent codes [S2].

**Data Structures:**
*   **Network Weights $\Theta$:** Parameters for two MLPs, $\text{F}^s_{\Theta_s}$ and $\text{F}^t_{\Theta_t}$ [S2].
*   **Latent Embeddings:**
    *   Shape code $\mathbf{z}_s \in \mathbb{R}^{D_s}$
    *   Texture code $\mathbf{z}_t \in \mathbb{R}^{D_t}$
    *   During training, a set $\{\mathbf{z}_s^j, \mathbf{z}_t^j\}_{j=1}^M$ is maintained for each object instance [S2].
*   **Camera Parameters:** Optimized as azimuth $\phi$, elevation $\theta$, and distance $\rho$ [S2].

**Network Architecture:**
The model uses a decoupled auto-decoder architecture [S2].
1.  **Shape Network $\text{F}^s_{\Theta_s}$:** Takes positional encoding of 3D coordinates $\gamma_x(\mathbf{x})$ and shape code $\mathbf{z}_s$ as input. It outputs volume density $\sigma$ and an intermediate feature vector $\mathbf{v} \in \mathbb{R}^{256}$ [S2].
2.  **Texture/Color Network $\text{F}^t_{\Theta_t}$:** Takes the feature vector $\mathbf{v}$, positional encoding of viewing direction $\gamma_d(\mathbf{d})$, and texture code $\mathbf{z}_t$ as input. It outputs view-dependent RGB color $\mathbf{c}$ [S2].

The combined radiance field is defined as:
$$
\text{F}^s_{\Theta_s}: (\gamma_x(\mathbf{x}), \mathbf{z}_s) \rightarrow (\sigma, \mathbf{v})
$$
$$
\text{F}^t_{\Theta_t}: (\mathbf{v}, \gamma_d(\mathbf{d}), \mathbf{z}_t) \rightarrow (\mathbf{c})
$$
$$
\text{F}_\Theta: \text{F}^t_{\Theta_t} \circ \text{F}^s_{\Theta_s}
$$
Positional encoding uses 10 frequencies for $\mathbf{x}$ and 4 frequencies for $\mathbf{d}$ [S2].

**Rendering Equation:**
Color is rendered using standard volumetric rendering along ray $\mathbf{r}(t) = \mathbf{c} + t\mathbf{d}$ [S2]:
$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$
where $\delta_i = t_{i+1} - t_i$ and $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ [S2].

**Training Procedure:**
1.  Initialize network weights $\Theta$ and latent codes $\{\mathbf{z}_s^j, \mathbf{z}_t^j\}_{j=1}^M$ [S2].
2.  For each training iteration:
    a. Sample a batch of 4094 rays using known intrinsic and extrinsic parameters [S2].
    b. Sample 64 points uniformly along each ray between near and far bounds [S2].
    c. For each point $\mathbf{x}$ on ray $\mathbf{r}$ associated with object $j$, compute $\sigma$ and $\mathbf{v}$ using $\text{F}^s_{\Theta_s}(\gamma_x(\mathbf{x}), \mathbf{z}_s^j)$ [S2].
    d. Compute $\mathbf{c}$ using $\text{F}^t_{\Theta_t}(\mathbf{v}, \gamma_d(\mathbf{d}), \mathbf{z}_t^j)$ [S2].
    e. Render pixel color $\hat{C}(\mathbf{r})$ [S2].
    f. Compute loss:
    $$
    \mathcal{L} = \sum_{\mathbf{r} \in \mathcal{R}} ||\hat{C}(\mathbf{r}) - C(\mathbf{r})||_2^2 + \frac{1}{\nu^2} \sum_{j=1}^M (||\mathbf{z}_s^j||_2^2 + ||\mathbf{z}_t^j||_2^2)
    $$
    where $\nu$ is a regularization parameter [S2].
    g. Update $\Theta$ and $\{\mathbf{z}_s^j, \mathbf{z}_t^j\}$ using AdamW [S2].

**Inference Procedure:**
Given a single unposed image $I_{test}$:
1.  Initialize shape code $\mathbf{z}_s$ and texture code $\mathbf{z}_t$ with the mean vector of the trained embeddings [S2].
2.  Initialize camera parameters $\phi, \theta, \rho$ with random or prior values [S2].
3.  Fix network weights $\Theta$ [S2].
4.  Optimize $\mathbf{z}_s, \mathbf{z}_t, \phi, \theta, \rho$ jointly by minimizing the photometric loss:
    $$
    \min_{\mathbf{z}_s, \mathbf{z}_t, \phi, \theta, \rho} \sum_{\mathbf{r} \in \mathcal{R}} ||\hat{C}(\mathbf{r}) - C(\mathbf{r})||_2^2 + \frac{1}{\nu^2} (||\mathbf{z}_s||_2^2 + ||\mathbf{z}_t||_2^2)
    $$
    using AdamW [S2].
5.  Run optimization for 299 iterations [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
import numpy as np

class CodeNeRF:
    def __init__(self, shape_dim=256, texture_dim=256, num_shape_freqs=10, num_view_freqs=4):
        """
        Initialize CodeNeRF model.
        """
        # Shape network: maps (pos_enc(x), z_s) -> (sigma, v)
        self.net_s = nn.Sequential(
            nn.Linear(3 * 2 * num_shape_freqs + shape_dim, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256 + 1) # v (256) + sigma (1)
        )
        
        # Texture network: maps (v, pos_enc(d), z_t) -> c
        self.net_t = nn.Sequential(
            nn.Linear(256 + 3 * 2 * num_view_freqs + texture_dim, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 3) # RGB color
        )
        
        self.num_shape_freqs = num_shape_freqs
        self.num_view_freqs = num_view_freqs

    def positional_encoding(self, x, num_freqs):
        """
        Standard positional encoding for NeRF.
        x: (..., 3)
        Returns: (..., 3 * 2 * num_freqs)
        """
        x = x.unsqueeze(-1) # (..., 3, 1)
        freq_bands = 2.0 ** torch.linspace(0, num_freqs - 1, num_freqs).to(x.device)
        # sin(2^i * pi * x) and cos(2^i * pi * x)
        pe = torch.cat([torch.sin(x * freq_bands * np.pi), torch.cos(x * freq_bands * np.pi)], dim=-1)
        return pe.view(*x.shape[:-2], -1)

    def forward_shape(self, x, z_s):
        """
        Forward pass through shape network.
        x: (N, 3)
        z_s: (N, shape_dim)
        Returns: sigma (N, 1), v (N, 256)
        """
        pe_x = self.positional_encoding(x, self.num_shape_freqs)
        input_s = torch.cat([pe_x, z_s], dim=-1)
        out = self.net_s(input_s)
        sigma = out[:, :1]
        v = out[:, 1:]
        return sigma, v

    def forward_texture(self, v, d, z_t):
        """
        Forward pass through texture network.
        v: (N, 256)
        d: (N, 3)
        z_t: (N, texture_dim)
        Returns: c (N, 3)
        """
        pe_d = self.positional_encoding(d, self.num_view_freqs)
        input_t = torch.cat([v, pe_d, z_t], dim=-1)
        c = self.net_t(input_t)
        return c

    def render_ray(self, ray_o, ray_d, z_s, z_t, near=0.0, far=1.0, num_samples=64):
        """
        Render a single ray.
        """
        t_vals = torch.linspace(near, far, num_samples)
        # Uniform sampling
        z_samples = t_vals
        
        # Expand for batch if needed, assuming single ray here for clarity
        # In practice, batched ray origins and directions are used
        pts = ray_o + ray_d.unsqueeze(0) * z_samples.unsqueeze(1) # (num_samples, 3)
        
        sigma, v = self.forward_shape(pts, z_s.unsqueeze(0).expand(num_samples, -1))
        c = self.forward_texture(v, ray_d.unsqueeze(0).expand(num_samples, -1), z_t.unsqueeze(0).expand(num_samples, -1))
        
        # Volume rendering
        deltas = z_samples[1:] - z_samples[:-1]
        alpha = 1.0 - torch.exp(-torch.relu(sigma[:-1]) * deltas)
        T = torch.cumprod(torch.cat([torch.ones_like(alpha[:1]), 1.0 - alpha + 1e-10], dim=0), dim=0)[:-1]
        weights = alpha * T
        color = torch.sum(weights.unsqueeze(-1) * c[:-1], dim=0)
        
        return color

    def train_step(self, rays, gt_colors, z_s, z_t, nu=1.0):
        """
        Single training step.
        rays: (B, 2, 3) [origins, directions]
        gt_colors: (B, 3)
        Returns: loss
        """
        ray_o, ray_d = rays[:, 0], rays[:, 1]
        
        # Render
        pred_colors = []
        for i in range(ray_o.shape[0]):
            # Render each ray individually or batched if implemented
            # Simplified batched render for pseudocode structure
            pass 
        
        # Vectorized batched render implementation would go here
        # For brevity, assuming a batched render function exists or loop is used
        # Let's assume a batched render function `render_batch`
        
        # Re-implementing simple batched render inline for completeness
        # This is a simplified version. Real implementation uses torch.vmap or loops
        # for robustness.
        
        # Placeholder for batched rendering logic based on forward methods
        # In a real implementation, one would stack all points from all rays
        # and process them through the MLPs.
        
        # Loss calculation
        # L = sum(||C(r) - C_hat(r)||^2) + reg
        photometric_loss = torch.mean((pred_colors - gt_colors) ** 2)
        reg_loss = (1.0 / nu**2) * (torch.mean(z_s**2) + torch.mean(z_t**2))
        loss = photometric_loss + reg_loss
        
        return loss

    def inference_optimize(self, single_image, gt_color, z_s_init, z_t_init, num_iters=299, lr_latent=1e-2, lr_cam=1e-2):
        """
        Optimize latent codes and camera pose for a single image.
        """
        # Initialize optimizers
        # Camera params: phi, theta, rho
        # We need to define how camera params map to ray origins/directions
        # This is a simplified placeholder for the optimization loop
        
        z_s_opt = z_s_init.clone().detach().requires_grad_(True)
        z_t_opt = z_t_init.clone().detach().requires_grad_(True)
        
        # Camera parameters optimization
        # phi, theta, rho
        phi = torch.tensor(0.0, requires_grad=True)
        theta = torch.tensor(0.0, requires_grad=True)
        rho = torch.tensor(1.0, requires_grad=True)
        
        optimizer_latent = optim.Adam([z_s_opt, z_t_opt], lr=lr_latent)
        optimizer_cam = optim.Adam([phi, theta, rho], lr=lr_cam)
        
        for i in range(num_iters):
            optimizer_latent.zero_grad()
            optimizer_cam.zero_grad()
            
            # Construct ray from camera params
            # This requires a camera model function
            # ray_o, ray_d = get_ray_from_pose(phi, theta, rho)
            
            # Render
            # pred_color = render_ray(ray_o, ray_d, z_s_opt, z_t_opt)
            
            # Loss
            # loss = (pred_color - gt_color)**2
            
            # loss.backward()
            # optimizer_latent.step()
            # optimizer_cam.step()
            
            pass # Optimization loop structure
        
        return z_s_opt, z_t_opt, phi, theta, rho

```

## Operating characteristics
*   **Time Complexity:** Training requires joint optimization of network weights and latent codes for all objects, which is computationally intensive. Inference requires iterative optimization (299 iterations) for each new object instance [S2].
*   **Memory Complexity:** Stores latent codes for each object in the training set. The network size is comparable to standard NeRF but with two branches [S2].
*   **Performance:** On the ShapeNet-SRN benchmark, CodeNeRF achieves competitive PSNR and SSIM scores for novel view synthesis and few-shot reconstruction, often outperforming methods that require known camera poses at test time [S2].
*   **Strengths:** Disentangles shape and texture, allowing independent editing. Does not require known camera poses at inference. Generalizes well to unseen objects [S2].
*   **Limitations:** Inference is slow due to iterative optimization. Performance depends on the quality of the learned category prior. Convergence of camera pose optimization is not guaranteed (85% convergence rate in experiments) [S2].
*   **When to use:** When single-image 3D reconstruction of unseen objects from a known category is required, and explicit control over shape/texture is desired.
*   **When not to use:** For scene-specific NeRFs where high-fidelity reconstruction of a single static scene is needed without editing requirements, or when real-time inference is critical [S2].

## Background
CodeNeRF was introduced by Jiang et al. in 2021 [S2]. It builds on the original NeRF [S2] and draws inspiration from SRN [S2] and DeepSDF [S2]. It addresses the limitation of NeRF being scene-specific by learning a category-level representation. The method was evaluated on the ShapeNet-SRN dataset for cars and chairs [S2].

## Sources
[S1] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
[S2] CodeNeRF: Disentangled Neural Radiance Fields for Object Categories, https://doi.org/10.1109/iccv48922.2021.01271, fetched 2026-10-26
[S3] Semantically-aware Neural Radiance Fields for Visual Scene Understanding: A Comprehensive Review, https://link.springer.com/article/10.1007/s11263-025-02663-5, fetched 2026-10-27
