# Unified Neural Implicit Surface and Radiance Field Rendering

Also called: UNISURF rendering, Unified volume/surface rendering

## Overview
UNISURF is a framework for reconstructing accurate 3D surfaces from multi-view RGB images without requiring per-pixel object masks [S1]. It solves the ambiguity of volume-only rendering and the mask-dependency of surface-only rendering by unifying them: it parameterizes occupancy directly via a continuous field $o_\theta$ instead of volume density $\sigma$, allowing the model to render both through the volume (for coarse structure) and directly on the surface (for geometric precision) [S1]. Unlike [Neural Radiance Fields](52-neural-radiance-fields.md) which requires hierarchical two-stage sampling, UNISURF eliminates this need by using the occupancy field for root-finding and a decaying sampling interval [S1]. Unlike [IDR](13663-hybrid-surface-volume-and-mesh-representation.md) (a sibling in [Hybrid Surface-Volume and Mesh Representation](13663-hybrid-surface-volume-and-mesh-representation.md)), it does not rely on input masks [S1].

## How it works
**Inputs:** Multi-view RGB images with known camera intrinsics and extrinsics. No object masks are required.
**Outputs:** A rendered image $\hat{C}(\mathbf{r})$ for a given ray $\mathbf{r}$, and implicitly, a 3D surface defined by the zero-level set of the occupancy field.

**Data Structures:**
1.  **Occupancy Network ($o_\theta$):** An 8-layer MLP with Softplus activation and hidden dimension 256. Input: 3D position $\mathbf{x}$ (encoded with Fourier features, $k=6$ octaves). Output: scalar occupancy $o \in [0, 1]$.
2.  **Color Network ($c_\theta$):** A 4-layer MLP with ReLU activation. Input: 3D position $\mathbf{x}$ (encoded with Fourier features, $k=6$ octaves), surface normal $\mathbf{n}$, geometry feature vector $\mathbf{h}$ (from the geometry network's intermediate layers), and viewing direction $\mathbf{d}$ (encoded with Fourier features, $k=4$ octaves). Output: RGB color $\mathbf{c} \in [0, 1]^3$.
3.  **Ray Parameters:** Camera origin $\mathbf{o}$, direction $\mathbf{d}$, and current sampling interval width $\Delta$.

**Numbered Steps:**

1.  **Network Initialization:** Initialize the occupancy network such that the decision boundary is a sphere [S1]. [G] This typically involves setting the bias terms to achieve a specific radius, ensuring the initial surface is a reasonable prior for the object.

2.  **Surface Intersection (Root-Finding):**
    For a ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, find the surface intersection point $\mathbf{x}_s$.
    a. Sample 256 uniformly spaced points along the ray.
    b. Evaluate $o_\theta$ at these points.
    c. Identify the interval $[t_a, t_b]$ where the occupancy changes sign (or crosses a threshold, typically 0.5, though the paper implies root-finding on the continuous field).
    d. Apply the secant method with 8 steps to refine the root $t_s$ within this interval [S1].
    e. Compute $\mathbf{x}_s = \mathbf{o} + t_s \mathbf{d}$.

3.  **Surface Normal and Feature Extraction:**
    a. Compute the surface normal $\mathbf{n}_s$ using double backpropagation to get the gradient of the occupancy field:
       $$ \mathbf{n}(\mathbf{x}_s) = \frac{\nabla_{\mathbf{x}_s} o_\theta(\mathbf{x}_s)}{\|\nabla_{\mathbf{x}_s} o_\theta(\mathbf{x}_s)\|_2} $$
    b. Retrieve the geometry feature vector $\mathbf{h}_s$ from the intermediate layers of the occupancy network evaluated at $\mathbf{x}_s$ [S1].

4.  **Volume Rendering Sampling:**
    Determine the sampling interval $[t_s - \Delta, t_s + \Delta]$ centered at the surface depth $t_s$.
    a. **Interval Samples:** Draw $N=64$ query points $\{t_i\}_{i=1}^N$ inside the interval using stratified sampling:
       $$ t_i \sim \mathcal{U} \left[ t_s + \left( \frac{2i - 2}{N} - 1 \right) \Delta, t_s + \left( \frac{2i}{N} - 1 \right) \Delta \right] $$
       Compute $\mathbf{x}_i = \mathbf{o} + t_i \mathbf{d}$ for each $i$.
    b. **Free Space Samples:** Draw 32 points randomly in the free space between the camera and the lower bound of the interval ($t < t_s - \Delta$) to reduce free space artifacts [S1].
    c. **No Intersection Fallback:** If no surface intersection is found (ray does not intersect the object), use stratified sampling on the entire ray depth range [S1].

5.  **Color Evaluation:**
    For each sample point $\mathbf{x}_i$ in the interval:
    a. Compute normal $\mathbf{n}_i$ and feature $\mathbf{h}_i$ at $\mathbf{x}_i$ (using gradients and network features).
    b. Compute color $\mathbf{c}_i = c_\theta(\mathbf{x}_i, \mathbf{n}_i, \mathbf{h}_i, \mathbf{d})$.
    For free space samples, color is typically background or black (implementation dependent, but the paper focuses on the interval samples for the unified equation) [S1].

6.  **Unified Rendering Equation:**
    Compute the rendered color $\hat{C}_v(\mathbf{r})$ using the occupancy values $o_i = o_\theta(\mathbf{x}_i)$:
    $$ \hat{C}_v(\mathbf{r}) = \sum_{i=1}^N o_i \prod_{j < i} (1 - o_j) c_i $$
    This equation effectively selects the color of the first occupied sample along the ray, unifying volume integration with surface-like occlusion [S1].

7.  **Surface Rendering (Optional for Inference):**
    For faster inference, the color can be computed directly at the surface point:
    $$ \hat{C}_s(\mathbf{r}) = c_\theta(\mathbf{x}_s, \mathbf{n}_s, \mathbf{h}_s, \mathbf{d}) $$
    This is twice as fast as volume rendering [S1].

8.  **Loss Computation:**
    a. **Reconstruction Loss:** $\mathcal{L}_{rec} = \sum_{\mathbf{r} \in \mathcal{R}} \|\hat{C}_v(\mathbf{r}) - C(\mathbf{r})\|_1$, where $C(\mathbf{r})$ is the observed color [S1].
    b. **Surface Regularization Loss:** $\mathcal{L}_{reg} = \sum_{\mathbf{x}_s \in \mathcal{S}} \|\mathbf{n}(\mathbf{x}_s) - \mathbf{n}(\mathbf{x}_s + \boldsymbol{\epsilon})\|_2$, where $\boldsymbol{\epsilon}$ is a small random uniform 3D perturbation [S1].
    c. **Total Loss:** $\mathcal{L} = \mathcal{L}_{rec} + \lambda \mathcal{L}_{reg}$ [S1].

9.  **Optimization Schedule:**
    a. Start with a large interval $\Delta_{\max} = 1.0$ to capture coarse geometry.
    b. Decay $\Delta$ monotonically using the schedule:
       $$ \Delta_k = \max(\Delta_{\max} \exp(-k\beta), \Delta_{\min}) $$
       where $k$ is the iteration number, $\beta = 1.5 \times 10^{-5}$, and $\Delta_{\min} = 0.05$ [S1].
    c. Perform two decay steps after 200k and 400k iterations [S1].
    d. Optimize using Adam with learning rate 0.0001 for 450k total iterations, processing $M=1024$ pixels per iteration [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class UNISURF:
    def __init__(self, config):
        # Occupancy Network: 8-layer MLP, Softplus, hidden 256
        self.o_net = nn.Sequential(
            nn.Linear(3 + 2*6*2, 256), # + Fourier features for x
            nn.Softplus(),
            *[nn.Sequential(nn.Linear(256, 256), nn.Softplus()) for _ in range(7)]
        )
        # Color Network: 4-layer MLP, ReLU
        self.c_net = nn.Sequential(
            nn.Linear(3 + 2*6*2 + 3 + 256 + 2*4*2, 128), # x, n, h, d features
            nn.ReLU(),
            *[nn.Sequential(nn.Linear(128, 128), nn.ReLU()) for _ in range(3)],
            nn.Linear(128, 3)
        )
        self.config = config
        self.delta_max = 1.0
        self.delta_min = 0.05
        self.beta = 1.5e-5
        self.delta = self.delta_max

    def fourier_encode(self, x, k_octaves):
        # Standard Fourier feature mapping
        freq_bands = 2.0 ** torch.linspace(0, k_octaves - 1, k_octaves).to(x.device)
        x_proj = x.unsqueeze(-1) * freq_bands.unsqueeze(0)
        return torch.cat([torch.sin(x_proj), torch.cos(x_proj)], dim=-1).flatten(-2)

    def get_normal_and_features(self, x, o_net):
        x.requires_grad_(True)
        o = o_net(x)
        grad = torch.autograd.grad(o.sum(), x, create_graph=True)[0]
        normal = grad / (grad.norm(dim=-1, keepdim=True) + 1e-6)
        # Get features from intermediate layer (e.g., after first layer)
        # Simplified: assume we can access intermediate activations or re-run
        # For implementation, one might store features during forward pass
        # Here we assume a helper to get features
        features = self.get_geometry_features(x, o_net) 
        return normal, features

    def get_geometry_features(self, x, o_net):
        # Placeholder for extracting intermediate layer output
        # In practice, modify o_net forward to return features
        return torch.zeros_like(x) # [G] Simplified for pseudocode completeness

    def root_find(self, o_net, ray_o, ray_d, num_samples=256):
        # Sample 256 points along ray
        t_vals = torch.linspace(0, 10, num_samples).to(ray_o.device) # Assume max depth 10
        points = ray_o.unsqueeze(1) + t_vals.unsqueeze(0) * ray_d.unsqueeze(1)
        occs = o_net(self.fourier_encode(points, 6)).squeeze(-1)
        
        # Find sign change
        # [G] Robust root finding logic: find index where occs crosses 0.5 or sign changes
        # Secant method refinement
        # ... implementation of secant method ...
        t_s = 0.5 # Placeholder for found root
        x_s = ray_o + t_s * ray_d
        return t_s, x_s

    def render(self, ray_o, ray_d, o_net, c_net, iteration):
        # Update delta based on schedule
        k = iteration
        self.delta = max(self.delta_max * torch.exp(-k * self.beta), self.delta_min)
        
        # 1. Root finding
        t_s, x_s = self.root_find(o_net, ray_o, ray_d)
        
        # 2. Surface Normal and Features
        n_s, h_s = self.get_normal_and_features(x_s, o_net)
        
        # 3. Volume Rendering Sampling
        N = 64
        # Stratified sampling in interval [t_s - delta, t_s + delta]
        t_interval = torch.linspace(t_s - self.delta, t_s + self.delta, N + 1).to(ray_o.device)
        t_samples = t_interval[:-1] + torch.rand(N).to(ray_o.device) * (t_interval[1] - t_interval[0])
        x_samples = ray_o.unsqueeze(1) + t_samples.unsqueeze(0) * ray_d.unsqueeze(1)
        
        # Free space sampling (32 points)
        N_free = 32
        t_free = torch.rand(N_free).to(ray_o.device) * (t_s - self.delta)
        x_free = ray_o.unsqueeze(1) + t_free.unsqueeze(0) * ray_d.unsqueeze(1)
        
        # Combine samples
        x_all = torch.cat([x_samples, x_free], dim=1)
        
        # Compute occupancy for all samples
        x_all_enc = self.fourier_encode(x_all, 6)
        o_all = o_net(x_all_enc).squeeze(-1) # [batch, num_samples]
        
        # Compute alphas (occupancy acts as alpha here)
        # Note: The paper uses o directly in the product term as an indicator
        # alpha_i = o_i, but strictly speaking, volume rendering uses 1-exp(-sigma*delta)
        # UNISURF simplifies this to o_i for solid objects
        alpha = o_all
        
        # Compute colors
        # For interval samples, use normal and features
        # For free space, color is background (e.g., 0)
        # [G] Simplified: compute colors for all, mask free space later or set to 0
        n_all, h_all = self.get_normal_and_features(x_all, o_net)
        d_enc = self.fourier_encode(ray_d, 4)
        c_all = c_net(torch.cat([x_all_enc, n_all, h_all, d_enc], dim=-1))
        
        # Apply free space mask (set color to 0 or background)
        # [G] Assuming first N samples are interval, rest are free
        c_all[:, N:] = 0.0 
        
        # Volume Rendering Equation
        # T_i = product(1 - alpha_j) for j < i
        # [G] Cumulative product
        alpha_cumprod = torch.cumprod(1 - alpha, dim=1)
        weights = alpha * alpha_cumprod[:, :-1] # Weight for each sample
        # Handle first sample
        weights[:, 0] = alpha[:, 0]
        
        rendered_color = torch.sum(weights.unsqueeze(-1) * c_all, dim=1)
        
        # Surface rendering color (for loss or fast inference)
        c_surface = c_net(torch.cat([self.fourier_encode(x_s, 6), n_s, h_s, d_enc], dim=-1))
        
        return rendered_color, c_surface, x_s, n_s

    def compute_loss(self, rendered_color, observed_color, x_s, o_net):
        L_rec = torch.mean(torch.abs(rendered_color - observed_color))
        
        # Regularization
        eps = torch.randn_like(x_s) * 0.01
        x_s_pert = x_s + eps
        n_s_pert, _ = self.get_normal_and_features(x_s_pert, o_net)
        L_reg = torch.mean(torch.norm(n_s - n_s_pert, dim=1))
        
        return L_rec + 0.1 * L_reg # Lambda = 0.1 typical
```

## Subtopics

### Neural Surface Field
A resolution-independent representation for modeling 3D surfaces, formulated as an implicit function $f_s$ mapping 3D points $\mathbf{x}$ to a surface field value $s \in [0, 1]$ [S3]. Unlike UNISURF which unifies volume and surface rendering, Neural Surface Fields often isolate geometry and texture (e.g., PIFu predicts color on points where $f_s(\mathbf{x})=0.5$), which can hinder geometry optimization under texture supervision [S3].

## Operating characteristics
**Time Complexity:** Surface rendering is twice as fast as volume rendering [S1]. Volume rendering requires sampling $N=64$ points plus free space samples and computing gradients for normals/features at each point. Surface rendering only requires evaluation at the root $\mathbf{x}_s$.
**Memory Complexity:** Depends on the MLP sizes (8-layer for occupancy, 4-layer for color) and the number of samples per ray. No large voxel grids are stored, unlike traditional volume rendering.
**Performance:** On the DTU MVS dataset, UNISURF attains results competitive with state-of-the-art implicit neural reconstruction methods like IDR, which use strong mask supervision, despite not using masks [S1].
**Strengths:** Eliminates the need for hierarchical two-stage sampling (unlike NeRF) and per-pixel masks (unlike IDR). Combines coarse structure learning (volume) with precise surface refinement (surface).
**Limitations:** Requires careful tuning of the sampling interval decay schedule. Root-finding can be unstable if the occupancy field is not smooth.

## Background
UNISURF was introduced in 2021 by Yariv et al. in the paper "UNISURF: Unifying Neural Implicit Surfaces and Radiance Fields for Multi-View Reconstruction" [S1]. It builds on [Neural Radiance Fields](52-neural-radiance-fields.md) and [IDR](13663-hybrid-surface-volume-and-mesh-representation.md). The method addresses the limitation of surface rendering methods that require masks and the limitation of volume rendering methods that produce noisy surfaces. It is used for multi-view 3D reconstruction on datasets like DTU and BlendedMVS [S1].

## Sources
[S1] UNISURF: Unifying Neural Implicit Surfaces and Radiance Fields for Multi-View Reconstruction, https://doi.org/10.1109/iccv48922.2021.00554, fetched 2026-10-26
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S3] DoubleField: Bridging the Neural Surface and Radiance Fields for High-fidelity Human Reconstruction and Rendering, https://doi.org/10.1109/cvpr52688.2022.01541, fetched 2026-10-29
