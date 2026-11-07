# Optimization and Densification Strategy

Also called: Modified 3DGS Optimization

## Overview
This method adapts the standard 3D Gaussian Splatting (3DGS) densification pipeline to work with alternative rendering backends, specifically those using ray-splat intersection or inversion-free splat evaluation [S1]. It solves the problem of missing screen-space gradients and aliasing artifacts inherent in these alternative rasterization schemes by replacing screen-space gradient metrics with view-space gradients and opacity decay, while integrating visibility-based pruning [S1]. It differs from standard 3DGS by abandoning the opacity reset mechanism in favor of exponential decay and by utilizing view-space rather than image-space gradients for densification decisions [S1].

## How it works
The method modifies the standard densification loop to accommodate rendering pipelines that do not provide per-pixel screen-space gradients or use different blending formulations.

**Inputs and Outputs:**
*   **Input:** Set of Gaussians $\mathcal{G}$, camera parameters, training images, loss gradients $\nabla \mathcal{L}$.
*   **Output:** Updated set of Gaussians $\mathcal{G}'$ with modified positions, scales, opacities, and colors.

**Data Structures:**
*   **Gaussian Buffer:** Stores mean $\mu$, covariance $\Sigma$ (or scale/rotation), opacity $\alpha$, and color coefficients.
*   **Visibility Score:** A metric per Gaussian indicating its contribution to the image, used for pruning [S1].

**Parameters:**
*   $\lambda_o$: Opacity decay multiplier. Set to $0.9995$ [S1].
*   Decay Interval: $50$ iterations [S1].
*   Tile Size: $8 \times 8$ pixels to mitigate divergence in insertion sort for per-pixel core tracking [S1].
*   Core Size $K$: $16$ fragments [S1].
*   Threshold $\tau_K$: $0.05$ (minimum $\alpha$ for core consideration) [S1].

**Steps:**

1.  **Gradient Computation:**
    Compute the gradient of the loss with respect to the splat means. Since screen-space gradients are unavailable in the target rendering scheme, use the gradient of the splat means scaled by half the respective distance to the camera in view space [S1]. Let $\nabla_{\mu} \mathcal{L}$ be the raw gradient. The densification metric $M_i$ for Gaussian $i$ is:
    $$ M_i = \| \nabla_{\mu_i} \mathcal{L} \| \cdot \frac{1}{2} d_i $$
    where $d_i$ is the distance from the Gaussian mean $\mu_i$ to the camera center in view space [S1].

2.  **Opacity Management (Decay):**
    To prevent over-densification, apply exponential decay to opacities instead of resetting them [S1]. Every 50 iterations during densification:
    $$ \alpha_i \leftarrow \alpha_i \cdot \lambda_o $$
    where $\lambda_o = 0.9995$ [S1].

3.  **Pruning:**
    Apply visibility score-based pruning [S1]. Gaussians with a visibility score below a certain threshold (adapted from Niemeyer et al. [NMR*25]) are removed from the scene [S1].

4.  **Densification (Splitting/Cloning):**
    Adapt the densification changes from GOF [YSG*24] to the ray-splat intersection approach [S1].
    *   **Splitting:** Select Gaussians with high metric $M_i$. Replace each with two new Gaussians. The new means are derived from the original mean plus/minus a perturbation. The new scales are reduced (typically divided by a factor, e.g., 1.6, though exact factor depends on the specific GOF adaptation) [S1].
    *   **Cloning:** Select Gaussians with high metric $M_i$ but low gradient magnitude (to fill gaps). Clone the Gaussian with a small random perturbation to its mean [S1].

5.  **Rendering Pipeline Adjustments:**
    *   Use a fast CUDA implementation of the 3D filter by Yu et al. [YCH*24] to tackle aliasing, as the inversion-free splat evaluation prevents the use of the screen-space low-pass filter in EWA splatting [S1].
    *   Reduce tile size to $8 \times 8$ to mitigate divergence caused by insertion sort for per-pixel core tracking [S1].
    *   Use an optimized SSIM implementation [MGK*24] for loss computation [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_and_densify_step(gaussians, cameras, images, step):
    """
    One step of optimization and densification for Modified 3DGS.
    """
    # 1. Render and Compute Loss
    # Use ray-splat intersection or inversion-free splat evaluation
    rendered_images, alpha, meta_data = rasterize(gaussians, cameras)
    loss = compute_loss(rendered_images, images) # Using optimized SSIM [S1]
    
    # 2. Backpropagation
    loss.backward()
    
    # 3. Gradient Processing for Densification
    # Compute view-space gradient metric
    # M_i = || grad_mu_i || * 0.5 * distance_to_camera_view_space
    view_distances = compute_view_space_distances(gaussians.means, cameras)
    grad_magnitudes = torch.norm(gaussians.grad_means, dim=1)
    densification_metric = grad_magnitudes * 0.5 * view_distances
    
    # 4. Opacity Decay (Every 50 steps)
    if step % 50 == 0:
        gaussians.opacity = gaussians.opacity * 0.9995 # lambda_o [S1]
        
    # 5. Densification Trigger (e.g., every N steps, after warmup)
    if should_densify(step):
        # Splitting and Cloning based on GOF adaptation [S1]
        split_candidates = select_candidates(densification_metric, mode='split')
        clone_candidates = select_candidates(densification_metric, mode='clone')
        
        new_gaussians = split_and_clone(gaussians, split_candidates, clone_candidates)
        
        # 6. Pruning based on Visibility Score [S1]
        visibility_scores = compute_visibility_scores(new_gaussians, cameras)
        pruned_gaussians = remove_low_visibility(new_gaussians, visibility_scores, threshold=0.01) # Threshold adapted from [NMR*25]
        
        gaussians = pruned_gaussians
        
    # 7. Optimizer Step
    optimizer.step()
    optimizer.zero_grad()
    
    return gaussians
```

## Subtopics

### Anisotropy Regularization
This variant removes the "needle surgery" structural splitting and training calls, keeping only standard densification and pruning [S2, unofficial]. It introduces a log-ratio soft regularization to control anisotropy. Parameters include `lambda_aniso` (default 0.001) and `aniso_ratio_max` (default 5) [S2, unofficial].

### GaussianDreamer
A method combining 2D and 3D diffusion model priors for 3DGS optimization [S3]. It utilizes Shap-E [245] to generate the initial point cloud, which is relatively sparse [S3]. 3DGS is optimized using 2D SDS [S3]. To densify the sparse initial cloud, it proposes noisy point growth and color perturbation [S3].

### MCMC Densification
A Bayesian densification strategy that reformulates Gaussian Splatting densification as a Stochastic Gradient Langevin Dynamic (SGLD) update rule [S4, unofficial]. It adds noise to center locations $\mu$ of Gaussians. The update rule is:
$$ \mathcal{G} \leftarrow \mathcal{G} - \lambda_{\text{lr}} \cdot \nabla_{\mathcal{G}} \mathbf{E}_{\mathcal{I}} [\mathcal{L}_{\text{total}} (\mathcal{G}; \mathbf{I})] + \lambda_{\text{noise}} \cdot \epsilon $$
where $\lambda_{\text{lr}}$ is the learning rate, $\lambda_{\text{noise}}$ is the noise scale, and $\epsilon$ is the noise term [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Training speedup of $2.1\times$ compared to standard 3DGS [S1]. Rendering is $1.6\times$ faster on average [S1].
*   **Memory Complexity:** Uses a core size $K=16$ and tile size $8 \times 8$, which may increase memory overhead for tile management but reduces divergence [S1].
*   **Strengths:** Faster training and rendering than standard 3DGS; handles aliasing better in inversion-free splatting; prevents over-densification via opacity decay [S1].
*   **Limitations:** Image quality is comparable to 3DGS but outperformed by Zip-NeRF [S1]. Requires careful tuning of tile size and core thresholds to avoid divergence [S1].
*   **When to use:** When using alternative rendering backends (e.g., ray-splat intersection) that lack screen-space gradients or use different blending [S1].

## Background
Originated from modifications to the standard 3DGS pipeline to support efficient perspective-correct rendering [S1]. It incorporates techniques from GOF [YSG*24] for densification and Niemeyer et al. [NMR*25] for pruning. The opacity decay strategy is adapted from StopThePop [RSP*24] [S1].

## Sources
[S1] Efficient Perspective‐Correct 3D Gaussian Splatting Using Hybrid Transparency, https://doi.org/10.1111/cgf.70014, fetched 2026-10-30
[S2] hsiang0117/4DGS-Volume-Cloud, https://github.com/hsiang0117/4DGS-Volume-Cloud, fetched 2026-10-26 (unofficial)
[S3] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S4] gsplat: An Open-Source Library for Gaussian Splatting, https://doi.org/10.48550/arXiv.2409.06765, fetched 2026-10-29 (unofficial)
