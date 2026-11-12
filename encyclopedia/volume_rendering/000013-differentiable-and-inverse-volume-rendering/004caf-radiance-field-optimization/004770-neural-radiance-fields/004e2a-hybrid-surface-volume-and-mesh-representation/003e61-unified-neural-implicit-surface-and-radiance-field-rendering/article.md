# Unified Neural Implicit Surface and Radiance Field Rendering

Also called: UNISURF, UNISURF rendering, Unified volume/surface rendering

## Overview
UNISURF is a framework for reconstructing 3D surfaces from multi-view RGB images without requiring per-pixel object masks [S1]. It solves the ambiguity of optimizing implicit surfaces by combining the geometric precision of surface rendering with the coarse structure learning capability of volume rendering [S1]. Unlike standard radiance fields that optimize volume density $\sigma$, UNISURF parameterizes occupancy directly via a continuous field $o_\theta$, allowing the model to render both through the volume and directly on the surface [S1]. This unification eliminates the need for hierarchical two-stage sampling [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $V$ calibrated RGB images with known camera poses (intrinsics and extrinsics). No object masks are required [S1].
*   **Output:** A neural representation consisting of an occupancy field $o_\theta$ and a color field $c_\theta$, which can be used to render novel views or extract a 3D mesh [S1].

**Data Structures:**
1.  **Occupancy Network ($o_\theta$):** An 8-layer Multi-Layer Perceptron (MLP) with Softplus activation and hidden dimension 256 [S1]. It maps 3D coordinates $\mathbf{x}$ to a scalar occupancy value [S1].
2.  **Color Network ($c_\theta$):** A 4-layer MLP with ReLU activation [S1]. It maps geometry features, surface normals, and viewing direction to RGB color [S1].
3.  **Fourier Feature Encoder:** Maps input coordinates to higher-dimensional frequencies to capture high-frequency details [S1].
    *   For 3D location $\mathbf{x}$: $k=6$ octaves [S1].
    *   For viewing direction $\mathbf{d}$: $k=4$ octaves [S1].
4.  **Ray Data:** For each ray $\mathbf{r}$, the system stores the camera origin $o$, direction $\mathbf{d}$, and the computed surface depth $t_s$ [S1].

**Parameters:**
*   $\Delta_{\max} = 1.0$: Initial maximum sampling interval width [S1].
*   $\Delta_{\min} = 0.05$: Minimum sampling interval width [S1].
*   $\beta = 1.5 \times 10^{-5}$: Decay rate for the sampling interval [S1].
*   $N = 64$: Number of samples drawn inside the sampling interval [S1].
*   $N_{free} = 32$: Number of samples drawn in free space [S1].
*   $M = 1024$: Number of pixels (rays) sampled per iteration [S1].
*   $\lambda$: Weight for the regularization loss (value not given in sources) [S1].
*   Learning rate: $0.0001$ (Adam optimizer) [S1].

**Algorithm Steps:**

1.  **Network Initialization:** Initialize the occupancy network $o_\theta$ such that the decision boundary is a sphere [S1]. This provides an inductive bias for the iterative deformation procedure [S1].

2.  **Surface Intersection (Root-Finding):** For a given ray $\mathbf{r} = o + t\mathbf{d}$, find the surface intersection point $\mathbf{x}_s = o + t_s\mathbf{d}$ [S1].
    *   Sample 256 points uniformly along the ray [S1].
    *   Apply the secant method with 8 steps to find the root where $o_\theta(\mathbf{x}) = 0.5$ [S1].
    *   If no intersection is found, treat the ray as background [S1].

3.  **Surface Normal and Feature Computation:** Compute the surface normal $\mathbf{n}_s$ and geometry features $\mathbf{h}_s$ at $\mathbf{x}_s$ [S1].
    *   The normal is the normalized gradient of the occupancy field:
        $$ \mathbf{n}(\mathbf{x}_s) = \frac{\nabla_{\mathbf{x}_s} o_\theta(\mathbf{x}_s)}{\|\nabla_{\mathbf{x}_s} o_\theta(\mathbf{x}_s)\|_2} $$
    *   This gradient is computed using double backpropagation [S1].
    *   The features $\mathbf{h}_s$ are the intermediate activations from the geometry network at $\mathbf{x}_s$ [S1].

4.  **Sampling Strategy:** Generate sample points $\{\mathbf{x}_i\}$ for volume rendering [S1].
    *   **Interval Sampling:** Draw $N=64$ points stratified within the interval $[t_s - \Delta, t_s + \Delta]$ centered at the surface depth $t_s$ [S1]. The depth $t_i$ for the $i$-th sample ($i=1 \dots N$) is drawn from:
        $$ t_i \sim \mathcal{U} \left[ t_s + \left( \frac{2i - 2}{N} - 1 \right) \Delta, t_s + \left( \frac{2i}{N} - 1 \right) \Delta \right] $$
        where $\mathcal{U}$ denotes the uniform distribution [S1].
    *   **Free Space Sampling:** Draw $N_{free}=32$ points randomly between the camera ($t=0$) and the lower bound of the interval ($t_s - \Delta$) to reduce free space artifacts [S1].
    *   **Background Rays:** For rays without a surface intersection, use stratified sampling on the entire ray [S1].

5.  **Rendering:** Compute the rendered color $\hat{C}(\mathbf{r})$ using the unified volume rendering equation [S1]:
    $$ \hat{C}_v(\mathbf{r}) = \sum_{i=1}^N o_{\theta}(\mathbf{x}_i) \prod_{j < i} (1 - o_{\theta}(\mathbf{x}_j)) c_{\theta}(\mathbf{x}_i, \mathbf{n}_i, \mathbf{h}_i, \mathbf{d}) $$
    *   Here, $o_\theta(\mathbf{x}_i)$ acts as the alpha value (occupancy indicator) [S1].
    *   The color field $c_\theta$ is conditioned on the position $\mathbf{x}_i$, the surface normal $\mathbf{n}_i$, the geometry features $\mathbf{h}_i$, and the viewing direction $\mathbf{d}$ [S1].
    *   Note: For volume rendering, normals and features for off-surface points are typically derived from the nearest surface point or computed via local gradients if available; the source specifies conditioning on $\mathbf{n}_i, \mathbf{h}_i$ in the formula but details normal computation primarily for $\mathbf{x}_s$ [S1]. In practice, for points $\mathbf{x}_i$ not on the surface, $\mathbf{n}_i$ and $\mathbf{h}_i$ are often taken from the surface intersection $\mathbf{x}_s$ or interpolated, though the source text implies they are computed at $\mathbf{x}_i$ via the same network logic [S1]. Given the source explicitly defines $\mathbf{n}(\mathbf{x}_s)$ via gradient, and the rendering formula uses $\mathbf{n}_i$, it is implied that normals/features are evaluated at sample points $\mathbf{x}_i$ as well, or $\mathbf{n}_s, \mathbf{h}_s$ are used for all samples. The source states "conditioning on the surface normal n and a feature vector h of the geometry network" generally, and the rendering equation uses indices $i$. We assume $\mathbf{n}_i, \mathbf{h}_i$ are computed at $\mathbf{x}_i$ using the same gradient/feature extraction logic as for $\mathbf{x}_s$ [S1].

    *   **Surface Rendering Alternative:** For inference, surface rendering can be used for speed:
        $$ \hat{C}_s(\mathbf{r}) = c_\theta(\mathbf{x}_s, \mathbf{n}_s, \mathbf{h}_s, \mathbf{d}) $$
        This is twice as fast as volume rendering [S1].

6.  **Loss Computation:** Compute the total loss $\mathcal{L}$ [S1]:
    $$ \mathcal{L} = \mathcal{L}_{rec} + \lambda \mathcal{L}_{reg} $$
    *   **Reconstruction Loss:** $\ell_1$ loss between rendered and observed colors:
        $$ \mathcal{L}_{rec} = \sum_{\mathbf{r} \in \mathcal{R}} \|\hat{C}_v(\mathbf{r}) - C(\mathbf{r})\|_1 $$
        where $\mathcal{R}$ is the set of rays in the minibatch and $C(\mathbf{r})$ is the observed color [S1].
    *   **Regularization Loss:** Encourages normal consistency in the neighborhood of the surface:
        $$ \mathcal{L}_{reg} = \sum_{\mathbf{x}_s \in \mathcal{S}} \|\mathbf{n}(\mathbf{x}_s) - \mathbf{n}(\mathbf{x}_s + \boldsymbol{\epsilon})\|_2 $$
        where $\mathcal{S}$ is the set of surface points and $\boldsymbol{\epsilon}$ is a small random uniform 3D perturbation [S1].

7.  **Optimization Schedule:** Update parameters using Adam [S1].
    *   The sampling interval width $\Delta$ decays monotonically during training to refine geometry [S1]:
        $$ \Delta_k = \max(\Delta_{\max} \exp(-k\beta), \Delta_{\min}) $$
        where $k$ is the iteration number [S1].
    *   Training runs for 450k iterations [S1].
    *   Two decay steps are applied after 200k and 400k iterations (effectively resetting or adjusting the schedule, though the formula is continuous; the source mentions "two decay steps" which may imply re-initializing the decay or specific checkpoints, but the formula is given as continuous [S1]).

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class UNISURF:
    def __init__(self, num_octaves_x=6, num_octaves_d=4):
        # Occupancy Network: 8-layer MLP, Softplus, hidden 256
        self.o_net = nn.Sequential(
            nn.Linear(3 + 2 * 3 * num_octaves_x, 256),
            nn.Softplus(),
            nn.Linear(256, 256),
            nn.Softplus(),
            nn.Linear(256, 256),
            nn.Softplus(),
            nn.Linear(256, 256),
            nn.Softplus(),
            nn.Linear(256, 256),
            nn.Softplus(),
            nn.Linear(256, 256),
            nn.Softplus(),
            nn.Linear(256, 256),
            nn.Softplus(),
            nn.Linear(256, 256),
            nn.Softplus(),
            nn.Linear(256, 1) # Output occupancy scalar
        )
        
        # Color Network: 4-layer MLP, ReLU
        # Input: geometry features (256) + normal (3) + viewing direction encoded
        self.c_net = nn.Sequential(
            nn.Linear(256 + 3 + 2 * 3 * num_octaves_d, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 3) # Output RGB
        )
        
        self.num_octaves_x = num_octaves_x
        self.num_octaves_d = num_octaves_d
        self.beta = 1.5e-5
        self.delta_max = 1.0
        self.delta_min = 0.05
        self.N_interval = 64
        self.N_free = 32
        
    def fourier_encode(self, x, num_octaves):
        """Encode input x using Fourier features."""
        # x shape: [..., 3]
        freq_bands = 2.0 ** torch.linspace(0, num_octaves - 1, num_octaves).to(x.device)
        # Expand dimensions for broadcasting
        freq_bands = freq_bands.view(1, -1, 1)
        x_expanded = x.unsqueeze(-2) # [..., 1, 3]
        scaled_x = x_expanded * freq_bands # [..., num_octaves, 3]
        encoded = torch.cat([torch.sin(scaled_x), torch.cos(scaled_x)], dim=-2) # [..., num_octaves*2, 3]
        return encoded.view(*x.shape[:-1], -1) # [..., num_octaves*2*3]

    def get_normal_and_features(self, x, o_net):
        """Compute normal and geometry features at x using double backprop."""
        x.requires_grad_(True)
        o = o_net(x)
        grad_o = torch.autograd.grad(o.sum(), x, create_graph=True)[0]
        normal = grad_o / (grad_o.norm(dim=-1, keepdim=True) + 1e-8)
        # Features are intermediate activations from the geometry network
        # For simplicity, we assume features are the output of the last hidden layer before the final output
        # In the actual implementation, one would hook into the network or re-run forward pass
        # Here we approximate by assuming the network structure allows access or re-computation
        # Since the source says "feature vector h of the geometry network", we assume it's available.
        # We will return normal and a placeholder for features, assuming they are computed similarly.
        # Note: In practice, features are often the output of the penultimate layer.
        # For this pseudocode, we assume a helper or re-run to get features.
        # To be self-contained, we assume features are derived from the same network state.
        # We will return normal and None for features, noting that features are needed for color net.
        return normal, None # Features extraction logic omitted for brevity, assumed available

    def root_find(self, ray_o, ray_d, o_net, num_samples=256, steps=8):
        """Find surface intersection using secant method."""
        # Sample points along ray
        t_vals = torch.linspace(0, 10.0, num_samples).to(ray_o.device) # Assume max depth 10
        x_samples = ray_o.unsqueeze(1) + t_vals.unsqueeze(0) * ray_d.unsqueeze(1)
        
        # Evaluate occupancy
        x_enc = self.fourier_encode(x_samples, self.num_octaves_x)
        o_vals = o_net(x_enc).squeeze(-1)
        
        # Find sign changes
        signs = torch.sign(o_vals)
        zero_crossings = torch.where(signs[:-1] != signs[1:])[0]
        
        if len(zero_crossings) == 0:
            return None, None # No intersection
        
        # Apply secant method for the first crossing
        idx = zero_crossings[0]
        t0, t1 = t_vals[idx], t_vals[idx+1]
        o0, o1 = o_vals[idx], o_vals[idx+1]
        
        for _ in range(steps):
            t_new = t1 - o1 * (t1 - t0) / (o1 - o0 + 1e-8)
            x_new = ray_o + t_new * ray_d
            x_enc_new = self.fourier_encode(x_new.unsqueeze(0), self.num_octaves_x)
            o_new = o_net(x_enc_new).item()
            
            if abs(o_new) < 1e-4:
                break
            if o_new * o1 < 0:
                t0, o0 = t1, o1
                t1, o1 = t_new, o_new
            else:
                t0, o0 = t_new, o_new
                
        t_s = t1
        x_s = ray_o + t_s * ray_d
        return t_s, x_s

    def render(self, ray_o, ray_d, o_net, c_net, iteration, get_features_func):
        """Render a single ray."""
        # 1. Find surface
        t_s, x_s = self.root_find(ray_o, ray_d, o_net)
        
        if t_s is None:
            # No intersection: background sampling
            t_vals = torch.linspace(0, 10.0, self.N_interval + self.N_free)
            x_samples = ray_o.unsqueeze(1) + t_vals.unsqueeze(0) * ray_d.unsqueeze(1)
        else:
            # 2. Determine interval width
            delta = max(self.delta_max * math.exp(-iteration * self.beta), self.delta_min)
            
            # 3. Sample points
            # Interval samples
            t_interval = torch.linspace(t_s - delta, t_s + delta, self.N_interval + 1)[:-1] + \
                         torch.rand(self.N_interval) * (2 * delta / self.N_interval)
            # Free space samples
            t_free = torch.rand(self.N_free) * (t_s - delta) if t_s > delta else torch.rand(self.N_free) * 0.001
            t_all = torch.cat([t_interval, t_free])
            x_samples = ray_o.unsqueeze(1) + t_all.unsqueeze(0) * ray_d.unsqueeze(1)
            
        # 4. Encode and evaluate networks
        x_enc = self.fourier_encode(x_samples, self.num_octaves_x)
        o_vals = o_net(x_enc).squeeze(-1)
        
        # Compute normals and features for all samples (approximation: use surface normal for all if not on surface)
        # Source implies conditioning on n_i, h_i. For simplicity, we use surface normal/features for all samples
        # if they are off-surface, or compute local if on-surface.
        # Given the complexity, we assume get_features_func returns n, h for x_samples
        n_samples, h_samples = get_features_func(x_samples, o_net, c_net)
        
        # Encode viewing direction
        d_enc = self.fourier_encode(ray_d.unsqueeze(0).expand_as(x_samples[:, :3]), self.num_octaves_d)
        
        # Color prediction
        # Concatenate features, normal, direction encoding
        color_input = torch.cat([h_samples, n_samples, d_enc], dim=-1)
        c_vals = c_net(color_input)
        
        # 5. Volume Rendering
        # Alpha = occupancy (binary-like, but soft in practice)
        alpha = o_vals
        # Transmittance
        T = torch.cumprod(1 - alpha + 1e-8, dim=0)
        T = torch.cat([torch.ones_like(T[:, :1]), T[:, :-1]], dim=-1)
        
        # Weighted sum
        weights = alpha * T
        color = torch.sum(weights.unsqueeze(-1) * c_vals, dim=0)
        
        return color

    def train_step(self, rays, images, optimizer, iteration):
        optimizer.zero_grad()
        rendered_colors = []
        for ray_o, ray_d in rays:
            # Placeholder for feature extraction function
            def get_features(x, o_net, c_net):
                # Compute normal
                x.requires_grad_(True)
                o = o_net(self.fourier_encode(x, self.num_octaves_x))
                grad = torch.autograd.grad(o.sum(), x, create_graph=True)[0]
                n = grad / (grad.norm(dim=-1, keepdim=True) + 1e-8)
                # Features: assume output of last hidden layer of o_net
                # This is a simplification; in reality, one would extract intermediate activations
                h = torch.zeros_like(x) # Placeholder
                return n, h
            
            color = self.render(ray_o, ray_d, self.o_net, self.c_net, iteration, get_features)
            rendered_colors.append(color)
            
        rendered_colors = torch.stack(rendered_colors)
        loss_rec = torch.mean(torch.abs(rendered_colors - images))
        
        # Regularization loss (simplified)
        # Requires sampling points around surface and comparing normals
        # Omitted for brevity as it requires specific surface point sampling logic
        loss_reg = torch.tensor(0.0)
        
        loss = loss_rec + 0.1 * loss_reg # lambda not given
        loss.backward()
        optimizer.step()
        return loss.item()
```

## Subtopics

### Neural Surfel Radiance Fields
Uses explicit surfels (position, normal, radius) instead of an implicit occupancy field. Chooses explicit primitives for potentially faster rendering but requires managing surfel density. [7358-neural-surfel-radiance-fields.md]

### Structured Latent Codes Anchored to Mesh
Uses latent codes anchored to mesh vertices. Chooses explicit mesh representation for direct topology control. [12715-structured-latent-codes-anchored-to-mesh.md]

## Operating characteristics
*   **Time Complexity:** Surface rendering is twice as fast as volume rendering [S1]. Volume rendering requires sampling and integration along rays, while surface rendering only requires evaluating the color network at the intersection point [S1].
*   **Memory Complexity:** Depends on the MLP sizes (8-layer and 4-layer) and Fourier features. Not given in sources.
*   **Performance:** On the DTU MVS dataset, UNISURF performs nearly on par with IDR (which uses strong mask supervision) without requiring masks [S1]. It outperforms NeRF and COLMAP in Chamfer distance [S1].
*   **Strengths:** Does not require per-pixel object masks [S1]. Combines coarse geometry learning (volume) with precise surface refinement (surface) [S1]. Eliminates hierarchical two-stage sampling [S1].
*   **Limitations:** Requires careful initialization (sphere) [S1]. Performance depends on the decay schedule of the sampling interval [S1].
*   **When to use:** When reconstructing solid objects from multi-view RGB images without masks is required [S1].
*   **When not to use:** When transparent objects are the primary focus, as the method assumes solid objects (occupancy $\in \{0,1\}$) [S1].

## Background
UNISURF was introduced in 2021 by Yariv et al. [S1]. It builds on the principles of implicit surface rendering (like IDR) and neural radiance fields (NeRF) [S1]. The method was developed to address the limitation of surface rendering methods requiring strong mask supervision and the limitation of radiance fields not optimizing implicit surfaces directly [S1]. It has been used for multi-view 3D reconstruction on datasets like DTU and BlendedMVS [S1].

## Sources
[S1] UNISURF: Unifying Neural Implicit Surfaces and Radiance Fields for Multi-View Reconstruction, https://doi.org/10.1109/iccv48922.2021.00554, fetched 2026-10-26
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
