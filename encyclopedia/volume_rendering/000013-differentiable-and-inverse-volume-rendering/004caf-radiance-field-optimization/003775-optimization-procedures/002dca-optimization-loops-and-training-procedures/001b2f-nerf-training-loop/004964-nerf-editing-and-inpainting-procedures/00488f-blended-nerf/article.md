# Blended-NeRF

Also called: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields

## Overview
Blended-NeRF is a framework for editing a specific region of interest (ROI) in an existing Neural Radiance Field (NeRF) scene using text prompts and a user-defined 3D bounding box [S1]. It differs from methods that synthesize objects from scratch (e.g., Dream Fields) or require semantic masks by initializing a generator NeRF with the original scene's weights and blending the synthesized content with the original scene via volumetric blending along rays [S1]. It solves the problem of local 3D editing while preserving the unedited scene outside the ROI [S1].

## How it works
**Inputs:**
1.  Existing NeRF model $F_{\theta_O}$ mapping $(\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$ [S1].
2.  3D ROI box $B$ localized in the scene [S1].
3.  Text prompt $y$ [S1].
4.  Hyperparameters: $\alpha$ (blending strength), $\lambda_T, \lambda_D$ (loss weights), $\tau$ (max transmittance), $\rho$ (max depth variance), $\epsilon$ (stability constant) [S1].

**Outputs:**
Edited NeRF scene $x_e$ [S1].

**Data Structures:**
*   $F_{\theta_O}$: MLP parameters for original scene.
*   $F_{\theta_G}$: MLP parameters for generator, initialized with $F_{\theta_O}$ [S1].
*   $B$: Axis-aligned 3D box defined by center and dimensions [S1].
*   $E_{img}, E_{txt}$: Encoders from a language-image model (e.g., CLIP) [S1].

**Numbered Steps:**

1.  **Initialization:** Initialize $F_{\theta_G}$ with weights from $F_{\theta_O}$ [S1].
2.  **ROI Localization:** Define box $B$ using a GUI that renders $F_{\theta_O}$ and uses depth maps for 3D understanding [S1].
3.  **Training Loop:**
    a.  **Pose Sampling:** Sample a random camera pose $P$ around the scene with azimuth/elevation $(\theta, \phi)$. Calculate initial distance $d = \frac{e_{max}}{2 \tan(\gamma/2)}$ where $e_{max}$ is the max box dimension and $\gamma$ is camera FOV. Randomly sample radius $r$ around $d$ [S1].
    b.  **Ray Casting & Sampling:** Cast rays $r$ from $P$. Sample points $\mathbf{x}_i$ along rays.
    c.  **Generator Rendering (Inside ROI):** For points $\mathbf{x}_i \in B$, query $F_{\theta_G}$ for $\sigma_G(\mathbf{x}_i)$ and $\mathbf{c}_G(\mathbf{x}_i)$. For points outside $B$, set density to zero. Render generated content $I_{ROI}$ using only points inside $B$ [S1]:
        $$ C(r) = \begin{cases} \frac{\sum_{\mathbf{x}_i \in B} T_i (1 - e^{-\sigma_i \delta_i}) \mathbf{c}_i}{\sum_{\mathbf{x}_i \in B} T_i (1 - e^{-\sigma_i \delta_i})} & \exists \mathbf{x}_i \in r \text{ s.t. } \mathbf{x}_i \in B \\ 0 & \text{otherwise} \end{cases} $$
        where $T_i = \exp(-\sum_{j < i} \sigma_j \delta_i)$ [S1].
    d.  **Object Blending Mode (Optional):** If blending with existing content inside $B$, query both $F_{\theta_O}$ and $F_{\theta_G}$. Compute alphas: $\alpha_O(\mathbf{x}_i) = 1 - \exp(\phi(\sigma_O(\mathbf{x}_i)) \cdot \delta_i)$ and $\alpha_G(\mathbf{x}_i) = 1 - \exp(\phi(\sigma_G(\mathbf{x}_i)) \cdot \delta_i)$, where $\phi$ is a non-negative activation [S1]. Compute blended color:
        $$ \mathbf{c}(\mathbf{x}_i) = S\left(\frac{\mathbf{c}_O(\mathbf{x}_i) \cdot \alpha_O(\mathbf{x}_i) + \mathbf{c}_G(\mathbf{x}_i) \cdot \alpha_G(\mathbf{x}_i)}{\epsilon + \alpha_O(\mathbf{x}_i) + \alpha_G(\mathbf{x}_i)}\right) $$
        where $S$ is sigmoid [S1]. Compute blended density $\sigma(\mathbf{x}_i)$ using either sum-in-activation $\phi(\sigma_O + \sigma_G)$ or sum-out-activation $\phi(\sigma_O) + \phi(\sigma_G)$ [S1].
    e.  **Loss Computation:**
        i.  **Similarity Loss:** $L_{sim} = -E_{img}(I_{ROI})^T E_{txt}(y)$ [S1].
        ii. **Transmittance Loss:** $L_T = -\min(\tau, \text{mean}(T(P)))$ [S1].
        iii. **Depth Loss:** $L_D = -\min(\rho, \sigma^2(D(P)))$ where $\sigma^2$ is variance of disparity map [S1].
        iv. **Total Loss:** $L_{total} = L_{sim} + \lambda_T L_T + \lambda_D L_D$ [S1].
    f.  **Optimization:** Minimize $L_{total}$ with respect to $\theta_G$ [S1].
    g.  **Augmentations:** Use random backgrounds (Gaussian noise, checkerboard, Fourier textures) and directional dependent prompts (concatenating view-dependent text to original caption) [S1]. Anneal $L_T$ and $L_D$ introduction [S1].
4.  **Blending (Inference):**
    a.  For each view ray, query both $F_{\theta_O}$ and $F_{\theta_G}$ [S1].
    b.  Points outside $B$ use $F_{\theta_O}$ [S1].
    c.  Points inside $B$ blend densities and colors using distance smoothing:
        $$ f(\mathbf{x}) = 1 - \exp\left(-\alpha \frac{d(\mathbf{x})}{diag}\right) $$
        $$ \sigma_{blend}(\mathbf{x}) = f(\mathbf{x}) \cdot \sigma_O(\mathbf{x}) + (1 - f(\mathbf{x})) \cdot \sigma_G(\mathbf{x}) $$
        $$ \mathbf{c}_{blend}(\mathbf{x}) = f(\mathbf{x}) \cdot \mathbf{c}_O(\mathbf{x}) + (1 - f(\mathbf{x})) \cdot \mathbf{c}_G(\mathbf{x}) $$
        where $d(\mathbf{x})$ is Euclidean distance from ROI center, $diag$ is box diagonal, and $\alpha$ controls blending strength [S1].
    d.  Render final view $x_e$ using standard volume rendering on the blended field [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch
import torch.nn as nn
import torch.nn.functional as F

class BlendedNeRF:
    def __init__(self, F_theta_O, B, text_prompt, alpha=2.0, lambda_T=0.1, lambda_D=0.1, tau=0.9, rho=1.0, epsilon=1e-6):
        """
        F_theta_O: Original NeRF model (MLP)
        B: 3D ROI box (center, dimensions)
        text_prompt: String prompt
        alpha: Blending strength hyperparameter
        lambda_T, lambda_D: Loss weights
        tau: Max transmittance for L_T
        rho: Max variance for L_D
        epsilon: Numerical stability constant
        """
        self.F_O = F_theta_O
        # Initialize Generator with Original weights
        self.F_G = copy_weights(F_theta_O) 
        self.B = B
        self.text_prompt = text_prompt
        self.alpha = alpha
        self.lambda_T = lambda_T
        self.lambda_D = lambda_D
        self.tau = tau
        self.rho = rho
        self.epsilon = epsilon
        
        # Load CLIP encoders
        self.E_img, self.E_txt = load_clip_encoders()
        
        # Compute ROI center and diagonal
        self.roi_center = B.center
        self.diag = B.diagonal_length

    def positional_encoding(self, x, L=10):
        """High frequency sinusoidal positional encoding."""
        # gamma(x) = [cos(2^l x), sin(2^l x)] for l in 0..L-1
        freq_bands = 2.0 ** torch.arange(L, device=x.device).float()
        x_expanded = x.unsqueeze(-1) * freq_bands.unsqueeze(0)
        pe = torch.cat([torch.sin(x_expanded), torch.cos(x_expanded)], dim=-1)
        return pe

    def render_ray_generator(self, ray_origin, ray_direction, num_samples, device):
        """
        Renders the generator's contribution inside the ROI box B.
        Returns color and transmittance for points inside B.
        """
        # Sample points along ray
        t_vals = torch.linspace(0, 1, num_samples, device=device)
        points = ray_origin.unsqueeze(0) + ray_direction.unsqueeze(0) * t_vals.unsqueeze(1)
        
        # Check which points are inside B
        inside_mask = self.is_inside_box(points, self.B)
        
        if not torch.any(inside_mask):
            return torch.zeros_like(ray_origin), torch.ones_like(ray_origin)

        # Query Generator for points inside B
        # Note: In practice, only pass points inside B to F_G to save compute, 
        # or pass all and mask outputs. Here we assume F_G handles input.
        points_inside = points[inside_mask]
        
        # Encode inputs
        pe_points = self.positional_encoding(points_inside)
        
        # Query F_G
        sigma_G, c_G = self.F_G(pe_points) # Assuming F_G outputs (sigma, color)
        
        # Apply non-negative activation if required for blending mode
        # phi is typically ReLU or similar
        sigma_G = torch.relu(sigma_G)
        
        # Compute transmittance and alpha for these points
        # delta is distance between samples (assume uniform for simplicity or compute from t_vals)
        delta = t_vals[1] - t_vals[0] 
        
        alpha_G = 1.0 - torch.exp(-sigma_G * delta)
        T_G = torch.exp(-torch.cumsum(sigma_G * delta, dim=0)) # Cumulative transmittance
        
        # Weighted color
        weights = T_G * alpha_G
        color = torch.sum(weights.unsqueeze(1) * c_G, dim=0)
        
        # Return only the contribution from inside B
        # For the final blend, we need to know the transmittance reaching the ROI
        # This is a simplified return; full volume rendering requires integrating outside too.
        # For training L_sim, we only care about I_ROI.
        return color, T_G[-1] if len(T_G) > 0 else torch.ones(1)

    def is_inside_box(self, points, box):
        """Check if points are inside axis-aligned box B."""
        # points shape: (N, 3)
        # box.min, box.max
        min_coords = box.min
        max_coords = box.max
        return torch.all((points >= min_coords) & (points <= max_coords), dim=1)

    def compute_loss(self, ray_origin, ray_direction, device):
        """Compute total loss for a single ray/pose."""
        # 1. Render Generator inside ROI
        c_gen, T_gen = self.render_ray_generator(ray_origin, ray_direction, 64, device)
        
        # 2. Similarity Loss
        # Render I_ROI (simplified as c_gen for this step)
        # In practice, I_ROI is the rendered image patch
        img_feat = self.E_img(c_gen) # Placeholder for actual image encoding
        txt_feat = self.E_txt(self.text_prompt)
        L_sim = -torch.dot(img_feat, txt_feat)
        
        # 3. Transmittance Loss
        # Mean transmittance of the view (simplified to T_gen for ROI)
        mean_T = torch.mean(T_gen)
        L_T = -torch.min(torch.tensor(self.tau), mean_T)
        
        # 4. Depth Loss
        # Compute disparity map variance (simplified placeholder)
        disparity = 1.0 / (ray_origin + ray_direction * 1.0) # Placeholder
        var_D = torch.var(disparity)
        L_D = -torch.min(torch.tensor(self.rho), var_D)
        
        L_total = L_sim + self.lambda_T * L_T + self.lambda_D * L_D
        return L_total

    def blend_and_render(self, ray_origin, ray_direction, num_samples, device):
        """
        Final inference blending.
        """
        t_vals = torch.linspace(0, 1, num_samples, device=device)
        points = ray_origin.unsqueeze(0) + ray_direction.unsqueeze(0) * t_vals.unsqueeze(1)
        
        sigma_out = torch.zeros_like(points[..., 0])
        c_out = torch.zeros_like(points[..., 0])
        
        inside_mask = self.is_inside_box(points, self.B)
        
        # Points outside B: Use Original
        outside_mask = ~inside_mask
        if torch.any(outside_mask):
            pts_out = points[outside_mask]
            pe_out = self.positional_encoding(pts_out)
            sigma_O_out, c_O_out = self.F_O(pe_out)
            sigma_out[outside_mask] = sigma_O_out
            c_out[outside_mask] = c_O_out
            
        # Points inside B: Blend
        if torch.any(inside_mask):
            pts_in = points[inside_mask]
            pe_in = self.positional_encoding(pts_in)
            
            sigma_O_in, c_O_in = self.F_O(pe_in)
            sigma_G_in, c_G_in = self.F_G(pe_in)
            
            # Distance smoothing factor
            dists = torch.norm(pts_in - self.roi_center, dim=1)
            f = 1.0 - torch.exp(-self.alpha * dists / self.diag)
            
            sigma_blend = f * sigma_O_in + (1.0 - f) * sigma_G_in
            c_blend = f * c_O_in + (1.0 - f) * c_G_in
            
            sigma_out[inside_mask] = sigma_blend
            c_out[inside_mask] = c_blend
            
        # Standard Volume Rendering
        delta = t_vals[1] - t_vals[0]
        alpha = 1.0 - torch.exp(-sigma_out * delta)
        T = torch.exp(-torch.cumsum(sigma_out * delta, dim=0))
        weights = T * alpha
        color = torch.sum(weights.unsqueeze(1) * c_out, dim=0)
        
        return color

    def train_step(self, ray_origin, ray_direction, optimizer):
        optimizer.zero_grad()
        loss = self.compute_loss(ray_origin, ray_direction, ray_origin.device)
        loss.backward()
        optimizer.step()
        return loss.item()
```

## Operating characteristics
*   **Time Complexity:** Training requires optimizing $F_{\theta_G}$ which is an MLP. Complexity is similar to standard NeRF training but restricted to rays passing through $B$ and augmented with CLIP encoding overhead. Inference involves querying two MLPs and blending.
*   **Memory Complexity:** Requires storing weights for both $F_{\theta_O}$ and $F_{\theta_G}$, plus CLIP encoders.
*   **Strengths:** Does not require 2D masks or semantic decomposition; works on any region defined by a 3D box; preserves the original scene outside the ROI; generates view-consistent results.
*   **Limitations:** Quality depends on the text prompt and CLIP alignment; may produce amorphous or transparent scenes without priors ($L_T, L_D$); blending artifacts may occur at ROI boundaries if $\alpha$ is not tuned.
*   **When to use:** When editing a specific object or region in an existing NeRF using text guidance without needing precise segmentation masks.
*   **When not to use:** When global scene editing is required, or when high-fidelity texture preservation outside the ROI is critical and the ROI box cannot be accurately localized.

## Background
Blended-NeRF was introduced by Ori Gordon, Omri Avrahami, and Dani Lischinski from The Hebrew University in 2023 [S1]. It builds on Dream Fields [27] by adapting its priors (pose sampling, background augmentation, transmittance/depth losses) to the context of editing existing NeRFs rather than generating new ones from scratch. It addresses limitations of methods like Kobayashi [29] and Benaim et al. [8] which require semantic decomposition or 2D masks [S1].

## Sources
[S1] Blended-NeRF: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields, https://arxiv.org/abs/2306.12760v2, fetched 2026-10-26
