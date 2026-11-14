# Unified Decompositional and Compositional NeRF

Also called: Joint decomposition and composition framework, Two-stage NeRF for joint decomposition and composition, UDC-NeRF, Unifed decompositional and compositional NeRF

## Overview
Unified Decompositional and Compositional NeRF (UDC-NeRF) is a framework for joint scene decomposition and composition that learns both object-level and scene-level implicit representations to enable novel view synthesis and object editing in a unified pipeline [S1]. Unlike methods that treat global synthesis and local editing as separate tasks (e.g., [ObjectNeRF](13525-dm-nerf.md)), UDC-NeRF models the interactions between them by using a coarse global field to guide sampling and a fine-grained stage to decompose the scene into distinct object/background components [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ multi-view posed images $\{x_i\}_{i=1}^N$ and their corresponding instance semantic masks $\{s_{ij}\}_{i=1, j=1}^{N, K+1}$ for a static scene, where $K$ is the number of object instances and the background is treated as instance index 1 [S1].
*   **Outputs:** A composited radiance field for novel view synthesis and decomposed object radiance fields for editing [S1].

**Data Structures:**
*   **Shared Backbone:** A 4-layer MLP that takes a 3D point coordinate $p=(x,y,z)$ and view direction $d$ as input, outputting a generic implicit scene feature vector $f$ of dimension 256 [S1].
*   **Heads:** Three-layer MLPs for the guidance head, background head, and object head. Each head predicts RGB color $c$ and volume density $\sigma$ [S1].
*   **Learnable Codes:** A set of learnable codes, one for each of the $K$ objects and the background, used as conditions for the heads [S1].
*   **Positional Encoding:** 10 levels for coordinates and 4 levels for ray directions [S1].

**Algorithm Steps:**

1.  **Coarse Stage (Guidance):**
    The guidance head predicts a global scene color $c_{guidance}$ and density $\sigma_{guidance}$ [S1]. This distribution is used to guide point sampling for the fine stage [S1].

2.  **Fine Stage Decomposition:**
    For each sampled point $p$ and direction $d$:
    a. The shared backbone predicts the generic feature $f$ [S1].
    b. $f$ and the learnable codes are passed to the separate background head and object heads to predict $(c^i_{object}, \sigma^i_{object})$ for each instance $i \in \{1, \dots, K+1\}$ [S1].

3.  **3D One-Hot Density Regularization:**
    To prevent background leakage into object regions, a binary occupancy mask $M_c$ identifies points where foreground objects dominate:
    $$ M_c = [\arg\max([\sigma^1_{object}, \sigma^2_{object}, \dots, \sigma^{K+1}_{object}]) > 1] $$
    A regularization loss constrains the background density ($\sigma^1_{object}$) at these points to a small negative value $\sigma_0 = -0.01$:
    $$ L_{one-hot} = M_c \|\sigma^1_{object} - \sigma_0\|^2_2 $$
    Note: ReLU is applied to $\sigma$ during rendering to ensure positivity, so negative raw predictions do not contribute to rendering [S1].

4.  **2D In-Painting Pseudo Supervision:**
    To handle occluded background regions unseen in training views, a pre-trained inpainting model (e.g., LaMa) generates pseudo-color $C^{pseudo}$ for uncertain background regions [S1]. A 2D spatial weight map $M^1_b(r)$ (value 0.05 for uncertain regions) weights the color loss:
    $$ L_{color\_pseudo} = M^1_b(r) \|\hat{C}^1_{object}(r) - C^{pseudo}(r)\|^2_2 $$
    Multi-view consistency suppresses ambiguities introduced by 2D inpainting [S1].

5.  **Intermediate Local Object Rendering:**
    Render color $\hat{C}^k_{object}(r)$ and opacity $\hat{O}^k_{object}(r)$ for each object $k$ using standard volume rendering equations along ray $r$ with $N$ samples:
    $$ \hat{C}^k_{object}(r) = \sum_{i=1}^N T^k_i (1 - \exp(-\sigma^k_{object} \delta_i)) c^k_{object} $$
    $$ \hat{O}^k_{object}(r) = \sum_{i=1}^N T^k_i (1 - \exp(-\sigma^k_{object} \delta_i)) $$
    $$ T^k_i = \exp\left(-\sum_{j=1}^{i-1} \sigma^k_{object} \delta_j\right) $$
    where $\delta_i$ is the distance between adjacent points [S1].

6.  **Object Loss:**
    Enforce consistency with semantic masks $M_o(r)^k$ (for objects) and $M_b(r)^1$ (for background):
    $$ L_{object} = \sum_{k=1}^{K+1} M_o(r)^k \|\hat{C}^k_{object}(r) - C(r)\|^2_2 + \lambda_b M_b(r)^1 \|\hat{O}^1_{object}(r) - M_o(r)^1\|^2_2 $$
    where $\lambda_b$ is a balancing weight [S1].

7.  **One-Hot Composition Weights:**
    To resolve gradient issues with $\arg\max$, use Gumbel-Softmax on densities. Calculate soft probability $p_i$ for object $i$:
    $$ p_i = \frac{\exp((\sigma^i_{object} + g_i)/\tau)}{\sum_j \exp((\sigma^j_{object} + g_j)/\tau)} $$
    where $g_i \sim \text{Gumbel}(0,1)$ and temperature $\tau = 0.1$ [S1].
    Compute one-hot composition weights $M_{obj}$ using a stop-gradient ($f_{sp}$) on $P$ (the vector of $p_i$):
    $$ M_{obj} = f_h(\arg\max[g_i + \log \sigma^i_{object}]) - f_{sp}(P) + P $$
    where $f_h$ is the one-hot function [S1].

8.  **Residual Composition:**
    Composite the object fields using $M_{obj}$ to get $(c_{comp}, \sigma_{comp})$. Combine this with the coarse stage residual $(c_{guidance}, \sigma_{guidance})$ using point-wise average pooling:
    $$ c_{final} = \text{AvgPool}(c_{comp}, c_{guidance}) $$
    $$ \sigma_{final} = \text{AvgPool}(\sigma_{comp}, \sigma_{guidance}) $$
    Apply color loss $L_{comp}$ on the final rendered pixels [S1].

9.  **Overall Optimization:**
    Minimize the total loss:
    $$ L_{overall} = L_{comp} + \lambda_1 L_{object} + \lambda_2 L_{color\_pseudo} + \lambda_3 L_{one-hot} $$
    where $\lambda_1, \lambda_2, \lambda_3$ are balancing weights [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class UDCNeRF:
    def __init__(self, num_objects, hidden_dim=256, num_coarse_samples=64, num_fine_samples=48):
        self.num_objects = num_objects + 1  # K objects + 1 background
        self.num_coarse_samples = num_coarse_samples
        self.num_fine_samples = num_fine_samples
        self.temperature = 0.1
        self.sigma_0 = -0.01
        self.lambda_b = 1.0  # Example weight for opacity loss
        
        # Shared Backbone: 4-layer MLP
        self.backbone = nn.Sequential(
            nn.Linear(6, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim)
        )
        
        # Heads: 3-layer MLPs
        self.guidance_head = nn.Sequential(
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, 4) # 3 for color, 1 for density
        )
        
        self.background_head = nn.Sequential(
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, 4)
        )
        
        self.object_heads = nn.ModuleList([
            nn.Sequential(
                nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
                nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
                nn.Linear(hidden_dim, 4)
            ) for _ in range(num_objects)
        ])
        
        # Learnable object codes
        self.object_codes = nn.Parameter(torch.randn(self.num_objects, hidden_dim))

    def forward(self, rays_o, rays_d, semantic_masks, pseudo_colors=None):
        """
        rays_o: (B, 3) ray origins
        rays_d: (B, 3) ray directions
        semantic_masks: (B, H, W, K+1) binary masks for objects/background
        pseudo_colors: (B, H, W, 3) optional pseudo-color for background inpainting
        """
        # 1. Coarse Stage
        # Sample points along rays (simplified: uniform sampling for demonstration)
        t_coarse = torch.linspace(0, 1, self.num_coarse_samples).to(rays_o.device)
        points_coarse = rays_o.unsqueeze(1) + rays_d.unsqueeze(1) * t_coarse.unsqueeze(0).unsqueeze(-1)
        points_coarse = points_coarse.view(-1, 3)
        dirs_coarse = rays_d.unsqueeze(1).expand(-1, self.num_coarse_samples, -1).view(-1, 3)
        
        # Encode
        # Note: In practice, positional encoding is applied to points and dirs
        feats_coarse = self.backbone(torch.cat([points_coarse, dirs_coarse], dim=-1))
        guidance_out = self.guidance_head(feats_coarse)
        c_guidance = guidance_out[..., :3]
        sigma_guidance = F.relu(guidance_out[..., 3:])
        
        # Use coarse distribution for fine sampling (importance sampling logic omitted for brevity, assume fine_points are sampled)
        # For implementation completeness, we assume fine_points and fine_dirs are provided or sampled based on coarse sigma
        # Here we assume a helper function `sample_fine_points` exists or use uniform for simplicity in this block
        # [G] In a real implementation, fine points are sampled based on the PDF of sigma_guidance
        
        # 2. Fine Stage Decomposition
        # Assume fine_points and fine_dirs are available from sampling step
        # If not provided in this simplified block, we fall back to coarse for structure, 
        # but the method requires distinct fine sampling. 
        # Let's assume we have fine_points and fine_dirs from the sampling routine.
        
        feats_fine = self.backbone(torch.cat([fine_points, fine_dirs], dim=-1))
        
        # Predict for background (index 0 in object_codes list if we map 1->0, but let's stick to 1-based index 1 for background as per paper)
        # Paper: background is index 1. Let's adjust object_codes indexing: 
        # object_codes[0] -> Object 1, ..., object_codes[K-1] -> Object K, object_codes[K] -> Background
        # Or simpler: separate background code. Paper says "set of learnable object codes... background as additional instance".
        # Let's assume object_codes has size K+1. Index 0 is Object 1, Index K is Background.
        
        # Background Head
        bg_code = self.object_codes[-1] # Last code is background
        bg_input = feats_fine + bg_code
        bg_out = self.background_head(bg_input)
        c_bg = bg_out[..., :3]
        sigma_bg = bg_out[..., 3:]
        
        # Object Heads
        c_objs = []
        sigma_objs = []
        for i in range(self.num_objects - 1):
            obj_code = self.object_codes[i]
            obj_input = feats_fine + obj_code
            obj_out = self.object_heads[i](obj_input)
            c_objs.append(obj_out[..., :3])
            sigma_objs.append(obj_out[..., 3:])
            
        c_objs = torch.stack(c_objs, dim=1) # (B*N, K, 3)
        sigma_objs = torch.stack(sigma_objs, dim=1) # (B*N, K, 1)
        
        # Stack all: K objects + 1 background
        # Order: [Obj1, Obj2, ..., ObjK, Background]
        c_all = torch.cat([c_objs, c_bg.unsqueeze(1)], dim=1)
        sigma_all = torch.cat([sigma_objs, sigma_bg.unsqueeze(1)], dim=1)
        
        # 3. One-Hot Composition Weights
        # Gumbel-Softmax
        g = torch.distributions.gumbel.Gumbel(0, 1).sample(sigma_all.shape).to(sigma_all.device)
        # Add log sigma to gumbel noise for argmax approximation
        # sigma_all is (B*N, K+1, 1). We need to apply softmax over the object dimension (dim=1)
        # Paper: p_i = exp((sigma + g)/tau) / sum(exp((sigma + g)/tau))
        
        # Reshape for softmax
        sigma_flat = sigma_all.view(-1, self.num_objects) # (B*N, K+1)
        g_flat = g.view(-1, self.num_objects)
        
        logits = (sigma_flat + g_flat) / self.temperature
        P = F.softmax(logits, dim=1) # Soft probabilities
        
        # One-hot weights M_obj
        # M_obj = fh(argmax[g + log sigma]) - fsp(P) + P
        # argmax is over the object dimension
        argmax_indices = torch.argmax(g_flat + torch.log(sigma_flat + 1e-8), dim=1)
        M_onehot = F.one_hot(argmax_indices, num_classes=self.num_objects).float()
        
        # Stop gradient on P
        M_obj = M_onehot - P.detach() + P
        
        # 4. Compose Radiance Fields
        # c_comp = sum(M_obj_k * c_k)
        c_comp = torch.sum(M_obj.unsqueeze(-1) * c_all, dim=1)
        sigma_comp = torch.sum(M_obj.unsqueeze(-1) * sigma_all, dim=1)
        
        # 5. Residual Composition
        # Average pooling on color and density
        # c_guidance and sigma_guidance are from coarse stage, need to be aligned with fine points or averaged
        # Paper: "point-wise average pooling operation applied on the color and density"
        # This implies combining the fine composited result with the coarse residual at the same spatial location
        # Assuming coarse predictions are interpolated or sampled at fine points
        c_final = (c_comp + c_guidance) / 2.0
        sigma_final = (sigma_comp + sigma_guidance) / 2.0
        
        return c_final, sigma_final, M_obj, P, sigma_all

    def compute_loss(self, c_final, sigma_final, M_obj, P, sigma_all, rays_o, rays_d, semantic_masks, pseudo_colors, gt_colors):
        # Render final image
        # [G] Standard volume rendering integration is required here to get rendered color C_final from c_final, sigma_final
        # C_final = integrate(c_final, sigma_final)
        # For the sake of the article, we assume C_final is the rendered color
        
        # L_comp
        L_comp = F.mse_loss(C_final, gt_colors)
        
        # L_object
        # Render intermediate object colors and opacities
        # [G] Helper function render_object(c_all, sigma_all, M_obj) -> C_obj, O_obj
        C_obj, O_obj = render_object(c_all, sigma_all, M_obj)
        
        # Semantic masks: M_o(r)^k for objects, M_b(r)^1 for background
        # Assuming semantic_masks are provided as (B, H, W, K+1)
        M_o = semantic_masks[:, :, :, :-1] # Objects
        M_b = semantic_masks[:, :, :, -1:] # Background
        
        # Color consistency for objects
        L_obj_color = 0
        for k in range(self.num_objects - 1):
            mask = M_o[:, :, :, k]
            if mask.sum() > 0:
                L_obj_color += F.mse_loss(C_obj[:, :, :, k][mask], gt_colors[mask])
                
        # Opacity consistency for background
        # L_obj_opacity = lambda_b * M_b * ||O_obj_bg - M_o_bg||^2
        # Note: Paper uses M_o(r)^1 for background opacity loss target? 
        # Paper: "||\hat{O}^1_{object}(r) - M_o(r)^1||^2_2" where M_o(r)^1 is likely the semantic mask for background
        # Let's assume M_b is the semantic mask for background
        L_obj_opacity = self.lambda_b * F.mse_loss(O_obj[:, :, :, -1] * M_b, M_b)
        
        L_object = L_obj_color + L_obj_opacity
        
        # L_color_pseudo
        L_color_pseudo = 0
        if pseudo_colors is not None:
            # M^1_b(r) is 0.05 for uncertain regions, 0 otherwise
            # Assuming pseudo_colors are only valid where M_b is active or uncertain
            # Paper: M^1_b(r) weight map
            weight_map = torch.where(M_b > 0, 0.05, 0.0).to(pseudo_colors.device)
            L_color_pseudo = F.mse_loss(C_obj[:, :, :, -1] * weight_map, pseudo_colors * weight_map)
            
        # L_one_hot
        # M_c = [argmax(sigma_all) > 1]
        # argmax over object dimension
        argmax_sigma = torch.argmax(sigma_all, dim=1) # (B*N, K+1) -> indices
        # Mask where foreground objects (indices 0 to K-1) have max density
        # Background is index K. If argmax != K, then foreground is dominant
        is_foreground = (argmax_sigma != (self.num_objects - 1)).float()
        
        # L_one_hot = M_c * ||sigma_bg - sigma_0||^2
        sigma_bg = sigma_all[:, -1, :] # (B*N, 1)
        L_one_hot = torch.mean(is_foreground.unsqueeze(-1) * (sigma_bg - self.sigma_0)**2)
        
        L_overall = L_comp + L_object + L_color_pseudo + L_one_hot
        
        return L_overall, L_comp, L_object, L_color_pseudo, L_one_hot
```

**Edge Cases and Pitfalls:**
*   **Gradient Backpropagation:** The $\arg\max$ operation is non-differentiable. The method uses Gumbel-Softmax on volume density to provide a continuous approximation [S1].
*   **Occluded Regions:** Regions occluded by objects in all training views are ambiguous. The 2D inpainting pseudo-supervision handles this, but multi-view consistency is required to suppress ambiguities introduced by 2D inpainting in regions seen in other views [S1].
*   **Background Density:** The background density is constrained to be negative in foreground-dominated regions to prevent it from contributing to the final render, as ReLU is applied during rendering [S1].

## Operating characteristics
*   **Time and Memory Complexity:** The method uses a two-stage framework. The coarse stage adds overhead for guidance sampling, while the fine stage requires maintaining $K+1$ separate radiance fields and computing Gumbel-Softmax weights for composition. Memory usage scales with $K$ due to the learnable codes and separate heads [S1].
*   **Measured Performance:** On the ToyDesk2 scene, the full model achieved an SSIM of 0.8126 and LPIPS of 0.4488, outperforming ablations such as removing 3D one-hot regularization (SSIM 0.8116, LPIPS 0.5452) or residual composition (SSIM 0.8003, LPIPS 0.4589) [S1].
*   **Strengths:** Unifies decomposition and composition, allowing both novel view synthesis and object editing. The one-hot regularization and inpainting pseudo-supervision improve rendering quality in occluded regions [S1].
*   **Limitations:** Requires semantic masks for training. The method relies on a pre-trained inpainting model (LaMa) for pseudo-supervision [S1].
*   **When to use:** When both high-quality novel view synthesis and object-level editing are required, and semantic masks are available.
*   **When not to use:** When semantic masks are not available or when object editing is not needed, simpler NeRF variants may be more efficient.

## Background
The method was proposed in "Learning Unified Decompositional and Compositional NeRF for Editable Novel View Synthesis" [S1]. It addresses the limitations of prior works like ObjectNeRF, which learn separate networks for global and local representations without modeling their interactions [S1]. The framework builds on the standard NeRF volume rendering pipeline and incorporates techniques from ObjectNeRF for object-specific representations [S1].

## Sources
[S1] Learning Unified Decompositional and Compositional NeRF for Editable Novel View Synthesis, https://arxiv.org/abs/2308.02840v1, fetched 2026-10-26
