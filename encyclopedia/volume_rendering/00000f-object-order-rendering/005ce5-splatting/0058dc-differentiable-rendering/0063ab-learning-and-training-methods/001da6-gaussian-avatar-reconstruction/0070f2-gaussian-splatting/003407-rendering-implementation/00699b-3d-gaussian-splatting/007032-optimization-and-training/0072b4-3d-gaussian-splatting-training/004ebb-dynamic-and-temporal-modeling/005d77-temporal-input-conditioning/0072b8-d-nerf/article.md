# D-NeRF

Also called: Deformable Neural Radiance Fields, Nerfies, Neural Radiance Fields for Dynamic Scenes

## Overview
D-NeRF synthesizes novel views at arbitrary time instants for dynamic scenes with complex non-rigid geometries, optimizing a deformable volumetric function from a sparse set of monocular input images without ground-truth geometry or multi-view stereo data [S1]. It sets itself apart from naive extensions of NeRF (which map 6D coordinates directly to radiance) and T-NeRF (which uses a 6D input without canonical mapping) by decomposing the representation into a static canonical scene and a time-dependent deformation field, effectively exploiting temporal redundancy [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sparse set of $T$ RGB images $\{I_t\}_{t=1}^T$ captured by a monocular camera, where each image $I_t \in \mathbb{R}^{H \times W \times 3}$ is associated with a camera pose matrix $T_t \in \mathbb{R}^{4 \times 4}$ (SE(3)) at time $t$. The method assumes a single view per time instant [S1].
*   **Outputs:** A rendered RGB pixel color $C'(p, t)$ for a pixel $p$ at time $t$, given a camera ray defined by origin $o$ and direction $d$ [S1].

**Data Structures:**
The model consists of two deep fully connected networks (MLPs) without convolutional layers [S1]:
1.  **Deformation Network ($\Psi_t$):** Maps a 3D point $x$ and time $t$ to a displacement vector $\Delta x$.
    *   Architecture: 8-layer MLP with ReLU activations.
    *   Output: No non-linearity applied to $\Delta x$ [S1].
2.  **Canonical Network ($\Psi_x$):** Maps a canonical 3D point $x'$ and viewing direction $d$ to volume density $\sigma$ and emitted color $c$.
    *   Architecture: 8-layer MLP with ReLU activations.
    *   Output: Sigmoid non-linearity applied to $c$ and $\sigma$ [S1].

**Positional Encoding:**
Raw coordinates are encoded into a higher-dimensional space using $\gamma(p) = \langle (\sin(2^l \pi p), \cos(2^l \pi p)) \rangle_{l=0}^{L-1}$, applied independently to each coordinate component [S4].
*   For spatial coordinates $x$: $L = 10$ [S1].
*   For viewing direction $d$ and time $t$: $L = 4$ [S1].

**Numbered Steps:**

1.  **Canonical Configuration Definition:** Define the canonical scene as the scene state at time $t=0$. Enforce that the deformation displacement is zero at this time:
    $$ \Psi_t(x, 0) = 0 \quad \text{for all } x $$
    For $t \neq 0$, $\Psi_t(x, t) = \Delta x$ [S1].

2.  **Ray Sampling:** For a pixel $p$ at time $t$, define the camera ray $x(h) = o + h d$, where $o$ is the camera center and $d$ is the viewing direction. Sample $N$ points $\{h_n\}_{n=1}^N$ along the ray interval $[h_n, h_f]$ using stratified sampling from evenly-spaced bins [S1].

3.  **Deformation Transform:** For each sample point $x(h_n)$ at time $t$, compute the displacement $\Delta x$ using the deformation network:
    $$ \Delta x = \Psi_t(\gamma(x(h_n)), \gamma(t)) $$
    Transform the point to the canonical space:
    $$ p(h_n, t) = x(h_n) + \Delta x $$
    [S1].

4.  **Canonical Query:** Query the canonical network with the transformed point and viewing direction to obtain density and color:
    $$ [c(p(h_n, t), d), \sigma(p(h_n, t))] = \Psi_x(\gamma(p(h_n, t)), \gamma(d)) $$
    [S1].

5.  **Volume Rendering Approximation:** Compute the accumulated transmittance and opacity for each sample. Let $\delta_n = h_{n+1} - h_n$ be the distance between quadrature points [S1].
    *   Opacity: $\alpha(h_n, t, \delta_n) = 1 - \exp(-\sigma(p(h_n, t)) \delta_n)$
    *   Transmittance: $T'(h_n, t) = \exp\left(-\sum_{m=1}^{n-1} \sigma(p(h_m, t)) \delta_m\right)$
    [S1].

6.  **Pixel Color Computation:** Approximate the pixel color $C'(p, t)$ as the weighted sum of emitted colors:
    $$ C'(p, t) = \sum_{n=1}^{N} T'(h_n, t) \alpha(h_n, t, \delta_n) c(p(h_n, t), d) $$
    [S1].

7.  **Training Optimization:** Minimize the mean squared error (MSE) between the rendered pixel $C'(p, t)$ and the ground truth pixel $\hat{C}(p, t)$:
    $$ \text{Loss} = \frac{1}{N_s} \sum_{i=1}^{N_s} \|\hat{C}(p_i, t) - C'(p_i, t)\|^2 $$
    where $N_s$ is the batch size in rays [S1].

**Parameters:**
*   **Canonical Time:** $t=0$ [S1].
*   **Positional Encoding Depth:** $L=10$ for $x$, $L=4$ for $d$ and $t$ [S1].
*   **Network Depth:** 8 layers for both $\Psi_t$ and $\Psi_x$ [S1].
*   **Training Resolution:** $400 \times 400$ images [S1].
*   **Iterations:** 800,000 [S1].
*   **Batch Size:** $N_s = 4096$ rays [S1].
*   **Samples per Ray:** 64 [S1].
*   **Optimizer:** Adam with learning rate $5 \times 10^{-4}$, $\beta_1=0.9$, $\beta_2=0.999$, decaying exponentially to $5 \times 10^{-5}$ [S1].
*   **Curriculum Learning:** Input images are sorted by timestamp and added incrementally to improve convergence [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class PositionalEncoding(nn.Module):
    def __init__(self, L):
        super().__init__()
        self.L = L

    def forward(self, x):
        # x shape: (..., 3) or (..., 1)
        # Apply encoding independently to each dimension
        freq_bands = 2.0 ** torch.linspace(0, self.L - 1, self.L).to(x.device)
        # Expand dimensions for broadcasting
        # x: (..., D) -> (..., D, 1)
        # freq_bands: (L,) -> (1, 1, L)
        # result: (..., D, 2*L)
        x_expanded = x.unsqueeze(-1)
        freq_bands = freq_bands.view(1, 1, -1)
        sines = torch.sin(x_expanded * freq_bands * torch.pi)
        cosines = torch.cos(x_expanded * freq_bands * torch.pi)
        return torch.cat([sines, cosines], dim=-1).view(*x.shape[:-1], -1)

class DeformationNetwork(nn.Module):
    def __init__(self, input_dim, hidden_dim=256, num_layers=8):
        super().__init__()
        layers = []
        for i in range(num_layers):
            layers.append(nn.Linear(input_dim if i == 0 else hidden_dim, hidden_dim))
            layers.append(nn.ReLU())
        self.network = nn.Sequential(*layers)

    def forward(self, x_enc, t_enc):
        # x_enc: (..., 30) for L=10, t_enc: (..., 8) for L=4
        # Concatenate encoded spatial and temporal inputs
        input_vec = torch.cat([x_enc, t_enc], dim=-1)
        # Output displacement without non-linearity
        return self.network(input_vec)

class CanonicalNetwork(nn.Module):
    def __init__(self, input_dim, hidden_dim=256, num_layers=8):
        super().__init__()
        layers = []
        for i in range(num_layers):
            layers.append(nn.Linear(input_dim if i == 0 else hidden_dim, hidden_dim))
            layers.append(nn.ReLU())
        # Final layer outputs density and color
        layers.append(nn.Linear(hidden_dim, 4)) # 1 for sigma, 3 for RGB
        self.network = nn.Sequential(*layers)

    def forward(self, x_enc, d_enc):
        # x_enc: (..., 30), d_enc: (..., 8)
        input_vec = torch.cat([x_enc, d_enc], dim=-1)
        out = self.network(input_vec)
        sigma = out[..., :1]
        rgb = out[..., 1:]
        # Apply sigmoid to sigma and rgb
        sigma = torch.sigmoid(sigma)
        rgb = torch.sigmoid(rgb)
        return sigma, rgb

def render_ray(rays_o, rays_d, near, far, N_samples, deform_net, canon_net, pos_enc_x, pos_enc_d, pos_enc_t):
    """
    Renders a single ray.
    rays_o: (3,)
    rays_d: (3,)
    near: float
    far: float
    t: float (time)
    """
    # Stratified sampling
    t_vals = torch.linspace(0., 1., N_samples + 1)
    t_vals = t_vals[:-1]
    # Add noise for stratified sampling
    noise = torch.rand(N_samples)
    t_vals = t_vals + noise / N_samples
    
    # Interpolate points along ray
    # x(h) = o + h * d
    # We map t_vals from [0,1] to [near, far]
    h_vals = near + t_vals * (far - near)
    points = rays_o.unsqueeze(0) + h_vals.unsqueeze(1) * rays_d.unsqueeze(0) # (N_samples, 3)
    
    # Encode inputs
    points_enc = pos_enc_x(points) # (N_samples, 30)
    d_enc = pos_enc_d(rays_d)      # (8,)
    t_enc = pos_enc_t(torch.tensor([t])) # (8,)
    
    # Deformation
    delta_x = deform_net(points_enc, t_enc) # (N_samples, 3)
    
    # Canonical points
    canonical_points = points + delta_x # (N_samples, 3)
    canonical_points_enc = pos_enc_x(canonical_points) # (N_samples, 30)
    
    # Canonical Query
    sigma, rgb = canon_net(canonical_points_enc, d_enc) # (N_samples, 1), (N_samples, 3)
    
    # Volume Rendering
    # Distances between samples
    dists = h_vals[1:] - h_vals[:-1]
    dists = torch.cat([dists, torch.tensor([1e10]).to(points.device)]) # Last sample distance doesn't matter much
    
    # Alpha and Transmittance
    alpha = 1.0 - torch.exp(-sigma.squeeze(-1) * dists) # (N_samples,)
    transmittance = torch.cumprod(torch.cat([torch.ones(1).to(points.device), 1.0 - alpha + 1e-10]), dim=0)[:-1] # (N_samples,)
    
    # Contribution
    contribution = transmittance * alpha.unsqueeze(-1) * rgb # (N_samples, 3)
    
    # Final color
    color = torch.sum(contribution, dim=0) # (3,)
    return color

def train_step(batch_rays_o, batch_rays_d, batch_colors, batch_times, near, far, deform_net, canon_net, optimizer):
    optimizer.zero_grad()
    
    # batch_rays_o: (Ns, 3)
    # batch_rays_d: (Ns, 3)
    # batch_colors: (Ns, 3)
    # batch_times: (Ns,)
    
    rendered_colors = []
    for i in range(batch_rays_o.shape[0]):
        # Render one ray
        # Note: In practice, this is vectorized, but for clarity we loop
        # Assuming helper functions defined above are accessible
        # This loop is inefficient for implementation but clear for logic
        # A real implementation would vectorize the ray processing
        pass 
        
    # Vectorized implementation logic:
    # 1. Sample points for all rays
    # 2. Encode all points, directions, and times
    # 3. Compute deformations
    # 4. Query canonical network
    # 5. Render
    
    # Simplified vectorized steps for pseudocode completeness:
    N_samples = 64
    Ns = batch_rays_o.shape[0]
    
    # Sample depths
    t_vals = torch.linspace(0., 1., N_samples + 1).to(batch_rays_o.device)
    t_vals = t_vals[:-1] + torch.rand(N_samples, 1).to(batch_rays_o.device) / N_samples
    h_vals = near + t_vals * (far - near) # (Ns, N_samples)
    
    # Points
    rays_o_exp = batch_rays_o.unsqueeze(1) # (Ns, 1, 3)
    rays_d_exp = batch_rays_d.unsqueeze(1) # (Ns, 1, 3)
    points = rays_o_exp + h_vals.unsqueeze(2) * rays_d_exp # (Ns, N_samples, 3)
    
    # Encodings
    points_enc = pos_enc_x(points) # (Ns, N_samples, 30)
    d_enc = pos_enc_d(batch_rays_d) # (Ns, 8)
    t_enc = pos_enc_t(batch_times) # (Ns, 8)
    
    # Deformation
    delta_x = deform_net(points_enc, t_enc) # (Ns, N_samples, 3)
    
    # Canonical Points
    canonical_points = points + delta_x
    canonical_points_enc = pos_enc_x(canonical_points)
    
    # Canonical Query
    # Repeat d_enc for each sample
    d_enc_exp = d_enc.unsqueeze(1).repeat(1, N_samples, 1) # (Ns, N_samples, 8)
    sigma, rgb = canon_net(canonical_points_enc, d_enc_exp) # (Ns, N_samples, 1), (Ns, N_samples, 3)
    
    # Rendering
    dists = h_vals[:, 1:] - h_vals[:, :-1] # (Ns, N_samples-1)
    dists = torch.cat([dists, torch.tensor([1e10]).to(points.device)].unsqueeze(0).repeat(Ns, 1), dim=1) # (Ns, N_samples)
    
    alpha = 1.0 - torch.exp(-sigma.squeeze(-1) * dists) # (Ns, N_samples)
    transmittance = torch.cumprod(torch.cat([torch.ones(Ns, 1).to(points.device), 1.0 - alpha + 1e-10], dim=1)[:, :-1], dim=1) # (Ns, N_samples)
    
    contribution = transmittance.unsqueeze(-1) * alpha.unsqueeze(-1) * rgb # (Ns, N_samples, 3)
    rendered_colors = torch.sum(contribution, dim=1) # (Ns, 3)
    
    loss = F.mse_loss(rendered_colors, batch_colors)
    loss.backward()
    optimizer.step()
    return loss.item()
```

## Subtopics

### [Dynamic NeRF Guided by Projected Colors](8882-dynamic-nerf-guided-by-projected-colors.md)
Replaces explicit temporal coordinate $t$ with multi-view projected color statistics to improve point correspondence estimation during on-the-fly training, mapping inputs to a "space-color" coordinate system.

## Operating characteristics
*   **Complexity:** Time and memory complexity are dominated by the two 8-layer MLPs and the ray sampling process. The method requires storing parameters for two networks.
*   **Performance:** On the 'Hell Warrior' scene, D-NeRF achieves MSE $31 \times 10^{-4}$, PSNR 23.19, SSIM 0.93, LPIPS 0.08, compared to NeRF's MSE $44 \times 10^{-3}$, PSNR 13.52, SSIM 0.81, LPIPS 0.25 [S1]. On the 'T-Rex' scene, D-NeRF achieves MSE $5 \times 10^{-4}$, PSNR 32.80, SSIM 0.98, LPIPS 0.03, compared to NeRF's MSE $21 \times 10^{-3}$, PSNR 18.19, SSIM 0.89, LPIPS 0.14 [S1].
*   **Scaling:** Training uses 400x400 images, 800k iterations, batch size 4096 rays, sampled 64 times per ray, trained on a single Nvidia GTX 1080 for 2 days [S1].
*   **Strengths:** Can handle rigid, articulated, and non-rigid motions from a sparse set of monocular views without ground-truth geometry [S1].
*   **Limitations:** Deformation fields are difficult to learn when complex scene motions exist, leading to blurry results similar to Neural Volumes [S6]. It cannot easily handle topology changes because the deformation field must be continuous [S7].

## Background
D-NeRF was proposed by Pumarola et al. in November 2020 [S1]. It extends NeRF to handle dynamic scenes by introducing time as an additional input and splitting the representation into a deformation network and a canonical radiance network [S2]. It is compared against Neural Volumes (NV), Scene Representation Networks (SRN), and T-NeRF [S1]. T-NeRF represents the scene with a 6D input $(x,y,z,\theta,\phi,t)$ directly without canonical mapping, which D-NeRF improves upon by using an intermediate canonical configuration [S1].

## Sources
[S1] D-NeRF: Neural Radiance Fields for Dynamic Scenes, https://doi.org/10.1109/cvpr46437.2021.01018, fetched 2026-10-26
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S3] Neural Rendering and Its Hardware Acceleration: A Review, https://doi.org/10.48550/arxiv.2402.00028, fetched 2026-10-26 (unofficial)
[S4] Neural Radiance Fields-Comprehensive Survey, https://zenodo.org/records/7597137, fetched 2026-10-26
[S5] Neural Volume Rendering: NeRF And Beyond, https://arxiv.org/abs/2101.05204v2, fetched 2026-10-26 (unofficial)
[S6] Representing Volumetric Videos as Dynamic MLP Maps, https://arxiv.org/abs/2304.06717, fetched 2026-10-29
[S7] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
