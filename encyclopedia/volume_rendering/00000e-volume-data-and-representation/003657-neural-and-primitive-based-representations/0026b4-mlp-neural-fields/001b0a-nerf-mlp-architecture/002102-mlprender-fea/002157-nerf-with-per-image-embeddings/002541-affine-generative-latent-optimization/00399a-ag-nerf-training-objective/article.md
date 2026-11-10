# AG-NeRF Training Objective

Also called: AG-NeRF Loss, Attention-Guided NeRF Objective

## Overview
The AG-NeRF training objective is a differentiable loss function designed to optimize the volume rendering of large-scale outdoor scenes using multi-height source images [S1]. It solves the problem of reconstructing geometry and texture across varying camera altitudes (from drone to satellite) without requiring a priori camera height knowledge or multi-stage training pipelines [S1]. Unlike BungeeNeRF, which relies on manual data splitting and sequential stage training, AG-NeRF uses an attention-based feature fusion module to extract relevant features from source images, enabling end-to-end optimization [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   Ray $r$ defined by origin and direction [S1].
    *   Set of $V$ source images (typically $V=10$) providing scene priors [S1].
    *   3D point coordinates $q_k = (x, y, z)$ and viewing directions $d_k = (\theta, \phi)$ for sample points along the ray [S1].
*   **Outputs:**
    *   Rendered color $\tilde{C}(r)$ for the ray [S1].
    *   Loss value $L$ for backpropagation [S1].

**Data Structures**
*   **Sample Points:** Two sets of points per ray: coarse ($N_c$) and fine ($N_f$) [S1].
    *   $N_c = 64$ coarse points.
    *   $N_f = 64$ fine points.
    *   Points are biased towards the object surface [S1].
*   **Feature Vector:** Dimension $d_{fi} = 35$ [S1].
*   **MLP1 (Density/Feature Network):** 4 layers, 64 feature dimensions [S1].
*   **MLP2 (Color Network):** 1 layer, 64 feature dimensions [S1].
*   **Feature Fusion Module:** 2-layer transformer block [S1].

**Numbered Steps**

1.  **Positional Encoding:** Map 3D coordinates $q_k$ and directions $d_k$ into high-dimensional space using positional encoding $\gamma$ [S1].
    $$ \gamma(q_k), \gamma(d_k) $$

2.  **Feature Extraction and Fusion:**
    *   Extract features from $V$ source images using a U-Net-like architecture with a truncated ResNet34 encoder (after layer 3) and a decoder with convolutions and skip-connections [S1].
    *   Fuse features for the query point $q_k$ using an attention-based mechanism to produce fused features $f_{att}(q_k)$ [S1].
    *   *Note:* The attention module requires source image inputs; without them ($V=0$), it fails to take effect [S1].

3.  **Density and Intermediate Feature Prediction:** Concatenate positional encoding of coordinates and fused features, then pass through MLP1 to obtain density $\sigma_k$ and intermediate feature $h_k$ [S1].
    $$ \sigma_k, h_k = \text{MLP1}(\gamma(q_k), f_{att}(q_k)) \quad \text{(Eq. 4)} $$

4.  **Color Prediction:** Concatenate positional encoding of direction and intermediate feature, then pass through MLP2 to obtain color $c_k$ [S1].
    $$ c_k = \text{MLP2}(\gamma(d_k), h_k) \quad \text{(Eq. 4)} $$

5.  **Volume Rendering:** Compute the predicted color $\tilde{C}(r)$ for ray $r$ by integrating along $N$ sample points (coarse and fine) [S1].
    $$ \tilde{C}(r) = \sum_{k=1}^{N} T_k (1 - \exp(-\sigma_k \delta_k)) c_k \quad \text{(Eq. 5)} $$
    Where:
    *   $\delta_k$ is the distance between adjacent sample points [S1].
    *   $T_k$ is the accumulated transmittance:
        $$ T_k = \exp\left(-\sum_{j=1}^{k-1} \sigma_j \delta_j\right) $$
    *   $N = N_c + N_f = 128$ total samples per ray [S1].

6.  **Loss Calculation:** Minimize the total squared error between rendered colors and ground truth colors for both coarse and fine predictions over a batch of rays $R$ [S1].
    $$ L = \sum_{r \in R} \left( \| C(r) - \tilde{C}_c(r) \|^2 + \| C(r) - \tilde{C}_f(r) \|^2 \right) \quad \text{(Eq. 6)} $$
    Where:
    *   $C(r)$ is the ground truth color [S1].
    *   $\tilde{C}_c(r)$ is the coarse predicted color [S1].
    *   $\tilde{C}_f(r)$ is the fine predicted color [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class AGNeRFTrainingObjective:
    def __init__(self, num_source_images=10, feature_dim=35, 
                 coarse_samples=64, fine_samples=64, 
                 mlp1_layers=4, mlp1_width=64, 
                 mlp2_layers=1, mlp2_width=64,
                 lr_feature=1e-3, lr_mlp=5e-4):
        self.num_source_images = num_source_images
        self.feature_dim = feature_dim
        self.coarse_samples = coarse_samples
        self.fine_samples = fine_samples
        
        # [G] Feature extraction uses a U-Net-like architecture with ResNet34 encoder truncated after layer 3.
        # [G] Feature fusion uses a 2-layer transformer block.
        # [G] Exact architectural implementations are not detailed in the sources.
        self.feature_extractor = None
        self.feature_fusion = None
        
        # MLP1: 4 layers, 64 width. Input dim = PE_dim + feature_dim. Output dim = 64 + 64 (density + h)
        self.mlp1 = nn.Sequential(*[nn.Linear(mlp1_width if i > 0 else (0+feature_dim), mlp1_width) for i in range(mlp1_layers)])
        # MLP2: 1 layer, 64 width. Input dim = PE_dim + 64. Output dim = 3 (RGB)
        self.mlp2 = nn.Linear(0+64, 3)
        
        self.optimizer_feature = torch.optim.Adam(self.feature_extractor.parameters(), lr=lr_feature)
        self.optimizer_fusion = torch.optim.Adam(self.feature_fusion.parameters(), lr=lr_feature)
        self.optimizer_mlp = torch.optim.Adam(list(self.mlp1.parameters()) + list(self.mlp2.parameters()), lr=lr_mlp)

    def positional_encoding(self, x, num_freq_bands=10):
        pe = []
        for i in range(num_freq_bands):
            pe.append(torch.sin(x * (2 ** i)))
            pe.append(torch.cos(x * (2 ** i)))
        return torch.cat(pe, dim=-1)

    def volume_render(self, density, color, delta):
        alpha = 1.0 - torch.exp(-density * delta)
        T = torch.ones_like(alpha)
        T[:, 1:] = torch.cumprod(1.0 - alpha[:, :-1] + 1e-10, dim=-1)
        weights = T * alpha
        return torch.sum(weights.unsqueeze(-1) * color, dim=-2)

    def forward(self, rays, source_images, ground_truth_colors):
        # [G] The sources do not specify the exact ray sampling algorithm beyond stating points are biased towards the object surface.
        coarse_points, coarse_dirs = self.sample_rays(rays, self.coarse_samples) # [G] Implementation not provided in sources.
        fine_points, fine_dirs = self.sample_rays(rays, self.fine_samples) # [G] Implementation not provided in sources.

        source_features = self.feature_extractor(source_images)
        fused_coarse = self.feature_fusion(source_features, coarse_points)
        fused_fine = self.feature_fusion(source_features, fine_points)

        pe_coarse_coords = self.positional_encoding(coarse_points)
        pe_coarse_dirs = self.positional_encoding(coarse_dirs)
        pe_fine_coords = self.positional_encoding(fine_points)
        pe_fine_dirs = self.positional_encoding(fine_dirs)

        input_mlp1_coarse = torch.cat([pe_coarse_coords, fused_coarse], dim=-1)
        input_mlp1_fine = torch.cat([pe_fine_coords, fused_fine], dim=-1)
        out_mlp1_coarse = self.mlp1(input_mlp1_coarse)
        out_mlp1_fine = self.mlp1(input_mlp1_fine)
        density_coarse, h_coarse = out_mlp1_coarse.chunk(2, dim=-1)
        density_fine, h_fine = out_mlp1_fine.chunk(2, dim=-1)

        input_mlp2_coarse = torch.cat([pe_coarse_dirs, h_coarse], dim=-1)
        input_mlp2_fine = torch.cat([pe_fine_dirs, h_fine], dim=-1)
        color_coarse = self.mlp2(input_mlp2_coarse)
        color_fine = self.mlp2(input_mlp2_fine)

        delta_coarse = torch.diff(torch.cat([rays[..., :1, :], coarse_points], dim=-2), dim=-2).norm(dim=-1)
        delta_fine = torch.diff(torch.cat([rays[..., :1, :], fine_points], dim=-2), dim=-2).norm(dim=-1)
        rendered_coarse = self.volume_render(density_coarse, color_coarse, delta_coarse)
        rendered_fine = self.volume_render(density_fine, color_fine, delta_fine)

        loss_coarse = torch.sum((ground_truth_colors - rendered_coarse) ** 2)
        loss_fine = torch.sum((ground_truth_colors - rendered_fine) ** 2)
        return loss_coarse + loss_fine

    def sample_rays(self, rays, num_samples):
        # [G] The sources do not specify the exact ray sampling algorithm beyond stating points are biased towards the object surface.
        pass
```

**Parameters and Typical Values**
*   $N_c = 64$ (coarse samples per ray) [S1].
*   $N_f = 64$ (fine samples per ray) [S1].
*   $V = 10$ (number of source images) [S1].
*   $d_{fi} = 35$ (feature vector dimension) [S1].
*   MLP1: 4 layers, 64 feature dimensions [S1].
*   MLP2: 1 layer, 64 feature dimensions [S1].
*   Optimizer: Adam [S1].
*   Learning Rate (Feature Extraction Network): $10^{-3}$ [S1].
*   Learning Rate (Tiny MLP): $5 \times 10^{-4}$ [S1].
*   Total Iterations: 200,000 [S1].
*   Rays per Iteration: 2,048 [S1].
*   Hardware: Single RTX 4090 GPU [S1].

**Edge Cases and Pitfalls**
*   **Source Image Dependency:** The attention-based feature extraction and fusion module fails to take effect without inputs of source images ($V=0$) [S1].
*   **Source Image Count:** When the number of source images exceeds 10, there is a tiny margin on 3D reconstruction result, so 10 is chosen to balance computational efficiency and rendering quality [S1].
*   **Pooling Alternatives:** Using AvgPool or MaxPool instead of attention-based fusion results in lower PSNR (1.7 dB and 1.5 dB decrease, respectively) [S1].

## Subtopics

### Depth Conditioning
Conditioning the NeRF MLP on the difference between sample location and estimated depth to serve as a strong prior for visual opacity [S2]. Uses TransMVSNet architecture to estimate depth from sparse observations by inputting four images $I(i)$ to predict per-view depth maps $D(i)$ and standard deviations $D_{std}(i)$ [S2]. The calculated difference $\Delta z$ is input to the pixelNeRF MLP $f_1$ as additional conditioning: $V(i) = f_1(x(i), d(i), \omega(i), \gamma(\Delta z))$ [S2].

## Operating characteristics
*   **Time Complexity:** AG-NeRF converges in 1 day on a single RTX 4090 GPU, compared to BungeeNeRF which takes up to 5 days for four stages [S1].
*   **Memory Complexity:** Not given in the sources [S1].
*   **Measured Performance:**
    *   **56 Leonard Dataset (4k iterations):** PSNR 24.570, SSIM 0.837, LPIPS 0.259 [S1].
    *   **Transamerica Dataset (4k iterations):** PSNR 25.063, SSIM 0.846, LPIPS 0.262 [S1].
    *   **56 Leonard Dataset (200k iterations):** PSNR 30.450, SSIM 0.963, LPIPS 0.065 [S1].
    *   **Transamerica Dataset (200k iterations):** PSNR 30.599, SSIM 0.959, LPIPS 0.076 [S1].
*   **Scaling:** Increasing source images from 2 to 14 on 56 Leonard yields about 1.3 dB gain in PSNR [S1].
*   **Strengths:**
    *   End-to-end differentiable training without a priori camera height knowledge [S1].
    *   Handles varying camera altitudes (drone to satellite) [S1].
    *   Significantly faster training than multi-stage methods like BungeeNeRF (1 day vs 5 days) [S1].
    *   Achieves ~6 dB PSNR increase over BungeeNeRF [S1].
*   **Limitations:**
    *   Requires source images; performance drops significantly (~11 dB PSNR decrease) without them [S1].
    *   Performance plateaus after 10 source images [S1].
*   **When to Use:** Large-scale outdoor scene reconstruction with varying altitudes where source images are available [S1].
*   **When Not to Use:** Scenes without source images or where computational efficiency is critical and source image selection is not feasible [S1].

## Background
AG-NeRF was proposed by Guo, Jingfeng, et al. in the paper "AG-NeRF: Attention-Guided Neural Radiance Fields for Multi-height Large-Scale Outdoor Scene Rendering" [S1]. It addresses limitations of existing NeRF-based approaches, which typically require hours to days to train and struggle with large-scale outdoor scenes due to low-level rendering requirements, complex parameter tuning, or computational bottlenecks [S1]. The method builds on BungeeNeRF but improves upon it by eliminating the need for manual data splitting and multi-stage training [S1].

## Sources
[S1] AG-NeRF: Attention-Guided Neural Radiance Fields for Multi-height Large-Scale Outdoor Scene Rendering, https://doi.org/10.1007/978-981-97-8508-7_8, fetched 2026-10-30
[S2] DINER: Depth-aware Image-based NEural Radiance fields, https://doi.org/10.1109/cvpr52729.2023.01198, fetched 2026-10-29
