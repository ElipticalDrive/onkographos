# Aug-NeRF

Also called: Aug-NeRF Training Pipeline, Augmented NeRF, Triple-Level Physically-Grounded Augmentations

## Overview
Aug-NeRF is a training framework for Neural Radiance Fields (NeRF) that injects worst-case adversarial perturbations at three distinct levels of the rendering pipeline—input coordinates, intermediate MLP features, and pre-rendering output—to implicitly regularize the model with physical grounds [S1]. It addresses the generalization gap in NeRF, where standard optimization often yields inconsistent and non-smooth geometries when interpolating to unseen views [S1]. Unlike explicit regularizers like Laplacian or Total Variation that aggressively constrain MLP flexibility, Aug-NeRF uses data-adaptive smoothness priors to enhance generalization without sacrificing representation fidelity [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of $N$ training images $I = \{I_i \in \mathbb{R}^M\}_{i=1}^N$ and their corresponding camera poses $P = \{\phi_i \in \mathbb{R}^{3 \times 4}\}_{i=1}^N$. The neural radiance field $F_\Theta$ is parameterized by weights $\Theta$.
*   **Outputs:** Optimized network parameters $\Theta$ that define the radiance field.

**Data Structures**
*   **Ray Representation:** A ray is defined as $r = (o, d, \theta)$, where $o \in \mathbb{R}^3$ is the optical center, $d \in \mathbb{R}^3$ is the direction, and $\theta \in [-\pi, \pi]^2$ is the angular view direction [S1].
*   **Radiance Field:** $F_\Theta: (x, \theta) \mapsto (c, \sigma)$, where $x \in \mathbb{R}^3$ is the spatial coordinate, $c \in \mathbb{R}^3$ is the RGB color, and $\sigma \in \mathbb{R}^+$ is the volume density [S1].
*   **MLP Architecture:** $F_\Theta = \gamma \circ \text{MLP}_\Theta$, where $\gamma$ is a Fourier feature mapping network [S1]. The MLP consists of a shared feature branch $f(\cdot)$ mapping coordinates to a $D$-dimensional feature vector, and two output branches: $g(\cdot)$ for color and $h(\cdot)$ for density [S1].
*   **Perturbation Vectors:**
    *   Coordinate perturbation $\delta_p = (\delta_t, \delta_{xyz}, \delta_\theta)$, where $\delta_t \in \mathbb{R}$, $\delta_{xyz} \in \mathbb{R}^3$, $\delta_\theta \in \mathbb{R}^2$ [S1].
    *   Feature perturbation $\delta_f \in \mathbb{R}^D$ [S1].
    *   Output perturbation $\delta_r = (\delta_c, \delta_\sigma)$, where $\delta_c \in \mathbb{R}^3$, $\delta_\sigma \in \mathbb{R}$ [S1].

**Parameters**
*   $K$: Total number of depth intervals sampled per ray (sum of coarse and fine samples).
*   $t_n, t_f$: Near and far bounds of the ray segment [S1].
*   $\lambda$: Regularization weight for the adversarial term, set to 0.1 [S1].
*   $\epsilon$: Radius of the perturbation search space (constraint magnitude) [S1].
*   PGD Configuration:
    *   Coordinate Level: 3 PGD steps, step size $10^{-2}$ [S1].
    *   Feature Level: 1 PGD step, step size $10^{-3}$ [S1].
    *   Output Level: 1 PGD step, step size $10^{-5}$ [S1] [S1].
*   Training Iterations: 500,000 [S1].

**Numbered Steps**
1.  **Ray Casting:** For each pixel in each training image $I_i$, cast a ray $r_i = (o_i, d_i, \theta_i)$ using the inverse projection of camera pose $\phi_i$. Collect ground-truth colors $\hat{C}_i$.
2.  **Sampling:**
    *   **Coarse Sampling:** Sample $K/2$ depth intervals $t_k$ uniformly along the ray within $[t_n, t_f]$.
    *   **Fine Sampling:** Sample $K/2$ depth intervals $t_k$ proportional to the densities from the coarse pass [S1].
3.  **Adversarial Perturbation Search:** For the current batch of rays, compute worst-case perturbations $\delta$ by maximizing the loss using Projected Gradient Descent (PGD) within defined constraint sets [S1].
4.  **Perturbed Coordinate Computation:**
    *   Compute perturbed depth: $t^\dag_k = t_k + \delta_{t,k}$.
    *   Compute perturbed view direction: $\theta^\dag_i = \theta_i + \delta_\theta$.
    *   Compute unperturbed point: $p_k = o_i + t_k d_i$.
    *   Compute perturbed point: $p^\dag_k = o_i + t^\dag_k d_i + \delta_{xyz}$ [S1].
5.  **Network Query (Unperturbed):** Compute standard color and density: $(c_k, \sigma_k) = g(f(p_k, \theta_i)), h(f(p_k))$ [S1].
6.  **Network Query (Perturbed):**
    *   Compute perturbed color: $c^\dag_k = g(f(p^\dag_k) + \delta_f, \theta^\dag_i) + \delta_c$.
    *   Compute perturbed density: $\sigma^\dag_k = h(f(p^\dag_k) + \delta_f) + \delta_\sigma$ [S1].
7.  **Volumetric Rendering:**
    *   Compute intervals: $\Delta t_k = t_{k+1} - t_k$, $\Delta t^\dag_k = t^\dag_{k+1} - t^\dag_k$.
    *   Compute transmittance: $T(k) = \exp(-\sum_{l=1}^{k-1} \sigma_l \Delta t_l)$ and $T^\dag(k) = \exp(-\sum_{l=1}^{k-1} \sigma^\dag_l \Delta t^\dag_l)$.
    *   Render unperturbed color: $C_i = \sum_{k=1}^K T(k)(1 - \exp(-\sigma_k \Delta t_k))c_k$.
    *   Render perturbed color: $C^\dag_i = \sum_{k=1}^K T^\dag(k)(1 - \exp(-\sigma^\dag_k \Delta t^\dag_k))c^\dag_k$ [S1].
8.  **Loss Computation:** Calculate the total loss combining photometric error and adversarial regularization:
    $$ \mathcal{L} = \|C_i - \hat{C}_i\|_2^2 + \lambda \|C^\dag_i - \hat{C}_i\|_2^2 $$
9.  **Optimization:** Update network parameters $\Theta$ via gradient descent on $\mathcal{L}$ [S1].

**Formulas**
*   **Volume Rendering:**
    $$ C(r | \Theta) = \sum_{k=1}^{K} T(k) (1 - \exp(-\sigma_k \Delta t_k)) c_k $$
    where $T(k) = \exp(-\sum_{l=1}^{k-1} \sigma_l \Delta t_l)$ and $\Delta t_k = t_{k+1} - t_k$ [S1].
*   **Min-Max Optimization Objective:**
    $$ \min_{\Theta} \mathbb{E}_{(r, \hat{C}) \sim P(\mathcal{R})} \max_{\delta} \| C^\dag(r | \Theta, \delta) - \hat{C} \|_2^2 $$
    where $\delta = (\delta_p, \delta_f, \delta_r)$ represents perturbations for coordinate, feature, and output respectively [S1].
*   **Full Training Objective:**
    $$ \mathbb{E}_{(r, \hat{C}) \sim P(\mathcal{R})} \left[ \| C(r | \Theta) - \hat{C} \|_2^2 + \lambda \max_{\delta} \| C^\dag(r | \Theta, \delta) - \hat{C} \|_2^2 \right] $$
    [S1].
*   **PGD Update Rule:**
    $$ \delta^{(t+1)} = \text{Proj}_{\|\delta\|_\infty \le \epsilon} \left[ \delta^{(t)} + \alpha \cdot \text{sgn}(\nabla_\delta \mathcal{L}(\Theta | \mathcal{R}, \delta)) \right] $$
    where $\alpha$ is the step size and $\epsilon$ is the radius [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def aug_nerf_train(images, poses, F_theta, K, t_n, t_f, lambda_reg, pgd_config):
    """
    images: List of N images.
    poses: List of N camera poses (3x4).
    F_theta: Neural Radiance Field function F_theta(p, theta) -> (c, sigma).
             F_theta consists of f(p) -> feature, g(feature, theta) -> color, h(feature) -> sigma.
    K: Total number of samples per ray.
    t_n, t_f: Near and far bounds.
    lambda_reg: Weight for adversarial loss (0.1).
    pgd_config: Dict with keys 'coord', 'feat', 'output', each containing {'steps', 'step_size', 'epsilon'}.
    """
    rays = []
    gt_colors = []
    
    # 1. Ray Casting
    for i in range(len(images)):
        for pixel in images[i].pixels:
            o, d, theta = cast_ray(pixel, poses[i])
            rays.append((o, d, theta))
            gt_colors.append(pixel.color)
            
    # 2. Optimization Loop
    while not converged:
        # Sample a batch of rays
        batch_rays, batch_gt = sample_batch(rays, gt_colors)
        
        # 3. Adversarial Perturbation Search (Inner Max)
        # Initialize perturbations to zero
        delta_t = zeros(len(batch_rays))
        delta_xyz = zeros(len(batch_rays), 3)
        delta_theta = zeros(len(batch_rays), 2)
        delta_f = zeros(len(batch_rays), D) # D is hidden dimension
        delta_c = zeros(len(batch_rays), 3)
        delta_sigma = zeros(len(batch_rays))
        
        # Coordinate PGD
        for step in range(pgd_config['coord']['steps']):
            # Compute loss w.r.t coordinate perturbations
            loss_coord = compute_perturbed_loss(batch_rays, batch_gt, 
                                                delta_t, delta_xyz, delta_theta, 
                                                delta_f, delta_c, delta_sigma, F_theta)
            grad_t, grad_xyz, grad_theta = gradients(loss_coord, [delta_t, delta_xyz, delta_theta])
            
            # Update
            delta_t = delta_t + pgd_config['coord']['step_size'] * sign(grad_t)
            delta_xyz = delta_xyz + pgd_config['coord']['step_size'] * sign(grad_xyz)
            delta_theta = delta_theta + pgd_config['coord']['step_size'] * sign(grad_theta)
            
            # Project to constraints
            delta_t = clamp(delta_t, -pgd_config['coord']['epsilon'], pgd_config['coord']['epsilon'])
            delta_xyz = clamp(delta_xyz, -pgd_config['coord']['epsilon'], pgd_config['coord']['epsilon'])
            delta_theta = clamp(delta_theta, -pgd_config['coord']['epsilon'], pgd_config['coord']['epsilon'])

        # Feature PGD
        for step in range(pgd_config['feat']['steps']):
            loss_feat = compute_perturbed_loss(batch_rays, batch_gt, 
                                               delta_t, delta_xyz, delta_theta, 
                                               delta_f, delta_c, delta_sigma, F_theta)
            grad_f = gradients(loss_feat, delta_f)
            delta_f = delta_f + pgd_config['feat']['step_size'] * sign(grad_f)
            delta_f = clamp(delta_f, -pgd_config['feat']['epsilon'], pgd_config['feat']['epsilon'])

        # Output PGD
        for step in range(pgd_config['output']['steps']):
            loss_out = compute_perturbed_loss(batch_rays, batch_gt, 
                                              delta_t, delta_xyz, delta_theta, 
                                              delta_f, delta_c, delta_sigma, F_theta)
            grad_c, grad_sigma = gradients(loss_out, [delta_c, delta_sigma])
            delta_c = delta_c + pgd_config['output']['step_size'] * sign(grad_c)
            delta_sigma = delta_sigma + pgd_config['output']['step_size'] * sign(grad_sigma)
            delta_c = clamp(delta_c, -pgd_config['output']['epsilon'], pgd_config['output']['epsilon'])
            delta_sigma = clamp(delta_sigma, -pgd_config['output']['epsilon'], pgd_config['output']['epsilon'])

        # 4. Forward Pass with Perturbations (Outer Min)
        # Coarse Sampling
        t_coarse = uniform_sample(K // 2, t_n, t_f)
        # Fine Sampling (proportional to coarse densities - simplified for pseudocode)
        # In practice, query F_theta with coarse points to get densities, then sample fine points
        t_fine = proportional_sample(K // 2, t_coarse, F_theta)
        t_all = concatenate(t_coarse, t_fine)
        
        C_clean_list = []
        C_pert_list = []
        
        for idx, (o, d, theta) in enumerate(batch_rays):
            # Perturbed Coordinates
            t_pert = t_all + delta_t[idx]
            theta_pert = theta + delta_theta[idx]
            p_pert = o + t_pert * d + delta_xyz[idx]
            
            # Unperturbed Query
            p_clean = o + t_all * d
            c_clean, sigma_clean = F_theta(p_clean, theta)
            
            # Perturbed Query
            # Note: delta_f is added to the feature output of f(p)
            # c_pert = g(f(p_pert) + delta_f, theta_pert) + delta_c
            # sigma_pert = h(f(p_pert) + delta_f) + delta_sigma
            feature_pert = F_theta.f(p_pert) + delta_f[idx]
            c_pert = F_theta.g(feature_pert, theta_pert) + delta_c[idx]
            sigma_pert = F_theta.h(feature_pert) + delta_sigma[idx]
            
            # Clamp outputs to valid ranges [0, 1]
            c_pert = clamp(c_pert, 0, 1)
            sigma_pert = clamp(sigma_pert, 0, 1) # Density is positive, but clamp to avoid numerical issues
            
            # Volumetric Rendering
            dt = t_all[1:] - t_all[:-1]
            dt_pert = t_pert[1:] - t_pert[:-1]
            
            # Transmittance
            T_clean = exp(-cumsum(sigma_clean[:-1] * dt))
            T_pert = exp(-cumsum(sigma_pert[:-1] * dt_pert))
            
            # Weights
            alpha_clean = 1 - exp(-sigma_clean * dt)
            alpha_pert = 1 - exp(-sigma_pert * dt_pert)
            
            # Render
            C_clean = sum(T_clean * alpha_clean * c_clean)
            C_pert = sum(T_pert * alpha_pert * c_pert)
            
            C_clean_list.append(C_clean)
            C_pert_list.append(C_pert)
            
        # 5. Loss and Update
        loss_clean = mean_squared_error(C_clean_list, batch_gt)
        loss_pert = mean_squared_error(C_pert_list, batch_gt)
        total_loss = loss_clean + lambda_reg * loss_pert
        
        # Gradient descent on Theta
        F_theta.update_gradients(gradients(total_loss, F_theta.parameters))

def cast_ray(pixel, pose):
    # Inverse projection to get ray origin o, direction d, and view angle theta
    # Implementation depends on camera model
    pass

def uniform_sample(n, t_n, t_f):
    return linspace(t_n, t_f, n)

def proportional_sample(n, t_coarse, F_theta):
    # Query densities at coarse points, then sample fine points based on density
    # Simplified placeholder for the specific sampling strategy
    pass

def clamp(x, min_val, max_val):
    return max(min_val, min(x, max_val))

def sign(x):
    return 1 if x > 0 else (-1 if x < 0 else 0)
```

**Edge Cases and Pitfalls**
*   **Perturbation Strength:** Using excessively strong worst-case perturbations (e.g., PGD-10) can deteriorate training performance instead of improving it [S1].
*   **Batch Size:** The algorithm description assumes a batch size of 1 for simplicity; implementation must handle batching correctly [S1].
*   **Output Clamping:** Perturbed color and density values must be clamped to ensure they lie within valid ranges (e.g., $[0, 1]$ for color) to prevent numerical instability [S1].
*   **Feature Injection Point:** Injecting perturbation into the middle layer of the NeRF’s MLP contributes the most significantly to performance gains; other layers may be less effective [S1].

## Operating characteristics
*   **Time Complexity:** The method adds an inner optimization loop (PGD) to each training step, increasing computational cost proportional to the number of PGD steps per level (typically 5 steps total: 3 for coord, 1 for feat, 1 for output) [S1].
*   **Memory Complexity:** Requires storing gradients for perturbations $\delta$ in addition to network parameters $\Theta$ [S1].
*   **Performance:** Achieves up to 1.5dB PSNR gain in novel view synthesis compared to baseline NeRF [S1]. On LLFF and 360° NeRF-Synthetic datasets, it reduces average error by 14.3%–26.0% and 12.5%–44.7%, respectively [S1]. Specific PSNR improvements include fern (+1.34), orchids (+1.24), trex (+1.37), drums (+1.33), ship (+0.53), chair (+0.87) [S1].
*   **Robustness:** When trained on noisy supervision images (Gaussian and Shot noise), Aug-NeRF achieves 6.5%–26.9% average error reduction compared to vanilla NeRF [S1].
*   **Hardware:** Tested on NVIDIA RTX A6000 GPU with 48 GB memory [S1].
*   **Strengths:** Implicitly enforces smoothness without aggressively constraining MLP flexibility, preserving fine details and geometry better than explicit regularizers like Laplacian or Total Variation [S1].
*   **Limitations:** Requires careful tuning of PGD hyperparameters (steps, step sizes, epsilon) [S1].

## Background
*   **Origin:** Proposed by Tianlong Chen, Peihao Wang, Zhiwen Fan, and Zhangyang Wang from The University of Texas at Austin [S1].
*   **Paper:** "Aug-NeRF: Training Stronger Neural Radiance Fields with Triple-Level Physically-Grounded Augmentations" [S1].
*   **Date:** arXiv:2207.01164v1, July 4, 2022 [S1].
*   **Context:** NeRF often yields inconsistent and visually non-smooth geometric results when interpolating novel views, representing a generalization gap [S1]. Previous attempts to fix this using explicit regularizers (Laplacian, TV) were found to constrain representation flexibility too aggressively [S1]. Aug-NeRF draws inspiration from robust data augmentations in CNNs to inject worst-case perturbations into the NeRF pipeline [S1].

## Sources
[S1] Aug-NeRF: Training Stronger Neural Radiance Fields with Triple-Level Physically-Grounded Augmentations, https://doi.org/10.1109/cvpr52688.2022.01476, fetched 2026-10-26
