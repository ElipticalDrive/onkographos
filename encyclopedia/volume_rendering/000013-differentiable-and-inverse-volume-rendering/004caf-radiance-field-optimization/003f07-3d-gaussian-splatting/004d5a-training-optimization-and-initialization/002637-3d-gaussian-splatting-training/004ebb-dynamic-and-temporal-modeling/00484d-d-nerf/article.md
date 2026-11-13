# D-NeRF

Also called: Deformable Neural Radiance Fields, Dynamic NeRF, Neural Radiance Fields for Dynamic Scenes

## Overview
D-NeRF is a method for synthesizing novel views at an arbitrary point in time of dynamic scenes with complex non-rigid geometries, optimized from a sparse set of input monocular views without ground-truth geometry or multi-view images [S1]. It solves the problem of extending static Neural Radiance Fields (NeRF) to dynamic environments by decomposing the learning process into two modules: a deformation network that maps scene points to a canonical space, and a canonical network that regresses density and color from that space [S1, S3]. Unlike naive extensions of NeRF that map $(x, y, z, t)$ directly to density and radiance, D-NeRF exploits temporal redundancy by anchoring all time instances to a common canonical configuration (typically $t=0$), allowing it to handle rigid, articulated, and non-rigid motions from a single camera moving around the scene [S1, S3].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sparse set of $T$ RGB images $\{I_t\}_{t=1}^T$ and their corresponding camera pose matrices $\{T_t\}_{t=1}^T$, where $I_t \in \mathbb{R}^{H \times W \times 3}$ and $T_t \in \mathbb{R}^{4 \times 4}$ is an SE(3) transformation [S1, S3]. Each time instant $t$ is acquired by a single camera view [S1].
*   **Outputs:** Rendered RGB images $C'(p, t)$ for arbitrary pixel coordinates $p$, viewing directions $d$, and time instances $t$ [S1].

**Data Structures:**
The model consists of two deep fully connected networks (MLPs) without convolutional layers [S1, S3]:
1.  **Deformation Network $\Psi_t$:**
    *   **Architecture:** 8-layer MLP with ReLU activations.
    *   **Input:** Encoded 3D position $x$ and time $t$.
    *   **Output:** Displacement vector $\Delta x \in \mathbb{R}^3$. No non-linearity is applied to the output [S1, S3].
    *   **Constraint:** $\Psi_t(x, 0) = 0$ [S1].
2.  **Canonical Network $\Psi_x$:**
    *   **Architecture:** 8-layer MLP with ReLU activations.
    *   **Input:** Encoded canonical position $x' = x + \Delta x$ and viewing direction $d$.
    *   **Output:** Volume density $\sigma \in \mathbb{R}$ and view-dependent RGB color $c \in \mathbb{R}^3$. A final sigmoid non-linearity is applied to both outputs [S1, S3].

**Positional Encoding:**
Raw coordinates are encoded into a higher-dimensional space using the positional encoding function $\gamma(p)$ [S1, S3]:
$$ \gamma(p) = \langle (\sin(2^l \pi p), \cos(2^l \pi p)) \rangle_{l=0}^{L-1} $$
The encoder is applied independently to each coordinate and camera view component [S1].
*   For spatial coordinates $x$: $L = 10$ [S1, S3].
*   For viewing direction $d$ and time $t$: $L = 4$ [S1, S3].

**Numbered Steps:**

1.  **Sampling:** At each training batch, sample a random set of $N_s$ pixels $\{p_{t,i}\}_{i=1}^{N_s}$ corresponding to rays cast from camera position $T_t$ to pixels $i$ of the RGB image $I_t$ at time $t$ [S1, S3].
2.  **Ray Definition:** For each pixel $p$, define the camera ray $x(h) = o + h d$, where $o$ is the camera center, $d$ is the viewing direction, and $h \in [h_n, h_f]$ represents the distance along the ray between near and far bounds [S1, S3].
3.  **Deformation:** For each sample point $x(h)$ at time $t$, compute the displacement $\Delta x$ using the deformation network:
    $$ \Delta x = \Psi_t(\gamma(x(h)), \gamma(t)) $$
    Transform the point to canonical space:
    $$ p(h, t) = x(h) + \Delta x $$
    Note: By definition, if $t=0$, $\Delta x = 0$ [S1, S3].
4.  **Radiance Field Query:** Query the canonical network with the canonical point $p(h, t)$ and direction $d$ to obtain density and color:
    $$ [c(p(h, t), d), \sigma(p(h, t))] = \Psi_x(\gamma(p(h, t)), \gamma(d)) $$
5.  **Volume Rendering Integration:** Approximate the volume rendering integral using numerical quadrature with stratified sampling. Draw $N$ samples $\{h_n\}_{n=1}^N$ from evenly-spaced ray bins in $[h_n, h_f]$ [S1, S3].
    *   Compute the distance between samples: $\delta_n = h_{n+1} - h_n$ [S1].
    *   Compute the transmittance at sample $n$:
        $$ T'(h_n, t) = \exp \left( - \sum_{m=1}^{n-1} \sigma(p(h_m, t)) \delta_m \right) $$
    *   Compute the opacity at sample $n$:
        $$ \alpha(h_n, t, \delta_n) = 1 - \exp(-\sigma(p(h_n, t)) \delta_n) $$
    *   Compute the rendered pixel color $C'(p, t)$:
        $$ C'(p, t) = \sum_{n=1}^{N} T'(h_n, t) \alpha(h_n, t, \delta_n) c(p(h_n, t), d) $$
6.  **Loss Calculation:** Minimize the mean squared error between the rendered pixels $C'$ and the ground truth pixels $\hat{C}$:
    $$ \text{Loss} = \frac{1}{N_s} \sum_{i=1}^{N_s} \| \hat{C}(p_i, t) - C'(p_i, t) \|^2 $$
7.  **Optimization:** Simultaneously update the parameters of $\Psi_x$ and $\Psi_t$ using the Adam optimizer [S1, S3].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch
import torch.nn as nn
import torch.optim as optim

class PositionalEncoding(nn.Module):
    def __init__(self, L):
        super().__init__()
        self.L = L

    def forward(self, x):
        # x shape: (..., 3) or (..., 1)
        # Apply encoding independently to each component
        out = []
        for i in range(x.shape[-1]):
            xi = x[..., i:i+1]
            freqs = 2**torch.arange(self.L, device=x.device) * math.pi
            # sin(2^l * pi * p) and cos(2^l * pi * p)
            sin_vals = torch.sin(freqs * xi)
            cos_vals = torch.cos(freqs * xi)
            out.append(torch.cat([sin_vals, cos_vals], dim=-1))
        return torch.cat(out, dim=-1)

class DeformationNetwork(nn.Module):
    def __init__(self, input_dim, hidden_dim=256, num_layers=8):
        super().__init__()
        layers = []
        for _ in range(num_layers):
            layers.append(nn.Linear(input_dim, hidden_dim))
            layers.append(nn.ReLU())
            input_dim = hidden_dim
        self.layers = nn.Sequential(*layers)
        self.output_layer = nn.Linear(hidden_dim, 3) # Output delta x

    def forward(self, x_enc, t_enc):
        # x_enc: (..., 30) for L=10, 3 coords
        # t_enc: (..., 8) for L=4, 1 time coord
        input_enc = torch.cat([x_enc, t_enc], dim=-1)
        delta_x = self.output_layer(self.layers(input_enc))
        return delta_x

class CanonicalNetwork(nn.Module):
    def __init__(self, input_dim, hidden_dim=256, num_layers=8):
        super().__init__()
        layers = []
        for _ in range(num_layers):
            layers.append(nn.Linear(input_dim, hidden_dim))
            layers.append(nn.ReLU())
            input_dim = hidden_dim
        self.layers = nn.Sequential(*layers)
        self.output_layer = nn.Linear(hidden_dim, 4) # Output sigma (1) and c (3)

    def forward(self, x_enc, d_enc):
        input_enc = torch.cat([x_enc, d_enc], dim=-1)
        out = self.output_layer(self.layers(input_enc))
        sigma = torch.sigmoid(out[..., 0:1])
        color = torch.sigmoid(out[..., 1:4])
        return sigma, color

def render_ray(ray_o, ray_d, near, far, N_samples, deform_net, canon_net, pos_enc_x, pos_enc_d, pos_enc_t):
    # ray_o: (3), ray_d: (3)
    # Sample points along the ray
    t_vals = torch.linspace(near, far, N_samples)
    # Stratified sampling: add uniform noise in [0, 1) and scale by bin width
    bins = t_vals.unsqueeze(-1) - (t_vals.unsqueeze(-1) - t_vals.unsqueeze(0)).unsqueeze(-1) # Simplified bin logic
    # Standard stratified sampling from NeRF
    samples = near + (far - near) * (torch.rand(N_samples) + torch.linspace(0, 1, N_samples, requires_grad=False)) / N_samples
    # Actually, standard stratified:
    # lower = near + (far - near) * (i / N)
    # upper = near + (far - near) * ((i + 1) / N)
    # sample = lower + uniform(0, upper - lower)
    
    # Re-implementing standard stratified sampling for clarity
    t_vals = torch.linspace(near, far, N_samples + 1)
    t_vals = t_vals[:-1] + (t_vals[1:] - t_vals[:-1]) * torch.rand(N_samples)
    
    # 3D points on ray
    pts = ray_o.unsqueeze(0) + t_vals.unsqueeze(1) * ray_d.unsqueeze(0) # (N_samples, 3)
    
    # Encode inputs
    pts_enc = pos_enc_x(pts)
    d_enc = pos_enc_d(ray_d)
    
    # We need time t for deformation. Assume t is passed as a scalar or batch dim.
    # For a single ray at time t:
    # Note: The pseudocode assumes a batch of rays. Here we process one ray's samples.
    # To match the paper, we need the time t for the deformation network.
    # Let's assume t is a scalar tensor.
    t_enc = pos_enc_t(torch.tensor([t]).expand(N_samples, 1))
    
    # Deformation
    delta_x = deform_net(pts_enc, t_enc) # (N_samples, 3)
    canonical_pts = pts + delta_x
    canonical_pts_enc = pos_enc_x(canonical_pts)
    
    # Canonical Network
    sigma, color = canon_net(canonical_pts_enc, d_enc.expand(N_samples, -1))
    
    # Volume Rendering
    # Distances between samples
    dists = t_vals[1:] - t_vals[:-1]
    dists = torch.cat([dists, torch.tensor([1e10], device=t_vals.device)]) # Last sample distance doesn't matter much if alpha is low, but paper uses delta_n = h_{n+1} - h_n
    
    # Alpha and Transmittance
    alpha = 1.0 - torch.exp(-sigma.squeeze() * dists)
    transmittance = torch.cumprod(torch.cat([torch.ones(1, device=alpha.device), 1.0 - alpha]), dim=0)[:-1]
    
    # Color
    weights = alpha * transmittance
    color_rendered = torch.sum(weights.unsqueeze(-1) * color.squeeze(), dim=0)
    
    return color_rendered

def train_step(images, poses, times, deform_net, canon_net, optimizer, pos_enc_x, pos_enc_d, pos_enc_t, N_samples=64, batch_size=4096):
    # images: list of T images
    # poses: list of T poses
    # times: list of T times
    # Select a random time t
    t_idx = torch.randint(0, len(images), (1,)).item()
    img = images[t_idx]
    pose = poses[t_idx]
    t = times[t_idx]
    
    # Sample batch_size rays from the image
    # H, W = img.shape[:2]
    # pixels = random.sample(range(H*W), batch_size)
    # For simplicity, assume we have ray origins and directions pre-computed for the batch
    # ray_os, ray_ds = sample_rays(pose, pixels, H, W)
    
    # Placeholder for ray sampling logic
    # ray_os: (batch_size, 3), ray_ds: (batch_size, 3)
    
    rendered_colors = []
    for i in range(batch_size):
        # Render each ray
        # This is inefficient but matches the "step-by-step" requirement without external libraries
        # In practice, this is vectorized.
        c = render_ray(ray_os[i], ray_ds[i], near, far, N_samples, deform_net, canon_net, pos_enc_x, pos_enc_d, pos_enc_t, t)
        rendered_colors.append(c)
        
    rendered_colors = torch.stack(rendered_colors)
    ground_truth = img_pixels # (batch_size, 3)
    
    loss = torch.mean((rendered_colors - ground_truth) ** 2)
    
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
    
    return loss.item()
```

**Parameters:**
*   **Canonical Time:** $t=0$ [S1, S3].
*   **Positional Encoding Depth:** $L=10$ for $x$, $L=4$ for $d$ and $t$ [S1, S3].
*   **Network Depth:** 8 layers for both $\Psi_x$ and $\Psi_t$ [S1, S3].
*   **Training Images:** $400 \times 400$ resolution [S1].
*   **Iterations:** 800,000 [S1].
*   **Batch Size:** $N_s = 4096$ rays [S1].
*   **Samples per Ray:** 64 [S1].
*   **Optimizer:** Adam with learning rate $5 \times 10^{-4}$, $\beta_1=0.9$, $\beta_2=0.999$, exponential decay to $5 \times 10^{-5}$ [S1].

**Edge Cases and Pitfalls:**
*   **Complex Motions:** Deformation fields are difficult to learn when complex scene motions exist, leading to blurry results similar to Neural Volumes [S4].
*   **Topological Changes:** Deformation fields are continuous and cannot model topological changes (e.g., objects appearing/disappearing) because they require discontinuities [S5].
*   **Single View per Time:** The method assumes a single camera view per time instant. It does not use multi-view stereo information at a specific time $t$ [S1].
*   **Curriculum Learning:** To improve convergence, input images should be sorted by timestamp and added incrementally (curriculum learning) [S1].

## Subtopics

### [Dynamic NeRF Guided by Projected Colors](8882-dynamic-nerf-guided-by-projected-colors.md)
This method is a dynamic Neural Radiance Field (NeRF) variant that replaces the explicit temporal coordinate $t$ with multi-view projected color statistics to improve point correspondence estimation during on-the-fly training. Unlike standard space-time dynamic NeRFs that optimize irregular temporal deformation fields, this approach maps inputs to a "space-color" coordinate system where correspondence is easier to track due to color invariance under motion.

## Operating characteristics
*   **Time Complexity:** Training requires 800,000 iterations on a single Nvidia GTX 1080 for 2 days [S1]. Rendering time is not explicitly quantified in milliseconds in the primary source, but it involves querying two MLPs and performing volume integration per ray [S1].
*   **Memory Complexity:** The model stores two 8-layer MLPs. The size is not explicitly given in MB, but it is significantly smaller than voxel-based methods like Neural Volumes [S4].
*   **Performance:** On the 'Hell Warrior' scene, D-NeRF achieves MSE $31 \times 10^{-4}$, PSNR 23.19, SSIM 0.93, LPIPS 0.08, compared to NeRF's MSE $44 \times 10^{-3}$, PSNR 13.52, SSIM 0.81, LPIPS 0.25 [S1]. On the 'T-Rex' scene, D-NeRF achieves MSE $5 \times 10^{-4}$, PSNR 32.80, SSIM 0.98, LPIPS 0.03, compared to NeRF's MSE $21 \times 10^{-3}$, PSNR 18.19, SSIM 0.89, LPIPS 0.14 [S1].
*   **Strengths:** Can synthesize novel views at arbitrary times from a sparse set of monocular views; handles rigid, articulated, and non-rigid motions; produces complete 3D meshes as a side product [S1].
*   **Limitations:** Struggles with complex scene motions resulting in blurry details; cannot handle topological changes; deformation fields are hard to learn [S4, S5].
*   **When to Use:** Scenes with non-rigid deformations where multi-view stereo is not available at each time step, and the scene does not undergo topological changes.
*   **When Not to Use:** Scenes with complex, high-frequency deformations that cause blurring, or scenes with objects appearing/disappearing (topological changes) [S4, S5].

## Background
D-NeRF was proposed by Albert Pumarola, Enric Corona, Gerard Pons-Moll, and Francesc Moreno-Noguer from Institut de Robòtica i Informàtica Industrial (CSIC-UPC) and Max Planck Institute for Informatics [S1]. It was presented at CVPR 2021 (published online November 2020) [S1]. The method extends the original NeRF [26] by introducing time as an additional input and decomposing the representation into a deformation network and a canonical radiance network, addressing the limitations of naive 6D extensions (T-NeRF) which failed to exploit temporal redundancy [S1, S3]. It is compared against Neural Volumes (NV), Scene Representation Networks (SRN), and Layered Light Field Fusion (LLFF) [S3, S4].

## Sources
[S1] D-NeRF: Neural Radiance Fields for Dynamic Scenes, https://doi.org/10.1109/cvpr46437.2021.01018, fetched 2026-10-26
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S3] Neural Radiance Fields-Comprehensive Survey, https://zenodo.org/records/7597137, fetched 2026-10-26
[S4] Representing Volumetric Videos as Dynamic MLP Maps, https://arxiv.org/abs/2304.06717, fetched 2026-10-29
[S5] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
