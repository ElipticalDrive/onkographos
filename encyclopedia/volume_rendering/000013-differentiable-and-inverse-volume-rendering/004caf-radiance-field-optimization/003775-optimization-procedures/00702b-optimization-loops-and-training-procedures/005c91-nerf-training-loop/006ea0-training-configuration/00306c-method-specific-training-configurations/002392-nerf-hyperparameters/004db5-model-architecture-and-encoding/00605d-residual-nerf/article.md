# Residual-NeRF

Also called: Residual-NeRF-S (ablation variant)

## Overview
Residual-NeRF is a method for improving depth perception and training speed for transparent objects in mostly static scenes by leveraging a static scene prior [S1]. It solves the ambiguity of depth estimation for transparent objects, where standard NeRFs struggle due to multiple surfaces radiating light along a ray, leading to noisy depth maps and perceived holes that hinder robotic grasp planning [S1]. Unlike methods that require pre-defined spatial areas for fusion (e.g., Block-NeRF, NeRFusion), Residual-NeRF learns to mix a pre-trained background NeRF and a residual NeRF at every point in space using a learned Mixnet, resulting in fewer holes and less noise in depth maps [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   $I_{bg}$: Set of images of the scene without transparent objects (used to train the background NeRF).
    *   $I_{res}$: Set of images of the scene with transparent objects present (used to train the residual NeRF and Mixnet).
    *   Camera poses $P$ for $I_{bg}$ and $I_{res}$.
    *   Ray parameters $r(\lambda) = o + \lambda d$ for rendering.
*   **Outputs:**
    *   Rendered color $\hat{C}(r)$ for a given ray.
    *   Depth map $\hat{D}(r)$ for the scene.

**Data Structures**
The method utilizes three Multi-Layer Perceptrons (MLPs):
1.  **Background NeRF ($\Phi_{bg}$):**
    *   **Input:** Encoded position and view direction $h(x, y, z, \theta, \phi)$.
    *   **Output:** Pre-activation color $\mathbf{c}'_{bg}$ and volume density $\sigma_{bg}$.
2.  **Residual NeRF ($\Phi_{res}$):**
    *   **Input:** Encoded position and view direction $h(x, y, z, \theta, \phi)$.
    *   **Output:** Pre-activation color $\mathbf{c}'_{res}$ and volume density $\sigma_{res}$.
3.  **Mixnet ($\Phi_{mix}$):**
    *   **Input:** Encoded position $h(x, y, z)$. The sources specify a "location-dependent multi-resolution hash encoding" is used for the Mixnet input [S1].
    *   **Output:** Mixing weight $\beta \in [0, 1]$.

**Numbered Steps**
1.  **Train Background NeRF:** Train $\Phi_{bg}$ on images $I_{bg}$ (scene without transparent objects) to infer $\sigma_{bg}$ and $\mathbf{c}'_{bg}$ [S1].
2.  **Train Residual NeRF and Mixnet:** Train $\Phi_{res}$ and $\Phi_{mix}$ on images $I_{res}$ (scene with transparent objects). The background NeRF $\Phi_{bg}$ is frozen and not updated during this phase [S1].
3.  **Compute Mixing Weight:** For each point along a ray, the Mixnet infers $\beta$ using the encoded position [S1].
4.  **Render Color:** Combine the outputs of $\Phi_{bg}$ and $\Phi_{res}$ using $\beta$ in the volumetric rendering equation to render the final color [S1].
5.  **Compute Depth:** Estimate depth using a density threshold $m$ [S1].

**Formulas**
The rendering process combines the background and residual fields. The final color $\hat{C}(r)$ is computed by integrating along the ray [S1]:

$$ \hat{C}(r) = \sum_{i=1}^{N} T_i (1 - \exp(-((1 - \beta)\sigma_{bg} + \beta\sigma_{res})_i \delta_i)) (\mathbf{c}_{bg} + \mathbf{c}_{res})_i $$

Where:
*   $T_i = \exp \left( -\sum_{j=1}^{i-1} \sigma_j \delta_j \right)$ is the accumulated transmittance up to sample $i$.
*   $\delta_i = t_{i+1} - t_i$ is the segment length between samples.
*   $\sigma_{bg}$ and $\sigma_{res}$ are the densities from the background and residual NeRFs, respectively.
*   $\beta$ is the mixing weight from the Mixnet.
*   $(\mathbf{c}_{bg} + \mathbf{c}_{res})_i$ is the combined color at sample $i$.

The combined color is a sigmoid-weighted sum of the pre-activation colors [S1]:

$$ \mathbf{c}_{bg+res} = S((1.0 - \beta)\mathbf{c}'_{bg} + \beta\mathbf{c}'_{res}) $$

The mixing weight $\beta$ is the sigmoid output of the Mixnet MLP applied to the encoded position [S1]:

$$ \beta = S(\Phi_{mix}(h(x, y, z))) $$

The sigmoid function $S(x)$ is defined as [S1]:

$$ S(x) = \frac{1}{1 + e^{-x}} $$

The individual network outputs are defined as [S1]:

$$ (\mathbf{c}'_{bg}, \sigma_{bg}) = \Phi_{bg}(h(x, y, z, \theta, \phi)) $$
$$ (\mathbf{c}'_{res}, \sigma_{res}) = \Phi_{res}(h(x, y, z, \theta, \phi)) $$

**Depth Estimation**
Standard depth estimation using the expected distance is inaccurate for transparent objects due to multiple surfaces radiating light [S1]. Instead, the method finds the point closest to the camera where the density exceeds a threshold $m$ [S1]:

$$ \hat{D}(r) = t_n \quad \text{where} \quad \sigma_n \ge m \quad \text{and} \quad \forall j < n, \sigma_j < m $$

**Parameters**
*   $m$: A tunable parameter for depth estimation threshold. It is used to find the closest point to the camera where density $\sigma_i \ge m$. A single tuned value is reused across scenes. Typical value: 3.0 for Blender scenes; tuned per real environment [S1].
*   $\beta$: The mixing weight inferred by the Mixnet, constrained to $[0, 1]$ via sigmoid activation [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class ResidualNeRF:
    def __init__(self, bg_mlp, res_mlp, mix_mlp, m_threshold=3.0):
        """
        Initialize Residual-NeRF.
        
        Args:
            bg_mlp: Background NeRF MLP (Phi_bg).
            res_mlp: Residual NeRF MLP (Phi_res).
            mix_mlp: Mixnet MLP (Phi_mix).
            m_threshold: Density threshold for depth estimation (m).
        """
        self.bg_mlp = bg_mlp
        self.res_mlp = res_mlp
        self.mix_mlp = mix_mlp
        self.m_threshold = m_threshold
        # Note: In training, bg_mlp is frozen after Step 1.

    def encode_position(self, x, y, z):
        """
        Encode position coordinates.
        Note: The sources specify a 'location-dependent multi-resolution hash encoding' 
        for the Mixnet, and standard positional encoding for the NeRFs.
        This function is a placeholder for the specific encoding implementation 
        (e.g., NeRF positional encoding or Hash grid lookup).
        """
        # [G] Standard NeRF positional encoding is assumed for bg/res inputs.
        # Hash encoding is assumed for mix input.
        pass

    def forward(self, ray_origins, ray_directions, num_samples, near, far):
        """
        Render a batch of rays.
        
        Args:
            ray_origins: Tensor of shape (..., 3).
            ray_directions: Tensor of shape (..., 3).
            num_samples: Number of samples per ray.
            near: Near plane distance.
            far: Far plane distance.
            
        Returns:
            color: Rendered color tensor.
            depth: Estimated depth tensor.
        """
        # 1. Sample points along rays
        t_vals = torch.linspace(0., 1., num_samples)
        z_vals = near * (1. - t_vals) + far * t_vals
        points = ray_origins.unsqueeze(-2) + ray_directions.unsqueeze(-2) * z_vals.unsqueeze(-1)
        
        # 2. Encode inputs
        # Position encoding for bg/res
        pos_enc = self.encode_position(points[..., 0], points[..., 1], points[..., 2])
        # Direction encoding (standard NeRF)
        dir_enc = self.encode_direction(ray_directions) # [G] Standard NeRF dir encoding
        
        # Mixnet uses only position encoding (hash grid)
        mix_pos_enc = self.encode_position_hash(points[..., 0], points[..., 1], points[..., 2]) # [G] Hash encoding
        
        # 3. Query Background and Residual NeRFs
        # bg_mlp and res_mlp take (pos_enc, dir_enc)
        bg_output = self.bg_mlp(torch.cat([pos_enc, dir_enc], dim=-1))
        res_output = self.res_mlp(torch.cat([pos_enc, dir_enc], dim=-1))
        
        c_prime_bg, sigma_bg = bg_output[..., :3], bg_output[..., 3]
        c_prime_res, sigma_res = res_output[..., :3], res_output[..., 3]
        
        # 4. Query Mixnet
        # mix_mlp takes mix_pos_enc
        beta = torch.sigmoid(self.mix_mlp(mix_pos_enc))
        
        # 5. Compute combined density and color
        # Density: weighted sum
        sigma_combined = (1.0 - beta) * sigma_bg + beta * sigma_res
        
        # Color: sigmoid-weighted sum of pre-activation colors
        c_combined = torch.sigmoid((1.0 - beta) * c_prime_bg + beta * c_prime_res)
        
        # 6. Volumetric Rendering
        # Compute transmittance
        deltas = z_vals[..., 1:] - z_vals[..., :-1]
        delta_inf = 1e10 * torch.ones_like(deltas[..., :1])
        deltas = torch.cat([deltas, delta_inf], -1)
        
        alpha = 1.0 - torch.exp(-sigma_combined * deltas)
        T = torch.cumprod(torch.exp(-alpha), dim=-2)
        T = torch.cat([torch.ones_like(T[..., :1]), T], -1)
        
        weights = alpha * T[..., :-1]
        color = torch.sum(weights.unsqueeze(-1) * c_combined, dim=-2)
        
        # 7. Depth Estimation
        # Find closest point where sigma >= m
        # [G] Implementation detail: This requires finding the first index i where sigma_combined[i] >= m_threshold
        # along each ray.
        depth = self.compute_depth(z_vals, sigma_combined)
        
        return color, depth

    def compute_depth(self, z_vals, sigma):
        """
        Compute depth as the first point where density >= m_threshold.
        """
        # [G] Logic to find the first index i where sigma[i] >= m_threshold
        # If no such point exists, return far or handle appropriately.
        # This is a simplified representation of the logic described in [S1].
        mask = sigma >= self.m_threshold
        # Find the first True in each row
        indices = torch.argmax(mask.float(), dim=-1)
        # Check if any mask was True
        any_mask = torch.any(mask, dim=-1)
        depths = z_vals[torch.arange(z_vals.shape[0]), indices]
        # Handle cases where no point exceeds threshold
        depths = torch.where(any_mask, depths, z_vals[..., -1])
        return depths

    def encode_direction(self, directions):
        """
        Standard NeRF positional encoding for directions.
        """
        # [G] Standard implementation of gamma(p) for directions
        pass

    def encode_position_hash(self, x, y, z):
        """
        Multi-resolution hash encoding for Mixnet.
        """
        # [G] Torch-NGP hash grid lookup implementation
        pass
```

**Edge Cases and Pitfalls**
*   **Transparent Objects:** Standard depth estimation (expected distance) is inaccurate for transparent objects because multiple surfaces radiate light along the ray [S1].
*   **Mixnet Necessity:** Without the Mixnet (using simple addition, referred to as Residual-NeRF-S), the residual NeRF must deduce the background NeRF's predictions in previously empty regions. This leads to unpredictable behavior, significant noise in view reconstruction, and holes in depth maps [S1].
*   **Depth Map Holes:** Perceived holes in depth maps can lead to robot collisions, while hallucinating non-existent surfaces may lead to no available grasp location [S1].
*   **Threshold Tuning:** The parameter $m$ must be tuned. Increasing $m$ beyond the optimal point leads to more noise without additional gains in hole filling [S1].

## Operating characteristics
*   **Time and Memory Complexity:** The method uses three MLPs ($\Phi_{bg}, \Phi_{res}, \Phi_{mix}$). The Mixnet uses a multi-resolution hash grid (via Torch-NGP), which typically offers faster training and inference compared to standard MLPs [S1].
*   **Measured Performance:** On synthetic data, Residual-NeRF achieves 46.1% lower RMSE and 29.5% lower MAE compared to baselines (NeRF, Dex-NeRF) [S1].
    *   RMSE on 'Bowl' scene: NeRF 0.1900, Dex-NeRF 0.1065, Residual-NeRF 0.0699 [S1].
    *   MAE on 'Bowl' scene: NeRF 0.1453, Dex-NeRF 0.0195, Residual-NeRF 0.0140 [S1].
*   **Scaling:** The method is designed for mostly static scenes where the background can be extensively observed (e.g., 100 images) while the foreground (transparent objects) is observed with fewer viewpoints (e.g., 50 images) [S1].
*   **Strengths:** Improves depth perception of transparent objects, reduces holes and noise in depth maps, speeds up training compared to training a single NeRF on the complex scene, and facilitates robust grasp planning for robots [S1].
*   **Limitations:** Requires a static background scene. Performance depends on the quality of the background NeRF prior. The threshold $m$ requires tuning [S1].
*   **When to use:** When depth perception of transparent objects is required in a mostly static environment, particularly for robotic manipulation tasks [S1].
*   **When not to use:** In highly dynamic scenes where the background changes significantly, or when transparent objects are not the primary focus of depth estimation [S1].

## Background
Residual-NeRF was proposed by Ichnowski et al. in 2024 [S1]. It addresses the challenge of dexterous manipulation of transparent objects, where common depth sensors struggle [S1]. The method builds on the observation that robot work areas (kitchens, shelves) are mostly static and opaque, allowing a background NeRF to serve as a strong prior [S1]. It is implemented using Torch-NGP (PyTorch version of InstantNGP) with multi-resolution hash grids, running on hardware such as RTX 4090 and AMD Ryzen Threadripper PRO 5965WX [S1]. The method is compatible with grasp planners like Dex-Net [S1].

## Sources
[S1] Residual-NeRF: Learning Residual NeRFs for Transparent Object Manipulation, https://doi.org/10.1109/icra57147.2024.10611224, fetched 2026-10-26
