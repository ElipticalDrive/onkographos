# SAMURAI

Also called: Joint optimization of shape, BRDF, cameras, and illuminations, Shape and Material from Unconstrained Real-World Arbitrary Image Collections

## Overview
SAMURAI is a joint optimization framework that estimates 3D shape, BRDF material properties, per-image illumination, camera poses, and camera intrinsics from unconstrained in-the-wild image collections with varying backgrounds and illuminations [S1]. It distinguishes itself from sibling methods like [UrbanIR](16200-urbanir.md) and [See-NeRF](18115-see-nerf.md) by explicitly decomposing radiance into reflectance and illumination, enabling relightable assets, and by employing a camera multiplex strategy to handle extremely coarse initial pose annotations without requiring perfect correspondences or GAN-style training [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** A collection of $q$ object images $C_j \in \mathbb{R}^{s_j \times 3}$ with varying resolutions, backgrounds, and illuminations [S1]. Rough camera pose quadrants annotated with 3 binary questions: Left vs. Right, Above vs. Below, Front vs. Back [S1]. Automatically estimated foreground segmentation masks $M_j \in \{0, 1\}^{s \times 1}$ (e.g., via U2-Net), which may be noisy [S1].
*   **Outputs:** A 3D neural volume representing shape and BRDF, explicit meshes with BRDF textures, and optimized per-image camera parameters and latent illumination vectors [S1].

**Data Structures and Layout**
*   **Neural Volume:** At each point $\mathbf{x} \in \mathbb{R}^3$, the network predicts:
    *   BRDF parameters $\mathbf{b} \in \mathbb{R}^5$ for the Cook-Torrance model: base color $\mathbf{bc} \in \mathbb{R}^3$, metallic $b_m \in \mathbb{R}^1$, roughness $b_r \in \mathbb{R}$ [S1].
    *   Unit-length surface normal $\mathbf{n} \in \mathbb{R}^3$ [S1].
    *   Volume density $\sigma \in \mathbb{R}$ [S1].
*   **Per-Image Latent Illumination:** Vectors $\mathbf{z}_{jl} \in \mathbb{R}^{128}$ for each image $j$ [S1].
*   **Camera Parameters:** Stored as offsets to initial parameters using a 'look-at' parameterization:
    *   3D look-at vector $\mathbf{r}_j \in \mathbb{R}^3$ [S1].
    *   Scalar up rotation $u_j \in \mathbb{R}[-\pi, \pi]$ [S1].
    *   3D camera position $\mathbf{t}_j \in \mathbb{R}^3$ [S1].
    *   Focal length $f_j \in \mathbb{R}$, stored in compressed form $\hat{f}_j = f_j / \sqrt{h}$, where $h$ is image height in pixels [S1].
*   **Network Architecture:** A single "Fine/Decomposition Network" (no coarse network) consisting of an MLP with 8 layers of 128 features [S1].
    *   **Density Branch:** A layer with 1 feature output for density $\sigma$ [S1].
    *   **Radiance Branch:** An MLP with a hidden layer for view/appearance conditioned radiance [S1].
    *   **BRDF Decoder:** Compresses main network features to 16 features, then expands to BRDF parameters [S1].

**Ray Sampling and Bounds**
*   **Sampling:** 128 points sampled along the ray in fixed steps [S1].
*   **Bounds:** Defined by a world-space sphere with radius $v$ (diameter 1). For a camera origin $\mathbf{o}$, the near bound is $|\mathbf{o}| - v$ and the far bound is $|\mathbf{o}| + v$ [S1]. This allows arbitrary camera distances, unlike fixed near/far planes in standard NeRF [S1].

**Optimization Steps**
1.  **Initialization:**
    *   Initialize camera poses based on quadrant annotations and an initial Field of View of 53.13 degrees [S1].
    *   Initialize Fourier embedding frequencies with logarithmic spacing, adding Gaussian distributed noise as offsets to prevent axis-aligned artifacts [S1].
2.  **Camera Multiplex Setup:**
    *   For each image, optimize 4 randomly jittered poses around the quadrant center [S1].
    *   Maintain a memory bank and momentum for camera multiplex weights $\mathbf{S}_j \in \mathbb{R}^4$ across batches [S1].
3.  **Forward Pass & Rendering:**
    *   Cast rays through the volume. Compute radiance using the predicted BRDF, illumination $\mathbf{z}_{jl}$, and view direction.
    *   Compute volumetric rendering for color $\tilde{\mathbf{c}}$ (direct color, used early) and $\hat{\mathbf{c}}$ (BRDF-based color) [S1].
4.  **Loss Calculation:**
    *   Compute individual losses (defined below).
    *   Apply **Camera Multiplex Weights**: $\mathbf{S}_j = \text{softmax}(-\lambda_s \mathbf{L}_{\text{Mask}, ij})$, where $\lambda_s$ increases during optimization. Update network loss: $\mathbf{L}_{\text{Network}, j} = \sum_i S_{ji} \mathbf{L}_{\text{Network}, ij}$ [S1].
    *   Apply **Posterior Scaling**: Maintain a circular buffer of ~1000 recent mask and image losses. Calculate mean $\mu_l$ and standard deviation $\sigma_l$. Compute scaling factor $sp_j = \max(\tanh(\frac{\mu_l - (\mathbf{L}_{\text{Mask}, j} + \mathbf{L}_{\text{Image}, j})}{\sigma_l}) + 1, 1)$. Apply: $\mathbf{L}_{\text{Network}, j} = sp_j \mathbf{L}_{\text{Network}, j}$ [S1].
    *   *Note:* Camera optimization uses the same losses but does *not* apply posterior scaling to allow badly initialized poses to recover [S1].
5.  **Backpropagation:**
    *   Optimize networks using Adam (LR $10^{-4}$, decay $10\times$ every 300k steps) [S1].
    *   Optimize cameras using Adam (LR $3 \times 10^{-3}$, decay $10\times$ every 70k steps) [S1].
    *   Schedule $\lambda_b$ (BRDF vs direct color), $\lambda_a$ (initialization loss), and $\lambda_c$ (resolution/multiplex reduction) [S1].

**Formulas**
*   **Mask Loss:** $\mathbf{L}_{\text{Mask}} = \mathbf{L}_{\text{BCE}} + \mathbf{L}_{\text{Background}}$, where $\mathbf{L}_{\text{Background}}$ forces background samples to 0 [S1].
*   **Initialization Loss:** $\mathbf{L}_{\text{Init}} = \mathbf{L}_{\text{MSE}}(C_s, \mathbf{bc})$, preventing illumination local minima [S1].
*   **Decomposition Sparsity:** $\mathbf{L}_{\text{Dec Sparsity}} = \frac{1}{N} \sum_i |e_i|$, where $e_i$ is the BRDF embedding [S1].
*   **Decomposition Smoothness:** $\mathbf{L}_{\text{Dec Smooth}} = \frac{1}{N} \sum_i |f(\theta; e_i) - f(\theta; e_i + \epsilon)|$, with $\epsilon \sim \mathcal{N}(0, 0.01^2)$ [S1].
*   **Total Network Loss:**
    $$ \mathbf{L}_{\text{Network}} = \lambda_b \mathbf{L}_{\text{Image}}(C_s, \tilde{\mathbf{c}}) + (1 - \lambda_b) \mathbf{L}_{\text{Image}}(C_s, \hat{\mathbf{c}}) + \mathbf{L}_{\text{Mask}} + \lambda_a \mathbf{L}_{\text{Init}} + \lambda_{\text{ndir}} \mathbf{L}_{\text{ndir}} + \lambda_{\text{Smooth}} \mathbf{L}_{\text{Smooth}} + \lambda_{\text{Dec Smooth}} \mathbf{L}_{\text{Dec Smooth}} + \lambda_{\text{Dec Sparsity}} \mathbf{L}_{\text{Dec Sparsity}} $$
    where $\mathbf{L}_{\text{Image}}$ is the Chabonnier loss $\sqrt{(g-p)^2 + 0.001^2}$ [S1].
*   **Camera Constraints:**
    *   Look-at loss: $\mathbf{L}_{\text{lookat}}$ (L1 loss on look-at vector $\mathbf{r}_j$) [S1].
    *   Bounds loss: $\mathbf{L}_{\text{Bounds}} = \max((v - |\mathbf{t}_j|)^2, 0)$ [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
import numpy as np

class SAMURAI:
    def __init__(self, images, masks, quadrants, num_images):
        self.num_images = num_images
        self.images = images
        self.masks = masks
        self.quadrants = quadrants
        
        # Initialize Network (8-layer MLP, 128 features)
        self.network = FineDecompositionNetwork()
        
        # Initialize Camera Parameters (Look-at parameterization)
        # r_j, u_j, t_j, f_j stored as offsets
        self.cam_r = nn.Parameter(torch.randn(num_images, 3))
        self.cam_u = nn.Parameter(torch.zeros(num_images))
        self.cam_t = nn.Parameter(torch.randn(num_images, 3))
        self.cam_f = nn.Parameter(torch.zeros(num_images)) # Compressed focal length
        
        # Camera Multiplex: 4 poses per image
        self.num_multiplex = 4
        self.cam_multiplex_weights = torch.ones(num_images, self.num_multiplex) / self.num_multiplex
        
        # Posterior Scaling Buffer
        self.loss_buffer = []
        self.max_buffer_size = 1000
        
        # Optimizers
        self.net_optimizer = optim.Adam(self.network.parameters(), lr=1e-4)
        self.cam_optimizer = optim.Adam([self.cam_r, self.cam_u, self.cam_t, self.cam_f], lr=3e-3)
        
        # Hyperparameters
        self.v = 0.5 # Sampling radius (diameter 1)
        self.epsilon_smooth = 0.01
        self.lambda_s = 1.0 # Scalar for multiplex softmax
        
    def get_camera_pose(self, idx, multiplex_idx=0):
        # Retrieve base parameters and apply offsets
        r = self.cam_r[idx]
        u = self.cam_u[idx]
        t = self.cam_t[idx]
        f_hat = self.cam_f[idx]
        
        # Reconstruct focal length from compressed storage
        # h is image height, assumed known or passed
        # f = f_hat * sqrt(h)
        # Note: In implementation, h must be tracked per image
        f = f_hat * np.sqrt(self.images[idx].shape[0]) 
        
        # Construct camera matrix from look-at, up rotation, position, focal length
        # This is a placeholder for the specific look-at to matrix conversion
        # described in the source, ensuring the camera looks at the origin/object
        cam_matrix = build_look_at_matrix(r, u, t, f)
        return cam_matrix

    def sample_rays(self, cam_matrix, img_h, img_w):
        # Generate rays based on camera matrix
        # Near bound: |o| - v, Far bound: |o| + v
        origin = cam_matrix[:3, 3]
        dist = torch.norm(origin)
        near = dist - self.v
        far = dist + self.v
        
        # Sample 128 points in fixed steps
        t_vals = torch.linspace(near, far, 128)
        directions = ... # Compute from pixel coordinates
        points = origin.unsqueeze(1) + directions.unsqueeze(1) * t_vals.unsqueeze(1)
        return points, directions, t_vals

    def forward(self, points, directions, img_idx, light_idx):
        # Encode points and directions (Fourier embedding with noise offsets)
        x_enc = self.network.pos_enc(points)
        d_enc = self.network.pos_enc(directions)
        
        # Main Network
        features = self.network.base(x_enc)
        
        # Density
        sigma = self.network.density_head(features)
        
        # BRDF Decoder
        brdf_features = self.network.brdf_compressor(features)
        brdf_params = self.network.brdf_expander(brdf_features)
        bc, bm, br = brdf_params # Base color, metallic, roughness
        
        # Radiance
        radiance = self.network.radiance_head(features, d_enc)
        
        # Illumination
        illumination = self.network.get_illumination(img_idx, light_idx)
        
        return sigma, bc, bm, br, radiance, illumination

    def compute_losses(self, pred_color, pred_sigma, pred_bc, img_color, mask, img_idx, cam_idx):
        # Image Loss (Chabonnier)
        l_image = torch.sqrt((pred_color - img_color)**2 + 0.001**2)
        
        # Mask Loss
        l_bce = ... # Binary Cross Entropy with estimated mask
        l_background = ... # Force background samples to 0
        l_mask = l_bce + l_background
        
        # Initialization Loss (MSE between input color and base color)
        l_init = torch.mean((img_color - pred_bc)**2)
        
        # BRDF Sparsity
        l_dec_sparsity = torch.mean(torch.abs(self.network.brdf_embeddings))
        
        # BRDF Smoothness
        eps = torch.randn_like(self.network.brdf_embeddings) * self.epsilon_smooth
        pred_smooth = self.network.brdf_decoder(self.network.brdf_compressor(self.network.base(self.network.pos_enc(...))))
        l_dec_smooth = torch.mean(torch.abs(pred_smooth - pred_smooth_with_noise))
        
        # Normal Direction Loss
        l_ndir = ... # Gradient of density for normals
        
        # Total Network Loss components
        # Note: lambda_b, lambda_a are scheduling variables
        l_network = (1 - self.lambda_b) * l_image + l_mask + self.lambda_a * l_init + \
                    l_dec_sparsity + l_dec_smooth + l_ndir
                    
        return l_network, l_mask, l_image, l_init

    def update_multiplex_weights(self, mask_losses):
        # mask_losses: tensor of shape (num_images, num_multiplex)
        # Sj = softmax(-lambda_s * LMask_ij)
        weights = torch.softmax(-self.lambda_s * mask_losses, dim=1)
        # Update with momentum/memory bank (simplified)
        self.cam_multiplex_weights = 0.9 * self.cam_multiplex_weights + 0.1 * weights
        self.lambda_s += 0.001 # Gradually increase

    def update_posterior_scaling(self, mask_losses, image_losses):
        # Combine losses
        total_losses = mask_losses + image_losses
        self.loss_buffer.append(total_losses)
        if len(self.loss_buffer) > self.max_buffer_size:
            self.loss_buffer.pop(0)
            
        buffer_tensor = torch.stack(self.loss_buffer)
        mu_l = torch.mean(buffer_tensor, dim=0)
        sigma_l = torch.std(buffer_tensor, dim=0)
        
        # sp_j = max(tanh((mu_l - L_j)/sigma_l) + 1, 1)
        scaling_factor = torch.clamp(torch.tanh((mu_l - total_losses) / (sigma_l + 1e-8)) + 1, min=1.0)
        return scaling_factor

    def step(self, img_idx, light_idx):
        # 1. Get Camera Pose (with multiplex jitter)
        cam_matrix = self.get_camera_pose(img_idx)
        
        # 2. Sample Rays
        points, directions, _ = self.sample_rays(cam_matrix, self.images[img_idx].shape[0], self.images[img_idx].shape[1])
        
        # 3. Forward Pass
        sigma, bc, bm, br, radiance, illumination = self.forward(points, directions, img_idx, light_idx)
        
        # 4. Render Color (Volume Rendering)
        pred_color = volume_render(points, sigma, radiance, illumination)
        
        # 5. Compute Losses
        l_net, l_mask, l_img, l_init = self.compute_losses(pred_color, sigma, bc, self.images[img_idx], self.masks[img_idx], img_idx, light_idx)
        
        # 6. Apply Multiplex Weights
        # (Assuming mask_losses for all multiplex cameras are computed)
        self.update_multiplex_weights(...) 
        
        # 7. Apply Posterior Scaling
        sp = self.update_posterior_scaling(l_mask, l_img)
        l_net_scaled = sp * l_net
        
        # 8. Backprop
        self.net_optimizer.zero_grad()
        l_net_scaled.backward()
        self.net_optimizer.step()
        
        # Camera Optimization (separate step, no posterior scaling)
        self.cam_optimizer.zero_grad()
        # Compute camera-specific losses (L_lookat, L_bounds, etc.)
        l_cam = ...
        l_cam.backward()
        self.cam_optimizer.step()
```

## Operating characteristics
*   **Time Complexity:** Rendering requires 128 samples per ray. The use of a single fine network (no coarse network) reduces memory and compute compared to two-stage NeRFs [S1]. However, camera multiplexing (4 poses per image) increases the number of forward passes per batch, requiring dynamic loss re-weighting [S1].
*   **Memory Complexity:** Stores per-image latent illumination vectors ($\mathbb{R}^{128}$) and camera parameters. The circular buffer for posterior scaling stores ~1000 loss elements [S1].
*   **Strengths:** Handles unconstrained in-the-wild images with varying backgrounds and illuminations. Does not require perfect camera poses or correspondences. Produces relightable 3D assets (meshes with BRDF) [S1]. Robust to noisy segmentation masks via posterior scaling [S1].
*   **Limitations:** Optimization is highly non-convex and requires careful scheduling of $\lambda$ variables. Camera multiplexing can be noisy if not properly weighted. Relies on automatic segmentation which may be imperfect [S1].
*   **When to use:** When given a collection of in-the-wild images with unknown/exact camera poses and varying lighting, and the goal is to create a relightable 3D model.
*   **When not to use:** When high-quality, controlled studio captures with known poses are available (standard NeRF or COLMAP-based methods are more efficient). When real-time rendering is required without pre-computation (NeRF is inherently slow) [S1].

## Background
SAMURAI was proposed by Boss et al. in 2022 [S1]. It addresses the limitations of prior methods like NeRF (which assumes fixed near/far planes and equidistant cameras) and Neural-PIL (which requires perfect camera poses and segmentation masks) [S1]. It builds on the architecture of NeRD and Neural-PIL but introduces flexible camera parameterization, camera multiplex optimization, and posterior scaling to handle the severe under-constrained nature of in-the-wild image collections [S1]. It is compared against SCNeRF, BARF, NeRF--, and GNeRF, highlighting its ability to work without correspondences, with coarse poses, in multiple illuminations, without double training, and without GANs [S1].

## Sources
[S1] SAMURAI: Shape and Material from Unconstrained Real-World Arbitrary Image Collections, https://doi.org/10.52202/068431-1914, fetched 2026-10-29
