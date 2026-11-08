# DNGaussian

Also called: Depth Normalization Regularized Sparse-view 3D Gaussian Radiance Fields

## Overview
DNGaussian is a depth-regularized framework for few-shot novel view synthesis that combines Hard and Soft Depth Regularization with Global-Local Depth Normalization to restore accurate scene geometry under coarse monocular depth supervision while preserving fine-grained color appearance [S1]. It addresses the significant geometry degradation in 3D Gaussian Splatting (3DGS) caused by sparse input views, where Gaussian primitive positioning becomes incorrectly learned [S1]. Unlike NeRF-based methods that interpolate colors to invisible areas, 3DGS-based methods expose black background on empty spaces, allowing DNGaussian to focus on visible geometry reconstruction [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** Sparse set of input images (typically 3 views), camera poses, and intrinsics. Monocular depth maps are predicted from input images using a pre-trained estimator (DPT) [S1].
- **Outputs:** A set of 3D Gaussian primitives (centers $\mu$, scales $s$, rotations $q$, opacities $\alpha$, and color features) representing the radiance field, enabling novel view synthesis [S1].

**Data Structures:**
- **Gaussian Primitives:** Each primitive $i$ is defined by:
    - Center $\mu_i \in \mathbb{R}^3$
    - Scale $s_i \in \mathbb{R}^3$
    - Rotation $q_i \in \mathbb{R}^4$ (quaternion)
    - Opacity $\alpha_i \in [0, 1]$
    - Color features: Stored via a Neural Color Renderer (see below) instead of spherical harmonics [S1].
- **Neural Color Renderer:** A hash encoder (16 levels, resolution range 16 to 512, max size $2^{19}$) followed by a 5-layer MLP with hidden dimension 64 [S1]. During inference, intermediate results are stored, and only the last MLP layers are calculated to merge view direction for acceleration [S1].

**Numbered Steps:**
1. **Initialization:** Initialize a random set of Gaussian primitives using a uniform distribution. This is done instead of using point clouds from COLMAP or other SfM systems due to instability in sparse-view situations [S1].
2. **Monocular Depth Estimation:** Predict monocular depth maps $\tilde{D}$ for all input views using the DPT estimator [S1].
3. **Color Supervision:** Perform standard 3DGS gradient descent for color reconstruction. Replace spherical harmonics with the Neural Color Renderer to prevent overfitting on sparse views [S1]. This module handles primitive duplication and removal based on activity [S1].
4. **Hard Depth Regularization:**
    - Freeze scaling $s$ and rotation $q$.
    - Apply a large opacity value $\tau$ to all Gaussians to render a 'hard depth' dominated by nearest Gaussians [S1].
    - Optimize only center $\mu$ and opacity $\alpha$ to move Gaussians to correct positions [S1].
    - Compute Hard Depth $D_{hard}$ and loss $R_{hard}$ [S1].
5. **Soft Depth Regularization:**
    - Applied after 1,000 iterations for stability [S1].
    - Freeze Gaussian centers $\check{\mu}$ to avoid negative influence from center movement [S1].
    - Tune opacity $\alpha$ to ensure surface completeness and prevent hollowness [S1].
    - Compute Soft Depth $D_{soft}$ and loss $R_{soft}$ [S1].
6. **Depth Normalization:** Apply Global-Local Depth Normalization to the rendered depths ($D_{hard}, D_{soft}$) and monocular depth $\tilde{D}$ before loss calculation [S1].
7. **Loss Calculation:** Compute total loss $L = L_{color} + R_{hard} + R_{soft}$ [S1].
8. **Optimization:** Repeat steps 3-7 for 6,000 iterations [S1].

**Formulas:**
- **Hard Depth Rendering:**
  $$ D_{hard}(x_p) = \sum_{i \in N} \tau (1 - \tau)^{i-1} G^{proj}_i(x_p) ||\mu_i - o||_2 $$
  where $\tau$ is the applied opacity (0.95), $i$ is the ray hit order, $G^{proj}_i$ is the projected 2D Gaussian, $\mu_i$ is the center, and $o$ is the camera center [S1].

- **Soft Depth Rendering:**
  $$ D_{soft}(x_p) = \sum_{i \in N} ||\check{\mu}_i - o||_2 \times \tilde{\alpha}_i \prod_{j=1}^{i-1} (1 - \tilde{\alpha}_j) $$
  where $\check{\mu}$ is the frozen center, $\tilde{\alpha}$ is the opacity, and the product term represents cumulative transparency [S1].

- **Depth Normalization:**
  Global and Local Depth Normalization ($GN, LN$) are applied to rendered depth $D_T$ and monocular depth $\tilde{D}$ [S1]. The specific normalization formulas are not provided in the sources, but the loss uses normalized versions $D^{GN}_T, \tilde{D}^{GN}$ and $D^{LN}_T, \tilde{D}^{LN}$ [S1].

- **Depth Regularization Loss:**
  $$ R_T = L_2(D^{GN}_T, \tilde{D}^{GN}) + \gamma L_2(D^{LN}_T, \tilde{D}^{LN}) $$
  for $T \in \{hard, soft\}$, where $\gamma = 0.1$ [S1].

- **Total Loss:**
  $$ L = L_{color} + R_{hard} + R_{soft} $$
  where $L_{color} = L_1(\hat{I}, I) + \lambda_{LD-SSIM} L_{D-SSIM}(\hat{I}, I)$ [S1].

**Parameters:**
- Opacity for Hard Depth ($\tau$): 0.95 [S1].
- Global Loss Weight ($\gamma$): 0.1 [S1].
- Training Iterations: 6,000 [S1].
- Soft Depth Start Iteration: 1,000 [S1].

**Pitfalls:**
- Decreasing input views causes significant geometry degradation in 3DGS because scene geometry relies on Gaussian primitive positioning, which becomes incorrectly learned under sparse constraints [S1].
- Using COLMAP point clouds for initialization is unstable in sparse-view situations [S1].
- Spherical harmonics can overfit with sparse views; Neural Color Renderer is used to mitigate this [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def DNGaussian_Training(input_images, camera_poses, camera_intrinsics):
    # 1. Initialization
    gaussians = RandomInitializeGaussians(count=100000) # [G] count is typical, not specified in source
    
    # 2. Monocular Depth Estimation
    monocular_depths = {}
    for img in input_images:
        monocular_depths[img.id] = DPT_Estimate(img)
        
    # 3. Neural Color Renderer Setup
    color_renderer = NeuralColorRenderer(
        hash_levels=16, 
        hash_res_range=(16, 512), 
        hash_max_size=2**19, 
        mlp_layers=5, 
        mlp_hidden_dim=64
    )
    
    # 4. Optimization Loop
    for iteration in range(1, 6001):
        for img in input_images:
            # Color Supervision
            rendered_color, rendered_depth, alpha_composite = RenderGaussians(gaussians, img)
            color_loss = L1_Loss(rendered_color, img.color) + LD_SSIM_Loss(rendered_color, img.color)
            
            # Hard Depth Regularization
            # Freeze s, q. Optimize mu, alpha.
            hard_opacity = 0.95
            hard_depth = ComputeHardDepth(gaussians, img, hard_opacity)
            hard_depth_norm, monocular_depth_norm = GlobalLocalNormalize(hard_depth, monocular_depths[img.id])
            hard_loss = L2_Loss(hard_depth_norm, monocular_depth_norm) + 0.1 * L2_Loss(LocalNormalize(hard_depth), LocalNormalize(monocular_depths[img.id]))
            
            # Soft Depth Regularization (after 1000 iterations)
            if iteration > 1000:
                # Freeze mu. Optimize alpha.
                frozen_centers = gaussians.mu.clone().detach()
                soft_depth = ComputeSoftDepth(frozen_centers, gaussians.alpha, img)
                soft_depth_norm, monocular_depth_norm = GlobalLocalNormalize(soft_depth, monocular_depths[img.id])
                soft_loss = L2_Loss(soft_depth_norm, monocular_depth_norm) + 0.1 * L2_Loss(LocalNormalize(soft_depth), LocalNormalize(monocular_depths[img.id]))
            else:
                soft_loss = 0
                
            total_loss = color_loss + hard_loss + soft_loss
            BackpropagateAndOptimize(total_loss, gaussians, color_renderer)
            
            # Primitive Management (from 3DGS)
            gaussians = ManagePrimitives(gaussians, img)
            
    return gaussians

def ComputeHardDepth(gaussians, img, tau):
    # D_hard(x_p) = sum_{i in N} tau * (1 - tau)^(i-1) * G_proj_i(x_p) * ||mu_i - o||_2
    depth = 0
    for ray in RaysFromImage(img):
        sorted_gaussians = SortGaussiansByDistance(ray.origin, gaussians)
        ray_depth = 0
        for i, g in enumerate(sorted_gaussians):
            # G_proj_i is the projected 2D Gaussian contribution
            contribution = ProjectedGaussianContribution(g, ray)
            distance = Norm(g.mu - ray.origin)
            ray_depth += tau * ((1 - tau) ** i) * contribution * distance
        depth[ray.pixel] = ray_depth
    return depth

def ComputeSoftDepth(frozen_centers, alphas, img):
    # D_soft(x_p) = sum_{i in N} ||mu_i - o||_2 * alpha_i * product_{j=1}^{i-1} (1 - alpha_j)
    depth = 0
    for ray in RaysFromImage(img):
        sorted_gaussians = SortGaussiansByDistance(ray.origin, frozen_centers)
        ray_depth = 0
        cumulative_transparency = 1.0
        for i, g in enumerate(sorted_gaussians):
            distance = Norm(g.mu - ray.origin)
            alpha = alphas[g.id]
            ray_depth += distance * alpha * cumulative_transparency
            cumulative_transparency *= (1 - alpha)
        depth[ray.pixel] = ray_depth
    return depth

def GlobalLocalNormalize(rendered_depth, monocular_depth):
    # Global and Local Depth Normalization details not provided in source [G]
    # Placeholder for normalization logic
    return GlobalNormalize(rendered_depth), GlobalNormalize(monocular_depth), LocalNormalize(rendered_depth), LocalNormalize(monocular_depth)
```

## Subtopics

### Depth Regularization
The parent method family. DNGaussian builds on [Depth Regularization](7616-depth-regularization.md) by applying it to 3D Gaussian primitives with specific freezing strategies.

### 3D Gaussian Splatting Training
The base optimization framework. DNGaussian inherits the gradient descent and primitive management from [3D Gaussian Splatting Training](52-3d-gaussian-splatting-training.md) but replaces spherical harmonics with a Neural Color Renderer.

### Radiance-field Optimization
The broader category of optimizing radiance fields. DNGaussian is a sparse-view variant.

### Differentiable and inverse volume rendering
The underlying rendering technique. DNGaussian uses differentiable volume rendering for depth and color.

### Volume rendering
The foundational technique.

## Operating characteristics
- **Time Complexity**: 25× reduction in training time compared to state-of-the-art sparse-view NeRFs [S1].
- **Memory Complexity**: Significantly lower memory overhead compared to state-of-the-art sparse-view NeRFs [S1].
- **Rendering Speed**: Over 3000× faster rendering speed compared to state-of-the-art sparse-view NeRFs [S1].
- **Strengths**: Restores accurate scene geometry under coarse monocular depth supervision; preserves fine-grained color appearance; efficient training and rendering [S1].
- **Limitations**: Exposes black background on empty spaces (invisible areas), which can lower PSNR/SSIM metrics for those areas compared to NeRFs that interpolate colors [S1].
- **When to use**: Sparse-view novel view synthesis (e.g., 3 views) where geometry accuracy and rendering speed are critical [S1].
- **When not to use**: Dense-view scenarios where spherical harmonics might suffice, or when interpolation of invisible areas is required for metric consistency [S1].

## Background
DNGaussian was proposed by Xiao Bai et al. in 2024 [S1]. It addresses the challenge of novel view synthesis with sparse inputs, where 3D Gaussian Splatting suffers from geometry degradation due to incorrect Gaussian primitive positioning [S1]. The method combines insights from depth-regularized sparse-view NeRFs [36, 42] with the efficiency of 3D Gaussian Splatting [18]. It was published in CVPR 2024 [S1].

## Sources
[S1] DNGaussian: Optimizing Sparse-View 3D Gaussian Radiance Fields with Global-Local Depth Normalization, https://doi.org/10.1109/cvpr52733.2024.01963, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
