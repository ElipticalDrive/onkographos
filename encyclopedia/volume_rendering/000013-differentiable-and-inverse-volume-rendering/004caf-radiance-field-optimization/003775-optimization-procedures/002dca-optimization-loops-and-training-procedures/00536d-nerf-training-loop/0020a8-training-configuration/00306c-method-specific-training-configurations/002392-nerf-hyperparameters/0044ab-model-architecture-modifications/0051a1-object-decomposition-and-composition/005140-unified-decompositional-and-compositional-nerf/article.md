# Unified Decompositional and Compositional NeRF

Also called: Joint decomposition and composition framework, Two-stage NeRF for joint decomposition and composition, UDC-NeRF, Unifed decompositional and compositional NeRF

## Overview
Unified Decompositional and Compositional NeRF (UDC-NeRF) is a two-stage framework that jointly learns object-level and scene-level implicit representations to enable both novel view synthesis and scene editing [S1]. Unlike methods that treat global scene modeling and local object decomposition as separate tasks, UDC-NeRF unites them in a single pipeline where a coarse global field guides sampling for a fine-grained stage that decomposes the scene into distinct object and background radiance fields [S1]. It distinguishes itself from siblings like [DM-NeRF](13525-dm-nerf.md) by explicitly modeling the interaction between decomposition and composition through a one-hot activation mechanism and residual composition, allowing for disentangled object manipulation while maintaining global consistency [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ multi-view posed images $\{x_i\}_{i=1}^N$ and their corresponding instance semantic masks $\{s_{ij}\}_{i=1, j=1}^{N, K+1}$ for a static scene, where $K$ is the number of object instances and the background is treated as instance index 1 [S1].
*   **Outputs:** A composited radiance field $(c_{comp}, \sigma_{comp})$ for novel view synthesis, and individual object/background radiance fields $(c^k_{object}, \sigma^k_{object})$ for scene editing [S1].

**Data Structures:**
*   **Shared Backbone:** A 4-layer Multi-Layer Perceptron (MLP) [S1].
    *   Input: Positional encoded position $\gamma(\mathbf{p})$ and view direction $\gamma(\mathbf{d})$.
    *   Output: Generic implicit scene feature vector $\mathbf{f}$ with dimension 256 [S1].
*   **Guidance Head:** MLPs following the backbone in the coarse stage [S1].
    *   Output: Global color $c_{guidance}$ and density $\sigma_{guidance}$ [S1].
*   **Object/Background Heads:** Separate 3-layer MLPs for each instance $k \in \{1, \dots, K+1\}$ [S1].
    *   Input: Feature vector $\mathbf{f}$ concatenated with a learnable object code for instance $k$ [S1].
    *   Output: Instance color $c^k_{object}$ and density $\sigma^k_{object}$ [S1].
*   **Learnable Codes:** A set of vectors, one for each of the $K+1$ instances (background + $K$ objects) [S1].

**Algorithm:**
The method operates in two stages: Coarse Guidance and Fine Decomposition/Composition.

1.  **Coarse Stage (Guidance):**
    *   Sample points along camera rays.
    *   Pass points through the Shared Backbone and Guidance Head to predict global color $c_{guidance}$ and density $\sigma_{guidance}$ [S1].
    *   Use the distribution of $\sigma_{guidance}$ to guide point sampling for the fine stage (coarse-to-fine sampling) [S1].

2.  **Fine Stage (Decomposition):**
    *   For each sampled point $\mathbf{p}$ and direction $\mathbf{d}$, compute generic feature $\mathbf{f}$ using the Shared Backbone [S1].
    *   For each instance $k \in \{1, \dots, K+1\}$, concatenate $\mathbf{f}$ with the learnable code for instance $k$ and pass through the respective Head to predict $c^k_{object}$ and $\sigma^k_{object}$ [S1].
    *   Apply ReLU to $\sigma^k_{object}$ during rendering to ensure positivity [S1].

3.  **Composition (One-Hot Activation):**
    *   To combine the $K+1$ branches, UDC-NeRF uses a differentiable one-hot selection based on volume density [S1].
    *   Compute soft probabilities $p_i$ for each object $i$ using Gumbel-Softmax on the predicted densities:
        $$
        p_i = \frac{\exp((\sigma^i_{object} + g_i)/\tau)}{\sum_j \exp((\sigma^j_{object} + g_j)/\tau)}
        $$
        where $g_i$ are i.i.d. samples from Gumbel(0,1) and temperature $\tau = 0.1$ [S1].
    *   Compute one-hot composition weights $M_{obj}$ using a stop-gradient operation to allow backpropagation through the soft probabilities $P$ (where $P$ is the vector of $p_i$):
        $$
        M_{obj} = f_h(\arg\max[g_i + \log \sigma^i_{object}]) - f_{sp}(P) + P
        $$
        where $f_h$ is the one-hot function and $f_{sp}$ is the stop-gradient operator [S1].
    *   Compute the composited color and density:
        $$
        c_{comp} = \sum_{k=1}^{K+1} M_{obj}^k c^k_{object}, \quad \sigma_{comp} = \sum_{k=1}^{K+1} M_{obj}^k \sigma^k_{object}
        $$
        [S1].

4.  **Residual Composition:**
    *   Combine the composited radiance field $(c_{comp}, \sigma_{comp})$ with the residual prediction from the coarse stage $(c_{guidance}, \sigma_{guidance})$ using point-wise average pooling:
        $$
        c_{final} = \text{AvgPool}(c_{comp}, c_{guidance}), \quad \sigma_{final} = \text{AvgPool}(\sigma_{comp}, \sigma_{guidance})
        $$
        [S1].
    *   Render the final image using standard volume rendering with $c_{final}$ and $\sigma_{final}$ [S1].

**Loss Functions:**
The total loss is $L_{overall} = L_{comp} + \lambda_1 L_{object} + \lambda_2 L_{color\_pseudo} + \lambda_3 L_{one-hot}$ [S1].

1.  **Composition Loss ($L_{comp}$):** Standard color loss between rendered final pixels and ground truth [S1].
2.  **Object Loss ($L_{object}$):** Enforces consistency for individual object branches using semantic masks $M_o(r)^k$ and background mask $M_b(r)^1$:
    $$
    L_{object} = \sum_{k=1}^{K+1} M_o(r)^k \| \hat{C}^k_{object}(r) - C(r) \|^2_2 + \lambda_b M_b(r)^1 \| \hat{O}^1_{object}(r) - M_o(r)^1 \|^2_2
    $$
    where $\hat{C}^k_{object}$ and $\hat{O}^k_{object}$ are the rendered color and opacity for the $k$-th object branch:
    $$
    \hat{C}^k_{object}(r) = \sum_{i=1}^{N} T^k_i (1 - \exp(-\sigma^k_{object} \delta_i)) c^k_{object}
    $$
    $$
    \hat{O}^k_{object}(r) = \sum_{i=1}^{N} T^k_i (1 - \exp(-\sigma^k_{object} \delta_i))
    $$
    with transmittance $T^k_i = \exp(-\sum_{j=1}^{i-1} \sigma^k_{object} \delta_j)$ [S1].
3.  **One-Hot Density Regularization ($L_{one-hot}$):** Constrains the background branch (index 1) in regions occupied by foreground objects. Define binary mask $M_c$:
    $$
    M_c = [\arg\max([\sigma^1_{object}, \dots, \sigma^{K+1}_{object})] > 1]
    $$
    Then:
    $$
    L_{one-hot} = M_c \| \sigma^1_{object} - \sigma_0 \|^2_2
    $$
    where $\sigma_0 = -0.01$ [S1].
4.  **In-Painting Pseudo Supervision ($L_{color\_pseudo}$):** Handles occluded background regions using a pre-trained lama model to generate pseudo-color $C^1_{pseudo}(r)$ for uncertain background regions $M^1_b(r)$:
    $$
    L_{color\_pseudo} = M^1_b(r) \| \hat{C}^1_{object}(r) - C^1_{pseudo}(r) \|^2_2
    $$
    where $M^1_b(r)$ is a 2D spatial weight map with value 0.05 for uncertain background regions [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class UDCNeRF:
    def __init__(self, num_objects, feature_dim=256, num_coarse_samples=64, num_fine_samples=48):
        self.num_objects = num_objects + 1  # K objects + 1 background
        self.num_coarse_samples = num_coarse_samples
        self.num_fine_samples = num_fine_samples
        self.tau = 0.1
        self.sigma_0 = -0.01
        
        # Shared Backbone: 4-layer MLP
        self.backbone = nn.Sequential(
            nn.Linear(12 + 12, 256), # 6 pos enc levels * 2 + 3 view dir * 4 levels * 2? 
                                     # Note: Source says 10 levels coords, 4 levels view. 
                                     # Standard NeRF pos enc: 2*L+1 per dim. 
                                     # Assuming standard implementation details not fully specified, 
                                     # we define the structure.
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, feature_dim)
        )
        
        # Guidance Head (Coarse)
        self.guidance_head = nn.Sequential(
            nn.Linear(feature_dim, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 4) # 3 for color, 1 for density
        )
        
        # Object/Background Heads: Separate heads for each instance
        self.heads = nn.ModuleList([
            nn.Sequential(
                nn.Linear(feature_dim + 256, 256), # +256 for object code
                nn.ReLU(),
                nn.Linear(256, 256),
                nn.ReLU(),
                nn.Linear(256, 4)
            ) for _ in range(self.num_objects)
        ])
        
        # Learnable object codes
        self.object_codes = nn.Parameter(torch.randn(self.num_objects, 256))
        
    def positional_encoding(self, x, num_levels=10):
        # Standard NeRF positional encoding
        # x shape: (..., 3)
        scales = torch.tensor([2**i for i in range(num_levels)], dtype=x.dtype, device=x.device)
        shaped = x[..., None, :] * scales[None, ..., None]
        shaped = shaped.reshape(*x.shape[:-1], -1)
        return torch.cat([x, torch.sin(shaped), torch.cos(shaped)], dim=-1)

    def sample_points(self, rays_o, rays_d, near, far):
        # Coarse sampling
        t_coarse = torch.linspace(near, far, self.num_coarse_samples)
        points_coarse = rays_o[:, None, :] + rays_d[:, None, :] * t_coarse[:, None]
        
        # Predict coarse density for fine sampling
        # Flatten for MLP input
        points_coarse_flat = points_coarse.reshape(-1, 3)
        directions_flat = rays_d[:, None, :].expand(-1, self.num_coarse_samples, -1).reshape(-1, 3)
        
        # Encode
        pos_enc = self.positional_encoding(points_coarse_flat, num_levels=10)
        dir_enc = self.positional_encoding(directions_flat, num_levels=4)
        input_feat = torch.cat([pos_enc, dir_enc], dim=-1)
        
        # Backbone
        f_coarse = self.backbone(input_feat)
        # Guidance Head
        guidance_out = self.guidance_head(f_coarse)
        sigma_guidance = F.relu(guidance_out[..., 3])
        c_guidance = torch.sigmoid(guidance_out[..., :3])
        
        # Reshape back
        sigma_guidance = sigma_guidance.reshape(rays_o.shape[0], self.num_coarse_samples)
        
        # Fine sampling based on coarse density (importance sampling logic omitted for brevity, 
        # assuming standard NeRF fine sampling procedure is available)
        # For implementation completeness, we assume a function `sample_fine` exists 
        # that uses sigma_guidance to draw points.
        points_fine, t_fine = self.sample_fine(rays_o, rays_d, near, far, sigma_guidance)
        
        return points_coarse, points_fine, c_guidance, sigma_guidance

    def sample_fine(self, rays_o, rays_d, near, far, sigma_coarse):
        # Placeholder for importance sampling logic
        # In practice, this uses the cumulative density function of sigma_coarse
        # to sample points more densely where density is high.
        # Returns points_fine (B, N_fine, 3) and t_fine (B, N_fine)
        # Since the source does not provide the exact sampling algorithm, 
        # we note this step relies on standard NeRF importance sampling.
        pass

    def forward(self, rays_o, rays_d, near, far, semantic_masks=None):
        # 1. Coarse Stage
        points_coarse, points_fine, c_guidance, sigma_guidance = self.sample_points(rays_o, rays_d, near, far)
        
        # 2. Fine Stage Decomposition
        # Process fine points
        points_fine_flat = points_fine.reshape(-1, 3)
        directions_flat = rays_d[:, None, :].expand(-1, self.num_fine_samples, -1).reshape(-1, 3)
        
        pos_enc = self.positional_encoding(points_fine_flat, num_levels=10)
        dir_enc = self.positional_encoding(directions_flat, num_levels=4)
        input_feat = torch.cat([pos_enc, dir_enc], dim=-1)
        
        # Backbone
        f = self.backbone(input_feat) # (B*N, 256)
        
        # Predict for each object
        object_colors = []
        object_densities = []
        
        for k in range(self.num_objects):
            # Concatenate with object code
            code = self.object_codes[k] # (256,)
            input_head = torch.cat([f, code[None, :].expand(f.shape[0], -1)], dim=-1)
            
            out = self.heads[k](input_head)
            sigma = F.relu(out[..., 3])
            color = torch.sigmoid(out[..., :3])
            
            object_densities.append(sigma)
            object_colors.append(color)
            
        # Reshape to (B, N_fine, K+1, 3/1)
        object_densities = torch.stack(object_densities, dim=2) # (B, N, K+1)
        object_colors = torch.stack(object_colors, dim=2) # (B, N, K+1, 3)
        
        # 3. Composition
        # Gumbel-Softmax
        # Add noise
        g = torch.distributions.gumbel.Gumbel(0, 1)
        g_samples = g.sample(object_densities.shape).to(object_densities.device)
        
        # Soft probability
        logits = object_densities + g_samples
        p = F.softmax(logits / self.tau, dim=2) # (B, N, K+1)
        
        # One-hot weights
        # argmax on g + log(sigma)
        # log(sigma) might be -inf if sigma is 0. Add small epsilon.
        log_sigma = torch.log(object_densities + 1e-8)
        argmax_indices = torch.argmax(log_sigma + g_samples, dim=2, keepdim=True)
        
        # One-hot function
        one_hot = F.one_hot(argmax_indices.squeeze(2), num_classes=self.num_objects).float()
        
        # Stop gradient on P
        p_stop = p.detach()
        
        # M_obj = one_hot - stop_grad(P) + P
        M_obj = one_hot - p_stop + p # (B, N, K+1)
        
        # Compose
        # c_comp = sum(M_obj * c_obj)
        c_comp = torch.sum(M_obj[..., None] * object_colors, dim=2) # (B, N, 3)
        sigma_comp = torch.sum(M_obj * object_densities, dim=2) # (B, N)
        
        # 4. Residual Composition
        # Average pooling between coarse and fine predictions
        # Note: Coarse points and fine points are at different locations.
        # The source says "point-wise average pooling". This usually implies 
        # evaluating the residual field at the fine points or interpolating.
        # Assuming the guidance head can be evaluated at fine points directly.
        
        pos_enc_fine = self.positional_encoding(points_fine_flat, num_levels=10)
        dir_enc_fine = self.positional_encoding(directions_flat, num_levels=4)
        input_feat_fine = torch.cat([pos_enc_fine, dir_enc_fine], dim=-1)
        f_fine = self.backbone(input_feat_fine)
        guidance_out_fine = self.guidance_head(f_fine)
        
        c_guidance_fine = torch.sigmoid(guidance_out_fine[..., :3])
        sigma_guidance_fine = F.relu(guidance_out_fine[..., 3])
        
        c_guidance_fine = c_guidance_fine.reshape(rays_o.shape[0], self.num_fine_samples, 3)
        sigma_guidance_fine = sigma_guidance_fine.reshape(rays_o.shape[0], self.num_fine_samples)
        
        c_final = (c_comp + c_guidance_fine) / 2.0
        sigma_final = (sigma_comp + sigma_guidance_fine) / 2.0
        
        # 5. Volume Rendering
        # Standard NeRF rendering
        deltas = points_fine[:, :, 1:] - points_fine[:, :, :-1]
        deltas = torch.cat([deltas, torch.tensor([1e10], device=deltas.device).expand_as(deltas[..., :1])], dim=-1)
        
        # Transmittance
        alpha = 1.0 - torch.exp(-sigma_final * deltas)
        T = torch.cumprod(torch.exp(-sigma_final * deltas), dim=1)
        T = torch.cat([torch.ones_like(T[..., :1]), T[..., :-1]], dim=-1)
        
        weights = alpha * T
        rendered_color = torch.sum(weights[..., None] * c_final, dim=1)
        
        return rendered_color, {
            'object_densities': object_densities,
            'object_colors': object_colors,
            'M_obj': M_obj,
            'c_guidance': c_guidance_fine,
            'sigma_guidance': sigma_guidance_fine
        }
```

## Operating characteristics
*   **Time Complexity:** The method requires two forward passes through the backbone (coarse and fine) and $K+1$ forward passes through the heads per ray, making it computationally heavier than standard NeRF by a factor of roughly $K+1$ for the head predictions.
*   **Memory Complexity:** Requires storing learnable codes for $K+1$ objects and intermediate features for all object branches during the backward pass.
*   **Strengths:** Enables explicit object editing (removal, addition, repositioning) and novel view synthesis in a unified framework. The one-hot regularization and inpainting supervision improve background quality in occluded regions [S1].
*   **Limitations:** Performance degrades if the number of objects $K$ is large due to increased computational cost. Relies on accurate semantic masks for training [S1]. The 2D inpainting may introduce ambiguity for regions seen in other views, though multi-view consistency helps suppress this [S1].
*   **When to use:** When the goal is both high-quality novel view synthesis and detailed object-level editing/manipulation in a static scene with known object instances [S1].
*   **When not to use:** For dynamic scenes or scenes where object instances are unknown or not provided as semantic masks [S1].

## Background
UDC-NeRF was proposed in the paper "Learning Unified Decompositional and Compositional NeRF for Editable Novel View Synthesis" [S1]. It addresses the limitations of prior works like ObjectNeRF [S1], which learns separate networks for global scene and local object branches without modeling their interactions. UDC-NeRF unites decomposition and composition to model these interactions, improving the quality of both novel view synthesis and scene editing [S1]. The method builds on the standard NeRF architecture [S6, unofficial] and ObjectNeRF [S1], introducing a two-stage framework with a coarse guidance stage and a fine decomposition stage [S1].

## Sources
[S1] Learning Unified Decompositional and Compositional NeRF for Editable Novel View Synthesis, https://arxiv.org/abs/2308.02840v1, fetched 2026-10-26
