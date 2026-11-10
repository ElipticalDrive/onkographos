# Direct Voxel Grid Optimization

Also called: DVG, Direct Voxel Density Optimization, Direct Voxel Grid Optimization Implementation, DirectVoxGo (DVGO)

## Overview
Direct Voxel Grid Optimization (DVGO) is a method for reconstructing per-scene radiance fields from calibrated images by directly optimizing a dense 3D voxel grid for volume density and a feature voxel grid for color, achieving convergence in less than 15 minutes on a single GPU [S1, S3]. It sets itself apart from implicit methods like [Neural Radiance Fields](4972-neural-radiance-fields.md) by replacing the large spatial MLP with explicit voxel grids, and from previous explicit voxel methods by training the grid directly from scratch without requiring a conversion step from a pre-trained implicit model [S1, S3]. This direct optimization approach reduces training time from hours/days to minutes while maintaining or exceeding the visual quality of NeRF [S1].

## How it works
DVGO builds on the standard emission-absorption volume rendering equation [44-radiance-field-optimization.md]. It uses a hybrid representation: a density voxel grid for geometry and a feature voxel grid combined with a shallow MLP for view-dependent appearance [S1, S3].

### Inputs and Outputs
**Inputs:**
- A set of $N$ calibrated images with known camera poses.
- Camera intrinsics and extrinsics.
- Near and far clipping planes for ray sampling.

**Outputs:**
- Optimized density voxel grid $V(\sigma)$.
- Optimized feature voxel grid $V(\text{feat})$.
- Optimized shallow MLP parameters.
- Rendered novel views.

### Data Structures
1.  **Density Voxel Grid ($V(\sigma)$):**
    -   Layout: 3D array of raw density values.
    -   Resolution: Coarse stage $M(c) \approx 100^3$; Fine stage $M(f) \approx 160^3$ (default) or $256^3$ [S1, S3].
    -   Values: Raw densities $\sigma_{\text{raw}}$. These are not directly used as opacity; they are interpolated and then activated [S1, S3].
    -   Initialization: Initialized to yield opacities very close to zero everywhere. Specifically, the raw densities are set such that the activated alpha $\alpha_{\text{init}}^{(c)} = 10^{-6}$ in the coarse stage [S1, S3].

2.  **Feature Voxel Grid ($V(\text{feat})$):**
    -   Layout: 3D array of feature vectors.
    -   Resolution: Same as the corresponding density grid (coarse or fine) [S1, S3].
    -   Channels: $D = 12$ [S1, S3].
    -   Values: Feature vectors $\mathbf{f}$ queried at sample points.

3.  **Shallow MLP:**
    -   Architecture: 2 hidden layers, 128 channels per layer [S3].
    -   Inputs: Concatenation of positional embeddings of the 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$, and the feature vector $\mathbf{f}$ from the voxel grid.
    -   Outputs: Color $\mathbf{c}$ and potentially other view-dependent parameters (though DVGO primarily outputs color) [S1, S3].

4.  **Bounding Box (BBox):**
    -   Automatically determined to tightly enclose the volume of interest to ensure parsimony [S1, S3].

### Numbered Steps

1.  **Initialization:**
    -   Determine the BBox enclosing the scene [S1, S3].
    -   Initialize the coarse density grid $V(\sigma)^{(c)}$ with raw values corresponding to $\alpha_{\text{init}}^{(c)} = 10^{-6}$ [S1, S3].
    -   Initialize the fine density grid $V(\sigma)^{(f)}$ with raw values corresponding to $\alpha_{\text{init}}^{(f)} = 10^{-2}$ [S1, S3].
    -   Initialize the feature grids $V(\text{feat})^{(c)}$ and $V(\text{feat})^{(f)}$ (e.g., to zeros or small random values) [G].
    -   Initialize the shallow MLP parameters [G].

2.  **Coarse Stage Optimization:**
    -   Optimize $V(\sigma)^{(c)}$, $V(\text{feat})^{(c)}$, and the MLP for 10,000 iterations using the Adam optimizer [S1, S3].
    -   Base learning rate for voxel grids: 0.1; for MLP: $10^{-3}$ [S3].
    -   Batch size: 8,192 rays [S3].
    -   Apply exponential learning rate decay [S1, S3].
    -   **Priors for Robustness:**
        -   **Low-Density Initialization:** Ensures geometry is not biased toward camera near planes [S1, S3].
        -   **View-Count Learning Rate:** Voxels visible to fewer views receive a lower learning rate to avoid redundant "cloud" allocation for explaining observations from few views [S1, S3].
        -   **Free Space Threshold:** Define known free space using threshold $\tau_c = 10^{-3}$. Skip query points in this space [S1].

3.  **Fine Stage Optimization:**
    -   Optimize $V(\sigma)^{(f)}$, $V(\text{feat})^{(f)}$, and the MLP for 20,000 iterations [S1, S3].
    -   Base learning rates remain 0.1 (voxels) and $10^{-3}$ (MLP) [S3].
    -   **Progressive Scaling:** The fine-stage voxel grid is progressively scaled at checkpoints 1000, 2000, and 3000 [S1].
    -   **Unknown Space Threshold:** Skip low-density query points in unknown space using threshold $\tau_f = 10^{-4}$ [S1].
    -   Apply exponential learning rate decay, downscaled by 0.1 at 20,000 iterations [S1].

4.  **Rendering (Forward Pass):**
    -   For a ray $\mathbf{r}$, sample $K$ points between near and far planes.
    -   For each point $i$:
        -   Query density $\sigma_i$ from the density grid using **post-activation interpolation** (see Formulas).
        -   Query feature vector $\mathbf{f}_i$ from the feature grid using trilinear interpolation.
        -   Compute color $\mathbf{c}_i$ using the shallow MLP with $\mathbf{f}_i$, positional embeddings of position $\mathbf{x}_i$, and viewing direction $\mathbf{d}$ [S1, S3].
    -   Accumulate color using volume rendering quadrature [S1, S3].

### Formulas

**Volume Rendering:**
The rendered color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ is:
$$
\hat{C}(\mathbf{r}) = \left( \sum_{i=1}^K T_i \alpha_i \mathbf{c}_i \right) + T_{K+1} \mathbf{c}_{\text{bg}}
$$
where:
-   $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ is the opacity at sample $i$ [S1, S3].
-   $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ is the transmittance up to sample $i$ [S1, S3].
-   $\sigma_i$ is the volume density at sample $i$ [S1, S3].
-   $\delta_i$ is the distance to the adjacent sampled point [S1, S3].
-   $\mathbf{c}_i$ is the color at sample $i$ [S1, S3].
-   $\mathbf{c}_{\text{bg}}$ is the background color [S1, S3].

**Post-Activation Interpolation (Density):**
Unlike previous methods that interpolate activated opacity or use nearest-neighbor, DVGO interpolates raw densities and then applies the activation function:
1.  Trilinearly interpolate the raw density values $\sigma_{\text{raw}}$ from the voxel grid at the query point $\mathbf{x}$ to get $\bar{\sigma}_{\text{raw}}$.
2.  Apply the activation function (e.g., Softplus or ReLU) to produce the final density $\sigma$:
    $$
    \sigma = \text{activation}(\bar{\sigma}_{\text{raw}})
    $$
    This allows modeling sharp linear surfaces within a single grid cell [S1, S3].

**Loss Function:**
The photometric loss is:
$$
\mathcal{L}_{\text{photo}} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \left\| \hat{C}(\mathbf{r}) - C(\mathbf{r}) \right\|_2^2
$$
where $\mathcal{R}$ is the set of rays in a sampled mini-batch and $C(\mathbf{r})$ is the observed pixel color [S3].

Additional loss weights mentioned:
-   $w_{\text{photo}} = 1$ [S1].
-   $w_{\text{pt\_rgb}} = 10^{-1}$ (coarse), $10^{-2}$ (fine) [S1].
-   $w_{\text{bg}} = 10^{-2}$ (coarse), $10^{-3}$ (fine) [S1].

### Parameters and Typical Values
-   **Grid Resolution:** $M(c) = 100^3$ (coarse), $M(f) = 160^3$ (fine, default) or $256^3$ [S1, S3].
-   **Initial Alpha:** $\alpha_{\text{init}}^{(c)} = 10^{-6}$, $\alpha_{\text{init}}^{(f)} = 10^{-2}$ [S1, S3].
-   **Thresholds:** $\tau_c = 10^{-3}$ (coarse free space), $\tau_f = 10^{-4}$ (fine unknown space) [S1].
-   **Sampling Step Size:** $\delta^{(c)} = 0.5 \cdot s^{(c)}$, $\delta^{(f)} = 0.5 \cdot s^{(f)}$, where $s$ is voxel size [S3].
-   **MLP:** 2 hidden layers, 128 channels [S3].
-   **Positional Embeddings:** $k_x = 5$ frequencies for position, $k_d = 4$ frequencies for direction [S1].
-   **Optimizer:** Adam [S3].
-   **Learning Rates:** Voxel grids: 0.1; MLP: $10^{-3}$ [S3].
-   **Batch Size:** 8,192 rays [S3].
-   **Iterations:** Coarse: 10,000; Fine: 20,000 [S1, S3].

### Edge Cases and Pitfalls
-   **Suboptimal Geometry:** Direct voxel density optimization is prone to allocating "cloud" in free space to fit photometric loss instead of finding multi-view consistent geometry [S1, S3]. Mitigated by low-density initialization and view-count-based learning rates [S1, S3].
-   **Real-World Uncertainty:** Inconsistent lighting or SfM errors can lead to multi-view inconsistent and blurrier surfaces, lessening the advantage of post-activation for sharp surfaces [S1].
-   **Initialization Sensitivity:** Using $\alpha_{\text{init}}^{(c)} = 10^{-7}$ may require more iterations for the coarse stage in some scenes [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DVGO:
    def __init__(self, bbox_min, bbox_max, coarse_res=100, fine_res=160, 
                 feat_channels=12, mlp_hidden=128, num_mlp_layers=2,
                 kx=5, kd=4, alpha_init_c=1e-6, alpha_init_f=1e-2,
                 tau_c=1e-3, tau_f=1e-4):
        self.bbox_min = torch.tensor(bbox_min)
        self.bbox_max = torch.tensor(bbox_max)
        self.coarse_res = coarse_res
        self.fine_res = fine_res
        self.feat_channels = feat_channels
        self.kx = kx
        self.kd = kd
        
        # Initialize Density Grids
        # Raw densities initialized to yield alpha_init
        # alpha = 1 - exp(-sigma * delta) => sigma = -log(1-alpha)/delta
        # Assuming delta is roughly voxel size, we set raw sigma accordingly.
        # For simplicity, we initialize raw sigma to a small value that results in alpha_init after activation.
        # Note: The exact mapping depends on the activation function. 
        # If activation is identity or linear near 0, sigma_raw ~ -log(1-alpha)/delta.
        # DVGO uses Softplus/ReLU. We initialize raw grid to a value that, after interpolation and activation, gives alpha_init.
        # A common approximation for small alpha is sigma ~ alpha/delta.
        # We store raw densities.
        
        self.coarse_sigma_grid = nn.Parameter(torch.full((coarse_res, coarse_res, coarse_res), 
                                                         self._raw_sigma_from_alpha(alpha_init_c)))
        self.fine_sigma_grid = nn.Parameter(torch.full((fine_res, fine_res, fine_res), 
                                                       self._raw_sigma_from_alpha(alpha_init_f)))
        
        # Initialize Feature Grids
        self.coarse_feat_grid = nn.Parameter(torch.zeros(coarse_res, coarse_res, coarse_res, feat_channels))
        self.fine_feat_grid = nn.Parameter(torch.zeros(fine_res, fine_res, fine_res, feat_channels))
        
        # Initialize Shallow MLP
        layers = []
        in_dim = feat_channels + kx + kd # Feature + Pos Embedding Position + Pos Embedding Direction
        for _ in range(num_mlp_layers):
            layers.append(nn.Linear(in_dim, mlp_hidden))
            layers.append(nn.ReLU())
            in_dim = mlp_hidden
        layers.append(nn.Linear(mlp_hidden, 3)) # Output RGB
        self.mlp = nn.Sequential(*layers)
        
        # Optimizers
        self.optimizer = torch.optim.Adam([
            {'params': [self.coarse_sigma_grid, self.fine_sigma_grid, self.coarse_feat_grid, self.fine_feat_grid]},
            {'params': self.mlp.parameters(), 'lr': 1e-3}
        ], lr=0.1)
        
        self.tau_c = tau_c
        self.tau_f = tau_f

    def _raw_sigma_from_alpha(self, alpha):
        # Approximation: sigma_raw such that activation(sigma_raw) * delta ~ alpha
        # Assuming delta ~ 1 for normalization, and activation is Softplus/ReLU.
        # If activation is Softplus, sigma_raw ~ alpha for small alpha.
        return torch.tensor(alpha)

    def positional_embedding(self, x, k):
        # Standard positional embedding: [sin(x), cos(x), sin(2x), cos(2x), ...]
        embeddings = []
        for i in range(k):
            embeddings.append(torch.sin((2**i) * x))
            embeddings.append(torch.cos((2**i) * x))
        return torch.cat(embeddings, dim=-1)

    def query_density(self, sigma_grid, res, points):
        # points: (N, 3) in world coordinates
        # Normalize points to [0, 1] range relative to bbox
        points_norm = (points - self.bbox_min) / (self.bbox_max - self.bbox_min)
        
        # Clamp to [0, 1] to handle boundary cases
        points_norm = torch.clamp(points_norm, 0, 1)
        
        # Reshape for grid_sample: (N, 1, 1, 3) -> (N, 1, 1, 3)
        # grid_sample expects (N, H_out, W_out, 2) for 2D or (N, D_out, H_out, W_out, 3) for 3D
        # We need to interpolate 3D grid.
        # points_norm shape: (N, 3). We need to add batch and depth dims if necessary, 
        # but grid_sample for 3D takes (N, 1, 1, 1, 3) if we want single point? 
        # Actually, grid_sample for 3D: input (N, C, D, H, W), grid (N, ..., 3)
        # Let's assume we query a batch of points.
        
        # Reshape points to (N, 1, 1, 1, 3)
        points_grid = points_norm.unsqueeze(1).unsqueeze(1).unsqueeze(1)
        
        # Interpolate
        # sigma_raw: (Res, Res, Res) -> (1, 1, Res, Res, Res)
        sigma_raw = sigma_grid.unsqueeze(0).unsqueeze(0)
        
        # grid_sample mode='trilinear', align_corners=False
        sigma_interp = F.grid_sample(sigma_raw, points_grid, mode='trilinear', align_corners=False, padding_mode='border')
        
        # Remove batch/depth dims: (1, 1, 1, 1, 1) -> (N,)
        sigma_interp = sigma_interp.squeeze()
        
        # Post-activation
        # DVGO uses Softplus or ReLU. Let's use Softplus as it's smoother for gradients.
        sigma = F.softplus(sigma_interp)
        
        return sigma

    def query_color(self, feat_grid, res, points, directions):
        # Query features
        points_norm = (points - self.bbox_min) / (self.bbox_max - self.bbox_min)
        points_norm = torch.clamp(points_norm, 0, 1)
        points_grid = points_norm.unsqueeze(1).unsqueeze(1).unsqueeze(1)
        feat_raw = feat_grid.unsqueeze(0).unsqueeze(0)
        feat_interp = F.grid_sample(feat_raw, points_grid, mode='trilinear', align_corners=False, padding_mode='border').squeeze()
        
        # Positional Embeddings
        pos_emb = self.positional_embedding(points, self.kx)
        dir_emb = self.positional_embedding(directions, self.kd)
        
        # Concatenate
        input_vec = torch.cat([feat_interp, pos_emb, dir_emb], dim=-1)
        
        # MLP
        color = self.mlp(input_vec)
        return color

    def render_ray(self, ray_origins, ray_directions, near, far, sigma_grid, feat_grid, res):
        # Sample points along ray
        t_vals = torch.linspace(0, 1, 64) # K=64 samples
        t = near + t_vals * (far - near)
        points = ray_origins.unsqueeze(1) + ray_directions.unsqueeze(1) * t.unsqueeze(2)
        
        # Query density
        sigma = self.query_density(sigma_grid, res, points.view(-1, 3))
        sigma = sigma.view(ray_origins.shape[0], 64)
        
        # Query color
        directions = ray_directions.unsqueeze(1).expand(-1, 64, -1)
        color = self.query_color(feat_grid, res, points.view(-1, 3), directions.view(-1, 3))
        color = color.view(ray_origins.shape[0], 64, 3)
        
        # Volume Rendering
        # delta_t = (far - near) / 64
        delta_t = (far - near) / 64
        alpha = 1 - torch.exp(-sigma * delta_t)
        T = torch.cumprod(torch.cat([torch.ones_like(alpha[:, :1]), 1 - alpha[:, :-1]], dim=1), dim=1)
        
        weights = alpha * T
        color_rendered = torch.sum(weights.unsqueeze(2) * color, dim=1)
        
        # Background color (assume white or black, default to 0)
        # T_last = T[:, -1]
        # color_rendered += T_last.unsqueeze(1) * background_color
        
        return color_rendered

    def step(self, images, poses, near, far):
        # Sample rays from images
        # ... (ray sampling logic) ...
        # For brevity, assume rays_origins, rays_directions, target_colors are prepared
        
        # Coarse Stage
        color_coarse = self.render_ray(rays_origins, rays_directions, near, far, 
                                       self.coarse_sigma_grid, self.coarse_feat_grid, self.coarse_res)
        loss_coarse = F.mse_loss(color_coarse, target_colors)
        
        # Apply priors (e.g., view-count based LR adjustment)
        # ... (gradient scaling logic) ...
        
        self.optimizer.zero_grad()
        loss_coarse.backward()
        self.optimizer.step()
        
        # Fine Stage (similar, but with fine grids and progressive scaling)
        # ...
```

## Subtopics

### [Ref-DVGO](6572-ref-dvgo.md)
A reflection-aware variant that combines DVGO's fast hybrid representation with Ref-NeRF's reflection direction reparameterization to handle specular reflections in reflective scenes, reducing GPU memory and training time compared to Ref-NeRF.

## Operating characteristics
-   **Time Complexity:** Training is significantly faster than NeRF. Convergence is about two orders of magnitude faster, reducing training time from 10-20 hours to 15-22 minutes per scene on a single NVIDIA RTX 2080 Ti GPU [S1, S3].
-   **Memory Complexity:** Uses explicit voxel grids, which can be memory-intensive at high resolutions ($160^3$ or $256^3$), but avoids the large memory footprint of storing intermediate states for reverse-mode AD in large MLPs [S1, S3].
-   **Rendering Speed:** Approximately 45x faster than NeRF (0.64 seconds vs 29 seconds per 800x800 image) [S1].
-   **Quality:** Achieves PSNR of 31.95 (160^3) and 32.80 (256^3) on standard benchmarks, outperforming NeRF (31.01) and MVSNeRF (28.14) without cross-scene pretraining [S1].
-   **Strengths:** Extremely fast training, no need for cross-scene pretraining, high rendering speed, sharp surface modeling via post-activation interpolation [S1, S3].
-   **Limitations:** Prone to suboptimal geometry (cloud in free space) if priors are not applied; real-world data uncertainty can blur surfaces; limited to bounded scenes (not unbounded or forward-facing) [S1, S3].

## Background
Direct Voxel Grid Optimization (DVGO) was introduced by Cheng Sun, Min Sun, and Hwann-Tzong Chen in 2022 [S1, S3]. It was presented at CVPR 2022 [S1, S3]. The method addresses the slow training times of Neural Radiance Fields (NeRF) by directly optimizing a dense voxel grid for density and a feature grid for color, avoiding the need for implicit MLPs and conversion steps used in previous voxel-based methods [S1, S3]. It is used for novel view synthesis on datasets like Syn-NeRF, Syn-NSVF, BlendedMVS, and Tanks and Temples [S1, S3].

## Sources
[S1] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
[S2] ScanNeRF: a Scalable Benchmark for Neural Radiance Fields, https://doi.org/10.1109/wacv56688.2023.00088, fetched 2026-10-26
[S3] Direct Voxel Grid Optimization: Super-Fast Convergence ..., https://openaccess.thecvf.com/content/CVPR2022/papers/Sun_Direct_Voxel_Grid_Optimization_Super-Fast_Convergence_for_Radiance_Fields_Reconstruction_CVPR_2022_paper.pdf, fetched 2026-10-29
