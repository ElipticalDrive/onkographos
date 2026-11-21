# Two-stage training procedure for Relightable 3D Gaussians

Also called: R3DG training, inverse rendering optimization stages

## Overview
This method defines the optimization protocol for Relightable 3D Gaussians (R3DG), a technique that reconstructs relightable scenes from multi-view images by combining standard 3D Gaussian Splatting (3DGS) rendering with physically based rendering (PBR) and BRDF decomposition [S1]. It solves the problem of recovering geometry, material properties (albedo, roughness), and lighting from static multi-view inputs, enabling novel view synthesis and relighting [S1]. It sets itself apart from vanilla 3DGS by introducing normal gradient-based densification in the first stage and PBR-specific loss terms in the second stage, while differing from neural relightable methods by maintaining a discrete point-based representation [S1].

## How it works
The training procedure is divided into two distinct stages. Stage 1 establishes the geometric and photometric foundation using 3DGS settings augmented with normal constraints. Stage 2 refines the material and lighting properties by sampling incident rays for PBR evaluation [S1].

### Inputs and Outputs
**Inputs:**
- Multi-view images of a static scene.
- Camera poses for each input view.
- Initial 3D Gaussian points (typically derived from Structure-from-Motion or similar initialization, though specific initialization details are not provided in the source [G]).

**Outputs:**
- Optimized 3D Gaussian attributes: position, covariance, opacity, spherical harmonics coefficients (for view-dependent color), normal vectors, albedo, roughness, and environment map parameters.

### Data Structures
The method operates on a set of 3D Gaussians. Each Gaussian $i$ maintains attributes including:
- Position $\mathbf{p}_i$
- Covariance matrix $\Sigma_i$ (or rotation/scale representation)
- Opacity $\alpha_i$
- Spherical Harmonics coefficients $\mathbf{c}_i$
- Normal vector $\mathbf{n}_i$
- Albedo $\mathbf{a}_i$
- Roughness $\rho_i$
- Environment map parameters (for lighting)

*Note: The exact memory layout or bit-packing of these attributes is not specified in the sources [S1].*

### Numbered Steps

**Stage 1: Geometry and Normal Optimization**
1.  **Initialize** the 3D Gaussians.
2.  **Iterate** for 30,000 iterations [S1].
3.  **Densification:** Perform normal gradient-based densification. A new Gaussian is split or cloned if the normal gradient magnitude exceeds the threshold $T_n = 2 \times 10^{-9}$ [S1].
4.  **Rendering:** Render the scene from the input camera views using standard 3DGS alpha-blending.
5.  **Loss Calculation:** Compute the total loss $\mathcal{L}$ using Equation (16) [S1]:
    $$ \mathcal{L} = \lambda_1 \mathcal{L}_1 + \lambda_{ssim} \mathcal{L}_{ssim} + \lambda_n \mathcal{L}_n + \lambda_{s,n} \mathcal{L}_{s,n} + \lambda_O \mathcal{L}_O + \lambda_u \mathcal{L}_u $$
    Where:
    - $\mathcal{L}_1$: L1 loss between rendered and ground truth colors.
    - $\mathcal{L}_{ssim}$: SSIM loss.
    - $\mathcal{L}_n$: Normal consistency loss.
    - $\mathcal{L}_{s,n}$: Surface normal smoothness or consistency loss.
    - $\mathcal{L}_O$: Opacity regularization or loss.
    - $\mathcal{L}_u$: Uncertainty or other auxiliary loss.
    *Note: The exact mathematical definitions of $\mathcal{L}_n, \mathcal{L}_{s,n}, \mathcal{L}_O, \mathcal{L}_u$ are not provided in the source text [S1].*
6.  **Update:** Backpropagate and update Gaussian attributes using an optimizer (e.g., Adam) [G].

**Stage 2: PBR and Material Optimization**
1.  **Iterate** for 10,000 iterations [S1].
2.  **Ray Sampling:** For each Gaussian point, sample $N_s = 64$ incident rays to evaluate the PBR model [S1].
3.  **Rendering:** Render the scene using the PBR model, incorporating the sampled lighting and material properties.
4.  **Loss Calculation:** Compute the total loss $\mathcal{L}$ using Equation (17) [S1]:
    $$ \mathcal{L} = \lambda_1 \mathcal{L}_1 + \lambda_{ssim} \mathcal{L}_{ssim} + \lambda_l \mathcal{L}_l + \lambda_{s,b} \mathcal{L}_{s,b} + \lambda_{s,r} \mathcal{L}_{s,r} $$
    Where:
    - $\mathcal{L}_1$: L1 loss between rendered and ground truth colors.
    - $\mathcal{L}_{ssim}$: SSIM loss.
    - $\mathcal{L}_l$: Lighting loss (e.g., consistency with estimated environment map).
    - $\mathcal{L}_{s,b}$: BRDF smoothness or consistency loss for base color/albedo.
    - $\mathcal{L}_{s,r}$: BRDF smoothness or consistency loss for roughness.
    *Note: The exact mathematical definitions of $\mathcal{L}_l, \mathcal{L}_{s,b}, \mathcal{L}_{s,r}$ are not provided in the source text [S1].*
5.  **Update:** Backpropagate and update Gaussian attributes [G].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_r3dg(input_images, camera_poses, initial_gaussians, num_iterations_stage1=30000, num_iterations_stage2=10000):
    """
    Two-stage training procedure for Relightable 3D Gaussians.
    
    Args:
        input_images: List of input views (H, W, 3)
        camera_poses: List of camera poses (4, 4)
        initial_gaussians: Initial set of 3D Gaussians
        num_iterations_stage1: Number of iterations for Stage 1
        num_iterations_stage2: Number of iterations for Stage 2
        
    Returns:
        Optimized Gaussians with PBR attributes
    """
    gaussians = initial_gaussians
    
    # Stage 1: Geometry and Normal Optimization
    for i in range(num_iterations_stage1):
        # Sample a random view from input images
        view_idx = random.randint(0, len(input_images) - 1)
        gt_image = input_images[view_idx]
        pose = camera_poses[view_idx]
        
        # Normal gradient-based densification
        # Check normal gradients for each Gaussian
        for g in gaussians:
            if g.normal_gradient_magnitude > Tn: # Tn = 2e-9
                densify_gaussian(gaussians, g)
        
        # Render scene using standard 3DGS
        rendered_image = render_3dgs(gaussians, pose)
        
        # Compute Stage 1 Loss
        # L = lambda1 * L1 + lambda_ssim * Lssim + lambda_n * Ln + lambda_s,n * Ls,n + lambda_O * LO + lambda_u * Lu
        loss_l1 = compute_l1_loss(rendered_image, gt_image)
        loss_ssim = compute_ssim_loss(rendered_image, gt_image)
        loss_n = compute_normal_loss(gaussians, gt_normals) # Assuming gt_normals available or estimated
        loss_s_n = compute_surface_normal_smoothness(gaussians)
        loss_o = compute_opacity_loss(gaussians)
        loss_u = compute_auxiliary_loss(gaussians)
        
        loss_stage1 = (lambda1 * loss_l1 + 
                       lambda_ssim * loss_ssim + 
                       lambda_n * loss_n + 
                       lambda_s_n * loss_s_n + 
                       lambda_O * loss_o + 
                       lambda_u * loss_u)
        
        # Backpropagate and update
        loss_stage1.backward()
        optimizer.step()
        optimizer.zero_grad()
        
        # Regular 3DGS densification/pruning steps would occur here [G]

    # Stage 2: PBR and Material Optimization
    for i in range(num_iterations_stage2):
        view_idx = random.randint(0, len(input_images) - 1)
        gt_image = input_images[view_idx]
        pose = camera_poses[view_idx]
        
        # Sample incident rays for PBR
        # Ns = 64 rays per Gaussian point
        incident_rays = sample_incident_rays(gaussians, Ns=64)
        
        # Render scene using PBR model
        rendered_pbr_image = render_pbr(gaussians, pose, incident_rays)
        
        # Compute Stage 2 Loss
        # L = lambda1 * L1 + lambda_ssim * Lssim + lambda_l * Ll + lambda_s,b * Ls,b + lambda_s,r * Ls,r
        loss_l1 = compute_l1_loss(rendered_pbr_image, gt_image)
        loss_ssim = compute_ssim_loss(rendered_pbr_image, gt_image)
        loss_l = compute_lighting_loss(gaussians, incident_rays)
        loss_s_b = compute_brdf_smoothness_base(gaussians)
        loss_s_r = compute_brdf_smoothness_roughness(gaussians)
        
        loss_stage2 = (lambda1 * loss_l1 + 
                       lambda_ssim * loss_ssim + 
                       lambda_l * loss_l + 
                       lambda_s_b * loss_s_b + 
                       lambda_s_r * loss_s_r)
        
        # Backpropagate and update
        loss_stage2.backward()
        optimizer.step()
        optimizer.zero_grad()

    return gaussians

# Constants
Tn = 2e-9
Ns = 64
lambda1 = 0.8
lambda_ssim = 0.2
lambda_n = 0.01
lambda_s_n = 0.01
lambda_O = 0.01
lambda_u = 0.01
lambda_l = 0.0001
lambda_s_b = 0.01
lambda_s_r = 0.01
```

### Parameters
- $T_n$: Normal gradient densification threshold, set to $2 \times 10^{-9}$ [S1].
- $N_s$: Number of incident rays sampled per Gaussian point for PBR in Stage 2, set to 64 [S1].
- $\lambda_1$: L1 loss weight, set to 0.8 for both stages [S1].
- $\lambda_{ssim}$: SSIM loss weight, set to 0.2 for both stages [S1].
- $\lambda_n, \lambda_{s,n}, \lambda_O, \lambda_u$: Stage 1 auxiliary loss weights, each set to 0.01 [S1].
- $\lambda_l$: Stage 2 lighting loss weight, set to 0.0001 [S1].
- $\lambda_{s,b}, \lambda_{s,r}$: Stage 2 BRDF loss weights, each set to 0.01 [S1].

### Edge Cases and Pitfalls
- **Large-scale scenes:** The high density of points in large scenes can slow down optimization significantly because rays are sampled and PBR is performed at each point [S1]. This is a limitation of the current pipeline [S1].
- **Static objects only:** The method is designed for static objects; dynamic scenes are not supported [S1].
- **Precision:** The normal gradient threshold $T_n$ is very small ($2 \times 10^{-9}$), requiring careful numerical implementation to avoid noise-induced densification or failure to densify [S1].

## Operating characteristics
- **Time Complexity:** The method requires 40,000 total iterations (30,000 in Stage 1, 10,000 in Stage 2) [S1]. Stage 2 is computationally more expensive per iteration due to ray sampling ($N_s=64$) and PBR evaluation [S1].
- **Memory Complexity:** Memory usage scales with the number of Gaussians. The high point density required for accurate PBR can lead to high memory consumption, especially in large scenes [S1].
- **Measured Performance:** On the Synthetic4Relight dataset, R3DG achieves:
    - Novel View Synthesis (NVS) PSNR: 36.80, SSIM: 0.982, LPIPS: 0.028 [S1].
    - Relighting PSNR: 31.00, SSIM: 0.964, LPIPS: 0.050 [S1].
    - Albedo PSNR: 28.31, SSIM: 0.951, LPIPS: 0.058 [S1].
    - Training time: 0.90 hours [S1].
- **Scaling:** Performance degrades or optimization slows down in large-scale scenes due to point density [S1].
- **Strengths:** Excels in relighting capability compared to point-based methods [S1]. Shows significantly better NVS quality than other relightable methods [S1]. Achieves realistic PBR rendering, smooth normal estimation, and accurate visibility on discrete point clouds [S1].
- **Limitations:** NVS performance slightly trails vanilla 3DGS [S1]. Limited to static objects [S1]. High point density slows optimization in large scenes [S1].
- **When to use:** For inverse rendering and relighting of static objects from multi-view images where high-quality relighting and material recovery are prioritized over raw NVS PSNR [S1].
- **When not to use:** For large-scale dynamic scenes or when vanilla 3DGS NVS performance is the sole metric of interest [S1].

## Background
The method was introduced in the paper "Relightable 3D Gaussians: Realistic Point Cloud Relighting with BRDF Decomposition and Ray Tracing" by J. Gao et al. [S1]. It builds upon the 3D Gaussian Splatting framework [26] and extends it with PBR capabilities. The work was supported by the National Key R&D Program of China and NSFC [S1].

## Sources
[S1] Relightable 3D Gaussians: Realistic Point Cloud Relighting with BRDF Decomposition and Ray Tracing, https://doi.org/10.1007/978-3-031-72995-9_5, fetched 2026-10-26
