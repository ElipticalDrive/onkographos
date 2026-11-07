# GRAF

Also called: Generative Radiance Fields, GRAF

## Overview
GRAF is a generative model for 3D-aware image synthesis that learns a conditional radiance field from a collection of unposed 2D images using an adversarial framework [S1, unofficial]. It allows explicit control over camera rotation, translation, shape, and appearance by mapping latent codes and camera parameters to image patches via differentiable volume rendering [S1, unofficial]. Unlike voxel-based approaches (e.g., HoloGAN, PLATONIC GAN) which suffer from memory limitations and discretization artifacts, GRAF uses a continuous representation that scales well with image resolution and incorporates 3D consistency by design without learned projections [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Generator Input:** Camera matrix $K$, camera pose $\xi = [R|t]$, 2D sampling pattern $\nu = (u, s)$, shape code $z_s \in \mathbb{R}^{M_s}$, appearance code $z_a \in \mathbb{R}^{M_a}$ [S1, unofficial].
*   **Generator Output:** A predicted image patch $P_0$ of size $K \times K$ pixels [S1, unofficial].
*   **Discriminator Input:** A real image patch $P$ extracted from a real image $I$ and the predicted patch $P_0$ [S1, unofficial].
*   **Discriminator Output:** A scalar score indicating realness [S1, unofficial].

**Data Structures:**
1.  **Conditional Radiance Field $g_\theta$:** A fully connected neural network (MLP) [S1, unofficial].
    *   **Shape Encoding Head $h_\theta$:** Maps $(\gamma(x), z_s)$ to shape encoding $h \in \mathbb{R}^H$.
    *   **Density Head $\sigma_\theta$:** Maps $h$ to volume density $\sigma \in \mathbb{R}^+$.
    *   **Color Head $c_\theta$:** Maps $(h, \gamma(d), z_a)$ to RGB color $c \in \mathbb{R}^3$.
    *   All heads use ReLU activations [S1, unofficial].
2.  **Discriminator $D_\phi$:** A convolutional neural network (CNN) [S1, unofficial].
    *   Uses spectral normalization and instance normalization [S1, unofficial].
    *   Takes a $K \times K$ patch as input [S1, unofficial].

**Numbered Steps:**

1.  **Sample Inputs:**
    *   Sample camera pose $\xi = [R|t]$ from distribution $p_\xi$. In experiments, use a uniform distribution on the upper hemisphere for camera location facing the origin, varying distance uniformly [S1, unofficial].
    *   Sample shape code $z_s \sim p_s$ and appearance code $z_a \sim p_a$. In experiments, use standard Gaussian distributions for both [S1, unofficial].
    *   Sample 2D sampling pattern $\nu = (u, s)$ where center $u \sim U(\Omega)$ (uniform over image domain $\Omega$) and scale $s \sim U([1, S])$. Ensure the entire $K \times K$ patch is within $\Omega$. $S = \min(W, H)/K$ where $W, H$ are target image dimensions [S1, unofficial].

2.  **Define Patch Coordinates:**
    *   Determine the $K \times K$ patch $P(u, s)$ coordinates as continuous real numbers:
        $$ P(u, s) = (sx + u, sy + v) \quad \text{for } x, y \in \{-K/2, \dots, K/2 - 1\} $$
    *   This allows continuous evaluation of the radiance field [S1, unofficial].

3.  **Ray Sampling:**
    *   Determine 3D rays uniquely from $P(u, s)$, camera pose $\xi$, and intrinsics $K$.
    *   Denote normalized rays as $d_r$ and number of rays as $R$ ($R = K^2$ during training, $R = W \times H$ during inference) [S1, unofficial].

4.  **3D Point Sampling:**
    *   Sample $N$ points $\{x_{ir}\}$ along each ray $r$ using stratified sampling [S1, unofficial].

5.  **Evaluate Conditional Radiance Field:**
    *   Compute shape encoding: $h = h_\theta(\gamma(x), z_s)$ [S1, unofficial].
    *   Compute density: $\sigma = \sigma_\theta(h)$ [S1, unofficial].
    *   Compute color: $c = c_\theta(h, \gamma(d), z_a)$ [S1, unofficial].
    *   Note: $\sigma$ is computed independently of viewpoint $d$ and appearance code $z_a$ to encourage multi-view consistency [S1, unofficial].

6.  **Volume Rendering:**
    *   Perform volume rendering to obtain pixel color $c_r$ from $\{(c_{ir}, \sigma_{ir})\}$ along ray $r$ using the standard emission-absorption model (see [NeRF](4972-neural-radiance-fields.md) Shared principles) [S1, unofficial].
    *   Combine results of all $R$ rays to form predicted patch $P_0$ [S1, unofficial].

7.  **Discriminator Processing:**
    *   Extract real patch $P$ from real image $I$ using bilinear interpolation at coordinates $P(u, s)$ drawn from the same distribution $p_\nu$. Denote this as $\Gamma(I, \nu)$ [S1, unofficial].
    *   Compute discriminator scores for $P_0$ and $P$ [S1, unofficial].

8.  **Training Objective:**
    *   Train using non-saturating GAN objective with R1 regularization:
        $$ V(\theta, \phi) = \mathbb{E}[f(D_\phi(G_\theta(z_s, z_a, \xi, \nu)))] + \mathbb{E}[f(-D_\phi(\Gamma(I, \nu)))] - \lambda \|\nabla D_\phi(\Gamma(I, \nu))\|^2 $$
    *   Where $f(t) = -\log(1 + \exp(-t))$ and $\lambda$ controls the strength of the regularizer [S1, unofficial].

9.  **Inference:**
    *   Randomly sample $z_s, z_a, \xi$.
    *   Predict one color value for every pixel in the target image (full resolution $W \times H$) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class ConditionalRadianceField(nn.Module):
    """
    The conditional radiance field g_theta.
    Disentangles shape (via h and sigma) from appearance (via c).
    """
    def __init__(self, input_dim_x, input_dim_d, latent_dim_s, latent_dim_a, hidden_dim):
        super().__init__()
        # Shape encoding head: h_theta maps (gamma(x), z_s) to h
        self.h_net = nn.Sequential(
            nn.Linear(input_dim_x + latent_dim_s, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim) # Output h
        )
        
        # Density head: sigma_theta maps h to sigma
        self.sigma_net = nn.Sequential(
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, 1) # Output sigma
        )
        
        # Color head: c_theta maps (h, gamma(d), z_a) to c
        self.c_net = nn.Sequential(
            nn.Linear(hidden_dim + input_dim_d + latent_dim_a, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, 3) # Output RGB color
        )

    def forward(self, x, d, z_s, z_a):
        # x: [B, 3], d: [B, 3], z_s: [B, M_s], z_a: [B, M_a]
        # Note: Positional encoding gamma is assumed to be applied to x and d before passing here,
        # or handled within the network if gamma is part of the input dimension.
        # Based on source, inputs are gamma(x), gamma(d), z_s, z_a.
        
        # 1. Compute shape encoding h
        h_input = torch.cat([x, z_s], dim=-1)
        h = self.h_net(h_input)
        
        # 2. Compute density sigma (independent of d and z_a)
        sigma = self.sigma_net(h)
        
        # 3. Compute color c
        c_input = torch.cat([h, d, z_a], dim=-1)
        c = self.c_net(c_input)
        
        return c, sigma

def positional_encoding(x, min_deg, max_deg):
    """
    Standard positional encoding for NeRF.
    x: [B, D]
    Returns: [B, D * (2 * max_deg + 1)]
    """
    scales = torch.tensor([2**i for i in range(min_deg, max_deg + 1)], dtype=x.dtype, device=x.device)
    scaled_x = x.unsqueeze(-1) * scales.unsqueeze(0)
    freq_bands = torch.sin(torch.cat([scaled_x, scaled_x + 0.5 * torch.pi], dim=-1))
    return freq_bands.view(x.shape[0], -1)

def volume_rendering(rays_o, rays_d, points, colors, sigmas, near, far):
    """
    Standard volume rendering (DVR) as per NeRF.
    rays_o: [R, 3] ray origins
    rays_d: [R, 3] ray directions (normalized)
    points: [R, N, 3] sampled points along rays
    colors: [R, N, 3] predicted colors
    sigmas: [R, N] predicted densities
    near, far: float, near and far clipping planes
    """
    # Compute deltas
    # points are sorted along the ray
    dists = torch.norm(points[:, 1:] - points[:, :-1], dim=-1)
    dists = torch.cat([dists, torch.tensor([1e10], device=points.device).expand(dists.shape[0], 1)], dim=-1)
    
    # Alpha
    alpha = 1.0 - torch.exp(-sigmas * dists)
    
    # Transmittance
    T = torch.cumprod(1.0 - alpha + 1e-10, dim=1)
    T = torch.cat([torch.ones_like(T[:, :1]), T[:, :-1]], dim=-1)
    
    # Weights
    weights = alpha * T
    
    # Composite color
    comp_color = torch.sum(weights.unsqueeze(-1) * colors, dim=1)
    
    return comp_color

class Discriminator(nn.Module):
    """
    Convolutional neural network discriminator.
    Uses spectral normalization and instance normalization.
    """
    def __init__(self, patch_size, channels=3, hidden_channels=64):
        super().__init__()
        # Simplified CNN structure for demonstration
        self.conv1 = nn.Conv2d(channels, hidden_channels, 4, stride=2, padding=1)
        self.conv2 = nn.Conv2d(hidden_channels, hidden_channels*2, 4, stride=2, padding=1)
        self.conv3 = nn.Conv2d(hidden_channels*2, hidden_channels*4, 4, stride=2, padding=1)
        self.fc = nn.Linear(hidden_channels*4 * (patch_size//8) * (patch_size//8), 1)
        
        # Spectral normalization is applied to conv layers in practice
        # Instance normalization is applied after conv layers
        self.norm1 = nn.InstanceNorm2d(hidden_channels)
        self.norm2 = nn.InstanceNorm2d(hidden_channels*2)
        self.norm3 = nn.InstanceNorm2d(hidden_channels*4)

    def forward(self, x):
        # x: [B, 3, K, K]
        x = F.leaky_relu(self.norm1(self.conv1(x)), 0.2)
        x = F.leaky_relu(self.norm2(self.conv2(x)), 0.2)
        x = F.leaky_relu(self.norm3(self.conv3(x)), 0.2)
        x = x.view(x.size(0), -1)
        x = self.fc(x)
        return x

def graf_train_step(generator, discriminator, real_image, K, W, H, N, lambda_r1):
    """
    One training step for GRAF.
    """
    # 1. Sample latent codes and camera parameters
    z_s = torch.randn(8, 128).cuda() # Example dims
    z_a = torch.randn(8, 128).cuda()
    
    # Sample pose (simplified)
    # In practice, sample R, t from uniform distribution on upper hemisphere
    R = torch.eye(3).unsqueeze(0).repeat(8, 1, 1).cuda()
    t = torch.zeros(8, 3).cuda()
    xi = torch.cat([R.view(8, 9), t], dim=-1)
    
    # Sample 2D sampling pattern nu = (u, s)
    # u ~ U([0, W], [0, H])
    # s ~ U([1, S]) where S = min(W, H) / K
    S = min(W, H) / K
    u_x = torch.rand(8) * W
    u_y = torch.rand(8) * H
    s = torch.rand(8) * (S - 1) + 1
    
    # 2. Generate Patch
    # Generator takes K, xi, nu, z_s, z_a
    # Internal logic:
    # - Define patch coordinates P(u, s)
    # - Cast rays from P(u, s) using K, xi
    # - Sample N points along rays
    # - Evaluate Conditional Radiance Field
    # - Volume Render
    pred_patch = generator(K, xi, (u_x, u_y, s), z_s, z_a) # Returns [8, 3, K, K]
    
    # 3. Sample Real Patch
    # Extract real patch from real_image using bilinear interpolation at P(u, s)
    # This is equivalent to torch.nn.functional.grid_sample with specific coordinates
    real_patch = extract_real_patch(real_image, u_x, u_y, s, K, W, H)
    
    # 4. Discriminator Forward
    score_gen = discriminator(pred_patch)
    score_real = discriminator(real_patch)
    
    # 5. Compute Losses
    # Non-saturating GAN loss
    # f(t) = -log(1 + exp(-t))
    # E[f(D(G(z)))]
    loss_gen = -torch.mean(torch.log(1 + torch.exp(-score_gen)))
    
    # E[f(-D(I))]
    loss_disc_real = -torch.mean(torch.log(1 + torch.exp(-score_real)))
    
    # R1 Regularization: lambda * || grad D(I) ||^2
    # Compute gradient of discriminator output w.r.t real patch
    grad_real = torch.autograd.grad(
        outputs=score_real.sum(),
        inputs=real_patch,
        create_graph=True,
        retain_graph=True,
        only_inputs=True
    )[0]
    r1_reg = lambda_r1 * (grad_real.pow(2).mean())
    
    loss_disc = loss_disc_real + r1_reg
    
    # 6. Backprop
    loss_disc.backward()
    # generator loss backward
    loss_gen.backward()
    
    return loss_gen.item(), loss_disc.item()

def extract_real_patch(image, u_x, u_y, s, K, W, H):
    """
    Extracts a KxK patch from image at continuous coordinates u scaled by s.
    image: [B, C, H, W]
    u_x, u_y: [B]
    s: [B]
    Returns: [B, C, K, K]
    """
    B = u_x.shape[0]
    # Create grid for the patch
    x_grid = torch.linspace(-K/2, K/2 - 1, K, device=image.device)
    y_grid = torch.linspace(-K/2, K/2 - 1, K, device=image.device)
    xx, yy = torch.meshgrid(x_grid, y_grid, indexing='ij')
    
    # Reshape for broadcasting
    xx = xx.unsqueeze(0).unsqueeze(0) # [1, 1, K, K]
    yy = yy.unsqueeze(0).unsqueeze(0)
    
    # Scale and shift
    # P(u, s) = (sx + u, sy + v)
    # Normalize to [-1, 1] for grid_sample
    # Assuming image coordinates are [0, W] x [0, H]
    # grid_sample expects normalized coordinates [-1, 1]
    
    # Centered coordinates relative to patch center
    # x_coord = s * x + u_x
    # y_coord = s * y + u_y
    
    # We need to do this per batch item
    # Reshape for broadcasting
    u_x = u_x.view(B, 1, 1, 1)
    u_y = u_y.view(B, 1, 1, 1)
    s = s.view(B, 1, 1, 1)
    
    x_coords = s * xx + u_x
    y_coords = s * yy + u_y
    
    # Normalize to [-1, 1]
    # x in [0, W] -> [-1, 1] => 2*x/W - 1
    # y in [0, H] -> [-1, 1] => 2*y/H - 1
    # Note: grid_sample uses (x, y) corresponding to (W, H)
    # x is horizontal, y is vertical
    
    x_norm = 2 * x_coords / W - 1
    y_norm = 2 * y_coords / H - 1
    
    grid = torch.stack([x_norm, y_norm], dim=-1) # [B, K, K, 2]
    
    # grid_sample expects grid of shape [B, H_out, W_out, 2]
    # Our grid is [B, K, K, 2], so H_out=K, W_out=K
    patch = F.grid_sample(image, grid, mode='bilinear', padding_mode='zeros', align_corners=False)
    
    return patch
```

**Parameters:**
*   **Patch size $K \times K$:** Fixed size of the image patch predicted by the generator during training. Chosen such that the principal point is in the center of the image [S1, unofficial].
*   **$S$:** Maximum scale for the virtual patch, $S = \min(W, H)/K$ [S1, unofficial].
*   **$N$:** Number of 3D points sampled along each ray for numerical integration [S1, unofficial].
*   **$\lambda$:** Strength of the R1 regularization term [S1, unofficial].
*   **Learning Rate:** 0.0005 for the generator, 0.0001 for the discriminator [S1, unofficial].
*   **Batch Size:** 8 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Training Cost:** Predicting one color value per pixel is too expensive at training time. The solution is to predict a fixed $K \times K$ patch randomly scaled and rotated to provide gradients for the entire field [S1, unofficial].
*   **Continuous Coordinates:** The patch coordinates $P(u, s)$ are continuous real numbers, not discrete integers, allowing continuous evaluation of the radiance field [S1, unofficial].
*   **Discriminator Scale:** The discriminator is similar to PatchGAN but allows continuous displacements $u$ and scales $s$. It does not downsample the real image $I$ based on $s$, but queries sparse locations to retain high-frequency details [S1, unofficial].

## Operating characteristics
*   **Time and Memory Complexity:** The continuous representation scales well with image resolution and memory consumption compared to voxel-based methods [S1, unofficial]. Training uses a fixed patch size $K \times K$ regardless of the target image resolution, reducing computational cost [S1, unofficial].
*   **Measured Performance:**
    *   FID scores on Cars dataset at $64^2$ resolution: 34 (GRAF) vs 199 (PLATONIC GAN), 59 (HoloGAN), 59 (2D GAN) [S1, unofficial].
    *   FID scores on Faces dataset at $512^2$ resolution: 25 (GRAF) vs 134 (HoloGAN), 321 (PLATONIC GAN), 66 (2D GAN) [S1, unofficial].
    *   Reconstruction Accuracy (MMD-CD) on Cars: 0.044 (GRAF) vs 0.109 (HoloGAN) [S1, unofficial].
*   **Strengths:**
    *   Continuous representation avoids discretization artifacts and memory limitations of voxel-based methods [S1, unofficial].
    *   Incorporates 3D consistency by design through differentiable volume rendering, avoiding entangled latents common in learned projection methods [S1, unofficial].
    *   Allows explicit control over camera rotation, translation, shape, and appearance [S1, unofficial].
*   **Limitations:**
    *   Requires adversarial training which can be unstable [S1, unofficial].
    *   Relies on the quality of the conditional radiance field representation [S1, unofficial].
*   **When to Use:** Suitable for 3D-aware image synthesis from unposed 2D images where explicit control over camera and appearance is desired, and high resolution is needed [S1, unofficial].
*   **When Not to Use:** When posed images of a single scene are available (use standard NeRF), or when computational resources for adversarial training are limited [S1, unofficial].

## Background
GRAF (Generative Radiance Fields) was introduced in the paper "GRAF: Generative Radiance Fields for 3D-Aware Image Synthesis" [S1, unofficial]. It builds on the principles of Neural Radiance Fields (NeRF) [S1, unofficial]. The method addresses the problem of 3D-aware image synthesis from unposed 2D images, a setting where traditional NeRF cannot be directly applied due to the lack of camera poses and the need for a generative model [S1, unofficial]. It was evaluated on synthetic datasets (Chairs, Cars) and real-world datasets (Faces, Cats, Birds) [S1, unofficial].

## Sources
[S1] GRAF: Generative Radiance Fields for 3D-Aware Image Synthesis, https://doi.org/10.48550/arxiv.2007.02442, fetched 2026-10-26 (unofficial)
