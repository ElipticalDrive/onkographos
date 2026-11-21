# Adaptive Time-step Training Framework

Also called: Pretraining-based Adaptive Time-step Adjustment (PATA), PATA: Adaptive Time-step Training Framework, temporal-budget learning, Adaptive Time-step Training for Spike-Based Neural Radiance Fields, Adaptive Time-step Training

## Overview
Adaptive Time-step Training (PATA) is a training strategy for spike-based Neural Radiance Fields (NeRF) that reduces inference time steps by learning a scene-wise target time step $t^*$ while maintaining rendering quality through distillation and loss functions [S1, unofficial]. Unlike sample-level or layer-level dynamic inference strategies which cause irregular execution, PATA uses a scene-wise temporal budget to preserve the parallel rendering pattern required by dense ray sampling [S1, unofficial]. It converts time-step selection from a manually tuned hyperparameter into a learned scene-wise budget, reducing temporal computation while preserving rendering behavior [S1, unofficial].

## How it works
PATA is a two-stage training framework that learns a scene-wise temporal budget and applies a shared target time step to all sampling points within a scene [S1, unofficial]. It parameterizes the target inference time step as a trainable variable shared by all ray samples within a scene [S1, unofficial].

### Inputs and Outputs
*   **Inputs:** Training rays and ground-truth colors $\mathcal{D}$, maximum time step $T$, first-stage iterations $N_1$, second-stage iterations $N_2$, loss weights $\alpha$ and $\beta$ [S1, unofficial].
*   **Outputs:** Optimized network parameters $\Theta$, PLIF decay factors $\tau$, and scene-wise target time step $t^*_r$ [S1, unofficial].

### Data Structures
*   **SNN-NeRF Parameters $\Theta$:** Weights and biases of the spike-based neural network (e.g., INGP-NeRF or TensoRF backbone) [S1, unofficial].
*   **PLIF Decay Factors $\tau$:** Parameters controlling the decay in Parametric Leaky Integrate-and-Fire neurons [S1, unofficial].
*   **Target Time Step $t^*$:** A single learnable scalar variable representing the temporal budget for a specific scene [S1, unofficial].
*   **Rendered Colors $C$:** Output color images for different time steps ($C_T, C_{t^*_r}$, etc.) [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pata_training(
    dataset, max_time_step=8, stage1_iters=10000, stage2_iters=30000,
    alpha=1e-6, beta=5e-7, gamma0=5e-5, gamma1=5e-2, gamma2=1e-6
):
    # Initialize parameters [G] Standard PyTorch initialization
    theta = torch.nn.Parameter(torch.randn_like(initial_weights))
    tau = torch.nn.Parameter(torch.ones_like(initial_tau))
    t_star = torch.nn.Parameter(torch.tensor(float(max_time_step - 1.0), requires_grad=True))
    optimizer = torch.optim.Adam([theta, tau, t_star])

    # Stage 1: Pretraining
    for k in range(stage1_iters):
        batch = next(iter(dataset))
        # [S1, unofficial] Unroll spike-based NeRF for T time steps to render full-step output C_T
        C_T = render_spike_nerf(batch, theta, tau, T=max_time_step)
        
        mse_loss = torch.mean((C_T - batch.gt) ** 2)
        l_consist = gamma0 * torch.mean((C_T - batch.gt) ** 2) + gamma1 * (1 - torch.cosine_similarity(C_T.flatten(), batch.gt.flatten(), dim=0))
        sigma = get_density_predictions(theta) # [S1, unofficial] Extract density predictions sigma_i
        l_cauchy = gamma2 * torch.sum(torch.log(1 + 2 * sigma))
        
        l_render = mse_loss + l_consist + l_cauchy
        optimizer.zero_grad()
        l_render.backward()
        optimizer.step()

    # Stage 2: Adaptive Time-step Optimization
    current_max_t = 2 # Max Time-Step Scaling: start reduced to save memory
    for k in range(stage2_iters):
        if k == 1: current_max_t = max_time_step # Restore to T once sampling stabilizes
        
        batch = next(iter(dataset))
        
        t_hat_star = torch.clamp(t_star, 1.0, float(current_max_t))
        t_star_r = torch.round(t_hat_star)
        t_star_f = torch.floor(t_hat_star)
        t_star_c = torch.ceil(t_hat_star)
        delta = t_hat_star - t_star_f
        
        # [S1, unofficial] Unroll for T time steps to obtain teacher and adjacent step outputs
        C_T = render_spike_nerf(batch, theta, tau, T=current_max_t)
        C_t_f = render_spike_nerf(batch, theta, tau, T=int(t_star_f))
        C_t_c = render_spike_nerf(batch, theta, tau, T=int(t_star_c))
        C_t_r = render_spike_nerf(batch, theta, tau, T=int(t_star_r))
        
        M_t_f = torch.mean((C_t_f - batch.gt) ** 2)
        M_t_c = torch.mean((C_t_c - batch.gt) ** 2)
        M_smooth = (1 - delta) * M_t_f + delta * M_t_c
        
        l_consist_smooth = gamma0 * torch.mean((C_t_r - batch.gt) ** 2) + gamma1 * (1 - torch.cosine_similarity(C_t_r.flatten(), batch.gt.flatten(), dim=0))
        l_cauchy = gamma2 * torch.sum(torch.log(1 + 2 * get_density_predictions(theta)))
        l_render_hat = M_smooth + l_consist_smooth + l_cauchy
        
        # L_extra approximation: energy of weights/colors after t_star_r
        w_extra = compute_weight_energy_after_t_star(theta, t_star_r)
        c_extra = compute_color_energy_after_t_star(theta, t_star_r)
        w_norm = torch.norm(theta)
        c_norm = torch.norm(C_T)
        l_extra = (w_extra / (w_norm**2 + 1e-8)) + (c_extra / (c_norm**2 + 1e-8)) + torch.exp(tau)
        
        M_T = torch.mean((C_T - batch.gt) ** 2)
        l_penalty = torch.stop_gradient(M_T / (M_smooth + 1e-8)) * torch.exp(t_hat_star)
        l_budg = alpha * l_extra + beta * l_penalty
        
        l_distill = torch.mean((C_T - C_t_r) ** 2)
        l_total = l_render_hat + l_budg + l_distill
        
        optimizer.zero_grad()
        l_total.backward()
        optimizer.step()

    t_star_r_final = torch.round(t_star)
    return theta, tau, t_star_r_final
```

### Parameters
*   **$\gamma_0$:** Coefficient for L2 consistency loss. Set to $5 \times 10^{-5}$ [S1, unofficial].
*   **$\gamma_1$:** Coefficient for cosine similarity loss. Set to $5 \times 10^{-2}$ [S1, unofficial].
*   **$\gamma_2$:** Coefficient for Cauchy regularization. Set to $1 \times 10^{-6}$ [S1, unofficial].
*   **$\alpha$:** Hyperparameter for $L_{extra}$. Set to $1 \times 10^{-6}$ [S1, unofficial].
*   **$\beta$:** Loss weight for $L_{penalty}$. Controls the compression pressure [S1, unofficial].
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
*   **When to Use:** When using spike-based NeRF backbones and seeking to reduce inference energy while maintaining quality [S1, unofficial].

## Background
*   **Origin:** Proposed by Ranxi Lin, Canming Yao, Jiayi Li, et al. in "Adaptive Time-step Training for Spike-Based Neural Radiance Fields" (2025) [S1, unofficial].
*   **History:** Developed to address the coupling of learning a reliable radiance field and reducing its temporal budget in spike-based NeRF, which makes optimization prone to unstable early truncation [S1, unofficial].
*   **Usage:** Validated on INGP-NeRF (hash-grid) and TensoRF (tensor-factorized) backbones [S1, unofficial].

## Sources
[S1] Adaptive Time-step Training for Enhancing Spike-Based Neural Radiance Fields, https://arxiv.org/abs/2507.23033v2, fetched 2026-10-27 (unofficial)
