# Plenoxels Optimization Configuration

Also called: Plenoxels optimizer settings, RMSProp vs SGD for Plenoxels

## Overview
Plenoxels Optimization Configuration defines the hyperparameter settings, specifically the choice of gradient-based optimizer and learning rate schedules, required to train the sparse voxel grid representation of Plenoxels [S1]. It solves the problem of efficiently converging voxel opacity ($\sigma$) and spherical harmonic (SH) coefficients from multi-view images without neural networks [S1]. It sets itself apart from its siblings by focusing on the specific numerical stability and convergence properties of optimizing SH coefficients, which benefit more from adaptive methods than standard neural network weights [S1].

## How it works
The optimization procedure updates the voxel parameters using gradient descent [S1]. The inputs are multi-view images and camera poses; the output is the optimized voxel grid containing opacity values and SH coefficients [S1].

**Data Structures:**
*   **Voxel Grid:** A 3D grid of resolution $N^3$ (typically $256^3$) [S1].
*   **Voxel Data:** Each voxel stores:
    *   Opacity $\sigma$: A single scalar value [G].
    *   SH Coefficients: A vector of coefficients representing view-dependent color [G]. The number of coefficients depends on the degree $L$ of the spherical harmonics [G]. For degree $L$, there are $(L+1)^2$ coefficients per color channel (RGB) [G].

**Parameters:**
*   **Optimizer:** RMSProp is the recommended default. SGD is an alternative but yields lower quality [S1].
*   **Learning Rate (LR):** Base learning rates for $\sigma$ and SH coefficients [G].
*   **Schedule:**
    *   **Delayed Schedule:** The learning rate for $\sigma$ is delayed (starts later or is scaled down) relative to SH coefficients. This yields the best results [S1].
    *   **Constant Schedule:** Constant learning rates for both $\sigma$ and SH [S1].

**Steps:**
1.  **Initialization:** Initialize voxel $\sigma$ to small positive values and SH coefficients to zero or average view color [G].
2.  **Sampling:** For each training step, sample rays through the voxel grid [G].
3.  **Rendering:** Compute the rendered color and opacity for each ray using volume rendering integration [G].
4.  **Loss Calculation:** Compute the loss between rendered and ground truth images (e.g., L1 or MSE) [G].
5.  **Backpropagation:** Compute gradients for $\sigma$ and SH coefficients [G].
6.  **Update:** Apply the optimizer update step according to the chosen schedule and optimizer type [S1].

**Formulas:**
*   **Volume Rendering:**
    $$ C(r) = \sum_{i=1}^n T_i (1 - \exp(-\sigma_i \delta_i)) c_i $$
    where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$, $\delta_i$ is the segment length, $\sigma_i$ is the opacity, and $c_i$ is the color computed from SH coefficients at direction $\omega$ [G].
*   **RMSProp Update:**
    $$ v_t = \rho v_{t-1} + (1-\rho) g_t^2 $$
    $$ \theta_t = \theta_{t-1} - \frac{\eta}{\sqrt{v_t + \epsilon}} g_t $$
    where $\rho$ is the decay rate (typically 0.99), $\eta$ is the learning rate, $g_t$ is the gradient, and $\epsilon$ is a small constant for numerical stability [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_plenoxels(grid, images, cameras, optimizer='RMSProp', schedule='delayed'):
    sigma = grid.opacity
    sh = grid.sh
    lr_sigma = 1e-3
    lr_sh = 1e-3
    rho = 0.99
    eps = 1e-8
    m_sigma = zeros_like(sigma)
    v_sigma = zeros_like(sigma)
    m_sh = zeros_like(sh)
    v_sh = zeros_like(sh)
    delay_step = 1000  # Step where sigma LR activates [G]

    for step in range(10000):
        for img, cam in zip(images, cameras):
            rendered = render_volume(grid, cam)
            loss = l1_loss(rendered, img)
            loss.backward()

            if schedule == 'delayed':
                current_lr_sigma = lr_sigma if step > delay_step else lr_sigma * 0.1
            else:
                current_lr_sigma = lr_sigma

            if optimizer == 'RMSProp':
                m_sigma = rho * m_sigma + (1 - rho) * sigma.grad
                v_sigma = rho * v_sigma + (1 - rho) * sigma.grad**2
                sigma = sigma - current_lr_sigma * sigma.grad / (sqrt(v_sigma) + eps)

                m_sh = rho * m_sh + (1 - rho) * sh.grad
                v_sh = rho * v_sh + (1 - rho) * sh.grad**2
                sh = sh - lr_sh * sh.grad / (sqrt(v_sh) + eps)
            elif optimizer == 'SGD':
                sigma = sigma - current_lr_sigma * sigma.grad
                sh = sh - lr_sh * sh.grad

            sigma.grad = zeros_like(sigma)
            sh.grad = zeros_like(sh)

def render_volume(grid, cam):
    # Standard volume rendering integration over sampled points [G]
    pass

def l1_loss(rendered, target):
    # Mean absolute difference [G]
    pass
```

**Edge Cases and Pitfalls:**
*   **Optimizer Choice:** Using SGD for both $\sigma$ and SH results in significantly lower PSNR (29.35 vs 30.57) and SSIM (0.932 vs 0.950) [S1].
*   **Learning Rate Schedule:** A constant schedule for both parameters yields lower quality (PSNR 30.13) compared to a delayed schedule for $\sigma$ (PSNR 30.57) [S1].
*   **Robustness:** The method is robust to variations in learning rate schedule and optimizer choice, though RMSProp with a delayed schedule is optimal [S1].

## Subtopics

### Space Carving and Opaque Voxels Optimization with Spherical Harmonics
A technique composed of two steps: space carving and opaque voxels optimization, used to generate neural radiance fields based on plenoxels [S2, unofficial]. It uses spherical harmonics of degree 9 to render the shading of each voxel [S2, unofficial]. Running the implementation `main_carve_sph.py` takes approximately 4 minutes on a single GPU [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of rays sampled and the resolution of the voxel grid [G].
*   **Memory Complexity:** Proportional to the number of voxels and the number of SH coefficients per voxel [G].
*   **Measured Performance:** On 8 synthetic NeRF scenes at $256^3$ resolution, RMSProp for both SH and $\sigma$ achieves PSNR 30.57, SSIM 0.950, LPIPS 0.065 [S1]. SGD for both drops to PSNR 29.35, SSIM 0.932, LPIPS 0.087 [S1].
*   **Strengths:** RMSProp provides better convergence for SH coefficients. The method is robust to variations in optimizer and schedule, though RMSProp with delayed schedule is optimal [S1].
*   **Limitations:** SGD is less effective, particularly for SH coefficients [S1].
*   **When to Use:** Use RMSProp with a delayed learning rate schedule for $\sigma$ for best results. Use SGD only if computational resources for RMSProp are constrained, accepting lower quality [S1].

## Background
Plenoxels Optimization Configuration originates from the Plenoxels paper, which introduced a novel approach for fast radiance field reconstruction from sparse views using voxel grids without neural networks [S1]. The specific optimization settings were derived from empirical evaluation on synthetic NeRF scenes [S1]. The method builds on the assumption that voxel opacity and SH coefficients can be optimized directly using gradient-based methods [S1].

## Sources
[S1] Plenoxels: Radiance Fields without Neural Networks, https://doi.org/10.1109/cvpr52688.2022.00542, fetched 2026-10-26
[S2] nissmar/Radiance-Fields, https://github.com/nissmar/Radiance-Fields, fetched 2026-10-27 (unofficial)
