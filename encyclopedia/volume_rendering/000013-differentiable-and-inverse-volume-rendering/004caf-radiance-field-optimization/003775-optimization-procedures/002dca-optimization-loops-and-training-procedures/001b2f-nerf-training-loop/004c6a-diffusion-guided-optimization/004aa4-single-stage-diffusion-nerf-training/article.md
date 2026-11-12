# Single-Stage Diffusion NeRF Training

Also called: Joint optimization of NeRF and Diffusion Prior, SSDNeRF training

## Overview
Single-Stage Diffusion NeRF Training (SSDNeRF) is a method for jointly optimizing a differentiable NeRF decoder, a diffusion prior, and scene latent codes in a single unified training loop [S1]. It solves the problem of biased and noisy scene representations that arise in two-stage methods, where the prior is ignored during the initial autoencoder training phase [S1]. Unlike [DreamFusion](12087-dreamfusion.md), which optimizes a single NeRF from scratch using Score Distillation Sampling (SDS), SSDNeRF operates on a latent code space (e.g., triplanes) and uses a cached gradient strategy to balance the high computational cost of the diffusion loss with the rendering loss [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A dataset of observed images $\{y_{ij}^{gt}\}$ with corresponding rays $\{r_{ij}^{gt}\}$, where $i$ indexes the scene and $j$ indexes the ray/pixel [S1].
*   **Outputs:** Optimized scene codes $\{x_i\}$, diffusion prior parameters $\phi$, and NeRF decoder parameters $\psi$ [S1].

**Data Structures:**
*   **Scene Codes ($x_i$):** Latent representations of the 3D scene. In the source implementation, these are triplanes with shape $\mathbb{R}^{3 \times 6 \times 128 \times 128}$ [S1].
*   **Diffusion Prior ($p_\phi$):** A Gaussian diffusion model (U-Net based) trained on scene codes [S1].
*   **NeRF Decoder ($p_\psi$):** A decoder mapping scene codes and ray coordinates to pixel colors, typically using SH encoding and linear layers [S1].

**Formulas:**
The total loss function combines rendering fidelity and diffusion prior consistency [S1]:
$$ \mathcal{L} = \lambda_{rend} \mathcal{L}_{rend}(\{x_i\}, \psi) + \lambda_{diff} \mathcal{L}_{diff}(\{x_i\}, \phi) $$
where $\mathcal{L}_{rend}$ is the negative log-likelihood of the rendered images given the observations, and $\mathcal{L}_{diff}$ is the approximate upper bound of the diffusion negative log-likelihood (Score Distillation) [S1].

**Parameters:**
*   $\lambda_{rend}$: Rendering loss weight. Calculated as $\frac{c_{rend} (1 - e^{-0.1 N_v})}{N_v}$, where $c_{rend}$ is a constant scale and $N_v$ is the number of available views [S1]. This calibrates the loss to prevent linear scaling with ray count [S1].
*   $\lambda_{diff}$: Diffusion loss weight. Calculated as $\frac{c_{diff}}{\text{EMA} \|x_i\|_F^2}$, where $c_{diff}$ is a constant scale and $\text{EMA} \|x_i\|_F^2$ is the exponential moving average of the scene codes' Frobenius norms [S1].
*   $K_{out}$: Number of outer iterations [S1].
*   $K_{in}$: Number of inner iterations per outer step [S1].
*   $B_{sc}$: Batch of scenes [S1].
*   $B_{ray}$: Batch of rays [S1].

**Steps:**
1.  Initialize scene codes $\{x_i\}$, diffusion prior parameters $\phi$, and NeRF decoder parameters $\psi$ [S1].
2.  Loop for $K_{out}$ outer iterations:
    a. Sample a batch of scenes $i \in B_{sc}$ [S1].
    b. Compute gradients for the diffusion loss $\mathcal{L}_{diff}$ with respect to $\phi$ and $\{x_i\}$: $g_\phi, g_{x}^{diff} \leftarrow \nabla_{\phi, \{x_i\}_{B_{sc}}} \lambda_{diff} \mathcal{L}_{diff}$ [S1].
    c. Update diffusion parameters $\phi$ using Adam optimizer with gradient $g_\phi$: $\phi \leftarrow \phi - \text{Adam}(g_\phi)$ [S1].
    d. Cache the diffusion gradient for the scene codes: $g_{x}^{diff}$ [S1].
    e. Loop for $K_{in}$ inner iterations:
        i. Sample a batch of rays $j \in B_{ray}$ [S1].
        ii. Compute gradients for the rendering loss $\mathcal{L}_{rend}$ with respect to $\{x_i\}$: $g_{x}^{rend} \leftarrow \nabla_{\{x_i\}_{B_{sc}}} \lambda_{rend} \mathcal{L}_{rend}$ [S1].
        iii. Combine the cached diffusion gradient with the current rendering gradient: $g_{x} \leftarrow g_{x}^{rend} + g_{x}^{diff}$ [S1].
        iv. Update scene codes $\{x_i\}$ using Adam optimizer with total gradient $g_{x}$: $\{x_i\}_{B_{sc}} \leftarrow \{x_i\}_{B_{sc}} - \text{Adam}(g_{x})$ [S1].
        v. If $k_{in}$ equals $K_{in}$ (last inner iteration):
            1. Compute gradients for decoder parameters $\psi$ with respect to rendering loss: $g_\psi \leftarrow \nabla_\psi \lambda_{rend} \mathcal{L}_{rend}$ [S1].
            2. Update decoder parameters $\psi$ using Adam optimizer with gradient $g_\psi$: $\psi \leftarrow \psi - \text{Adam}(g_\psi)$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
x = initialize_scene_codes() # Shape: [N_scenes, 3, 6, 128, 128]
phi = initialize_diffusion_prior() # U-Net parameters
psi = initialize_nerf_decoder() # Decoder parameters
ema_norm_sq = 0.0
ema_decay = 0.99 # Not specified in source, assumed standard EMA

for k_out in range(K_out):
    # Outer loop: Update Diffusion Prior and Cache Prior Gradients
    batch_scenes = sample_batch_scenes(B_sc)
    
    # Compute diffusion loss and gradients
    # L_diff is the score distillation loss for the batch
    L_diff = compute_diffusion_loss(x[batch_scenes], phi)
    g_phi, g_x_diff = compute_gradients(L_diff, [phi, x[batch_scenes]])
    
    # Update Diffusion Prior
    phi = adam_update(phi, g_phi)
    
    # Cache the prior gradient for scene codes
    # Note: In practice, this gradient is accumulated or stored for reuse
    cached_g_x_diff = g_x_diff
    
    # Inner loop: Update Scene Codes and Decoder
    for k_in in range(K_in):
        batch_rays = sample_batch_rays(B_ray)
        
        # Compute rendering loss and gradients w.r.t scene codes
        # L_rend depends on x and psi
        L_rend = compute_rendering_loss(x[batch_scenes], psi, batch_rays)
        g_x_rend = compute_gradients(L_rend, x[batch_scenes])
        
        # Combine gradients
        # Lambda weights are computed dynamically
        lambda_rend = compute_lambda_rend(N_v, c_rend)
        lambda_diff = compute_lambda_diff(ema_norm_sq, c_diff)
        
        # Scale gradients by their respective lambdas if not included in loss computation
        # Assuming L_rend and L_diff are already weighted in the source logic
        # The source says: gx <- gx_rend + g_x_diff
        # where gx_rend is grad of lambda_rend * L_rend
        # and g_x_diff is grad of lambda_diff * L_diff
        
        g_x_total = g_x_rend + cached_g_x_diff
        
        # Update Scene Codes
        x[batch_scenes] = adam_update(x[batch_scenes], g_x_total)
        
        # Update EMA of scene code norms for lambda_diff
        # Compute Frobenius norm squared of current codes
        current_norm_sq = compute_frobenius_norm_squared(x[batch_scenes])
        ema_norm_sq = ema_decay * ema_norm_sq + (1 - ema_decay) * current_norm_sq
        
        # Update Decoder only on last inner iteration
        if k_in == K_in - 1:
            g_psi = compute_gradients(L_rend, psi)
            psi = adam_update(psi, g_psi)
```

**Edge Cases and Pitfalls:**
*   **Computational Cost:** The diffusion loss $\mathcal{L}_{diff}$ is significantly more expensive to evaluate than the rendering loss $\mathcal{L}_{rend}$ [S1]. The inner/outer loop structure with gradient caching is essential to mitigate this [S1].
*   **Weight Calibration:** The rendering loss weight $\lambda_{rend}$ must be calibrated by the number of views $N_v$ to prevent the loss from scaling linearly with the number of rays, which would bias the optimization [S1].
*   **Prior Bias:** Two-stage methods that ignore the prior during autoencoder training result in biased and noisy scene codes [S1]. Single-stage training prevents this by constraining codes with both terms simultaneously [S1].

## Operating characteristics
*   **Time Complexity:** The method is computationally intensive due to the diffusion model. The gradient caching strategy reduces the number of diffusion passes, but it remains slower than standard NeRF training [S1].
*   **Memory Complexity:** Requires storing scene codes, diffusion model parameters, and NeRF decoder parameters. The triplane representation ($\mathbb{R}^{3 \times 6 \times 128 \times 128}$) is memory-efficient compared to voxel grids [S1].
*   **Strengths:** Produces more generalizable and less noisy scene codes compared to two-stage methods [S1]. Handles a wide spectrum from single-view to dense observations [S1].
*   **Limitations:** High computational cost for the diffusion prior [S1]. Requires careful tuning of empirical weights $c_{rend}$ and $c_{diff}$ [S1].
*   **When to use:** When high-quality 3D generation and reconstruction are needed, and computational resources allow for joint optimization [S1].
*   **When not to use:** When real-time performance is critical or computational resources are limited [S1].

## Background
Single-Stage Diffusion NeRF was introduced in the paper "Single-Stage Diffusion NeRF: A Unified Approach to 3D Generation and Reconstruction" [S1]. It addresses limitations in previous two-stage generative neural field methods [S1]. The method builds on the concept of Score Distillation Sampling [S1] and integrates it with a differentiable NeRF decoder in a unified training framework.

## Sources
[S1] Single-Stage Diffusion NeRF: A Unified Approach to 3D Generation and Reconstruction, https://doi.org/10.1109/iccv51070.2023.00229, fetched 2026-10-29
