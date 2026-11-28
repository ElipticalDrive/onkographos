# Mip-NeRF 360 Optimization Setup

Also called: Depth Distortion Regularization, Gaussian Position Encoding, Mip-NeRF 360 Training Procedure, Position Warping for Unbounded Scenes

## Overview
Mip-NeRF 360 optimization setup defines the training configuration for unbounded neural radiance fields, specifically addressing the challenge of rendering large-scale urban scenes with limited video memory by warping the infinite scene domain into a finite sphere [S2]. It distinguishes itself from standard NeRF training by combining a specific scene contraction function, a multi-stage proposal network sampling strategy, and a robust learning schedule that prevents aliasing artifacts in unbounded spaces [S1, S2].

## How it works
The method optimizes a radiance field for unbounded scenes by warping coordinates, training a proposal network to guide sampling, and minimizing a composite loss function [S1, S2].

### Inputs and Outputs
*   **Inputs:** Camera poses $\mathbf{t}$ (estimated via COLMAP), input images $\mathbf{C}^*$, and scene coordinates $\mathbf{x}$ [S1, S2].
*   **Outputs:** Rendered color $\mathbf{C}(\mathbf{t})$, depth maps, and optimized network weights for the proposal and NeRF MLPs [S1, S2].

### Data Structures
1.  **Proposal MLP ($\Theta_{\text{prop}}$):**
    *   Architecture: 4 layers, 256 hidden units.
    *   Activations: ReLU for hidden layers, Softplus for density output $\tau$ [S1].
2.  **NeRF MLP ($\Theta_{\text{NeRF}}$):**
    *   Architecture: 8 layers, 1024 hidden units.
    *   Activations: ReLU for hidden layers, Softplus for density output $\tau$ [S1].
3.  **Scene Warping:**
    *   Input: 3D coordinate $\mathbf{x} \in \mathbb{R}^3$.
    *   Output: Warped coordinate $\mathbf{x}_{\text{encoding}} \in \mathbb{R}^3$ [S2].

### Numbered Steps

1.  **Scene Warping (Position Encoding):**
    For every 3D point $\mathbf{x}$ sampled along a ray, apply the contraction function to map unbounded space to a unit sphere [S2]:
    $$
    \mathbf{x}_{\text{encoding}} = \begin{cases} 
    \mathbf{x} & \text{if } \|\mathbf{x}\|_2 \leq 1 \\
    \left(2 - \frac{1}{\|\mathbf{x}\|_2}\right) \frac{\mathbf{x}}{\|\mathbf{x}\|_2} & \text{if } \|\mathbf{x}\|_2 > 1
    \end{cases}
    $$
    Points within the unit sphere ($\|\mathbf{x}\|_2 \leq 1$) remain unchanged; points outside are contracted towards the boundary [S2].

2.  **Proposal Network Sampling (Two Stages):**
    *   **Stage 0:** Evaluate the proposal MLP using 64 samples along the ray to produce interval midpoints $\hat{s}_0$ and weights $\hat{w}_0$ [S1].
    *   **Stage 1:** Resample based on $\hat{w}_0$ and evaluate the proposal MLP again with 64 samples to produce $\hat{s}_1$ and $\hat{w}_1$ [S1].

3.  **NeRF Network Sampling (One Stage):**
    *   Resample based on the final proposal weights $\hat{w}_1$ [S1].
    *   Evaluate the NeRF MLP using 32 samples to produce final interval midpoints $s$ and weights $w$ [S1].

4.  **Loss Computation:**
    Compute the total loss $\mathcal{L}$ averaged over all rays in the batch [S1]:
    $$
    \mathcal{L} = \mathcal{L}_{\text{recon}}(\mathbf{C}(\mathbf{t}), \mathbf{C}^*) + \lambda \mathcal{L}_{\text{dist}}(\mathbf{s}, \mathbf{w}) + \sum_{k=0}^1 \mathcal{L}_{\text{prop}}(\mathbf{s}, \mathbf{w}, \hat{\mathbf{s}}^k, \hat{\mathbf{w}}^k)
    $$
    *   $\mathcal{L}_{\text{recon}}$: Charbonnier loss $\sqrt{(x - x^*)^2 + \epsilon^2}$ with $\epsilon = 0.001$ [S1].
    *   $\mathcal{L}_{\text{dist}}$: Regularizer $R_{\text{dist}} = \sum_{u,v} H(u)H(v) \left| \frac{t_u + t_{u+1}}{2} - \frac{t_v + t_{v+1}}{2} \right|$ constraining weights to approach zero or concentrate at the same intersection [S2].
    *   $\mathcal{L}_{\text{prop}}$: Interlevel loss supervising the proposal network.
    *   $\lambda$: Hyperparameter balancing data and regularizer, set to $0.01$ [S1].

5.  **Gradient Stop-Gradient:**
    Apply stop-gradient to $\mathcal{L}_{\text{prop}}$ to ensure the optimization of $\Theta_{\text{prop}}$ is independent from $\Theta_{\text{NeRF}}$ [S1].

6.  **Optimization Update:**
    Update weights using the Adam optimizer [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def init_mlp_params(layers, hidden):
    """Initialize MLP weights and biases with He initialization."""
    params = {}
    for i in range(layers):
        in_dim = hidden if i > 0 else 3
        out_dim = hidden if i < layers - 1 else 4
        scale = np.sqrt(2.0 / in_dim)
        params[f'W{i}'] = np.random.randn(in_dim, out_dim) * scale
        params[f'b{i}'] = np.zeros(out_dim)
    return params

def init_adam_state(params):
    """Initialize Adam optimizer moments to zero."""
    m = {k: np.zeros_like(v) for k, v in params.items()}
    v = {k: np.zeros_like(v) for k, v in params.items()}
    return m, v

def sample_stratified(t_near, t_far, num_samples):
    """Stratified sampling along a ray segment."""
    t = np.linspace(t_near, t_far, num_samples + 1)
    return (t[:-1] + t[1:]) / 2.0

def forward_mlp(params, x):
    """Standard MLP forward pass with ReLU and Softplus for last layer."""
    h = x
    for i in range(len(params) // 2):
        W = params[f'W{i}']
        b = params[f'b{i}']
        h = h @ W + b
        if i < len(params) // 2 - 1:
            h = np.maximum(0, h)
        else:
            h = np.log1p(np.exp(h))
    return h

def compute_weights(density, delta):
    """Compute volumetric rendering weights from density and interval sizes."""
    alpha = 1.0 - np.exp(-density * delta)
    T = np.cumprod(1.0 - alpha, axis=0)
    T = np.concatenate([np.ones_like(T[:1]), T[:-1]], axis=0)
    return T * alpha

def resample_from_weights(t_bounds, weights, num_samples):
    """Resample positions based on discrete probability distribution."""
    cdf = np.cumsum(weights)
    cdf[-1] = 1.0
    u = np.random.uniform(0, 1, num_samples)
    indices = np.searchsorted(cdf, u)
    t_near = t_bounds[indices]
    t_far = t_bounds[indices + 1]
    return np.linspace(t_near, t_far, num_samples)

def volume_render(color, weights):
    """Accumulate color using volumetric rendering equation."""
    return np.sum(color * weights[:, np.newaxis], axis=0)

def compute_distortion_loss(midpoints, weights):
    """Compute depth distortion regularization R_dist."""
    loss = 0.0
    for i in range(len(midpoints)):
        for j in range(len(midpoints)):
            loss += weights[i] * weights[j] * np.abs(midpoints[i] - midpoints[j])
    return loss

def compute_interlevel_loss(s_nerf, w_nerf, s_prop, w_prop):
    """Compute interlevel loss between NeRF and proposal histograms."""
    return np.mean((w_nerf - w_prop) ** 2)

def train_mipnerf_360(rays, images, num_iterations=250000, batch_size=16384):
    theta_prop = init_mlp_params(4, 256)
    theta_nerf = init_mlp_params(8, 1024)
    m_prop, v_prop = init_adam_state(theta_prop)
    m_nerf, v_nerf = init_adam_state(theta_nerf)
    beta1, beta2, eps_adam = 0.9, 0.999, 1e-6
    lr_start, lr_end, warmup_iters = 2e-3, 2e-5, 512
    grad_clip_norm = 1e-3
    lambda_dist = 0.01
    epsilon_charb = 0.001

    for i in range(num_iterations):
        batch_indices = np.random.choice(len(rays), batch_size, replace=False)
        batch_rays = rays[batch_indices]
        batch_images = images[batch_indices]

        if i < warmup_iters:
            lr = lr_start * (i / warmup_iters)
        else:
            log_lr_start = np.log(lr_start)
            log_lr_end = np.log(lr_end)
            progress = (i - warmup_iters) / (num_iterations - warmup_iters)
            lr = np.exp(log_lr_start + progress * (log_lr_end - log_lr_start))

        # Stage 0: Proposal
        t_0 = sample_stratified(batch_rays.t_near, batch_rays.t_far, 64)
        x_0 = warp_position(transform_ray_to_world(batch_rays, t_0))
        out_0 = forward_mlp(theta_prop, x_0)
        density_0, _ = out_0[:, :1], out_0[:, 1:]
        delta_0 = np.diff(np.concatenate([batch_rays.t_near, t_0, batch_rays.t_far], axis=1), axis=1)
        w_0 = compute_weights(density_0, delta_0)
        s_0 = t_0

        # Stage 1: Proposal
        t_1 = resample_from_weights(np.concatenate([batch_rays.t_near, batch_rays.t_far], axis=1), w_0, 64)
        x_1 = warp_position(transform_ray_to_world(batch_rays, t_1))
        out_1 = forward_mlp(theta_prop, x_1)
        density_1, _ = out_1[:, :1], out_1[:, 1:]
        delta_1 = np.diff(np.concatenate([batch_rays.t_near, t_1, batch_rays.t_far], axis=1), axis=1)
        w_1 = compute_weights(density_1, delta_1)
        s_1 = t_1

        # Stage 2: NeRF
        t_nerf = resample_from_weights(np.concatenate([batch_rays.t_near, batch_rays.t_far], axis=1), w_1, 32)
        x_nerf = warp_position(transform_ray_to_world(batch_rays, t_nerf))
        out_nerf = forward_mlp(theta_nerf, x_nerf)
        density_nerf, color_nerf = out_nerf[:, :1], out_nerf[:, 1:]
        delta_nerf = np.diff(np.concatenate([batch_rays.t_near, t_nerf, batch_rays.t_far], axis=1), axis=1)
        w_nerf = compute_weights(density_nerf, delta_nerf)
        s_nerf = t_nerf

        C_recon = volume_render(color_nerf, w_nerf)
        L_recon = np.mean(np.sqrt((C_recon - batch_images)**2 + epsilon_charb**2))
        L_dist = compute_distortion_loss(s_nerf, w_nerf)
        L_prop_0 = compute_interlevel_loss(s_nerf, w_nerf, s_0, w_0)
        L_prop_1 = compute_interlevel_loss(s_nerf, w_nerf, s_1, w_1)
        L_total = L_recon + lambda_dist * L_dist + L_prop_0 + L_prop_1

        # Backprop & Update
        grad_prop = compute_gradients(L_total, theta_prop)
        grad_nerf = compute_gradients(L_total, theta_nerf)
        grad_prop = clip_gradients(grad_prop, grad_clip_norm)
        grad_nerf = clip_gradients(grad_nerf, grad_clip_norm)
        theta_prop, m_prop, v_prop = adam_update(theta_prop, grad_prop, m_prop, v_prop, lr, beta1, beta2, eps_adam)
        theta_nerf, m_nerf, v_nerf = adam_update(theta_nerf, grad_nerf, m_nerf, v_nerf, lr, beta1, beta2, eps_adam)

    return theta_prop, theta_nerf
```

### Parameters
*   **Optimizer:** Adam [S1].
*   **Beta 1:** $0.9$ [S1].
*   **Beta 2:** $0.999$ [S1].
*   **Epsilon:** $10^{-6}$ [S1].
*   **Learning Rate:** Annealed log-linearly from $2 \times 10^{-3}$ to $2 \times 10^{-5}$ [S1].
*   **Warm-up:** 512 iterations [S1].
*   **Gradient Clipping Norm:** $10^{-3}$ [S1].
*   **Training Iterations:** 250,000 [S1].
*   **Batch Size:** $2^{14}$ (16,384) rays [S1].
*   **Lambda ($\lambda$):** $0.01$ for $\mathcal{L}_{\text{dist}}$ [S1].
*   **Charbonnier Epsilon ($\epsilon$):** $0.001$ [S1].

### Edge Cases and Pitfalls
*   **Z-aliasing:** The standard interlevel loss can cause the proposal MLP to learn non-smooth mappings, leading to rays "skipping" scene content, especially with high-capacity backends like iNGP [S3].
*   **Unbounded Memory:** Without warping, rendering large-scale scenes exhausts video memory due to the infinite domain [S2]. The contraction function ensures all points are mapped to a finite sphere, preventing this [S2].
*   **Precision:** The contraction function is continuous but has a derivative discontinuity at $\|\mathbf{x}\|_2 = 1$ [S2].

## Subtopics

### [Anti-Aliased Interlevel Loss](3457-anti-aliased-interlevel-loss.md)
The Anti-Aliased Interlevel Loss is a proposal supervision loss function that measures the distance between a NeRF histogram and a proposal histogram by blurring the NeRF histogram into a piecewise-linear spline and resampling it into the proposal's interval endpoints. Unlike the piecewise constant loss used in mip-NeRF 360, which suffers from aliasing when NeRF histogram bins are narrow or translate relative to the proposal, this method produces a smooth loss surface that prevents aliasing artifacts during training.

### [Isotropic Gaussian Scale Contraction](5407-isotropic-gaussian-scale-contraction.md)
Isotropic Gaussian Scale Contraction is a technique for efficiently adapting the scale of isotropic Gaussians when a scene contraction function $C(\mathbf{x})$ is applied. It solves the problem of maintaining correct anti-aliasing properties in contracted coordinate spaces without the computational overhead of general multivariate Gaussian transformations. It sets itself apart from the standard mip-NeRF 360 approach by linearizing the contraction around each Gaussian center to derive an isotropic scale from the Jacobian determinant, rather than contracting a full covariance matrix.

### [Google Immersive View NeRF](13526-google-immersive-view-nerf.md)
A method for 3D reconstruction and novel view synthesis that distills a dense collection of indoor photos into a neural field to render photorealistic views from viewpoints not present in the original collection, specifically optimized for the scale, lighting variability, and privacy constraints of Google Maps Immersive View. It distinguishes itself from standard NeRF implementations by integrating low-dimensional generative latent optimization (GLO) vectors and metadata-derived exposure conditioning into the radiance network, thereby resolving cloudy geometry artifacts caused by lighting changes without sacrificing interpretability. Unlike [Anti-Aliased Interlevel Loss](3457-anti-aliased-interlevel-loss.md) which focuses on sampling strategies, this method focuses on appearance modeling and capture constraints; it builds on [Mip-NeRF 360 Optimization Setup](3426-mip-nerf-360-optimization-setup.md) as its architectural base.

### [Progressively Optimized Local Radiance Fields](15521-progressively-optimized-local-radiance-fields.md)
Progressively Optimized Local Radiance Fields (LocalRF) is a view synthesis method that jointly optimizes camera poses and multiple overlapping local radiance fields from a single long monocular video, avoiding Structure-from-Motion and scaling to unbounded scenes with fixed memory. It differs from global optimization approaches by dynamically allocating new local fields as the camera trajectory extends, preventing global failure from pose outliers and maintaining high resolution around the camera path. Unlike Mega-NeRF, it does not require pre-partitioned space, and unlike Mip-NeRF 360, it jointly estimates poses rather than relying on pre-computed ones.

### [Mip-NeRF RGB-D Volume Rendering and Loss](17857-mip-nerf-rgb-d-volume-rendering-and-loss.md)
The method minimizes geometric and photometric loss together on a set of frames using differentiable rendering functions to optimize network weights.

#### Smooth Proposal Supervision
A method to resolve 'z-aliasing' artifacts in proposal networks by replacing the discrete interlevel loss with a continuous, smooth loss function that penalizes partial overlaps between proposal and NeRF histograms [S3]. It identifies z-aliasing artifacts where the proposal MLP learns a non-smooth mapping, causing rays to 'skip' scene content, especially when using iNGP backends for rapid optimization [S3]. The original Mip-NeRF 360 interlevel loss assigns equivalent loss to NeRF and proposal histogram bins regardless of overlap extent, penalizing only complete failures to overlap, leading to non-smooth density maps [S3]. The new loss prevents the proposal network from learning non-smooth mappings that occur with the previous loss, particularly when the proposal network has high capacity (e.g., iNGP backend) [S3].

#### MipNeRF-360 Depth Rendering
An unbounded extension of MipNeRF that uses a contract function to parameterize 3D Euclidean space within a ball of radius 2, allowing direct rendering of images and depth maps in the contracted space [S4]. It renders images and depth maps in the contracted space using standard NeRF equations (Eq. 1 and Eq. 2) [S4].

#### Simple Multi-Space Radiance Field (Ablation Variant)
A simplified module that outputs K scalars and K RGB-G vectors (4 dimensions: 3 RGB + 1 gating scaler) without efficient multi-space composition strategy, leading to over-smoothing [S5]. Rendering uses the same integral equation as NeRF to accumulate the K RGB-G vectors $\{\hat{c}_{ki}\}$ and get the RGB-G maps of each sub-space [S5]. It splits the RGB-G maps into RGB maps and sub-space composition maps (gating maps), applies softmax over the gating maps along the sub-space channels, and uses it as the weight to compose sub-space RGB maps to form the final rendering results [S5]. The number of sub-spaces $K$ is set to 8 [S5]. The lack of an efficient multi-space composition strategy introduces an over-smoothing problem [S5].

#### Mip-NeRF360 Evaluation Configuration
Uses the scene warping scheme identical to DVGO and Mip-NeRF360 [S6]. Keeps other components (positional encoding, point sampling, etc.) identical to TensoRF [S6]. Uses 2 scales [S6]. Mip-NeRF360 results (Table 9): garden PSNR 24.13, room PSNR 28.11, Model Size avg 12.6MB [S6].

#### MipNeRF-360 Distillation Mode
Implementing the MipNeRF-360 distillation mode replaces the default settings with a light proposal MLP, reducing training and inference time [S7]. The distillation mode runs 30% faster in training and 40% faster in inference than default settings, with a PSNR drop of 0.33 [S7].

## Operating characteristics
*   **Time Complexity:** The multi-stage sampling (2 proposal stages + 1 NeRF stage) increases per-iteration cost compared to single-stage NeRF, but the proposal network reduces the number of NeRF evaluations needed for high quality [S1]. The distillation mode variant reduces this overhead, running 30% faster in training and 40% faster in inference [S7].
*   **Memory Complexity:** The scene warping function allows the method to handle unbounded scenes without increasing memory usage proportional to scene size, as all geometry is confined to a unit sphere [S2]. This makes it suitable for large-scale urban scenes where standard 3DGS or NeRF would exhaust video memory [S2].
*   **Performance:** On the Mip-NeRF 360 dataset, the method achieves a PSNR of 24.13 on the "garden" scene and 28.11 on the "room" scene, with an average model size of 12.6MB [S6].
*   **Strengths:** Robust to unbounded scenes, prevents aliasing artifacts through specific loss formulations and warping, and provides stable optimization via the Charbonnier loss and gradient clipping [S1, S2].
*   **Limitations:** Requires pre-computed camera poses (e.g., via COLMAP) [S1]. The warping function introduces a discontinuity in the derivative at the unit sphere boundary [S2].

## Background
Mip-NeRF 360 was introduced by Mildenhall et al. in 2022 [S1]. It extends the original mip-NeRF approach to handle "in the wild" photo collections with large scale variations [S1]. The optimization setup, including the specific MLP architectures, learning rate schedule, and scene warping, was designed to stabilize training for unbounded scenes [S1]. The scene warping technique, often referred to as Gaussian Position Encoding in subsequent works like HO-Gaussian, was inspired by this method to handle large-scale urban scenes [S2].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] HO-Gaussian: Hybrid Optimization of 3D Gaussian Splatting for Urban Scenes, https://doi.org/10.1007/978-3-031-73027-6_2, fetched 2026-10-26
[S3] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
[S4] Digging into Depth Priors for Outdoor Neural Radiance Fields, https://doi.org/10.1145/3581783.3612306, fetched 2026-10-30
[S5] MS-NeRF: Multi-Space Neural Radiance Fields, https://doi.org/10.1109/tpami.2025.3540074, fetched 2026-10-26
[S6] Strivec: Sparse Tri-Vector Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01611, fetched 2026-10-26
[S7] S-NeRF: Neural Radiance Fields for Street Views, https://doi.org/10.48550/arxiv.2303.00749, fetched 2026-10-29
[S8] PDF: Point Diffusion Implicit Function for Large-scale Scene Neural Representation, https://doi.org/10.48550/arXiv.2311.01773, fetched 2026-10-29 (unofficial)
[S9] PDF: Point Diffusion Implicit Function for Large-scale Scene Neural Representation, https://doi.org/10.48550/arXiv.2311.01773, fetched 2026-10-29
