# Direct Voxel Grid Optimization

Also called: DVG, Direct Voxel Density Optimization, Direct Voxel Grid Optimization Implementation, DirectVoxGo (DVGO)

## Overview
Direct Voxel Grid Optimization (DVGO) is a method for reconstructing per-scene radiance fields from calibrated images by directly optimizing a dense 3D voxel grid for volume density and a feature voxel grid for color, achieving convergence in less than 15 minutes on a single GPU [S1][S5, unofficial]. It sets itself apart from implicit NeRF-based methods by eliminating the need for lengthy MLP training and from previous voxel methods by training the grid directly from scratch without a conversion step from an implicit model [S1][S5, unofficial]. Unlike [Explicit Radiance Fields](806-explicit-radiance-fields.md) which often rely on pre-trained implicit models or complex hierarchical structures, DVGO uses a simple dense grid with post-activation interpolation to model sharp surfaces efficiently [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ calibrated images with known camera poses (intrinsic and extrinsic parameters) [S1, S6].
*   **Outputs:** A rendered novel view image $\hat{C}(\mathbf{r})$ for any camera ray $\mathbf{r}$, and optimized voxel grid parameters [S1, S6].

**Data Structures:**
The representation consists of three primary components stored in 3D grids:
1.  **Density Voxel Grid ($V^{(\sigma)}$):** Stores raw density values $\sigma$ [S1, S6].
    *   **Coarse Stage:** Resolution $M^{(c)} \times M^{(c)} \times M^{(c)}$, typically $100^3$ [S1, S6].
    *   **Fine Stage:** Resolution $M^{(f)} \times M^{(f)} \times M^{(f)}$, typically $160^3$ or $256^3$ [S1, S6].
2.  **Feature Voxel Grid ($V^{(\text{feat})}$):** Stores feature vectors used by the shallow MLP [S1, S6].
    *   **Fine Stage:** Resolution $M^{(f)} \times M^{(f)} \times M^{(f)}$ [S1, S6].
    *   **Channel Depth:** $D = 12$ channels [S1, S6].
3.  **Color Voxel Grid ($V^{(\text{color})}$):** (Used only in Coarse Stage) Stores view-invariant color [S4].
    *   **Coarse Stage:** Resolution $M^{(c)} \times M^{(c)} \times M^{(c)}$ [S4].

**Bounding Box (BBox):**
The method automatically finds a BBox tightly enclosing the volume of interest to allocate voxel grids, ensuring parsimony [S1, S6]. Coordinates are normalized to $[-1, 1]$ within this BBox [S4].

**Shallow MLP Architecture:**
A shallow Multi-Layer Perceptron maps features to view-dependent color [S1, S6].
*   **Input:** Concatenation of interpolated features from $V^{(\text{feat})}$ and positional embeddings of position $\mathbf{x}$ and viewing direction $\mathbf{d}$ [S1, S6].
*   **Structure:** 2 hidden layers, 128 channels per layer [S6].
*   **Output:** RGB color $\mathbf{c}$ [S1, S6].

**Positional Embedding:**
*   Position $\mathbf{x}$: $k_x = 5$ frequencies [S1, S6].
*   Direction $\mathbf{d}$: $k_d = 4$ frequencies [S1, S6].

**Rendering Model:**
For a ray $\mathbf{r}$, $K$ points are sampled between near and far planes [S1, S6]. The rendered color is calculated using volume rendering quadrature [S1, S6]:

$$
\hat{C}(\mathbf{r}) = \left( \sum_{i=1}^K T_i \alpha_i \mathbf{c}_i \right) + T_{K+1} \mathbf{c}_{\text{bg}}
$$

Where:
*   $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ is the opacity at point $i$ [S1, S6].
*   $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ is the accumulated transmittance [S1, S6].
*   $\sigma_i$ is the density at point $i$ [S1, S6].
*   $\delta_i$ is the distance to the adjacent sampled point [S1, S6].
*   $\mathbf{c}_i$ is the color at point $i$ [S1, S6].
*   $\mathbf{c}_{\text{bg}}$ is the background color [S1, S6].

**Optimization Steps:**
The optimization is divided into two phases: Coarse Geometry Searching and Fine Detail Reconstruction [S4].

1.  **Initialization:**
    *   Initialize density voxel grid to yield opacities very close to zero everywhere to avoid bias toward camera near planes [S1, S6].
    *   Coarse stage initial activated alpha: $\alpha_{\text{init}}^{(c)} = 10^{-6}$ [S1, S6].
    *   Fine stage initial activated alpha: $\alpha_{\text{init}}^{(f)} = 0.01$ [S1, S6].

2.  **Coarse Geometry Searching:**
    *   **Grids:** Use $V^{(\sigma)}$ (coarse) and $V^{(\text{color})}$ (coarse) [S4].
    *   **Sampling:** Step size $\delta^{(c)} = 0.5 \times s^{(c)}$, where $s^{(c)}$ is the voxel size [S6].
    *   **Free Space Handling:** Define known free space using threshold $\tau_c = 10^{-3}$. Skip query points where density is below this threshold [S1].
    *   **Learning Rate Prior:** Apply a lower learning rate to voxels visible to fewer views to avoid redundant voxels allocated just for explaining observations from a small number of views [S1, S6].
    *   **Training:** Optimize for 10,000 iterations using Adam optimizer [S1, S6].
    *   **Loss:** Photometric loss $\mathcal{L}_{\text{photo}}$ with weight $w_{\text{photo}} = 1$. Point-wise RGB loss weight $w_{\text{pt\_rgb}} = 10^{-1}$. Background loss weight $w_{\text{bg}} = 10^{-2}$ [S1].
    *   **Learning Rates:** Base learning rate for voxel grids is 0.1. Base learning rate for shallow MLP is $10^{-3}$ [S6].

3.  **Fine Detail Reconstruction:**
    *   **Grids:** Use $V^{(\sigma)}$ (fine) and $V^{(\text{feat})}$ (fine) [S4].
    *   **Sampling:** Step size $\delta^{(f)} = 0.5 \times s^{(f)}$, where $s^{(f)}$ is the voxel size [S6].
    *   **Unknown Space Handling:** Skip low-density query points in unknown space using threshold $\tau_f = 10^{-4}$ [S1].
    *   **Progressive Scaling:** Progressively scale the fine-stage voxel grid at checkpoints 1000, 2000, and 3000 iterations [S1].
    *   **Training:** Optimize for 20,000 iterations using Adam optimizer [S1, S6].
    *   **Loss:** Photometric loss $\mathcal{L}_{\text{photo}}$ with weight $w_{\text{photo}} = 1$. Point-wise RGB loss weight $w_{\text{pt\_rgb}} = 10^{-2}$. Background loss weight $w_{\text{bg}} = 10^{-3}$ [S1].
    *   **Learning Rates:** Base learning rate for voxel grids is 0.1. Base learning rate for shallow MLP is $10^{-3}$ [S6].
    *   **Schedule:** Apply exponential learning rate decay, downscaled by 0.1 at 20,000 iterations [S1].

**Post-Activation Interpolation:**
A key technique for sharp surface modeling. Instead of interpolating activated opacity or using nearest-neighbor, DVGO trilinearly interpolates the raw density values from the grid, then applies the activation function (e.g., Softplus or ReLU) to produce opacity [S1, S6]. This allows modeling sharp linear surfaces within a single grid cell [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def dvgo_optimize(images, poses, near_far, batch_size=8192):
    # 1. Setup BBox and Grids
    bbox = find_bounding_box(poses)
    M_c, M_f = 100, 160
    D_feat = 12
    
    # Initialize grids
    V_sigma_c = np.zeros((M_c, M_c, M_c))
    V_color_c = np.zeros((M_c, M_c, M_c, 3))
    V_sigma_f = np.zeros((M_f, M_f, M_f))
    V_feat_f = np.zeros((M_f, M_f, M_f, D_feat))
    
    # Initialize density to low opacity
    V_sigma_c[:] = low_density_init(alpha_init=1e-6, M=M_c, bbox=bbox)
    V_sigma_f[:] = low_density_init(alpha_init=0.01, M=M_f, bbox=bbox)
    
    # Optimizers
    opt_sigma_c = Adam(lr=0.1)
    opt_color_c = Adam(lr=0.1)
    opt_sigma_f = Adam(lr=0.1)
    opt_feat_f = Adam(lr=0.1)
    opt_mlp = Adam(lr=1e-3)
    
    # Shallow MLP for view-dependent color
    mlp = ShallowMLP(hidden_dim=128, num_layers=2, k_x=5, k_d=4)
    
    # 2. Coarse Stage
    for iter in range(10000):
        rays = sample_rays(images, poses, batch_size)
        loss = 0.0
        
        for ray in rays:
            points, deltas = sample_points_along_ray(ray, near_far, step_size=0.5 * (bbox.size / M_c))
            
            # Query density
            raw_sigma = trilinear_interpolate(V_sigma_c, points)
            sigma = activation(raw_sigma)
            
            # Query color (view-invariant in coarse stage)
            color = trilinear_interpolate(V_color_c, points)
            
            # Volume rendering for this ray
            alpha = 1 - np.exp(-sigma * deltas)
            T = np.concatenate([np.ones((1,)), np.cumprod(1 - alpha)[:-1]])
            rendered_color = np.sum(T * alpha * color, axis=0) + T[-1] * bg_color
            
            # Loss calculation
            target_color = get_pixel_color(ray)
            loss += np.linalg.norm(rendered_color - target_color)**2
            
            # Gradient accumulation for voxels
            view_count = count_views_visible_to_voxel(ray, points)
            lr_scale = 1.0 if view_count > 1 else 0.1
            grad_sigma = compute_density_gradient(loss, sigma, deltas, T, color)
            grad_color = compute_color_gradient(loss, T, alpha, color)
            opt_sigma_c.step(V_sigma_c, points, grad_sigma, lr_scale)
            opt_color_c.step(V_color_c, points, grad_color, lr_scale)
            
        opt_sigma_c.step()
        opt_color_c.step()
        
    # 3. Fine Stage
    for iter in range(20000):
        if iter in [1000, 2000, 3000]:
            V_sigma_f = scale_up_grid(V_sigma_f, M_f)
            V_feat_f = scale_up_grid(V_feat_f, M_f)
            
        rays = sample_rays(images, poses, batch_size)
        loss = 0.0
        
        for ray in rays:
            points, deltas = sample_points_along_ray(ray, near_far, step_size=0.5 * (bbox.size / M_f))
            
            # Filter points in known free space
            raw_sigma = trilinear_interpolate(V_sigma_f, points)
            sigma = activation(raw_sigma)
            valid_mask = sigma > 1e-4
            
            if not np.any(valid_mask):
                continue
                
            valid_points = points[valid_mask]
            valid_deltas = deltas[valid_mask]
            valid_sigma = sigma[valid_mask]
            
            # Query features
            features = trilinear_interpolate(V_feat_f, valid_points)
            
            # Query color via MLP (view-dependent)
            pos_embed = positional_encoding(valid_points, k=5)
            dir_embed = positional_encoding(ray.direction, k=4)
            color = mlp(features, pos_embed, dir_embed)
            
            # Volume rendering
            alpha = 1 - np.exp(-valid_sigma * valid_deltas)
            T = np.concatenate([np.ones((1,)), np.cumprod(1 - alpha)[:-1]])
            rendered_color = np.sum(T * alpha * color, axis=0) + T[-1] * bg_color
            
            target_color = get_pixel_color(ray)
            loss += np.linalg.norm(rendered_color - target_color)**2
            
            # Gradients
            grad_sigma = compute_density_gradient(loss, valid_sigma, valid_deltas, T, color)
            grad_feat = compute_feature_gradient(loss, mlp, features, pos_embed, dir_embed, T, alpha, color)
            opt_sigma_f.step(V_sigma_f, valid_points, grad_sigma)
            opt_feat_f.step(V_feat_f, valid_points, grad_feat)
            opt_mlp.step(mlp.params, valid_points, grad_feat)
            
        opt_sigma_f.step()
        opt_feat_f.step()
        opt_mlp.step()
        
        # Exponential decay at 20000 iterations
        if iter == 20000:
            opt_sigma_f.lr *= 0.1
            opt_feat_f.lr *= 0.1
            opt_mlp.lr *= 0.1

def low_density_init(alpha_init, M, bbox):
    delta = bbox.size / M
    sigma = -np.log(1 - alpha_init) / delta
    return np.full((M, M, M), sigma)

def trilinear_interpolate(grid, points):
    # points: (K, 3) in [-1, 1]
    pts = (points + 1) / 2 * (grid.shape[0] - 1)
    pts = np.clip(pts, 0, grid.shape[0] - 1)
    x0, y0, z0 = np.floor(pts).astype(int)
    x1, y1, z1 = np.minimum(x0 + 1, grid.shape[0] - 1), np.minimum(y0 + 1, grid.shape[1] - 1), np.minimum(z0 + 1, grid.shape[2] - 1)
    dx, dy, dz = pts - np.array([x0, y0, z0]).T
    q00 = grid[x0, y0, z0] * (1-dx) + grid[x1, y0, z0] * dx
    q10 = grid[x0, y1, z0] * (1-dx) + grid[x1, y1, z0] * dx
    q01 = grid[x0, y0, z1] * (1-dx) + grid[x1, y0, z1] * dx
    q11 = grid[x0, y1, z1] * (1-dx) + grid[x1, y1, z1] * dx
    return q00 * (1-dy) + q10 * dy, q01 * (1-dy) + q11 * dy # Simplified for brevity, returns interpolated values

def activation(x):
    return np.maximum(0, x) # ReLU/Softplus approximation

def positional_encoding(x, k):
    freqs = 2**np.arange(k) * np.pi
    return np.concatenate([np.sin(x[:, None] * freqs), np.cos(x[:, None] * freqs)], axis=-1)

def ShallowMLP(hidden_dim, num_layers, k_x, k_d):
    # Simplified MLP definition
    class MLP:
        def __init__(self):
            self.W1 = np.random.randn(hidden_dim * (k_x*2 + k_d*2 + 12), hidden_dim)
            self.b1 = np.zeros(hidden_dim)
            self.W2 = np.random.randn(hidden_dim, 3)
            self.b2 = np.zeros(3)
        def forward(self, features, pos, dir):
            x = np.concatenate([features, pos, dir], axis=-1)
            h = activation(x @ self.W1 + self.b1)
            return h @ self.W2 + self.b2
    return MLP()

def find_bounding_box(poses):
    # Returns a simple axis-aligned bounding box from camera centers
    centers = poses[:, :3, 3]
    return type('BBox', (), {'min': centers.min(axis=0), 'max': centers.max(axis=0), 'size': np.linalg.norm(centers.max(axis=0) - centers.min(axis=0))})()

def sample_rays(images, poses, batch_size):
    # Returns batch_size rays with origin and direction
    return [type('Ray', (), {'origin': np.array([0,0,-1]), 'direction': np.array([0,0,1])})() for _ in range(batch_size)]

def sample_points_along_ray(ray, near_far, step_size):
    t = np.arange(near_far[0], near_far[1], step_size)
    points = ray.origin + t[:, None] * ray.direction
    deltas = np.full_like(t, step_size)
    return points, deltas

def count_views_visible_to_voxel(ray, points):
    return 1 # Simplified view count

def compute_density_gradient(loss, sigma, deltas, T, color):
    return -loss * deltas * T * color # Simplified gradient direction

def compute_color_gradient(loss, T, alpha, color):
    return -loss * T * alpha # Simplified gradient direction

def compute_feature_gradient(loss, mlp, features, pos, dir, T, alpha, color):
    return -loss * T * alpha * color # Simplified gradient direction

def Adam(lr):
    class Optimizer:
        def __init__(self, lr): self.lr = lr
        def step(self, params, points, grad, scale=1.0):
            params[:] -= self.lr * scale * grad # Simplified update
    return Optimizer(lr)

def scale_up_grid(grid, new_size):
    # Simple nearest-neighbor upscale for progressive scaling
    old_size = grid.shape[0]
    new_grid = np.zeros((new_size, new_size, new_size))
    ratio = new_size / old_size
    for i in range(new_size):
        for j in range(new_size):
            for k in range(new_size):
                new_grid[i,j,k] = grid[int(i/ratio), int(j/ratio), int(k/ratio)]
    return new_grid

def get_pixel_color(ray):
    return np.array([0.5, 0.5, 0.5])

bg_color = np.array([0.0, 0.0, 0.0])
```

## Subtopics

### [Ref-DVGO](6572-ref-dvgo.md)
Ref-DVGO is a reflection-aware neural rendering method that combines the fast hybrid implicit-explicit representation of Direct Voxel Grid Optimization (DVGO) with the reflection direction reparameterization of Ref-NeRF to achieve high-quality photorealistic novel view synthesis of reflective scenes with reduced training and rendering time. It sets itself apart from DVGO by explicitly modeling specular reflections via a reflection-aware reparameterization and from Ref-NeRF by replacing large spatial MLPs with six distinct voxel grids, significantly reducing GPU memory requirements and training time. See [Ref-DVGO](6572-ref-dvgo.md).

### [VGOS](17769-vgos.md)
VGOS is a method for fast radiance field reconstruction from sparse inputs (3-10 views) that addresses overfitting, holes, and floaters in voxel grids by using an incremental voxel training strategy and voxel smoothing regularization. It sets itself apart from pre-training methods like SRF and MVSNeRF by training from scratch without pre-trained models or extra inputs, and from depth-dependent methods like DSNeRF by relying solely on RGB images. See [VGOS](17769-vgos.md).

### Incremental Voxel Training Strategy
A training strategy that prevents voxels from overfitting to training views by suppressing the optimization of peripheral voxels in the early stage of radiance field reconstruction [S7]. It freezes the optimization of peripheral voxels at the early stage of training and gradually thaws them as training progresses [S7]. This prevents voxels close to the cameras' near planes from overfitting to the training views [S7].

## Operating characteristics
*   **Time Complexity:** Training is significantly faster than implicit methods. Convergence is about two orders of magnitude faster than NeRF, reducing training time from 10-20 hours to 15 minutes [S1][S5, unofficial].
*   **Memory Complexity:** Uses dense voxel grids. For $M^{(f)}=160^3$, memory usage is manageable on a single GPU. Higher resolutions ($256^3$) increase memory but improve quality [S1].
*   **Measured Performance:** On NVIDIA RTX 2080 Ti, DVGO achieves 31.95 PSNR in 15 minutes ($160^3$) and 32.80 PSNR in 22 minutes ($256^3$) on inward-facing benchmarks [S1]. Rendering speed is ~45x faster than NeRF (0.64s vs 29s per 800x800 image) [S1].
*   **Strengths:** Extremely fast training from scratch; no need for cross-scene pre-training; high rendering speed; sharp surface modeling via post-activation [S1][S5, unofficial].
*   **Limitations:** Prone to suboptimal geometry (cloud in free space) if priors are not applied; real-world data uncertainty (lighting, SfM error) can lead to blurrier surfaces; limited to bounded scenes; resolution is fixed by grid size, potentially leading to aliasing at very high resolutions compared to adaptive methods [S1][S5, unofficial].
*   **When to Use:** When fast per-scene reconstruction is required and computational resources are limited (single GPU). Suitable for inward-facing scenes with known poses [S1][S5, unofficial].
*   **When Not to Use:** For unbounded or forward-facing scenes (not handled natively); when extreme geometric precision at very high resolutions is needed without hierarchical structures; when real-world data has significant multi-view inconsistency that cannot be mitigated by priors [S1][S5, unofficial].

## Background
Direct Voxel Grid Optimization was introduced by Cheng Sun, Min Sun, and Hwann-Tzong Chen [S1][S5, unofficial]. It addresses the bottleneck of lengthy training times in Neural Radiance Fields (NeRF) [37] by directly optimizing explicit voxel grids. Previous voxel methods often required conversion from trained implicit models or cross-scene pre-training [S1][S5, unofficial]. DVGO demonstrated that direct optimization, combined with post-activation interpolation and specific priors, could achieve NeRF-comparable quality in minutes. It is used on benchmarks such as Syn-NeRF, Syn-NSVF, BlendedMVS, and Tanks and Temples [S1][S5, unofficial].

## Sources
[S1] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
[S2] NeRF-QA: Neural Radiance Fields Quality Assessment Database, https://doi.org/10.1109/qomex58391.2023.10178625, fetched 2026-10-26
[S3] ScanNeRF: a Scalable Benchmark for Neural Radiance Fields, https://doi.org/10.1109/wacv56688.2023.00088, fetched 2026-10-26
[S4] Leveraging Neural Radiance Fields for Large-Scale 3D Reconstruction from Aerial Imagery, https://doi.org/10.3390/rs16244655, fetched 2026-10-26
[S5] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S6] Direct Voxel Grid Optimization: Super-Fast Convergence ..., https://openaccess.thecvf.com/content/CVPR2022/papers/Sun_Direct_Voxel_Grid_Optimization_Super-Fast_Convergence_for_Radiance_Fields_Reconstruction_CVPR_2022_paper.pdf, fetched 2026-10-29
[S7] VGOS: Voxel Grid Optimization for View Synthesis from Sparse Inputs, https://doi.org/10.24963/ijcai.2023/157, fetched 2026-10-26
