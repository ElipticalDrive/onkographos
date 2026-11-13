# Aug-NeRF

Also called: Aug-NeRF Training Pipeline, Augmented NeRF, Triple-Level Physically-Grounded Augmentations

## Overview
Aug-NeRF is a training framework for Neural Radiance Fields (NeRF) that injects worst-case adversarial perturbations at three distinct levels of the pipeline—input coordinates, intermediate MLP features, and pre-rendering outputs—to implicitly regularize the model with physical grounds [S1]. It addresses the generalization gap in NeRF where interpolation between seen views yields inconsistent and non-smooth geometries [S1]. Unlike explicit smoothness regularizers (e.g., Laplacian or Total Variation) that aggressively constrain MLP flexibility, Aug-NeRF uses data-adaptive perturbations to induce a smooth loss landscape, enhancing generalization to unseen views without sacrificing representation detail [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images $I = \{I_i \in \mathbb{R}^{M}\}_{i=1}^N$ and their corresponding camera poses $P = \{\phi_i \in \mathbb{R}^{3 \times 4}\}_{i=1}^N$.
*   **Outputs:** Optimized neural radiance field parameters $\Theta$.

**Data Structures:**
*   **Ray Representation:** A ray is defined as $r = (o, d, \theta)$, where $o \in \mathbb{R}^3$ is the optical center, $d \in \mathbb{R}^3$ is the direction, and $\theta \in [-\pi, \pi]^2$ is the angular view direction [S1].
*   **Radiance Field:** $F_\Theta: (p, \theta) \mapsto (c, \sigma)$, where $p \in \mathbb{R}^3$ is the spatial coordinate, $c \in \mathbb{R}^3$ is RGB color, and $\sigma \in \mathbb{R}^+$ is density [S1].
*   **MLP Architecture:** $F_\Theta = \gamma \circ \text{MLP}_\Theta$, where $\gamma$ is a Fourier feature mapping network [S1]. The MLP is structured with two branches: $g \circ f(p, \theta)$ for color and $h \circ f(p)$ for density, where $f$ maps coordinates to a $D$-dimensional feature vector [S1].
*   **Sampling:** $K$ total depth intervals are sampled per ray, split into $K/2$ coarse samples (uniform) and $K/2$ fine samples (proportional to coarse densities) [S1].

**Parameters:**
*   $K$: Total number of depth intervals.
*   $\lambda$: Regularization weight, set to $0.1$ [S1].
*   $\epsilon$: Perturbation radius constraints (specific values depend on the level; see below) [S1].
*   PGD Configuration: Coordinate level uses 3 steps with step size $10^{-2}$; Feature level uses 1 step with step size $10^{-3}$; Pre-rendering level uses 1 step with step size $10^{-5}$ [S1].
*   Training Iterations: 500,000 [S1].

**Formulas:**
The core optimization is a min-max game [S1]:
$$
\min _{\Theta } \mathbb{E}_{(r, \hat{C}) \sim P(\mathcal{R})} \max _{\delta } \left \| C^\dag (r | \Theta , \delta ) - \hat{C} \right \|_2^2
$$
where $\delta = (\delta_p, \delta_f, \delta_r)$ represents perturbations for coordinate, feature, and output respectively [S1].

The full training objective combines photometric loss and adversarial regularization [S1]:
$$
\mathbb{E}_{(r, \hat{C}) \sim P(\mathcal{R})} \left \| C(r | \Theta ) - \hat{C} \right \|_2^2 + \lambda \max _{\delta } \left \| C^\dag (r | \Theta , \delta ) - \hat{C} \right \|_2^2
$$

**Numbered Steps:**

1.  **Ray Casting:** For each pixel in each training image $I_i$, cast a ray $r_i = (o_i, d_i, \theta_i)$ using the camera pose $\phi_i$. Collect ground-truth colors $\hat{C}_i$.
2.  **Sampling:**
    *   Sample $K/2$ coarse depth intervals $t_k$ uniformly between near-far bounds $[t_n, t_f]$.
    *   Sample $K/2$ fine depth intervals $t_k$ proportional to the densities from the coarse samples.
3.  **Adversarial Perturbation Search:** For the current batch of rays, compute worst-case perturbations $\delta$ using Projected Gradient Descent (PGD) to maximize the loss. The update rule for perturbation $\delta$ at iteration $t$ is [S1]:
    $$
    \delta^{(t+1)} = \text{Proj}_{\|\delta\|_\infty \le \epsilon} \left [ \delta^{(t)} + \alpha \cdot \text{sgn}(\nabla_{\delta} \mathcal{L}(\Theta | \mathcal{R}, \delta)) \right ]
    $$
    where $\alpha$ is the step size and $\epsilon$ is the radius constraint specific to the perturbation level [S1].
4.  **Coordinate Perturbation (Level 1):**
    *   Compute perturbed depth: $t^\dag_k = t_k + \delta_t$.
    *   Compute perturbed view direction: $\theta^\dag = \theta + \delta_\theta$.
    *   Compute perturbed point: $p^\dag_k = o + t^\dag_k d + \delta_{xyz}$.
    *   Constraints: $\delta_t$ is bounded by $|\alpha_t (t_{k+1} - t_k)|$; $\delta_{xyz}$ is bounded by a ball $B(0, \epsilon_p)$; $\delta_\theta$ is bounded within a conical frustum $[-\epsilon_p/2f, \epsilon_p/2f]^2$ [S1].
5.  **Feature Perturbation (Level 2):**
    *   Inject noise $\delta_f$ into the intermediate feature vector $f(p^\dag_k)$.
    *   The feature perturbation $\delta_f$ is searched over $S_f = [-\epsilon_f, \epsilon_f]^D$ [S1].
6.  **Pre-Rendering Output Perturbation (Level 3):**
    *   Define output perturbation $\delta_r = (\delta_c, \delta_\sigma)$.
    *   $\delta_c \in [-\epsilon_c, \epsilon_c]^3$ and $\delta_\sigma \in [-\epsilon_\sigma, \epsilon_\sigma]$ [S1].
7.  **Network Query:**
    *   Compute unperturbed color and density: $(c_k, \sigma_k) = g \circ f(p_k, \theta), h \circ f(p_k)$.
    *   Compute perturbed color: $c^\dag_k = g(f(p^\dag_k) + \delta_f, \theta^\dag) + \delta_c$.
    *   Compute perturbed density: $\sigma^\dag_k = h(f(p^\dag_k + \delta_f)) + \delta_\sigma$.
    *   Note: $c_k$ and $\sigma_k$ values are clamped to $[0, 1]$ [S1].
8.  **Volumetric Rendering:**
    *   Compute intervals: $\Delta t_k = t_k - t_{k-1}$ and $\Delta t^\dag_k = t^\dag_k - t^\dag_{k-1}$.
    *   Compute transmittance: $T(k) = \exp(-\sum_{l=1}^{k-1} \sigma_l \Delta t_l)$ and $T^\dag(k) = \exp(-\sum_{l=1}^{k-1} \sigma^\dag_l \Delta t^\dag_l)$.
    *   Render unperturbed color: $C_i = \sum_{k=1}^K T(k)(1 - \exp(-\sigma_k \Delta t_k))c_k$.
    *   Render perturbed color: $C^\dag_i = \sum_{k=1}^K T^\dag(k)(1 - \exp(-\sigma^\dag_k \Delta t^\dag_k))c^\dag_k$.
9.  **Loss Computation and Update:**
    *   Compute loss: $\mathcal{L} = \|C_i - \hat{C}_i\|_2^2 + \lambda \|C^\dag_i - \hat{C}_i\|_2^2$.
    *   Update network parameters $\Theta$ via gradient descent on $\mathcal{L}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def aug_nerf_training(images, poses, network, K, lambda_reg, pgd_config):
    """
    images: List of N training images.
    poses: List of N camera poses (3x4 matrices).
    network: NeRF MLP parameters Theta.
    K: Total number of samples per ray.
    lambda_reg: Adversarial regularization weight (0.1).
    pgd_config: Dict with keys 'coord', 'feat', 'output', 
                each containing {'steps': int, 'step_size': float, 'epsilon': float}.
    """
    # Initialize rays
    rays = []
    for i in range(len(images)):
        for pixel in images[i].pixels:
            o, d, theta = cast_ray(pixel, poses[i])
            rays.append((o, d, theta, pixel.color))
            
    for iteration in range(500000):
        # Sample a batch of rays
        batch_rays = sample_batch(rays)
        
        # 1. Coarse and Fine Sampling
        coarse_samples = uniform_sample(K // 2, near_bound, far_bound)
        fine_samples = proportional_sample(K // 2, coarse_samples, network)
        all_depths = sort_and_concat(coarse_samples, fine_samples)
        
        # 2. Generate Adversarial Perturbations via PGD
        # Initialize perturbations to zero
        delta_t = zeros_like(all_depths)
        delta_xyz = zeros(3)
        delta_theta = zeros(2)
        delta_f = zeros(hidden_dim)
        delta_c = zeros(3)
        delta_sigma = zeros(1)
        
        # Coordinate PGD
        for step in range(pgd_config['coord']['steps']):
            # Compute loss w.r.t coordinate perturbations
            loss_coord = compute_perturbed_loss(batch_rays, all_depths, 
                                               delta_t, delta_xyz, delta_theta, 
                                               delta_f, delta_c, delta_sigma, network)
            grad_t = gradient(loss_coord, delta_t)
            grad_xyz = gradient(loss_coord, delta_xyz)
            grad_theta = gradient(loss_coord, delta_theta)
            
            # Update and Project
            delta_t = project_linf(delta_t + pgd_config['coord']['step_size'] * sign(grad_t), 
                                   epsilon=pgd_config['coord']['epsilon'])
            delta_xyz = project_ball(delta_xyz + pgd_config['coord']['step_size'] * sign(grad_xyz), 
                                     epsilon=pgd_config['coord']['epsilon'])
            delta_theta = project_frustum(delta_theta + pgd_config['coord']['step_size'] * sign(grad_theta), 
                                          epsilon=pgd_config['coord']['epsilon'])

        # Feature PGD
        for step in range(pgd_config['feat']['steps']):
            loss_feat = compute_perturbed_loss(batch_rays, all_depths, 
                                               delta_t, delta_xyz, delta_theta, 
                                               delta_f, delta_c, delta_sigma, network)
            grad_f = gradient(loss_feat, delta_f)
            delta_f = project_linf(delta_f + pgd_config['feat']['step_size'] * sign(grad_f), 
                                   epsilon=pgd_config['feat']['epsilon'])

        # Output PGD
        for step in range(pgd_config['output']['steps']):
            loss_out = compute_perturbed_loss(batch_rays, all_depths, 
                                              delta_t, delta_xyz, delta_theta, 
                                              delta_f, delta_c, delta_sigma, network)
            grad_c = gradient(loss_out, delta_c)
            grad_sigma = gradient(loss_out, delta_sigma)
            
            delta_c = project_linf(delta_c + pgd_config['output']['step_size'] * sign(grad_c), 
                                   epsilon=pgd_config['output']['epsilon'])
            delta_sigma = project_linf(delta_sigma + pgd_config['output']['step_size'] * sign(grad_sigma), 
                                       epsilon=pgd_config['output']['epsilon'])

        # 3. Forward Pass with Perturbations
        perturbed_colors = []
        clean_colors = []
        
        for ray in batch_rays:
            o, d, theta, gt_color = ray
            
            # Perturbed Coordinates
            t_dag = all_depths + delta_t
            theta_dag = theta + delta_theta
            p_dag = o + t_dag * d + delta_xyz
            
            # Query Network
            # Unperturbed
            p_clean = o + all_depths * d
            c_clean, sigma_clean = network.query(p_clean, theta)
            
            # Perturbed
            # Note: Feature perturbation is added to the output of the feature branch f
            features_perturbed = network.feature_branch(p_dag) + delta_f
            c_dag_raw = network.color_branch(features_perturbed, theta_dag)
            sigma_dag_raw = network.density_branch(features_perturbed)
            
            # Apply Output Perturbations
            c_dag = c_dag_raw + delta_c
            sigma_dag = sigma_dag_raw + delta_sigma
            
            # Clamp outputs
            c_dag = clamp(c_dag, 0, 1)
            sigma_dag = clamp(sigma_dag, 0, 1)
            c_clean = clamp(c_clean, 0, 1)
            sigma_clean = clamp(sigma_clean, 0, 1)
            
            # Volumetric Rendering
            dt = all_depths[1:] - all_depths[:-1]
            dt_dag = t_dag[1:] - t_dag[:-1]
            
            # Transmittance
            T_clean = exp(-cumsum(sigma_clean[:-1] * dt))
            T_dag = exp(-cumsum(sigma_dag[:-1] * dt_dag))
            
            # Render
            color_clean = sum(T_clean * (1 - exp(-sigma_clean * dt)) * c_clean)
            color_dag = sum(T_dag * (1 - exp(-sigma_dag * dt_dag)) * c_dag)
            
            clean_colors.append(color_clean)
            perturbed_colors.append(color_dag)
            
        # 4. Loss and Update
        clean_loss = mean_squared_error(clean_colors, [r[3] for r in batch_rays])
        adv_loss = mean_squared_error(perturbed_colors, [r[3] for r in batch_rays])
        total_loss = clean_loss + lambda_reg * adv_loss
        
        update_network(network, gradient(total_loss, network.params))

def cast_ray(pixel, pose):
    # Inverse projection to get ray origin, direction, and view angle
    # Returns o, d, theta
    pass

def uniform_sample(n, tn, tf):
    # Sample n points uniformly between tn and tf
    pass

def proportional_sample(n, coarse_depths, network):
    # Sample n points proportional to density at coarse_depths
    pass

def project_linf(x, epsilon):
    return clip(x, -epsilon, epsilon)

def project_ball(x, epsilon):
    if norm(x) > epsilon:
        return x * (epsilon / norm(x))
    return x

def project_frustum(x, epsilon):
    # Conical frustum constraint for view direction
    # Simplified as box constraint for implementation purposes based on source description
    return clip(x, -epsilon, epsilon)

def clamp(x, min_val, max_val):
    return max(min_val, min(x, max_val))
```

## Subtopics

### NeRF Ensemble with Conjoint Pruning
Uses ensemble averaging and pruning to improve robustness, differing from Aug-NeRF's adversarial perturbation approach. [Link](12608-nerf-ensemble-with-conjoint-pruning.md)

### NeRF in the Wild
Focuses on handling uncalibrated images and real-world noise, whereas Aug-NeRF assumes calibrated poses but adds robustness via perturbations. [Link](13541-nerf-in-the-wild.md)

### ContraNeRF
Uses contrastive learning to improve generalization, distinct from Aug-NeRF's explicit worst-case perturbation regularization. [Link](13659-contranerf.md)

### RoGUENeRF
Focuses on robustness to geometric errors and occlusions, using a different mechanism than triple-level adversarial training. [Link](13731-roguenerf.md)

## Operating characteristics
*   **Time Complexity:** The method adds significant computational overhead due to the inner maximization loop (PGD) for each training step. The number of PGD steps varies by level (3 for coordinates, 1 for features, 1 for output) [S1].
*   **Memory Complexity:** Requires storing gradients for the perturbations in addition to the network parameters. The source assumes a batch size of 1 for simplicity, but memory scales with batch size and the number of samples $K$ [S1].
*   **Performance:** Aug-NeRF reduces average error by 14.3%–26.0% on the LLFF dataset and 12.5%–44.7% on the 360° NeRF-Synthetic dataset compared to baseline NeRF [S1]. It achieves up to 1.5dB PSNR gain in novel view synthesis [S1].
*   **Strengths:** Improves geometry smoothness and generalization to unseen views; robust to noisy supervision images (Gaussian and Shot noise) [S1].
*   **Limitations:** Requires careful tuning of PGD parameters; excessively strong perturbations (e.g., PGD-10) can deteriorate performance [S1].
*   **Hardware:** Evaluated on NVIDIA RTX A6000 GPU with 48 GB memory [S1].

## Background
Aug-NeRF was proposed by Tianlong Chen, Peihao Wang, Zhiwen Fan, and Zhangyang Wang from The University of Texas at Austin [S1]. It was published in 2022 [S1]. The method is motivated by the observation that NeRF often yields inconsistent and non-smooth geometries when interpolating novel views, a generalization gap that explicit regularizers like Laplacian or Total Variation fail to address without sacrificing representation flexibility [S1]. It draws inspiration from robust data augmentations in convolutional neural networks [S1].

## Sources
[S1] Aug-NeRF: Training Stronger Neural Radiance Fields with Triple-Level Physically-Grounded Augmentations, https://doi.org/10.1109/cvpr52688.2022.01476, fetched 2026-10-26
