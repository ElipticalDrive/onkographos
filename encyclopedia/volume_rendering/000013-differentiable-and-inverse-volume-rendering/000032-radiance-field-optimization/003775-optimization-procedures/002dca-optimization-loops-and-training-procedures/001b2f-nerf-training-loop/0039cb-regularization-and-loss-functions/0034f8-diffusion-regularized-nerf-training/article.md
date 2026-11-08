# Diffusion-Regularized NeRF Training

Also called: DDM Regularization, DDM-regularized NeRF, Denoising Diffusion Model Regularization, DiffusioNeRF Regularization, Diffusion prior for NeRF, Diffusion-regularized NeRF, Score matching regularization

## Overview
Diffusion-Regularized NeRF Training mitigates the severe under-constraint of Neural Radiance Fields (NeRFs) when trained with limited input views, which typically leads to artifacts such as "floaters" (semi-transparent material floating in mid-air) and low-quality geometry [S1, S2]. It sets itself apart from siblings like [RegNeRF](8582-regnerf.md) by using a Denoising Diffusion Model (DDM) trained on RGBD patches to directly regularize geometry and color, whereas RegNeRF uses normalizing flows on RGB patches [S1]. Unlike [Depth and Geometry Supervision](8727-depth-and-geometry-supervision.md) which relies on explicit depth sensors or stereo matches, this method learns a probabilistic prior over RGBD distributions to guide the optimization [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of training images $I$ with known camera poses and intrinsics; a pre-trained Denoising Diffusion Model (DDM) $\epsilon_\theta$ trained on RGBD patches from a synthetic dataset (e.g., Hypersim).
*   **Outputs:** Optimized NeRF parameters (density $\sigma$ and color $\mathbf{c}$ fields).

**Data Structures:**
*   **NeRF Representation:** The method uses Instant NGP [S1], which employs multi-scale hash tables to store feature encodings of 3D coordinates in a fixed memory block, allowing varying spatial resolutions and reducing MLP size [S1].
*   **DDM State:** The DDM $\epsilon_\theta(\mathbf{x}_\tau, \tau)$ is a neural network estimating noise $\epsilon$ given a noisy input $\mathbf{x}_\tau$ and time step $\tau$ [S1, S2].

**Numbered Steps:**

1.  **DDM Training (Pre-processing):**
    *   Extract RGBD patches from a synthetic dataset (e.g., Hypersim). Patch sizes of $24 \times 24$ or $48 \times 48$ are used [S1].
    *   Train the DDM to predict the noise $\epsilon$ added during the forward diffusion process. The loss function is:
        $$ \mathbb{E}_{\mathbf{x}_0, \epsilon} \left[ \frac{\beta_\tau}{2\alpha_\tau (1-\bar{\alpha}_\tau)} || \epsilon - \epsilon_\theta \left( \sqrt{\bar{\alpha}_\tau} \mathbf{x}_0 + \sqrt{1 - \bar{\alpha}_\tau} \epsilon, \tau \right) || \right] $$
        where $\epsilon \sim \mathcal{N}(0, I)$, $\alpha_\tau = 1 - \beta_\tau$, and $\bar{\alpha}_\tau = \prod_{s=0}^\tau \alpha_s$ [S1].
    *   Typical training: 650,000 steps, batch size 32 [S1].

2.  **NeRF Optimization Loop:**
    *   Initialize NeRF parameters (density $\sigma$ and color $\mathbf{c}$) [S1].
    *   For each optimization step:
        a.  **Ray Sampling:** Cast rays from random camera poses (and 25% of the time from training poses) [S1]. Sample points $t_i$ along rays $r(t) = o + td$.
        b.  **Field Evaluation:** Evaluate $\sigma(\mathbf{r}(t_i))$ and $\mathbf{c}(\mathbf{r}(t_i), \mathbf{d})$ using the Instant NGP hash tables and MLPs [S1].
        c.  **Volumetric Rendering:** Compute weights $w_i$ and expected color $\mathbf{C}(r)$ and depth $D(r)$:
            $$ \rho(t_i) = 1 - \exp(-\sigma(\mathbf{r}(t_i))(t_{i+1} - t_i)) $$
            $$ T(t_i) = \prod_{j=1}^{i-1} (1 - \rho(t_j)) $$
            $$ w_i = T(t_i)\rho(t_i) $$
            $$ \mathbf{C}(r) \approx \sum_{i=1}^N w_i \mathbf{c}(\mathbf{r}(t_i), \mathbf{d}) + \left ( 1 - \sum_{i=1}^N w_i \right ) \mathbf{c}_\text {bg} $$
            $$ D(r) = \frac {\sum _{i=1}^N w_i t_i}{\sum _{i=1}^N w_i} $$
            where $\mathbf{c}_\text {bg}$ is white [S1].
        d.  **DDM Regularization:**
            i.  Form an RGBD patch from the rendered $\mathbf{C}(r)$, $D(r)$, and weights $w_i$ for the rays passing through a random $48 \times 48$ (or $24 \times 24$) pixel patch [S1].
            ii. If using training poses (25% of time), replace the rendered RGB component with the ground truth image RGB [S1].
            iii. Pass the patch through the DDM to get the noise prediction $\epsilon_\theta$.
            iv. The DDM output is proportional to the negative score function: $\epsilon_\theta(\mathbf{x}_\tau, \tau) \propto -\nabla_\mathbf{x} \log p(\mathbf{x})$ [S1].
        e.  **Gradient Computation:** Compute the total loss gradient:
            $$ \nabla \mathcal{L} = \nabla \mathcal{L}_{\text{photo}} + \lambda_{\text{fg}} \nabla \mathcal{L}_{\text{fg}} + \lambda_{\text{fr}} \nabla \mathcal{L}_{\text{fr}} + \lambda_{\text{dist}} \nabla \mathcal{L}_{\text{dist}} - \lambda_{\text{DDM}} \epsilon_\theta $$
            where $\mathcal{L}_{\text{photo}}$ is the photometric loss, $\mathcal{L}_{\text{fg}}, \mathcal{L}_{\text{fr}}, \mathcal{L}_{\text{dist}}$ are geometric regularizers (foreground, front-to-back, distribution), and $\lambda_{\text{DDM}}$ is the DDM weight [S1].
        f.  **Backpropagation:** Backpropagate $\nabla \mathcal{L}$ to update NeRF parameters [S1].

**Parameters:**
*   **Patch Size:** $48 \times 48$ pixels (enabled by Instant NGP speed); $24 \times 24$ performs better on LLFF but worse on DTU [S1].
*   **Diffusion Time $\tau$:** Schedules from 0.1 to 0 during the first 2500 steps of NeRF optimization. $\bar{\alpha}_\tau = \cos(0.5\pi(\tau + 0.008)/1.008)$ [S1].
*   **$\lambda_{\text{DDM}}$:** Trades off reconstruction accuracy around thin structures against depth smoothness. Max value $1 \times 10^{-4}$ for DTU, $1.5 \times 10^{-5}$ for LLFF, linearly increasing from 0 at step 3000 to max at step 8000 [S1].
*   **NeRF Steps:** 12,000 steps total [S1].
*   **$\lambda_{\text{dist}}$:** 0 for first 2500 steps, then linearly increases [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import math

# Standard math operations assumed available: sqrt, cos, exp, prod, sum, mean

def compute_transmittance_and_opacity(sigma, t, t_next):
    """
    Computes opacity rho and transmittance T for a segment.
    sigma: density at t
    t, t_next: sample positions
    """
    delta = t_next - t
    rho = 1.0 - torch.exp(-sigma * delta)
    return rho

def render_ray(rays_o, rays_d, sigma_fn, color_fn, t_near, t_far, num_samples, bg_color=torch.tensor([1.0, 1.0, 1.0])):
    """
    Renders a single ray using volumetric rendering.
    Returns: color C, depth D, weights w, positions t
    """
    t_vals = torch.linspace(t_near, t_far, num_samples)
    # Add small epsilon to avoid division by zero or degenerate cases
    t_vals = t_vals + (t_vals[1] - t_vals[0]) * 0.5
    
    # Sample densities and colors
    # Note: In Instant NGP, sigma_fn and color_fn handle the hash table lookup internally
    sigma_vals = sigma_fn(rays_o + rays_d * t_vals.unsqueeze(1)) # Shape: [batch, num_samples]
    # Color depends on direction d, which is constant for the ray
    color_vals = color_fn(rays_o + rays_vals * t_vals.unsqueeze(1), rays_d) # Shape: [batch, num_samples, 3]
    
    # Compute opacity and transmittance
    deltas = t_vals[1:] - t_vals[:-1]
    # Pad deltas for the last sample if necessary, or assume t_vals includes endpoints
    # Standard NeRF sampling: t_i are centers, t_{i+1} are boundaries
    # Here we assume t_vals are boundaries for simplicity of implementation logic from source
    # Re-aligning to source formula: t_0:N are samples. 
    # Let's assume t_vals are the sample positions t_i.
    # We need t_{i+1}. We can use the next sample or a fixed step.
    # For implementation from text:
    # rho(t_i) = 1 - exp(-sigma(r(t_i)) * (t_{i+1} - t_i))
    
    # Approximation: use uniform spacing if not provided
    step = (t_far - t_near) / num_samples
    deltas = torch.full_like(t_vals, step)
    deltas[-1] = 0 # Last sample doesn't contribute to opacity in standard discrete sum if not handled, 
                   # but source uses sum w_i. 
                   # Let's stick to the source formula strictly.
    
    # Re-implementation of source formulas (1)-(4)
    # t_0:N are samples. 
    # rho_i = 1 - exp(-sigma_i * (t_{i+1} - t_i))
    # T_i = prod_{j=1}^{i-1} (1 - rho_j)
    # w_i = T_i * rho_i
    
    # To make this self-contained, we assume t_vals are the t_i points.
    # We need t_{i+1}. We'll assume t_vals includes t_N as the end point or we extrapolate.
    # Source says t_0:N. Let's assume t_vals has N+1 points for boundaries, or N points for centers.
    # Source eq 1: sum_{i=1}^N. 
    # Let's assume t_vals are the N sample centers.
    # We need delta_i = t_{i+1} - t_i.
    
    # Simplified for code:
    deltas = torch.full((num_samples,), step)
    deltas[-1] = 0 # No contribution after last sample
    
    rho = 1.0 - torch.exp(-sigma_vals * deltas)
    
    # Cumulative product for T
    # T_i = product_{j=1}^{i-1} (1 - rho_j)
    # T_1 = 1
    # T_2 = 1 - rho_1
    # ...
    one_minus_rho = 1.0 - rho
    # Cumprod of (1-rho) shifted
    # T[i] = prod_{j=0}^{i-1} (1-rho[j])
    T = torch.cumprod(one_minus_rho, dim=1)
    # Shift T: T[i] should be product up to i-1. 
    # torch.cumprod gives prod up to i.
    # So T_shifted[i] = T[i-1] for i>0, T[0]=1
    T_shifted = torch.cat([torch.ones_like(T[:, :1]), T[:, :-1]], dim=1)
    
    w = T_shifted * rho
    
    C = torch.sum(w.unsqueeze(-1) * color_vals, dim=1) + (1.0 - torch.sum(w, dim=1)).unsqueeze(-1) * bg_color
    D = torch.sum(w * t_vals, dim=1) / torch.sum(w, dim=1)
    
    return C, D, w, t_vals

def ddm_score_epsilon(x_tau, tau, ddm_model):
    """
    Returns the noise prediction epsilon_theta from the DDM.
    This is proportional to -grad_x log p(x).
    """
    # ddm_model is the trained neural network epsilon_theta
    return ddm_model(x_tau, tau)

def train_step(nerf_params, ddm_model, training_images, camera_poses, patch_size=48, lambda_dmm=1e-5):
    """
    One step of the training loop.
    """
    # 1. Sample a random patch of rays
    # Select random camera pose and random patch coordinates
    # ... (implementation specific to ray sampling from patch)
    
    # 2. Render RGBD patch
    C_render, D_render, w_render, t_render = render_ray(...)
    
    # 3. Form RGBD input for DDM
    # x_tau is the RGBD patch. 
    # In practice, x_tau is the noisy version. 
    # However, the source says: "Pass the rendered RGBD patches into the trained DDM".
    # And eq 19 uses epsilon_theta directly.
    # The DDM expects a noisy input x_tau.
    # We must add noise to the rendered patch to get x_tau.
    
    # Sample noise
    epsilon = torch.randn_like(C_render) # Assuming C and D are concatenated or handled separately
    # Source eq 12: x_tau = sqrt(alpha_tau) * x_0 + sqrt(beta_tau) * epsilon
    # x_0 is the clean rendered patch (C_render, D_render)
    
    # Combine C and D into a single tensor x_0 for DDM input
    # Assuming DDM takes RGBD as a single tensor or concatenated channels
    x_0 = torch.cat([C_render, D_render.unsqueeze(-1)], dim=-1) 
    
    # Add noise
    x_tau = math.sqrt(alpha_tau) * x_0 + math.sqrt(beta_tau) * epsilon
    
    # 4. Get DDM gradient (noise prediction)
    epsilon_theta = ddm_score_epsilon(x_tau, tau, ddm_model)
    
    # 5. Compute Loss
    # L_photo
    # ... (compare C_render with ground truth RGB from training image)
    
    # L_geom (fg, fr, dist)
    # ... (compute geometric regularizers)
    
    # L_DDM term in gradient
    # The gradient contribution is -lambda_DDM * epsilon_theta
    # Note: epsilon_theta is already the gradient direction (proportional to -grad log p)
    
    total_grad = grad_photo + lambda_fg * grad_fg + lambda_fr * grad_fr + lambda_dist * grad_dist - lambda_dmm * epsilon_theta
    
    # 6. Backprop
    nerf_params.backward(total_grad)
```

**Edge Cases and Pitfalls:**
*   **Early Training Inaccuracy:** Rendered patches are inaccurate early on. Mitigation: 25% of the time, use a training pose and sample RGB directly from the training image [S1].
*   **Over-smoothing:** The DDM regularizer can over-smooth thin structures and hallucinate details in occluded regions, impacting reconstruction scores [S1].
*   **Scheduling:** Careful scheduling of $\tau$ and $\lambda_{\text{DDM}}$ is required; improper scheduling leads to poor results [S1].

## Operating characteristics
*   **Time Complexity:** Training time is dominated by the NeRF optimization (12,000 steps) and DDM pre-training (650,000 steps). On a single Nvidia A100 GPU, NeRF training takes approximately 30 minutes per scene [S1]. Instant NGP's fast rendering allows for larger patches ($48 \times 48$) without substantial time extension [S1].
*   **Memory Complexity:** Uses Instant NGP's multi-scale hash tables, which store features in a fixed memory block, reducing MLP size compared to standard MLP-based NeRFs [S1].
*   **Performance:** Improves geometry and generalization on LLFF and DTU datasets compared to baselines like mip-NeRF and RegNeRF [S1].
*   **Strengths:** Directly regularizes geometry via RGBD; handles limited views better than standard L2 loss; larger patch sizes possible with Instant NGP compared to RegNeRF's $8 \times 8$ [S1].
*   **Limitations:** Can over-smooth thin structures; hallucinates details in unseen regions; struggles with shiny objects compared to SDF/occupancy methods; requires careful hyperparameter tuning ($\tau$, $\lambda_{\text{DDM}}$) [S1].
*   **When to Use:** When training NeRFs with limited input views and high-quality geometry is required.
*   **When Not to Use:** When shiny/reflective surfaces are prominent; when computational resources for DDM pre-training are unavailable; when thin structure preservation is critical without post-processing.

## Background
*   **Origin:** Proposed in "DiffusioNeRF: Regularizing Neural Radiance Fields with Denoising Diffusion Models" (CVPR 2023) [S1].
*   **History:** Builds on the understanding that NeRFs are under-constrained with few views, leading to floaters [S1]. It leverages Denoising Diffusion Probabilistic Models (DDPMs) [8, 20, 31] which were originally developed for image generation [S1]. The connection between DDM noise prediction and score matching ($\nabla \log p$) allows the DDM to act as a regularizer [S1].
*   **Usage:** Implemented using torch-ngp with tiny-cuda-nn backend [S1]. Applied to novel-view synthesis and 3D reconstruction on LLFF and DTU datasets [S1].

## Sources
[S1] DiffusioNeRF: Regularizing Neural Radiance Fields with Denoising Diffusion Models, https://doi.org/10.1109/cvpr52729.2023.00407, fetched 2026-10-26
[S2] Rugularizing generalizable neural radiance field with limited-view images, https://link.springer.com/article/10.1007/s40747-024-01696-6, fetched 2026-10-29
