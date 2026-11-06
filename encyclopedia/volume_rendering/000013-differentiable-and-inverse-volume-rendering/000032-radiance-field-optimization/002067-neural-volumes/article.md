# Neural Volumes

Also called: Deformable Volumes, Neural Volumes: Learning Dynamic Renderable Volumes from Images, Volumetric Reconstruction with Ray Potentials

## Overview
Neural Volumes (NV) is a method for synthesizing novel views of dynamic scenes by optimizing a deep 3D convolutional network to predict a discretized RGB$\alpha$ voxel grid and a 3D warp field [S1, S2]. It solves the problem of rendering semi-transparent, dynamic objects (such as smoke or hair) from sparse multi-view video inputs without requiring explicit geometry reconstruction or depth sensors [S2]. Unlike implicit neural radiance fields that map coordinates to color/density via MLPs, NV uses an explicit voxel grid, trading memory for renderable structure and enabling real-time inference at the cost of fixed resolution limits [S1, S2]. It sets itself apart from static voxel methods like DeepVoxels by incorporating a learned warp field to model non-rigid motion and from MLP-based methods by maintaining an interpretable, differentiable volumetric representation [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A synchronized sequence of $K$ camera views (typically $K=3$ orthogonal views) of a dynamic scene within a bounded volume. Input images are downsampled (e.g., $334 \times 512$) [S2].
*   **Outputs:** Novel view images rendered from arbitrary camera poses and time steps [S2].

**Data Structures:**
1.  **Voxel Grid ($V$):** A discrete 3D grid of resolution $D \times D \times D$ (typically $128^3$) storing RGBA values. Each voxel $\mathbf{x}$ maps to $\mathbf{V}(\mathbf{x}) = (\mathbf{V}_{\text{rgb}}(\mathbf{x}), \mathbf{V}_{\alpha}(\mathbf{x}))$, where $\mathbf{V}_{\text{rgb}} \in \mathbb{R}^3$ and $\mathbf{V}_{\alpha} \in \mathbb{R}$ (differential opacity, range $[0, \infty)$) [S2].
2.  **Warp Grid ($W$):** A discrete 3D grid of resolution $32^3$ storing inverse warp vectors. This grid encodes the location within the template voxel grid from which color/opacity should be sampled, effectively modeling motion and increasing effective resolution [S1, S2].
3.  **Latent Code ($z$):** A 256-dimensional vector representing the scene state at a specific time instant [S2].

**Network Architecture:**
1.  **Encoder:** A CNN with dedicated branches for each of the $K$ input views. Outputs are concatenated and encoded into a latent code $z$. To ensure a smooth latent space, the encoder outputs parameters ($\mu, \sigma$) of a diagonal 256-dimensional Gaussian. A sample $z$ is generated via the reparameterization trick: $z = \mu + \sigma \odot \epsilon$, where $\epsilon \sim \mathcal{N}(0, I)$ [S2].
2.  **Decoder:** A 3D deconvolutional network (or linear basis decoder) that maps the latent code $z$ (optionally concatenated with a conditioning variable $c$) to the Voxel Grid $V$ and Warp Grid $W$ [S1, S2][S4, unofficial]. The decoder uses softplus activation to ensure non-negative opacity [S4, unofficial].

**Rendering (Differentiable Ray Marching):**
For a camera ray $\mathbf{r}$, the system performs front-to-back alpha compositing. Instead of sampling directly from $V$, it first samples the warp field to find the source location:
$$ \mathbf{x}_{\text{src}} = W^{-1}(\mathbf{r}(t)) $$
where $W^{-1}$ is the inverse warp field sampled at the ray position $\mathbf{r}(t)$. The color and opacity are then retrieved from the template volume:
$$ \mathbf{V}_{\text{sampled}} = V(\mathbf{x}_{\text{src}}) $$
The accumulated color $I_{\text{rgb}}$ and opacity $I_{\alpha}$ for pixel $p$ are computed as [S4, unofficial]:
$$ I_{\text{rgb}}(p) = \sum_{j} \mathbf{V}_{\text{rgb}}(r_j) \cdot \mathbf{V}_{\alpha}(r_j) \cdot \prod_{k<j} (1 - \mathbf{V}_{\alpha}(r_k)) $$
$$ I_{\alpha}(p) = 1 - \prod_{j} (1 - \mathbf{V}_{\alpha}(r_j)) $$
where $r_j$ are the sample points along the ray [S4, unofficial].

**Training:**
The network is trained end-to-end by minimizing the reconstruction loss against input images. The loss function is [S4, unofficial]:
$$ \mathcal{L}(\theta) = \| I_{\text{rgb}} - I^*_{\text{rgb}} \|_2^2 + \beta \cdot D_{KL}(z \| \mathcal{N}(0,1)) + \lambda_{tv} J_{tv}(\mathbf{V}_{\alpha}) + \lambda_B J_B(I_{\alpha}) $$
*   $D_{KL}$: KL-divergence regularization for the latent code [S2].
*   $J_{tv}$: Total variation regularization on opacity to encourage sharp boundaries [S4, unofficial].
*   $J_B$: Beta distribution prior on exit opacity to encourage binary transparency (fully transparent or opaque) [S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class NeuralVolumes:
    def __init__(self, voxel_res=128, warp_res=32, latent_dim=256, num_cameras=3):
        self.encoder = Encoder(num_cameras=num_cameras, latent_dim=latent_dim)
        self.decoder = Decoder(voxel_res=voxel_res, warp_res=warp_res, latent_dim=latent_dim)
        self.voxel_res = voxel_res
        self.warp_res = warp_res

    def encode(self, images):
        """
        images: Tensor of shape (B, K, H, W, 3)
        Returns: z (B, latent_dim)
        """
        # Encoder branches for each camera
        features = []
        for i in range(images.shape[1]):
            cam_feat = self.encoder.branches[i](images[:, i])
            features.append(cam_feat)
        
        # Concatenate and encode to latent space
        combined = torch.cat(features, dim=1)
        mu, log_var = self.encoder.head(combined)
        
        # Reparameterization trick
        std = torch.exp(0.5 * log_var)
        eps = torch.randn_like(std)
        z = mu + eps * std
        
        return z

    def decode_and_render(self, z, camera_params):
        """
        z: Latent code (B, latent_dim)
        camera_params: Dictionary containing ray origins and directions
        Returns: rendered_images (B, H, W, 3), rendered_opacities (B, H, W, 1)
        """
        # Generate Voxel Grid and Warp Grid
        V_rgb, V_alpha, W_inv = self.decoder(z)
        
        # Differentiable Ray Marching
        # Note: In practice, this involves sampling W_inv at ray positions
        # and then indexing V with the warped coordinates.
        
        # 1. Sample Warp Field
        # ray_positions: (B, N_samples, 3)
        # W_inv_sampled: (B, N_samples, 3) representing the source coordinates
        W_inv_sampled = F.grid_sample(
            W_inv, ray_positions, mode='bilinear', align_corners=True
        )
        
        # 2. Sample Volume using Warped Coordinates
        # V_sampled: (B, N_samples, 4) -> [R, G, B, Alpha]
        V_sampled = F.grid_sample(
            torch.cat([V_rgb, V_alpha.unsqueeze(1)], dim=1),
            W_inv_sampled,
            mode='bilinear',
            align_corners=True
        )
        
        V_rgb_sampled = V_sampled[:, :3, :, :]
        V_alpha_sampled = V_sampled[:, 3:, :, :]
        
        # 3. Front-to-Back Alpha Compositing
        # T_i = exp(-sum(sigma_j * delta_j)) approximated by product of (1-alpha)
        # Assuming uniform step size or pre-computed deltas
        
        # Initialize transmittance
        T = torch.ones_like(V_alpha_sampled[:, 0, :, :]) # (B, N_samples)
        
        accumulated_color = torch.zeros_like(V_rgb_sampled[:, 0, :, :])
        accumulated_opacity = torch.zeros_like(V_alpha_sampled[:, 0, :, :])
        
        for i in range(V_rgb_sampled.shape[1]): # Iterate over samples
            alpha = V_alpha_sampled[:, i, :, :]
            color = V_rgb_sampled[:, i, :, :]
            
            # Contribution: T_i * (1 - exp(-sigma * delta)) * c
            # Here alpha is differential opacity, so contribution is alpha * T * color
            contribution = T * alpha * color
            accumulated_color += contribution
            
            # Update transmittance: T_{i+1} = T_i * (1 - alpha)
            T = T * (1 - alpha)
            
            # Accumulate opacity for visualization
            accumulated_opacity += alpha * T # Note: This logic varies by implementation
            
        # Final color is accumulated_color
        # Final opacity is 1 - T
        final_opacity = 1 - T
        
        return accumulated_color, final_opacity

    def train_step(self, images, camera_params, optimizer):
        optimizer.zero_grad()
        z = self.encode(images)
        pred_rgb, pred_alpha = self.decode_and_render(z, camera_params)
        
        # Loss components
        recon_loss = F.mse_loss(pred_rgb, images[..., :3]) # Assuming images are RGB
        kl_loss = -0.5 * torch.sum(1 + log_var - mu**2 - log_var.exp())
        
        # Regularization terms (simplified)
        tv_loss = total_variation(pred_alpha)
        beta_loss = beta_prior(pred_alpha)
        
        loss = recon_loss + beta * kl_loss + lambda_tv * tv_loss + lambda_b * beta_loss
        loss.backward()
        optimizer.step()
        return loss.item()
```

## Subtopics

### [Voxel Grid Decoders](2219-voxel-grid-decoders.md)
Models the volume as a discrete 3D grid, trading memory for speed and enabling real-time rendering by avoiding per-ray MLP evaluations.

### [RenderNet Architecture](6691-rendernet-architecture.md)
A deep convolutional network that separates 3D feature extraction and 2D image decoding, using view-space resampling to make features camera-pose dependent.

### [GRU Volume Fusion](7543-gru-volume-fusion.md)
Uses Gated Recurrent Units with sparse 3D convolutions to incrementally aggregate local feature volumes into a global radiance field, adapting only relevant voxels.

## Operating characteristics
*   **Time Complexity:** Training is slow, requiring at least 12 hours per scene [S1]. Inference is real-time once the volume is decoded [S2].
*   **Memory Complexity:** High memory usage due to the explicit $128^3$ voxel grid and $32^3$ warp grid [S1].
*   **Performance:** Captures detailed volumetric geometry and appearance but is limited by the fixed voxel resolution, leading to blurry results for fine details compared to higher-resolution or implicit methods [S1, S3].
*   **Strengths:** Handles semi-transparent materials (smoke, hair) and dynamic scenes; interpretable volume; real-time inference [S2].
*   **Limitations:** Cannot scale to high resolutions without prohibitive memory costs; limited to bounded volumes with distinct backgrounds [S1].
*   **When to use:** Dynamic scenes with semi-transparent elements where real-time novel view synthesis is required and high-frequency detail is less critical than motion coherence [S2].
*   **When not to use:** Static scenes requiring ultra-high resolution; scenes with large unbounded backgrounds or complex lighting not captured by the voxel representation [S1].

## Background
Neural Volumes was introduced by Kellnhofer et al. in 2019 [S2]. It builds on earlier work in volumetric video and ray potentials, distinguishing itself by modeling voxel transparency rather than occupancy probability to enable rendering of dynamic, semi-transparent scenes [S2]. It is a baseline method in the family of differentiable volume rendering, preceding more recent implicit neural field approaches like NeRF [S1, S3].

## Sources
[S1] NeRF: Representing scenes as neural radiance fields for view synthesis, http://www.matthewtancik.com/nerf, fetched 2026-10-27
[S2] Neural Volumes: Learning Dynamic Renderable Volumes from Images, https://arxiv.org/abs/1906.07751, fetched 2026-10-26
[S3] Representing Volumetric Videos as Dynamic MLP Maps, https://arxiv.org/abs/2304.06717, fetched 2026-10-29
[S4] Neural Volumes: 从图像中学习动态可渲染体, https://www.alphaxiv.org/zh/abs/1906.07751, fetched 2026-10-30 (unofficial)
