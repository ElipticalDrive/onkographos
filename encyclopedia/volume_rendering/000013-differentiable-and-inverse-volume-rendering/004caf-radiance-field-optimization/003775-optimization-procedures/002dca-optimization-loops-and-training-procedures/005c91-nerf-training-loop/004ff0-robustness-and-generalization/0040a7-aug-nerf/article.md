# Aug-NeRF

Also called: Aug-NeRF Training Pipeline, Augmented NeRF, Triple-Level Physically-Grounded Augmentations

## Overview
Aug-NeRF is a training framework for Neural Radiance Fields (NeRF) that enhances generalization to unseen views and robustness to data inconsistencies by injecting worst-case adversarial perturbations at three distinct levels of the rendering pipeline: input coordinates, intermediate MLP features, and pre-rendering outputs [S1]. Unlike explicit smoothness regularizers (e.g., Laplacian or Total Variation) that aggressively constrain MLP flexibility and often degrade performance, Aug-NeRF uses these perturbations as a data-adaptive implicit regularization to induce a smooth loss landscape without sacrificing representation capacity [S1]. It builds on [NeRF](52-name.md) by treating the training objective as a min-max game where perturbations maximize the photometric loss, which is then minimized with respect to network parameters [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images $I = \{I_i\}_{i=1}^N$ with corresponding camera poses $P = \{\phi_i\}_{i=1}^N$. The neural radiance field $F_\Theta$ is parameterized by weights $\Theta$.
*   **Outputs:** Updated network weights $\Theta$ that minimize the combined photometric and adversarial regularization loss.

**Data Structures and Layout:**
*   **Ray Representation:** A ray is defined as $r = (o, d, \theta)$, where $o \in \mathbb{R}^3$ is the optical center, $d \in \mathbb{R}^3$ is the direction, and $\theta \in [-\pi, \pi]^2$ is the angular view direction [S1].
*   **Radiance Field:** $F_\Theta: (p, \theta) \mapsto (c, \sigma)$, where $p \in \mathbb{R}^3$ is the spatial coordinate, $c \in \mathbb{R}^3$ is RGB color, and $\sigma \in \mathbb{R}^+$ is density [S1].
*   **MLP Structure:** $F_\Theta = \gamma \circ \text{MLP}_\Theta$, where $\gamma$ is a Fourier feature mapping network. The MLP consists of a shared feature encoder $f(\cdot)$ mapping coordinates to a $D$-dimensional feature vector, followed by a color branch $g(\cdot)$ and a density branch $h(\cdot)$ [S1].
*   **Sampling:** Points are sampled along rays using coarse uniform sampling ($K/2$ points) and fine density-proportional sampling ($K/2$ points), totaling $K$ points per ray [S1].

**Formulas:**
1.  **Volume Rendering:**
    $$ C(r | \Theta) = \sum_{k=1}^{K} T(k) (1 - \exp(-\sigma_k \Delta t_k)) c_k $$
    where $T(k) = \exp(-\sum_{l=1}^{k-1} \sigma_l \Delta t_l)$ is the transmittance, and $\Delta t_k = t_{k+1} - t_k$ [S1].

2.  **Training Objective (Min-Max Game):**
    $$ \min_{\Theta} \mathbb{E}_{(r, \hat{C}) \sim P(\mathcal{R})} \left[ \| C(r | \Theta) - \hat{C} \|_2^2 + \lambda \max_{\delta} \| C^\dag(r | \Theta, \delta) - \hat{C} \|_2^2 \right] $$
    where $\delta = (\delta_p, \delta_f, \delta_r)$ represents perturbations for coordinates, features, and output respectively, and $\lambda = 0.1$ [S1].

3.  **Perturbation Search (Projected Gradient Descent):**
    $$ \delta^{(t+1)} = \text{Proj}_{\|\delta\|_\infty \le \epsilon} \left[ \delta^{(t)} + \alpha \cdot \text{sgn}(\nabla_\delta \mathcal{L}(\Theta | \mathcal{R}, \delta)) \right] $$
    where $\alpha$ is the step size and $\epsilon$ is the perturbation radius [S1].

**Parameters:**
*   $K$: Total number of depth intervals (sum of coarse and fine samples).
*   $\lambda$: Regularization weight, set to 0.1 [S1].
*   PGD Configuration: Coordinate level uses 3 steps with step size $10^{-2}$; Feature level uses 1 step with $10^{-3}$; Pre-rendering level uses 1 step with $10^{-5}$ [S1].
*   Perturbation Bounds:
    *   $\delta_t$: Constrained by $|\alpha_t (t_{k+1} - t_k)|$.
    *   $\delta_{xyz}$: Constrained to a ball $B(0, \epsilon_p)$ along the ray.
    *   $\delta_\theta$: Constrained to a conical frustum $[-\epsilon_p/2f, \epsilon_p/2f]^2$, where $f$ is focal length and $\epsilon_p$ is pixel size [S1].
    *   $\delta_c, \delta_\sigma$: Constrained to $[-\epsilon_c, \epsilon_c]^3 \times [-\epsilon_\sigma, \epsilon_\sigma]$, clamped to keep $c_k, \sigma_k \in [0, 1]$ [S1].
    *   $\delta_f$: Constrained to $[-\epsilon_f, \epsilon_f]^D$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def aug_nerf_train(images, poses, F_theta, K, lambda_reg, pgd_config):
    """
    images: List of N training images.
    poses: List of N camera poses (3x4 matrices).
    F_theta: Neural Radiance Field function F_theta(p, theta) -> (c, sigma).
    K: Total number of samples per ray.
    lambda_reg: Weight for adversarial regularization (0.1).
    pgd_config: Dict with keys 'coord', 'feat', 'output', each containing 
                {'steps': int, 'step_size': float, 'epsilon': float}.
    """
    rays = []
    for i in range(len(images)):
        # Cast rays for each pixel in image i using pose i
        # Returns list of rays r = (o, d, theta) and ground truth colors C_hat
        ray_pixels = cast_rays(images[i], poses[i])
        rays.extend(ray_pixels)
    
    # Initialize perturbations to zero
    delta_t = 0
    delta_xyz = 0
    delta_theta = 0
    delta_f = 0
    delta_c = 0
    delta_sigma = 0
    
    for iteration in range(500000): # 500K iterations
        # 1. Sample a batch of rays (batch size 1 assumed for simplicity)
        ray, C_hat = sample_ray(rays)
        o, d, theta, C_hat = ray
        
        # 2. Generate worst-case perturbations using PGD
        # Coordinate Perturbation
        delta_t, delta_xyz, delta_theta = pgd_search_coordinate(
            ray, F_theta, C_hat, pgd_config['coord']
        )
        
        # Feature Perturbation
        delta_f = pgd_search_feature(
            ray, F_theta, C_hat, pgd_config['feat']
        )
        
        # Pre-rendering Output Perturbation
        delta_c, delta_sigma = pgd_search_output(
            ray, F_theta, C_hat, pgd_config['output']
        )
        
        # 3. Coarse Sampling
        t_coarse = uniform_sample(K // 2, ray.t_near, ray.t_far)
        
        # 4. Fine Sampling (proportional to coarse densities)
        # Query coarse points to get densities for importance sampling
        p_coarse = o + t_coarse * d
        _, sigma_coarse = F_theta(p_coarse, theta)
        t_fine = importance_sample(K // 2, t_coarse, sigma_coarse)
        
        # Combine and sort all depth intervals
        t_all = sort(concat(t_coarse, t_fine))
        
        # 5. Compute Perturbed Points and Features
        # Apply coordinate perturbations
        t_dag = t_all + delta_t
        theta_dag = theta + delta_theta
        p_dag = o + t_dag * d + delta_xyz
        
        # Query network for unperturbed features (using unperturbed points for base query? 
        # Source says: c_k = g(f(p_dag) + delta_f, theta_dag) + delta_c
        # This implies features are queried at perturbed coordinates, then feature perturbation added.
        
        features = F_theta.feature_encoder(p_dag) # f(p_dag)
        
        # 6. Compute Perturbed Outputs
        # Color branch
        c_dag = F_theta.color_branch(features + delta_f, theta_dag) + delta_c
        # Density branch
        sigma_dag = F_theta.density_branch(features + delta_f) + delta_sigma
        
        # Clamp outputs to valid range [0, 1]
        c_dag = clamp(c_dag, 0, 1)
        sigma_dag = clamp(sigma_dag, 0, 1)
        
        # 7. Volumetric Rendering
        # Compute intervals
        dt_dag = diff(t_dag) # t_dag[k] - t_dag[k-1]
        
        # Compute transmittance
        # T_dag(k) = exp(-sum_{l=1}^{k-1} sigma_dag[l] * dt_dag[l])
        T_dag = compute_transmittance(sigma_dag, dt_dag)
        
        # Render perturbed color
        alpha_dag = 1 - exp(-sigma_dag * dt_dag)
        C_dag = sum(T_dag * alpha_dag * c_dag)
        
        # 8. Compute Loss
        # Photometric loss on clean ray (optional in some implementations, 
        # but source Eqn 6 shows L = ||C - C_hat||^2 + lambda ||C_dag - C_hat||^2)
        # Note: Source Algorithm 1 line 22: L = ||Ci - C_hat||^2 + lambda ||Ci_dag - C_hat||^2
        # However, Ci is rendered from unperturbed params? 
        # In Aug-NeRF, the "clean" render uses the current theta but unperturbed inputs?
        # Actually, the min-max is over delta. The outer min is over Theta.
        # The loss used for gradient update on Theta includes both terms.
        
        L_clean = mse(render_clean(ray, F_theta), C_hat)
        L_adv = mse(C_dag, C_hat)
        L = L_clean + lambda_reg * L_adv
        
        # 9. Update Network Parameters
        gradients = compute_gradients(L, F_theta)
        F_theta.update(gradients)

def pgd_search_coordinate(ray, F_theta, C_hat, config):
    delta_t = 0
    delta_xyz = 0
    delta_theta = 0
    for step in range(config['steps']):
        # Compute gradient of loss w.r.t perturbations
        # Perturb ray inputs
        t_dag = ray.t + delta_t
        theta_dag = ray.theta + delta_theta
        p_dag = ray.o + t_dag * ray.d + delta_xyz
        
        # Render with these perturbations (assuming other perturbations are 0 or handled separately)
        # For coordinate PGD, we typically isolate coordinate impact
        C_perturbed = render_ray_with_coords(ray, F_theta, delta_t, delta_xyz, delta_theta)
        grad = gradient(mse(C_perturbed, C_hat), [delta_t, delta_xyz, delta_theta])
        
        # Update
        delta_t += config['step_size'] * sign(grad.delta_t)
        delta_xyz += config['step_size'] * sign(grad.delta_xyz)
        delta_theta += config['step_size'] * sign(grad.delta_theta)
        
        # Project to constraints
        delta_t = project_delta_t(delta_t, ray)
        delta_xyz = project_ball(delta_xyz, config['epsilon'])
        delta_theta = project_cone(delta_theta, config['epsilon'], ray.focal_length)
    return delta_t, delta_xyz, delta_theta

def pgd_search_feature(ray, F_theta, C_hat, config):
    delta_f = 0
    for step in range(config['steps']):
        # Query features at perturbed coordinates (from coord PGD result)
        # ... (coordinate perturbations applied first)
        features = F_theta.feature_encoder(p_dag)
        c_perturbed = F_theta.color_branch(features + delta_f, theta_dag)
        sigma_perturbed = F_theta.density_branch(features + delta_f)
        C_perturbed = render_ray(c_perturbed, sigma_perturbed, ...)
        
        grad = gradient(mse(C_perturbed, C_hat), delta_f)
        delta_f += config['step_size'] * sign(grad)
        delta_f = project_box(delta_f, config['epsilon'])
    return delta_f

def pgd_search_output(ray, F_theta, C_hat, config):
    delta_c = 0
    delta_sigma = 0
    for step in range(config['steps']):
        # Perturb final color and density before integration
        # ... (coordinate and feature perturbations applied)
        # Render with perturbed c and sigma
        C_perturbed = render_ray(c + delta_c, sigma + delta_sigma, ...)
        
        grad_c = gradient(mse(C_perturbed, C_hat), delta_c)
        grad_sigma = gradient(mse(C_perturbed, C_hat), delta_sigma)
        
        delta_c += config['step_size'] * sign(grad_c)
        delta_sigma += config['step_size'] * sign(grad_sigma)
        
        delta_c = project_box(delta_c, config['epsilon'])
        delta_sigma = project_box(delta_sigma, config['epsilon'])
    return delta_c, delta_sigma
```

**Edge Cases and Pitfalls:**
*   **Perturbation Strength:** Using excessively strong worst-case perturbations (e.g., PGD-10) can deteriorate training performance instead of improving it [S1].
*   **Batch Size:** The algorithm description assumes a batch size of 1 for simplicity; implementation for larger batches requires averaging gradients or perturbations appropriately [S1].
*   **Feature Injection Point:** Injecting feature perturbation into the middle layer of the NeRF MLP contributes most significantly to performance gains; other layers may be less effective [S1].

## Subtopics

### [Occluder simulation augmentation](14621-occluder-simulation-augmentation.md)
A data augmentation technique that overlays randomly positioned and colored squares on training images to test and improve model robustness to in-the-wild transient occlusions. It sets itself apart from standard augmentation by explicitly simulating the physical presence of temporary objects that obscure underlying geometry, forcing the neural radiance field to learn to ignore or represent these artifacts as non-permanent features.

## Operating characteristics
*   **Time Complexity:** Training requires 500K iterations, which is comparable to standard NeRF but includes the overhead of PGD steps for perturbation generation at each iteration [S1].
*   **Memory Complexity:** Requires standard NeRF memory footprint plus storage for perturbation vectors $\delta$ during the inner maximization loop [S1].
*   **Performance:** On the LLFF dataset, Aug-NeRF reduces average error by 14.3%–26.0% and improves PSNR by up to 1.5dB (e.g., +1.34 on "fern", +1.37 on "trex") compared to baseline NeRF [S1]. On the 360° NeRF-Synthetic dataset, it reduces average error by 12.5%–44.7% (e.g., +1.33 on "drums", +0.87 on "chair") [S1].
*   **Robustness:** When trained on noisy supervision images (Gaussian and Shot noise), Aug-NeRF achieves 6.5%–26.9% average error reduction compared to vanilla NeRF [S1].
*   **Hardware:** Evaluated on NVIDIA RTX A6000 GPU with 48 GB memory [S1].
*   **Strengths:** Implicitly regularizes the model without sacrificing representation flexibility; improves geometry smoothness and generalization to unseen views; robust to noisy input data.
*   **Limitations:** Sensitive to perturbation strength; requires careful tuning of PGD parameters; computationally more expensive per iteration than vanilla NeRF due to adversarial search.

## Background
Aug-NeRF was proposed by Tianlong Chen, Peihao Wang, Zhiwen Fan, and Zhangyang Wang from The University of Texas at Austin in 2022 [S1]. It addresses the generalization gap in NeRF where interpolated novel views often yield inconsistent and visually non-smooth geometric results [S1]. The method is inspired by robust data augmentations used in convolutional neural networks, adapting the concept of worst-case adversarial perturbations to the specific physics of volumetric rendering [S1]. It contrasts with explicit regularizers like Laplacian or Total Variation, which are found to constrain MLP flexibility too aggressively [S1].

## Sources
[S1] Aug-NeRF: Training Stronger Neural Radiance Fields with Triple-Level Physically-Grounded Augmentations, https://doi.org/10.1109/cvpr52688.2022.01476, fetched 2026-10-26
