# Unified Decompositional and Compositional NeRF

Also called: Joint decomposition and composition framework, Two-stage NeRF for joint decomposition and composition, UDC-NeRF, Unifed decompositional and compositional NeRF

## Overview
Unified Decompositional and Compositional NeRF (UDC-NeRF) is a two-stage framework that jointly learns object-level and scene-level implicit representations to enable both novel view synthesis and instance-level scene editing [S1]. Unlike sibling methods such as [DM-NeRF](13525-dm-nerf.md) or [Detail-based Segmentation](15935-detail-based-segmentation.md) which may treat decomposition and composition as separate or loosely coupled tasks, UDC-NeRF unites them in a single pipeline where a coarse global field guides the sampling for a fine-grained decomposed field, which is then re-composed using differentiable one-hot activation weights [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ multi-view posed images $\{x_i\}_{i=1}^N$ and their corresponding instance semantic masks $\{s_{ij}\}_{i=1, j=1}^{N, K+1}$ for a static scene, where $K$ is the number of object instances and the background is treated as instance index 1 [S1].
*   **Outputs:** A composited radiance field $(c_{comp}, \sigma_{comp})$ for novel view synthesis, and individual object/background radiance fields $(c^k_{object}, \sigma^k_{object})$ for editing [S1].

**Data Structures:**
1.  **Shared Backbone:** A 4-layer Multi-Layer Perceptron (MLP) [S1].
    *   Input: 3D point coordinate $p=(x,y,z)$ and view direction $d$.
    *   Output: Generic implicit scene feature vector $f$ with dimension 256 [S1].
    *   Positional Encoding: 10 levels for coordinates, 4 levels for ray directions [S1].
2.  **Guidance Head:** Several MLPs following the backbone [S1].
    *   Output: Global RGB color $c_{guidance}$ and volume density $\sigma_{guidance}$ [S1].
3.  **Object/Background Heads:** Separate 3-layer MLPs [S1].
    *   Input: Feature vector $f$ concatenated with learnable object codes (one per instance $k \in [1, K+1]$) [S1].
    *   Output: RGB color $c^k_{object}$ and volume density $\sigma^k_{object}$ for instance $k$ [S1].
4.  **Learnable Object Codes:** A set of vectors, one for each of the $K$ objects and the background [S1].

**Numbered Steps:**

1.  **Coarse Stage (Global Guidance):**
    *   Sample points along camera rays.
    *   Pass points and directions through the Shared Backbone and Guidance Head to predict $c_{guidance}$ and $\sigma_{guidance}$ [S1].
    *   Use the distribution of $\sigma_{guidance}$ to guide point sampling for the fine stage (coarse-to-fine sampling) [S1].

2.  **Fine Stage (Decomposition):**
    *   Sample points along rays using the guidance from Step 1 [S1].
    *   Pass points and directions through the Shared Backbone to get feature $f$ [S1].
    *   For each instance $k \in [1, K+1]$:
        *   Concatenate $f$ with the learnable code for instance $k$.
        *   Pass through the specific Object/Background Head (separate heads for background and objects) to predict $c^k_{object}$ and $\sigma^k_{object}$ [S1].

3.  **One-Hot Activation (Composition Weights):**
    *   To determine which instance dominates at each point, compute soft probabilities $p_i$ using Gumbel-Softmax on the predicted densities [S1]:
        $$ p_i = \frac{\exp((\sigma^i_{object} + g_i)/\tau)}{\sum_j \exp((\sigma^j_{object} + g_j)/\tau)} $$
        where $g_i$ are i.i.d. samples from Gumbel(0,1) and temperature $\tau = 0.1$ [S1].
    *   Compute one-hot composition weights $M_{obj}$ using a stop-gradient operation to allow backpropagation through the soft probabilities $P$ (where $P$ is the vector of $p_i$) [S1]:
        $$ M_{obj} = f_h(\arg\max[g_i + \log \sigma^i_{object}]) - f_{sp}(P) + P $$
        where $f_h$ is the one-hot function and $f_{sp}$ is the stop-gradient operator [S1].

4.  **Residual Composition:**
    *   Compute the composited radiance field $(c_{comp}, \sigma_{comp})$ by weighting the individual instance fields with $M_{obj}$ [S1].
    *   Combine $(c_{comp}, \sigma_{comp})$ with the residual radiance field $(c_{guidance}, \sigma_{guidance})$ from the coarse stage using point-wise average pooling on color and density [S1].
    *   Render the final image from this combined field [S1].

**Formulas and Losses:**

*   **Binary Occupancy Mask:** Determines if foreground objects occupy a point [S1]:
    $$ M_c = [\arg\max([\sigma^1_{object}, \sigma^2_{object}, ..., \sigma^{K+1}_{object}]) > 1] $$
    Note: The source uses index 1 for background in the regularization context, but the argmax compares all $K+1$ instances [S1].

*   **3D One-Hot Density Regularization:** Constrains the background field (index 1) in regions occupied by foreground objects [S1]:
    $$ L_{one-hot} = M_c \|\sigma^1_{object} - \sigma_0\|^2_2 $$
    where $\sigma_0 = -0.01$ [S1]. ReLU is applied to $\sigma$ during rendering to ensure positivity [S1].

*   **2D In-painting Pseudo Supervision:** Handles occluded background regions using a pre-trained Lama model [S1]:
    $$ L_{color pseudo} = M^1_b(r) \|\hat{C}^1_{object}(r) - C^1_{pseudo}(r)\|^2_2 $$
    where $M^1_b(r)$ is a 2D spatial weight map (value 0.05 for uncertain background regions) and $C^1_{pseudo}$ is the inpainted color [S1].

*   **Intermediate Local Object Rendering:** Renders color and opacity for each instance $k$ [S1]:
    $$ \hat{C}^k_{object}(r) = \sum_{i=1}^N T^k_i (1 - \exp(-\sigma^k_{object} \delta_i)) c^k_{object} $$
    $$ \hat{O}^k_{object}(r) = \sum_{i=1}^N T^k_i (1 - \exp(-\sigma^k_{object} \delta_i)) $$
    where $T^k_i = \exp(-\sum_{j=1}^{i-1} \sigma^k_{object} \delta_j)$ is the transmittance, $N$ is the number of sampled points, and $\delta_i$ is the distance between adjacent points [S1].

*   **Object Loss:** Enforces color and opacity consistency using semantic masks [S1]:
    $$ L_{object} = \sum_{k=1}^{K+1} M_o(r)^k \|\hat{C}^k_{object}(r) - C(r)\|^2_2 + \lambda_b M_b(r)^1 \|\hat{O}^1_{object}(r) - M_o(r)^1\|^2_2 $$
    where $M_o(r)^k$ and $M_b(r)^1$ are 2D spatial weight maps derived from semantic masks [S1].

*   **Overall Optimization Loss:**
    $$ L_{overall} = L_{comp} + \lambda_1 L_{object} + \lambda_2 L_{color pseudo} + \lambda_3 L_{one-hot} $$
    where $\lambda_1, \lambda_2, \lambda_3$ are balancing weights [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class UDCNeRF:
    def __init__(self, num_objects, hidden_dim=256, num_coarse_samples=64, num_fine_samples=48):
        """
        Initialize the UDC-NeRF model.
        
        Args:
            num_objects (int): K, number of object instances. Background is index 1, objects 2..K+1.
            hidden_dim (int): Dimension of the shared backbone output.
            num_coarse_samples (int): Number of points sampled in the coarse stage.
            num_fine_samples (int): Number of points sampled in the fine stage.
        """
        self.K = num_objects
        self.num_instances = num_objects + 1  # K objects + 1 background
        
        # Positional Encoding parameters
        self.freq_coords = 10
        self.freq_dirs = 4
        
        # Shared Backbone: 4-layer MLP
        self.backbone = nn.Sequential(
            nn.Linear(6 + 6 * self.freq_coords + 6 * self.freq_dirs, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim)
        )
        
        # Guidance Head: Predicts global scene properties for sampling
        self.guidance_head = nn.Sequential(
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, 4) # 3 for color, 1 for density
        )
        
        # Learnable Object Codes
        self.object_codes = nn.Parameter(torch.randn(self.num_instances, hidden_dim))
        
        # Separate Heads for Background (index 0 in code, but conceptually 1) and Objects
        # Note: Source says separate background head and object head. 
        # We implement a generic head that takes the code as input.
        self.heads = nn.ModuleList([
            nn.Sequential(
                nn.Linear(hidden_dim * 2, hidden_dim), # Concat feature + code
                nn.ReLU(),
                nn.Linear(hidden_dim, hidden_dim),
                nn.ReLU(),
                nn.Linear(hidden_dim, 4) # 3 for color, 1 for density
            ) for _ in range(self.num_instances)
        ])
        
        # Hyperparameters
        self.tau = 0.1
        self.sigma_0 = -0.01
        self.lambda_1 = 1.0 # Placeholder for L_object weight
        self.lambda_2 = 1.0 # Placeholder for L_color_pseudo weight
        self.lambda_3 = 1.0 # Placeholder for L_one_hot weight
        
    def positional_encoding(self, x, num_freq_bands, use_identity=True):
        """Standard positional encoding."""
        f = [1.0]
        for i in range(1, num_freq_bands):
            f.append(2**i * x)
        return torch.cat(f, dim=-1)

    def forward(self, rays_o, rays_d, semantic_masks=None, pseudo_colors=None):
        """
        Forward pass for training.
        
        Args:
            rays_o: Origin of rays [N_rays, 3]
            rays_d: Direction of rays [N_rays, 3]
            semantic_masks: Semantic masks for supervision [N_rays, N_instances]
            pseudo_colors: Pseudo colors from inpainting for background [N_rays, 3]
            
        Returns:
            rendered_color: Final rendered color [N_rays, 3]
            losses: Dictionary of loss components
        """
        N_rays = rays_o.shape[0]
        
        # 1. Coarse Stage
        # Sample points along rays (simplified uniform sampling for demonstration)
        # In practice, this uses the guidance field distribution
        t_coarse = torch.linspace(0, 1, self.num_coarse_samples).to(rays_o.device)
        # Expand for rays
        t_coarse = t_coarse.unsqueeze(0).expand(N_rays, -1)
        points_coarse = rays_o.unsqueeze(1) + rays_d.unsqueeze(1) * t_coarse.unsqueeze(2)
        
        # Encode inputs
        # Assuming standard NeRF PE for coords and dirs
        # For brevity, assuming helper functions exist or inputs are already encoded
        # In a real implementation, encode points_coarse and rays_d
        
        # Predict guidance color and density
        # Guidance Head takes backbone output
        # Backbone input: encoded points + encoded dirs
        # Let's assume we have encoded features for simplicity in this pseudocode structure
        # Real implementation requires full PE pipeline
        
        # Placeholder for guidance prediction logic
        # sigma_guidance, c_guidance = self.guidance_head(backbone_output)
        
        # 2. Fine Stage Sampling
        # Use sigma_guidance to sample points (importance sampling)
        # For this pseudocode, we assume we have sampled points points_fine
        # and their corresponding t values t_fine
        
        # Encode fine points
        # points_fine, dirs_encoded = encode(points_fine, rays_d)
        
        # Get generic feature from backbone
        # f = self.backbone(points_fine) # Shape: [N_rays, N_samples, hidden_dim]
        
        # 3. Instance-specific Prediction
        # Expand object codes to match batch and samples
        # codes = self.object_codes # [N_instances, hidden_dim]
        
        # Predict color and density for each instance
        # c_objects, sigma_objects = [], []
        # for i in range(self.num_instances):
        #     input_to_head = torch.cat([f, codes[i].unsqueeze(0).unsqueeze(0).expand_as(f)], dim=-1)
        #     out = self.heads[i](input_to_head)
        #     c_objects.append(out[..., :3])
        #     sigma_objects.append(out[..., 3:])
        
        # 4. One-Hot Activation
        # Compute soft probabilities using Gumbel-Softmax
        # g = torch.distributions.gumbel.Gumbel(0, 1).sample(sigma_objects[0].shape).to(rays_o.device)
        # logits = torch.stack(sigma_objects, dim=-1) # [N_rays, N_samples, N_instances]
        # g = torch.stack([g for _ in range(self.num_instances)], dim=-1)
        # probs = torch.softmax((logits + g) / self.tau, dim=-1)
        
        # Compute one-hot weights M_obj
        # argmax_indices = torch.argmax(g + torch.log(torch.clamp(logits, min=1e-6)), dim=-1)
        # M_obj = F.one_hot(argmax_indices, num_classes=self.num_instances).float()
        # Stop gradient on probs for the residual connection
        # M_obj = M_obj - probs.detach() + probs
        
        # 5. Composition
        # Weighted sum of instance fields
        # c_objects_stacked = torch.stack(c_objects, dim=-1)
        # sigma_objects_stacked = torch.stack(sigma_objects, dim=-1)
        
        # c_comp = torch.sum(c_objects_stacked * M_obj.unsqueeze(-1), dim=-1)
        # sigma_comp = torch.sum(sigma_objects_stacked * M_obj, dim=-1)
        
        # 6. Residual Composition
        # Combine with guidance field
        # c_final = (c_comp + c_guidance) / 2
        # sigma_final = (sigma_comp + sigma_guidance) / 2
        
        # 7. Volume Rendering
        # Render final image from c_final, sigma_final
        # rendered_color = volume_render(c_final, sigma_final, t_fine)
        
        # 8. Loss Calculation
        # losses = self.compute_losses(rendered_color, c_final, sigma_final, semantic_masks, pseudo_colors)
        
        # Return rendered_color and losses
        pass

    def compute_losses(self, rendered_color, c_final, sigma_final, semantic_masks, pseudo_colors):
        """
        Compute the total loss.
        """
        # L_comp: Color loss between rendered and ground truth
        # L_comp = F.mse_loss(rendered_color, gt_color)
        
        # L_object: Color and opacity consistency
        # Requires rendering intermediate object fields
        # L_object = ...
        
        # L_color_pseudo: In-painting supervision for background
        # Requires identifying background pixels and comparing with pseudo colors
        # L_color_pseudo = ...
        
        # L_one_hot: Density regularization
        # Requires computing occupancy mask M_c
        # L_one_hot = ...
        
        # L_overall = L_comp + lambda1 * L_object + lambda2 * L_color_pseudo + lambda3 * L_one_hot
        
        return {}
```

**Edge Cases and Pitfalls:**
*   **Gradient Backpropagation with Argmax:** The $\arg\max$ operation is non-differentiable. This is addressed by using the Gumbel-Softmax function on volume density as a continuous approximation, combined with a stop-gradient operation on the soft probabilities to allow gradients to flow through the one-hot mask [S1].
*   **Occluded Background Regions:** Regions occluded by objects in all training views are ambiguous. This is handled by using a pre-trained Lama in-painting model to provide pseudo-color supervision for these uncertain 2D background regions [S1].
*   **In-painting Ambiguity:** 2D in-painting may introduce ambiguity in regions seen in other views, but this is suppressed by strong multi-view consistency supervision [S1].
*   **Density Positivity:** Volume density $\sigma$ must be non-negative for rendering. ReLU is applied to $\sigma$ during the rendering process [S1].

## Operating characteristics
*   **Time Complexity:** The method involves two stages (coarse and fine) and multiple MLP passes per instance ($K+1$). The fine stage requires computing Gumbel-Softmax and one-hot weights for all instances, adding overhead compared to single-field NeRF.
*   **Memory Complexity:** Requires storing learnable codes for $K+1$ instances and separate MLP heads for each instance (or a shared head with code conditioning). The source does not specify exact memory usage in MB/GB.
*   **Measured Performance:** On the ToyDesk2 scene, the full model achieved an SSIM of 0.8126 and LPIPS of 0.4488. Ablations show that removing 3D One-hot Density Regularization drops SSIM to 0.8116, removing Gumbel-Softmax Activation drops it to 0.7217, and removing 2D In-painting pseudo Supervision drops it to 0.8127 (SSIM) and increases LPIPS to 0.4516. Removing Residual Radiance Composition drops SSIM to 0.8003 [S1].
*   **Strengths:** Enables joint novel view synthesis and object editing in a unified pipeline. The one-hot regularization and in-painting supervision improve rendering and editing quality, especially for occluded regions [S1].
*   **Limitations:** Relies on semantic masks for supervision. The method assumes a static scene. The two-stage process adds complexity compared to single-stage NeRF [S1].
*   **When to use:** When both high-quality novel view synthesis and instance-level editing (removal, addition, transformation) are required for a static scene with known object instances [S1].
*   **When not to use:** For dynamic scenes, or when semantic masks are not available [S1].

## Background
*   **Origin:** Proposed in "Learning Unified Decompositional and Compositional NeRF for Editable Novel View Synthesis" (2023) [S1].
*   **History:** Builds upon ObjectNeRF [S1], which learns separate networks for global scene and local object branches without modeling interactions. UDC-NeRF unites decomposition and composition to model interactions and correlations between these tasks [S1].
*   **Usage:** Used for joint scene decomposition and composition to effectively learn object-level and scene-level implicit representations for effective scene modeling, allowing object editing and novel view synthesis in a unified pipeline [S1].

## Sources
[S1] Learning Unified Decompositional and Compositional NeRF for Editable Novel View Synthesis, https://arxiv.org/abs/2308.02840v1, fetched 2026-10-26
