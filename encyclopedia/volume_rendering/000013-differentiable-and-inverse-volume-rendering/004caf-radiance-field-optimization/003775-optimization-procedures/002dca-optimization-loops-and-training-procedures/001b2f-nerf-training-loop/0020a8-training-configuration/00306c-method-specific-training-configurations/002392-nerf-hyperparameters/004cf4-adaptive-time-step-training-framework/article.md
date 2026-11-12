# Adaptive Time-step Training Framework

Also called: Pretraining-based Adaptive Time-step Adjustment (PATA), PATA: Adaptive Time-step Training Framework, temporal-budget learning, Adaptive Time-step Training for Spike-Based Neural Radiance Fields

## Overview
Adaptive Time-step Training (PATA) is a training strategy for spike-based Neural Radiance Fields (NeRF) that reduces inference time steps by learning a scene-wise target time step $t^*$ while maintaining rendering quality through distillation and loss functions [S1, unofficial]. Unlike sample-level or layer-level dynamic inference strategies which cause irregular execution, PATA uses a scene-wise temporal budget to preserve the parallel rendering pattern required by dense ray sampling [S1, unofficial]. It converts time-step selection from a manually tuned hyperparameter into a learned scene-wise budget, reducing temporal computation while preserving rendering behavior [S1, unofficial].

## How it works
PATA is a two-stage training framework that learns a scene-wise temporal budget and applies a shared target time step to all sampling points within a scene [S1, unofficial]. It parameterizes the target inference time step as a trainable variable shared by all ray samples within a scene [S1, unofficial].

### Inputs and Outputs
*   **Inputs:** Training rays and ground-truth colors $\mathcal{D}$, maximum time step $T$, first-stage iterations $N_1$, second-stage iterations $N_2$, loss weights $\alpha$ and $\beta$.
*   **Outputs:** Optimized network parameters $\Theta$, PLIF decay factors $\tau$, and scene-wise target time step $t^*_r$.

### Data Structures
*   **SNN-NeRF Parameters $\Theta$:** Weights and biases of the spike-based neural network (e.g., INGP-NeRF or TensoRF backbone).
*   **PLIF Decay Factors $\tau$:** Parameters controlling the decay in Parametric Leaky Integrate-and-Fire neurons.
*   **Target Time Step $t^*$:** A single learnable scalar variable representing the temporal budget for a specific scene.
*   **Rendered Colors $C$:** Output color images for different time steps ($C_T, C_{t^*_r}$, etc.).

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def pata_training(
    training_data: Dataset,
    max_time_step: int = 8,
    stage1_iters: int = 10000,
    stage2_iters: int = 30000,
    alpha: float = 1e-6,
    beta: float = 5e-7,
    gamma0: float = 5e-5,
    gamma1: float = 5e-2,
    gamma2: float = 1e-6,
    surrogate_alpha: float = 4.0
):
    """
    Pretraining-based Adaptive Time-step Adjustment (PATA) training loop.
    
    Args:
        training_data: Dataset of rays and ground truth colors.
        max_time_step (T): Maximum temporal budget for unrolling.
        stage1_iters (N1): Number of iterations for pretraining.
        stage2_iters (N2): Number of iterations for adaptive optimization.
        alpha: Weight for L_extra in temporal budget loss.
        beta: Weight for L_penalty in temporal budget loss (controls compression).
        gamma0: Coefficient for L2 consistency loss.
        gamma1: Coefficient for cosine similarity loss.
        gamma2: Coefficient for Cauchy regularization.
        surrogate_alpha: Hyperparameter for surrogate gradient of spike activation.
        
    Returns:
        theta: Optimized network parameters.
        tau: Optimized PLIF decay factors.
        t_star_r: Learned scene-wise target time step.
    """
    
    # Initialize model parameters
    theta = initialize_snn_nerf_parameters()
    tau = initialize_plif_decay_factors()
    
    # Stage 1: Pretraining
    # Train a fixed full-step spike-based renderer at maximum time step T
    for k in range(1, stage1_iters + 1):
        batch = sample_mini_batch(training_data)
        
        # Unroll spike-based NeRF for T time steps
        # C_T is the rendered color using full temporal budget
        C_T = render_spike_nerf(batch, theta, tau, T=max_time_step)
        
        # Compute first-stage rendering loss
        # L_Render(T) = MSE + Consistency + Cauchy Regularization
        mse_loss = mean_squared_error(C_T, batch.gt)
        l_consist = compute_consistency_loss(C_T, batch.gt, gamma0, gamma1)
        l_cauchy = compute_cauchy_regularization(theta, gamma2)
        
        l_render_t = mse_loss + l_consist + l_cauchy
        
        # Update parameters
        l_render_t.backward()
        optimizer_step(theta, tau)
        zero_grad()

    # Stage 2: Adaptive Time-step Optimization
    # Initialize learnable target time step
    t_star = torch.tensor(max_time_step - 1.0, requires_grad=True)
    
    # Max Time-Step Scaling: Start with reduced T to save memory, then restore
    current_max_t = 2
    for k in range(1, stage2_iters + 1):
        # Restore max time step after initial epoch if needed
        if k == 1:
            current_max_t = max_time_step
            
        batch = sample_mini_batch(training_data)
        
        # Clamp and Round target time step
        t_hat_star = torch.clamp(t_star, 1.0, float(current_max_t))
        t_star_r = torch.round(t_hat_star)
        
        # Compute fractional part for smoothing
        t_star_f = torch.floor(t_hat_star)
        t_star_c = torch.ceil(t_hat_star)
        delta = t_hat_star - t_star_f
        
        # Unroll for T time steps to get teacher and adjacent step outputs
        # Note: Even though we optimize for t_star_r, we need C_T for distillation
        # and C_t_star_f, C_t_star_c for smoothing.
        C_T = render_spike_nerf(batch, theta, tau, T=current_max_t)
        C_t_star_f = render_spike_nerf(batch, theta, tau, T=int(t_star_f))
        C_t_star_c = render_spike_nerf(batch, theta, tau, T=int(t_star_c))
        C_t_star_r = render_spike_nerf(batch, theta, tau, T=int(t_star_r))
        
        # Compute Smoothed Rendering Loss
        # M_smooth(t_hat*) = (1 - delta) * M_t_star_f + delta * M_t_star_c
        M_t_star_f = mean_squared_error(C_t_star_f, batch.gt)
        M_t_star_c = mean_squared_error(C_t_star_c, batch.gt)
        M_smooth = (1 - delta) * M_t_star_f + delta * M_t_star_c
        
        # Compute L_Render(t_hat*) by replacing M_T in Eq 13 with M_smooth
        # L_Render(t_hat*) = M_smooth + L_consist(t_hat*) + L_Cauchy
        # Note: Consistency loss is typically computed against GT. 
        # The source implies using the smoothed error for the main loss term.
        l_consist_smooth = compute_consistency_loss(C_t_star_r, batch.gt, gamma0, gamma1)
        l_cauchy = compute_cauchy_regularization(theta, gamma2)
        
        l_render_t_hat = M_smooth + l_consist_smooth + l_cauchy
        
        # Compute Temporal Budget Loss
        # L_extra = (||w_{i,t>t_r*}||^2 / ||W_points||^2) + (||c_{t>t_r*}||^2 / ||C_T||^2) + exp(tau)
        # Note: Exact implementation of L_extra depends on specific spike-based 
        # weight/color accumulation definitions. We assume access to weights w and colors c 
        # for time steps > t_star_r.
        w_extra = compute_weight_energy_after_t_star(theta, t_star_r)
        c_extra = compute_color_energy_after_t_star(theta, t_star_r)
        w_norm = compute_norm_weights(theta)
        c_norm = compute_norm_colors(C_T)
        
        l_extra = (w_extra / (w_norm**2 + 1e-8)) + (c_extra / (c_norm**2 + 1e-8)) + torch.exp(tau)
        
        # L_penalty = sg(M_T / M_smooth(t_hat*)) * exp(t_hat*)
        # sg denotes stop-gradient
        l_penalty = torch.stop_gradient(M_T / (M_smooth + 1e-8)) * torch.exp(t_hat_star)
        
        l_budg = alpha * l_extra + beta * l_penalty
        
        # Compute Temporal Distillation Loss
        # L_distill = || C_T - C_t_star_r ||^2_2
        l_distill = mean_squared_error(C_T, C_t_star_r)
        
        # Total Objective
        l_total = l_render_t_hat + l_budg + l_distill
        
        # Update parameters
        l_total.backward()
        optimizer_step(theta, tau, t_star)
        zero_grad()

    # Inference
    t_star_r_final = torch.round(t_star)
    return theta, tau, t_star_r_final

def compute_consistency_loss(C_pred, gt, gamma0, gamma1):
    l2 = gamma0 * torch.mean((C_pred - gt)**2)
    # Cosine similarity
    dot_product = torch.sum(C_pred * gt)
    norm_pred = torch.norm(C_pred)
    norm_gt = torch.norm(gt)
    cosine_sim = dot_product / (norm_pred * norm_gt + 1e-8)
    l_cos = gamma1 * (1 - cosine_sim)
    return l2 + l_cos

def compute_cauchy_regularization(theta, gamma2):
    # sigma_i are density predictions
    # Assuming access to density predictions sigma from the model
    sigma = get_density_predictions(theta)
    l_cauchy = gamma2 * torch.sum(torch.log(1 + 2 * sigma))
    return l_cauchy

def render_spike_nerf(batch, theta, tau, T):
    # Placeholder for the specific spike-based unrolling logic
    # This function unrolls the SNN for T time steps
    # Returns the rendered color image C
    pass

def compute_weight_energy_after_t_star(theta, t_star_r):
    # Computes squared norm of weights contributing to time steps > t_star_r
    pass

def compute_color_energy_after_t_star(theta, t_star_r):
    # Computes squared norm of colors contributing to time steps > t_star_r
    pass

def compute_norm_weights(theta):
    # Computes total norm of weights
    pass

def compute_norm_colors(C_T):
    # Computes total norm of full-step color output
    pass
```

### Parameters
*   **$\gamma_0$:** Coefficient for L2 consistency loss. Set to $5 \times 10^{-5}$ [S1, unofficial].
*   **$\gamma_1$:** Coefficient for cosine similarity loss. Set to $5 \times 10^{-2}$ [S1, unofficial].
*   **$\gamma_2$:** Coefficient for Cauchy regularization. Set to $1 \times 10^{-6}$ [S1, unofficial].
*   **$\alpha$:** Hyperparameter for $L_{extra}$. Set to $1 \times 10^{-6}$ [S1, unofficial].
*   **$\beta$:** Loss weight for $L_{penalty}$. Controls the compression pressure. Typical value: $5 \times 10^{-7}$ [S1, unofficial].
*   **$T$:** Maximum time step. Typical value: 8 [S1, unofficial].
*   **$N_1$:** First-stage iterations. Typical value: 10,000 [S1, unofficial].
*   **$N_2$:** Second-stage iterations. Typical value: 30,000 [S1, unofficial].
*   **Surrogate Gradient $\alpha$:** Predefined hyperparameter for surrogate gradient of spike activation. Defaults to 4 [S1, unofficial].
*   **Hybrid Input Mode:** Default input mechanism combining stable accumulation with controllable temporal weighting. Decay mode scales input contribution at each time step by $1/\tau$. No-decay mode avoids attenuation but lacks explicit temporal control. Hybrid mode uses weight function:
    $$
    w(t, \tau) = \begin{cases} 
    (1 - \tau^{-1})^{T-t-1}, & \text{if } t = 0 \\
    \frac{1}{\tau} (1 - \tau^{-1})^{T-t-1} \left( \sum_{j=0}^{T-1} (1 - \tau^{-1})^j \right)^{-1}, & \text{if } t > 0 
    \end{cases}
    $$
    This maintains $\sum_{t=0}^{T-1} w(t, \tau) = 1$ for all $\tau > 1$ [S1, unofficial].

### Formulas
*   **Consistency Loss:**
    $$
    \mathcal{L}_{consist}(T) = \gamma_0 \|C_T - gt\|^2 + \gamma_1 \left( 1 - \frac{C_T \cdot gt}{\|C_T\| \|gt\| + \epsilon} \right)
    $$
    [S1, unofficial]
*   **Cauchy Regularization:**
    $$
    \mathcal{L}_{Cauchy} = \gamma_2 \sum_i \log(1 + 2\sigma_i)
    $$
    where $\sigma_i$ are density predictions [S1, unofficial].
*   **Extra Energy Loss:**
    $$
    \mathcal{L}_{extra} = \frac{\|w_{i,t>t^*_r}\|^2}{\|W_{points}\|^2} + \frac{\|c_{t>t^*_r}\|^2}{\|C_T\|^2} + e^\tau
    $$
    [S1, unofficial]
*   **Temporal Budget Loss:**
    $$
    \mathcal{L}_{budg} = \alpha \mathcal{L}_{extra} + \beta \mathcal{L}_{penalty}
    $$
    [S1, unofficial]
*   **Temporal Distillation Loss:**
    $$
    \mathcal{L}_{distill} = \| C_T - C_{t^*_r} \|^2_2
    $$
    [S1, unofficial]
*   **Smoothed Rendering Loss:**
    $$
    M_{smooth}(\hat{t}^*) = (1 - \delta) M_{t^*_f} + \delta M_{t^*_c}
    $$
    where $t^*_f = \lfloor \hat{t}^* \rfloor$, $t^*_c = \lceil \hat{t}^* \rceil$, and $\delta = \hat{t}^* - t^*_f$ [S1, unofficial].
*   **Penalty Loss:**
    $$
    \mathcal{L}_{penalty} = \text{sg}\left( \frac{M_T}{M_{smooth}(\hat{t}^*)} \right) e^{\hat{t}^*}
    $$
    where sg denotes stop-gradient [S1, unofficial].
*   **Total Second-Stage Objective:**
    $$
    \mathcal{L} = \mathcal{L}_{Render}(\hat{t}^*) + \mathcal{L}_{budg} + \mathcal{L}_{distill}
    $$
    [S1, unofficial]

### Edge Cases and Pitfalls
*   **Abrupt Changes:** Abrupt changes in the training objective when $t^*$ crosses rounding boundaries are prevented by using the smoothed loss $M_{smooth}(\hat{t}^*)$ which evaluates the local quality landscape between adjacent integer time steps [S1, unofficial].
*   **High Memory Cost:** High memory cost during early training due to dense unfiltered sampling points is handled by Max Time-Step Scaling: starting with a reduced maximum time step (e.g., 2) and restoring to $T$ once sampling stabilizes [S1, unofficial].
*   **Loss Component Sensitivity:** Removing $\mathcal{L}_{extra}$ drives the time step to 1.625 with a PSNR drop. Removing $\mathcal{L}_{penalty}$ increases the time step to 6.625. Removing $\mathcal{L}_{budg}$ weakens control [S1, unofficial].

## Operating characteristics
*   **Time and Memory Complexity:** The method reduces inference time steps, thereby reducing temporal computation. However, during training, it requires unrolling to the maximum time step $T$ for distillation and smoothing, which increases memory usage during early training stages [S1, unofficial].
*   **Measured Performance:**
    *   On INGP-NeRF: Reduces average time step from $T=8$ to 3.625 (SyntheticNeRF) and 2.0 (TensoRF variant), achieving up to 57.57% energy reduction on SyntheticNeRF and 37.46% on Mip-NeRF 360 [S1, unofficial].
    *   On TensoRF: Achieves up to 40.66% energy reduction on SyntheticNeRF and 68.90% on LLFF [S1, unofficial].
    *   PATA reduces estimated inference energy by up to 57.57% on INGP-NeRF and 68.90% on TensoRF across Synthetic-NeRF, Mip-NeRF 360, and LLFF benchmarks [S1, unofficial].
*   **Strengths:** Preserves parallel rendering pattern; backbone-general (validates on INGP-NeRF and TensoRF); stable adaptive time-step training [S1, unofficial].
*   **Limitations:** Requires two-stage training; sensitive to loss weights; early training memory intensive [S1, unofficial].
*   **When to Use:** When using spike-based NeRF backbones and seeking to reduce inference energy while maintaining quality.
*   **When Not to Use:** When non-spike-based NeRF backbones are used without modification; when memory is extremely constrained during early training without scaling strategies [S1, unofficial].

## Background
*   **Origin:** Proposed by Ranxi Lin, Canming Yao, Jiayi Li, et al. in "Adaptive Time-step Training for Spike-Based Neural Radiance Fields" (2025) [S1, unofficial].
*   **History:** Developed to address the coupling of learning a reliable radiance field and reducing its temporal budget in spike-based NeRF, which makes optimization prone to unstable early truncation [S1, unofficial].
*   **Usage:** Validated on INGP-NeRF (hash-grid) and TensoRF (tensor-factorized) backbones [S1, unofficial].

## Sources
[S1] Adaptive Time-step Training for Enhancing Spike-Based Neural Radiance Fields, https://arxiv.org/abs/2507.23033v2, fetched 2026-10-27 (unofficial)
