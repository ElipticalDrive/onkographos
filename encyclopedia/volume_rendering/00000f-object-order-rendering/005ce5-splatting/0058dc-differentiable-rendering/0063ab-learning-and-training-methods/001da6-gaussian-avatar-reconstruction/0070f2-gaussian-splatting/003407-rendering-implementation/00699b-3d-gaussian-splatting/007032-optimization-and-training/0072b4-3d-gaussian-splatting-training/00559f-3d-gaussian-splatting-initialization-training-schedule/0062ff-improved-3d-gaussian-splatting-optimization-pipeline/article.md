# Improved 3D Gaussian Splatting Optimization Pipeline

Also called: DS-DST optimization pipeline, accelerated 3DGS training

## Overview
This method defines a structured training schedule for 3D Gaussian Splatting that combines multi-scale rendering, adaptive densification, and optimized spherical harmonics evaluation to improve reconstruction quality and convergence speed. It addresses the slow convergence and high computational cost of standard 3D Gaussian Splatting by introducing a resolution-switching strategy and specific hyperparameter tuning for the learning rate and densification thresholds. It builds on [3D Gaussian Splatting Initialization & Training Schedule](15507-3d-gaussian-splatting-initialization-training-schedule.md) by refining the optimization dynamics after initialization.

## How it works
**Inputs:**
- Training images $\{I\}$
- Camera parameters $\{C\}$
- Initial Gaussian set $\{G\}$ (positions $\mu$, covariances $\Sigma$, colors $C$, opacity $\alpha$, SH coefficients)

**Outputs:**
- Optimized Gaussian model $\{G^*\}$

**Data Structures:**
- Gaussian $G_i$: Contains position $\mu_i \in \mathbb{R}^3$, covariance $\Sigma_i \in \mathbb{R}^{3 \times 3}$ (or quaternion/scale representation), opacity $\alpha_i \in [0,1]$, spherical harmonics coefficients $c_i$, and color $C_i$.
- Loss weights: $\lambda_1, \lambda_2, \lambda_3, \lambda_4$ for L1, SSIM, VGG, and LPIPS respectively.
- Thresholds: Opacity threshold $\tau_\alpha = 0.005$, gradient threshold $\tau_g$, size threshold $\tau_S$.

**Parameters:**
- Total iterations: $T_{total} = 30,000$ [S1].
- Resolution switch point: $T_{switch} = 5,000$ iterations [S1].
- Low resolution: $R_{low} = (540, 960)$ [S1].
- High resolution: $R_{high} = (1080, 1920)$ [S1].
- Optimizer: Adam [S1].
- Learning Rate Schedule: CosineAnnealingLR [S1].

**Algorithm Steps:**
1. Initialize Gaussians $G$ with positions $\mu$, covariances $\Sigma$, colors $C$, opacity $\alpha$, and SH coefficients.
2. Set initial learning rate $LR_{initial}$ and configure CosineAnnealingLR schedule.
3. For iteration $t = 1$ to $30,000$:
    a. Determine resolution $R_t$:
       - If $t < 5,000$, $R_t = (540, 960)$.
       - Else, $R_t = (1080, 1920)$.
    b. Render training images at resolution $R_t$ using current Gaussian parameters.
    c. Compute loss:
       $$ L = \lambda_1 L_1 + \lambda_2 (1 - \text{SSIM}) + \lambda_3 \text{VGG} + \lambda_4 \text{LPIPS} $$
       where $L_1$ is the L1 loss, SSIM is Structural Similarity Index, VGG is perceptual loss, and LPIPS is Learned Perceptual Image Patch Similarity [S1].
    d. Compute gradients $\nabla L$ with respect to Gaussian parameters.
    e. Update parameters using Adam optimizer with learning rate from CosineAnnealingLR.
    f. **Adaptive Pruning:** For each Gaussian $G_i$:
       - If $\alpha_i < 0.005$ or the size of $\Sigma_i$ exceeds a threshold, remove $G_i$ from $G$.
    g. **Improved Densification:** For each Gaussian $G_i$:
       - If gradient magnitude $\|\nabla L(G_i)\| > \tau_g$:
         - If size of $\Sigma_i > \tau_S$:
           - Split $G_i$ into $G_{new}$ with $\Sigma_{new} = 0.8 \times \Sigma_i$.
         - Else:
           - Clone $G_{new}$ at $\mu_i + \text{noise}(\sigma)$.
         - Add $G_{new}$ to $G$.
    h. **Optimized Spherical Harmonics:** Compute SH-to-RGB conversion outside the rasterizer for each $G_i$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def improved_3dgs_optimization(Images, Cameras, Initial_Gaussians):
    G = Initial_Gaussians
    LR_initial = 0.0001  # [G] Typical Adam LR for 3DGS
    T_total = 30000
    T_switch = 5000
    R_low = (540, 960)
    R_high = (1080, 1920)
    alpha_threshold = 0.005
    
    # Configure Adam and CosineAnnealingLR
    optimizer = Adam(G.parameters(), lr=LR_initial)
    scheduler = CosineAnnealingLR(optimizer, T_max=T_total)
    
    for t in range(1, T_total + 1):
        # 1. Resolution Scheduling
        if t < T_switch:
            current_res = R_low
        else:
            current_res = R_high
            
        # 2. Rendering and Loss Computation
        rendered_images = render(G, Cameras, current_res)
        loss = compute_loss(rendered_images, Images)
        # Loss = lambda1 * L1 + lambda2 * (1 - SSIM) + lambda3 * VGG + lambda4 * LPIPS
        
        # 3. Optimization Step
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        scheduler.step()
        
        # 4. Adaptive Pruning
        gaussians_to_remove = []
        for i, g in enumerate(G):
            if g.opacity < alpha_threshold or get_size(g.covariance) > size_threshold:
                gaussians_to_remove.append(i)
        for i in reversed(gaussians_to_remove):
            G.pop(i)
            
        # 5. Improved Densification
        gaussians_to_add = []
        for i, g in enumerate(G):
            grad_norm = get_gradient_norm(g)
            if grad_norm > gradient_threshold:
                if get_size(g.covariance) > size_threshold:
                    # Split
                    new_g = split_gaussian(g, scale_factor=0.8)
                    gaussians_to_add.append(new_g)
                else:
                    # Clone
                    noise = random_noise(sigma=0.01) # [G] Typical noise scale
                    new_g = clone_gaussian(g, offset=noise)
                    gaussians_to_add.append(new_g)
        G.extend(gaussians_to_add)
        
        # 6. Optimized SH Evaluation
        # Compute SH-to-RGB outside rasterizer
        for g in G:
            g.color = evaluate_sh(g.sh_coeffs, g.direction)
            
    return G
```

## Subtopics

### Accelerated Position Learning Rate Schedule
A variant that reduces total training iterations to 3,000 (from 30,000) to achieve up to 23x faster training [S2]. It requires explicitly tuning the maximum step count of the position learning rate scheduler to match the 3k budget. Applying the original 30k schedule to this reduced budget keeps the learning rate too high, preventing convergence and degrading quality metrics [S2].

## Operating characteristics
- **Time Complexity:** The method reduces training time significantly, especially when combined with the accelerated schedule (up to 23x faster) [S2]. The multi-scale rendering (low res for first 5k iterations) reduces early-stage computational load [S1].
- **Memory Complexity:** Adaptive pruning removes low-opacity and oversized Gaussians, controlling memory usage [S1]. The accelerated schedule results in a lower final Gaussian count (e.g., 1.86M vs 3.92M in some comparisons) [S2].
- **Performance:** Achieves competitive PSNR, SSIM, and LPIPS scores compared to standard 3DGS and other variants [S1, S2]. The accelerated schedule maintains high quality (SSIM ~0.821, LPIPS ~0.197) with fewer Gaussians [S2].
- **Strengths:** Faster convergence, reduced memory footprint, improved image quality metrics (SSIM, LPIPS) [S1, S2].
- **Limitations:** Requires careful tuning of hyperparameters (thresholds, learning rate schedule) [S1, S2]. The accelerated schedule may not converge if the original learning rate schedule is not adjusted [S2].

## Background
The improved pipeline was introduced in "Optimization of 3D Gaussian Splatting for Accurate Image Reconstruction" (DS-DST, 2025) [S1]. It builds upon the original 3D Gaussian Splatting method by adding structured optimization steps. The accelerated variant was proposed in "Faster 3D Gaussian Splatting Convergence via Structure-Aware Densification" (2025) [S2].

## Sources
[S1] Optimization of 3D Gaussian Splatting for Accurate Image Reconstruction, https://doi.org/10.59232/dst-v4i2p101, fetched 2026-10-26
[S2] Faster 3D Gaussian Splatting Convergence via Structure-Aware Densification, https://doi.org/10.1145/3799902.3811212, fetched 2026-10-30
