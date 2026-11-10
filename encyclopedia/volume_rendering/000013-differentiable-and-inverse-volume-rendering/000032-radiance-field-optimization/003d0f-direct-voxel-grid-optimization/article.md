# Direct Voxel Grid Optimization

Also called: DVG, Direct Voxel Density Optimization, Direct Voxel Grid Optimization Implementation, DirectVoxGo (DVGO)

## Overview
Direct Voxel Grid Optimization (DVGO) is a method for reconstructing per-scene radiance fields from calibrated images by directly optimizing a dense 3D voxel grid for volume density and a feature voxel grid for color, achieving convergence in less than 15 minutes on a single GPU [S1, S4]. It sets itself apart from implicit methods like [Neural Radiance Fields](4972-neural-radiance-fields.md) by eliminating the need for lengthy MLP training, and from earlier explicit voxel methods by training the voxel grid directly from scratch without requiring a conversion step from a pre-trained implicit model [S1, S4]. Unlike [Plenoxels](806-explicit-radiance-fields.md) which optimizes colors directly in the grid, DVGO uses a shallow MLP to handle view-dependent appearance, balancing speed and quality [S2, S4].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ calibrated images with known camera poses (intrinsic and extrinsic parameters), normalized such that camera centers lie in $[-1, 1]$ [S3, S4].
*   **Outputs:** A rendered novel view image $\hat{C}(\mathbf{r})$ for any ray $\mathbf{r}$, and optimized voxel grids representing the scene geometry and appearance.

**Data Structures:**
The representation consists of two primary voxel grids and a shallow MLP:
1.  **Density Voxel Grid ($V_{\text{dens}}$):** Stores raw density values $\sigma_{\text{raw}}$.
    *   **Layout:** 3D array of size $M \times M \times M$.
    *   **Values:** Float32.
    *   **Initialization:** Initialized to yield opacities very close to zero (see parameters).
2.  **Feature Voxel Grid ($V_{\text{feat}}$):** Stores feature vectors for the shallow MLP.
    *   **Layout:** 3D array of size $M \times M \times M$.
    *   **Values:** Float32, with $D$ channels (typically $D=12$).
    *   **Initialization:** Zeros or small random values.
3.  **Color Voxel Grid ($V_{\text{color}}$):** (Used only in the coarse stage).
    *   **Layout:** 3D array of size $M \times M \times M$.
    *   **Values:** Float32, 3 channels (RGB).
4.  **Shallow MLP:**
    *   **Architecture:** 2 hidden layers, 128 channels per layer [S4].
    *   **Inputs:** Concatenation of positional encoding of 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$, and the feature vector from $V_{\text{feat}}$.
    *   **Output:** View-dependent color $\mathbf{c}$.

**Bounding Box (BBox):**
The method automatically finds a BBox tightly enclosing the volume of interest to allocate the voxel grids, ensuring parsimony [S1, S4]. The voxel grid coordinates are mapped to the BBox space.

**Parameters:**
*   **Grid Resolution:**
    *   Coarse stage ($M_c$): Typically $100^3$ [S1, S4].
    *   Fine stage ($M_f$): Typically $160^3$ (default) or $256^3$ [S1, S4].
*   **Sampling:**
    *   Coarse step size $\delta_c = 0.5 \times s_c$, where $s_c$ is the voxel size [S4].
    *   Fine step size $\delta_f = 0.5 \times s_f$, where $s_f$ is the voxel size [S4].
    *   Number of samples $K$ per ray.
*   **Initialization:**
    *   $\alpha_{\text{init}}^{(c)} = 10^{-6}$ (Initial activated alpha for coarse stage) [S1, S4].
    *   $\alpha_{\text{init}}^{(f)} = 0.01$ (Initial activated alpha for fine stage) [S1, S4].
*   **Thresholds:**
    *   $\tau_c = 10^{-3}$ (Threshold for defining known free space in coarse stage) [S1].
    *   $\tau_f = 10^{-4}$ (Threshold for skipping low-density query points in fine stage) [S1].
*   **Optimization:**
    *   Optimizer: Adam [S4].
    *   Base learning rate for voxel grids: $0.1$ [S1, S4].
    *   Base learning rate for MLP: $10^{-3}$ [S4].
    *   Batch size: 8,192 rays [S1, S4].
    *   Coarse iterations: 10,000 [S1, S4].
    *   Fine iterations: 20,000 [S1, S4].
    *   Learning rate schedule: Exponential decay, downscaled by 0.1 at 20,000 iterations [S1].
*   **Loss Weights:**
    *   $w_{\text{photo}} = 1$ [S1].
    *   $w_{\text{pt\_rgb}}$: $10^{-1}$ (coarse), $10^{-2}$ (fine) [S1].
    *   $w_{\text{bg}}$: $10^{-2}$ (coarse), $10^{-3}$ (fine) [S1].
*   **Positional Encoding:**
    *   $k_x = 5$ frequencies for position [S1].
    *   $k_d = 4$ frequencies for direction [S1].

**Algorithm Steps:**

The optimization proceeds in two phases: Coarse Geometry Searching and Fine Detail Reconstruction.

**1. Coarse Geometry Searching:**
1.  **Initialize:** Create $V_{\text{dens}}$ of size $M_c^3$. Initialize raw density values such that the activated alpha is $\alpha_{\text{init}}^{(c)} = 10^{-6}$. This avoids biasing geometry toward camera near planes [S1, S4].
2.  **Create Auxiliary Grids:** Initialize $V_{\text{color}}$ (for view-invariant color) and $V_{\text{feat}}$ (zeros).
3.  **Define Free Space:** Identify points in free space using threshold $\tau_c = 10^{-3}$. Skip query points in this space during optimization [S1].
4.  **Optimize:**
    *   For 10,000 iterations:
        *   Sample rays.
        *   For each ray, sample $K$ points with step size $\delta_c = 0.5 s_c$.
        *   Query $V_{\text{dens}}$ for raw density $\sigma_{\text{raw}}$.
        *   **Post-Activation Interpolation:** Trilinearly interpolate $\sigma_{\text{raw}}$ from the grid, then apply activation (e.g., Softplus or ReLU) to produce opacity $\alpha$. This allows modeling sharp surfaces within a single grid cell [S1, S4].
        *   Query $V_{\text{color}}$ for color $\mathbf{c}$ (trilinear interpolation).
        *   Compute rendered color $\hat{C}(\mathbf{r})$ using volume rendering.
        *   Compute loss $\mathcal{L}_{\text{coarse}} = w_{\text{photo}} \mathcal{L}_{\text{photo}} + w_{\text{pt\_rgb}} \mathcal{L}_{\text{pt\_rgb}} + w_{\text{bg}} \mathcal{L}_{\text{bg}}$.
        *   **View-Count Prior:** Apply a lower learning rate to voxels visible to fewer views to avoid redundant voxels allocated for explaining observations from a small number of views [S1, S4].
        *   Update parameters using Adam.

**2. Fine Detail Reconstruction:**
1.  **Scale Grid:** Progressively scale the fine-stage voxel grid resolution (e.g., at checkpoints 1000, 2000, 3000) [S1].
2.  **Initialize Fine Grid:** Create $V_{\text{dens}}$ of size $M_f^3$ (e.g., $160^3$). Initialize raw density values such that activated alpha is $\alpha_{\text{init}}^{(f)} = 0.01$ [S1, S4].
3.  **Define Unknown Space:** Skip low-density query points where $\sigma < \tau_f = 10^{-4}$ [S1].
4.  **Optimize:**
    *   For 20,000 iterations:
        *   Sample rays.
        *   For each ray, sample $K$ points with step size $\delta_f = 0.5 s_f$.
        *   Query $V_{\text{dens}}$ for raw density $\sigma_{\text{raw}}$.
        *   **Post-Activation Interpolation:** Trilinearly interpolate $\sigma_{\text{raw}}$, then apply activation to produce $\alpha$ [S1, S4].
        *   Query $V_{\text{feat}}$ for feature vector $\mathbf{f}$ (trilinear interpolation).
        *   Compute positional encoding for position $\mathbf{x}$ ($k_x=5$) and direction $\mathbf{d}$ ($k_d=4$).
        *   Pass $\mathbf{f}$, pos($\mathbf{x}$), pos($\mathbf{d}$) through the shallow MLP to get view-dependent color $\mathbf{c}$ [S4].
        *   Compute rendered color $\hat{C}(\mathbf{r})$.
        *   Compute loss $\mathcal{L}_{\text{fine}}$.
        *   **View-Count Prior:** Apply lower learning rate to voxels visible to fewer views [S1, S4].
        *   Update parameters using Adam.

**Volume Rendering Equation:**
The rendered color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ is calculated as [S1, S4]:
$$
\hat{C}(\mathbf{r}) = \left( \sum_{i=1}^K T_i \alpha_i \mathbf{c}_i \right) + T_{K+1} \mathbf{c}_{\text{bg}}
$$
where:
*   $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ is the opacity at sample $i$.
*   $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ is the accumulated transmittance up to sample $i$.
*   $\sigma_i$ is the density at sample $i$.
*   $\delta_i$ is the distance to the adjacent sampled point.
*   $\mathbf{c}_i$ is the color at sample $i$.
*   $\mathbf{c}_{\text{bg}}$ is the background color.

**Loss Function:**
The photometric loss is [S4]:
$$
\mathcal{L}_{\text{photo}} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \left\| \hat{C}(\mathbf{r}) - C(\mathbf{r}) \right\|_2^2
$$
where $\mathcal{R}$ is the set of rays in the mini-batch and $C(\mathbf{r})$ is the observed pixel color.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class ShallowMLP(nn.Module):
    def __init__(self, d_feat=12, k_x=5, k_d=4):
        super().__init__()
        # Positional encoding dimensions
        self.dim_x = 3 * (2 * k_x) + 3  # 3 coords * 2k + 3 original
        self.dim_d = 3 * (2 * k_d) + 3
        self.dim_feat = d_feat
        self.dim_in = self.dim_x + self.dim_d + self.dim_feat
        
        # 2 hidden layers, 128 channels
        self.fc1 = nn.Linear(self.dim_in, 128)
        self.fc2 = nn.Linear(128, 128)
        self.fc3 = nn.Linear(128, 3)  # RGB output
        
    def forward(self, x, d, feat):
        # x: (N, 3), d: (N, 3), feat: (N, D)
        pe_x = positional_encoding(x, k=self.k_x)
        pe_d = positional_encoding(d, k=self.k_d)
        input_vec = torch.cat([pe_x, pe_d, feat], dim=-1)
        
        h = F.relu(self.fc1(input_vec))
        h = F.relu(self.fc2(h))
        color = self.fc3(h)
        return color

def positional_encoding(x, k):
    # Standard positional encoding for 3D vector x
    # x: (N, 3) -> (N, 3 * 2k + 3)
    # Implementation details omitted for brevity, assumed standard
    pass

def trilinear_interpolate(grid, coords):
    # grid: (M, M, M, C)
    # coords: (N, 3) normalized to [-1, 1] or [0, 1] depending on grid layout
    # Returns: (N, C)
    # Using F.grid_sample or manual interpolation
    pass

def post_activation_density(raw_density):
    # Apply activation (e.g., Softplus) to raw density
    return F.softplus(raw_density)

def volume_render(rays, V_dens, V_feat, mlp, bg_color, alpha_init, tau, step_size):
    # rays: (N_rays, 2, 3) [origin, direction]
    # V_dens: (M, M, M)
    # V_feat: (M, M, M, D)
    
    N_rays = rays.shape[0]
    K = 128  # Number of samples per ray
    origins = rays[:, 0]
    directions = rays[:, 1]
    
    # Sample points along rays
    t_vals = torch.linspace(0, 1, K).to(rays.device)
    # Adjust t_vals based on near/far planes of the ray and BBox
    # For simplicity, assume uniform sampling within BBox bounds
    # In practice, compute intersection with BBox
    
    points = origins.unsqueeze(1) + directions.unsqueeze(1) * t_vals.unsqueeze(2)
    # points: (N_rays, K, 3)
    
    # Query Density Grid
    # Normalize points to grid coordinates if necessary
    raw_densities = trilinear_interpolate(V_dens, points) # (N_rays, K)
    
    # Post-activation
    densities = post_activation_density(raw_densities)
    
    # Compute alpha
    # alpha = 1 - exp(-sigma * delta)
    # delta is step_size along the ray
    deltas = step_size * torch.norm(directions, dim=1, keepdim=True) # (N_rays, 1)
    alphas = 1.0 - torch.exp(-densities * deltas)
    
    # Transmittance
    # T_i = prod_{j=1}^{i-1} (1 - alpha_j)
    # Use cumulative product
    one_minus_alpha = 1.0 - alphas
    T = torch.cumprod(one_minus_alpha, dim=1)
    # T[:, 0] is 1.0 (transmittance before first sample)
    # We need T_i for each sample i
    # T_i in formula is product up to i-1
    # So T[:, i-1] corresponds to T_i for sample i
    # Let's align: T_accum[i] = prod_{j=0}^{i-1} (1-alpha[j])
    # The formula uses T_i = prod_{j=1}^{i-1} (1-alpha_j)
    # So for sample i (0-indexed), T_i = T_accum[i]
    
    # Query Feature Grid
    features = trilinear_interpolate(V_feat, points) # (N_rays, K, D)
    
    # Get directions for each sample (same for all samples on a ray)
    ray_dirs = directions.unsqueeze(1).expand(-1, K, -1) # (N_rays, K, 3)
    
    # Compute colors using MLP
    colors = mlp(points, ray_dirs, features) # (N_rays, K, 3)
    
    # Volume Rendering
    # C = sum(T_i * alpha_i * c_i) + T_{K+1} * c_bg
    # weights = T * alpha
    weights = T * alphas
    # T_{K+1} is the transmittance after the last sample
    T_last = torch.cat([T, torch.ones_like(T[:, -1:])], dim=1)[:, -1]
    
    color_rendered = torch.sum(weights.unsqueeze(-1) * colors, dim=1) + T_last.unsqueeze(-1) * bg_color
    
    return color_rendered

def optimize_dvgo(images, poses, V_dens_c, V_feat_c, V_color_c, V_dens_f, V_feat_f, mlp):
    # Coarse Stage
    # Initialize V_dens_c to alpha_init_c = 1e-6
    # Initialize V_color_c and V_feat_c to 0
    
    for i in range(10000):
        # Sample rays from images
        rays = sample_rays(images, poses)
        
        # Render
        color_coarse = volume_render(rays, V_dens_c, V_color_c, bg_color=0.0, alpha_init=1e-6, tau=1e-3, step_size=0.5*s_c)
        
        # Loss
        loss = photometric_loss(color_coarse, images)
        
        # Backprop and update
        # Apply view-count prior to learning rate
        optimizer_coarse.step()
        
    # Fine Stage
    # Initialize V_dens_f to alpha_init_f = 0.01
    # V_feat_f is optimized
    
    for i in range(20000):
        # Sample rays
        rays = sample_rays(images, poses)
        
        # Render
        color_fine = volume_render(rays, V_dens_f, V_feat_f, mlp, bg_color=0.0, alpha_init=0.01, tau=1e-4, step_size=0.5*s_f)
        
        # Loss
        loss = photometric_loss(color_fine, images)
        
        # Backprop and update
        # Apply view-count prior
        optimizer_fine.step()
```

## Subtopics

### [Ref-DVGO](6572-ref-dvgo.md)
Ref-DVGO is a reflection-aware neural rendering method that combines the fast hybrid implicit-explicit representation of Direct Voxel Grid Optimization (DVGO) with the reflection direction reparameterization of Ref-NeRF to achieve high-quality photorealistic novel view synthesis of reflective scenes with reduced training and rendering time. It sets itself apart from DVGO by explicitly modeling specular reflections via a reflection-aware reparameterization and from Ref-NeRF by replacing large spatial MLPs with six distinct voxel grids, significantly reducing GPU memory requirements and training time.

## Operating characteristics
*   **Time Complexity:** Training is significantly faster than implicit methods. Convergence is about two orders of magnitude faster than NeRF [S1].
*   **Memory Complexity:** Uses dense voxel grids. Memory usage scales with $M^3$. For $M=160$, this is manageable on modern GPUs.
*   **Measured Performance:**
    *   Training time: ~15 minutes on NVIDIA RTX 2080 Ti for $M=160^3$ (31.95 PSNR) [S1].
    *   Training time: ~22 minutes on NVIDIA RTX 2080 Ti for $M=256^3$ (32.80 PSNR) [S1].
    *   Rendering speed: ~45x faster than NeRF (0.64s vs 29s per 800x800 image) [S1].
*   **Strengths:**
    *   Extremely fast convergence from scratch (no pre-training or conversion needed) [S1, S4].
    *   High rendering speed [S1].
    *   Sharp surface modeling via post-activation interpolation [S1, S4].
*   **Limitations:**
    *   Prone to suboptimal geometry (cloud in free space) if priors are not applied [S1].
    *   Real-world data uncertainty (lighting, SfM error) can lead to blurrier surfaces, reducing the benefit of post-activation [S1].
    *   Not designed for unbounded or forward-facing scenes [S1].
*   **When to use:** When fast per-scene training is required and the scene is bounded and inward-facing.
*   **When not to use:** For unbounded scenes, or when cross-scene generalization is needed without per-scene optimization.

## Background
Direct Voxel Grid Optimization was introduced by Cheng Sun, Min Sun, and Hwann-Tzong Chen in 2022 [S1, S4]. It addresses the slow training times of NeRF by directly optimizing explicit voxel grids. The method builds on the idea of explicit volumetric representations but improves upon previous voxel methods by training from scratch and using post-activation interpolation [S1, S4]. It is used for novel view synthesis on benchmarks like Syn-NeRF, Syn-NSVF, BlendedMVS, and Tanks and Temples [S1, S4].

## Sources
[S1] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
[S2] ScanNeRF: a Scalable Benchmark for Neural Radiance Fields, https://doi.org/10.1109/wacv56688.2023.00088, fetched 2026-10-26
[S3] Leveraging Neural Radiance Fields for Large-Scale 3D Reconstruction from Aerial Imagery, https://doi.org/10.3390/rs16244655, fetched 2026-10-26
[S4] Direct Voxel Grid Optimization: Super-Fast Convergence ..., https://openaccess.thecvf.com/content/CVPR2022/papers/Sun_Direct_Voxel_Grid_Optimization_Super-Fast_Convergence_for_Radiance_Fields_Reconstruction_CVPR_2022_paper.pdf, fetched 2026-10-29
