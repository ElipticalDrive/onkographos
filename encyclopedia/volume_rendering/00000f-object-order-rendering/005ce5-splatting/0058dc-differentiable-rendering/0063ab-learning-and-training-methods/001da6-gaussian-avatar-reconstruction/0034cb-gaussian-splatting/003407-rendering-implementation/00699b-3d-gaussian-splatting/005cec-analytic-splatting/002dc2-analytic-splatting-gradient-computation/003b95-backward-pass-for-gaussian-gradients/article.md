# Backward Pass for Gaussian Gradients

Also called: Gaussian Splatting Backpropagation, Gradient Computation for Gaussian Parameters, Gaussian Rasterization Backward Pass, Splatter Backpropagation, Splatter backward pass

## Overview
Computes the gradients of a scalar loss with respect to the input 3D Gaussian parameters (color, opacity, mean, scale, rotation) by propagating pixel-level loss gradients backward through the differentiable rasterization process [S2, unofficial]. It enables end-to-end training of Gaussian Splatting models by providing exact analytic gradients for the 2D covariance matrix, 2D means, and 3D attributes via the chain rule [S2, unofficial]. Unlike parent methods that focus on forward projection, this method specifically handles the reverse accumulation of gradients through the alpha-compositing and projection steps [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Gradient of the loss $\mathcal{L}$ with respect to the rendered pixel colors $\frac{\partial \mathcal{L}}{\partial C_i} \in \mathbb{R}^3$ for each pixel $i$ [S2, unofficial]. Saved tensors from the forward pass: `colors_precomp`, `means3D`, `scales`, `rotations`, `cov3Ds_precomp`, `radii`, `dc`, `sh`, `geomBuffer`, `binningBuffer`, `imgBuffer`, `sampleBuffer` [S1, unofficial].
*   **Output:** A tuple of gradients: `grad_means3D`, `grad_means2D`, `grad_dc`, `grad_sh`, `grad_colors_precomp`, `grad_opacities`, `grad_scales`, `grad_rotations`, `grad_cov3Ds_precomp`, and three `None` values [S1, unofficial].

**Data Structures:**
The backward pass operates on the same Gaussian attributes as the forward pass. Key intermediate values saved from the forward pass include the transmittance $T_n$ and the accumulated color contribution $S_n$ for each Gaussian $n$ contributing to a pixel [S2, unofficial].

**Numbered Steps:**

1.  **Initialization:** Restore `num_rendered`, `num_buckets`, `raster_settings`, and the saved tensors from the forward pass [S1, unofficial]. Pack arguments for the C++ backward kernel call [S1, unofficial].
2.  **Pixel to Gaussian Gradient Propagation (Back-to-Front):** Iterate through the Gaussians contributing to each pixel in reverse order (back to front) [S2, unofficial].
    *   **Transmittance Update:** Compute $T_{n-1}$ from the saved $T_n$ using:
        $$T_{n-1} = \frac{T_n}{1 - \alpha_{n-1}} \quad (23)$$
        where $\alpha_n$ is the opacity of Gaussian $n$ at the pixel [S2, unofficial].
    *   **Accumulated Color Update:** Compute $S_{n-1}$ iteratively:
        $$S_N(k) = 0$$
        $$S_{n-1}(k) = c_n(k)\alpha_n T_n + S_n(k) \quad (25)$$
        where $c_n(k)$ is the color of Gaussian $n$ in channel $k$ [S2, unofficial].
    *   **Color Gradient:** Compute $\frac{\partial \mathcal{L}}{\partial c_n(k)}$:
        $$\frac{\partial C_i(k)}{\partial c_n(k)} = \alpha_n \cdot T_n \quad (22)$$
    *   **Opacity Gradient:** Compute $\frac{\partial \mathcal{L}}{\partial \alpha_n}$:
        $$\frac{\partial C_i(k)}{\partial \alpha_n} = c_n(k) \cdot T_n - \frac{S_n(k)}{1 - \alpha_n} \quad (24)$$
    *   **Opacity Parameter Gradient:** Compute $\frac{\partial \mathcal{L}}{\partial o_n}$ and $\frac{\partial \mathcal{L}}{\partial \sigma_n}$ (where $\sigma_n$ is the exponent term in the Gaussian PDF):
        $$\frac{\partial \alpha_n}{\partial o_n} = \exp(-\sigma_n)$$
        $$\frac{\partial \alpha_n}{\partial \sigma_n} = -o_n \exp(-\sigma_n) \quad (26)$$
3.  **2D Covariance Gradient:** Compute the gradient of $\sigma_n$ with respect to the inverse 2D covariance matrix $\mathbf{Y} = (\boldsymbol{\Sigma}'_n)^{-1}$:
    $$\frac{\partial \sigma_n}{\partial \mathbf{Y}} = \frac{1}{2} \Delta_n \Delta_n^\top \quad (28)$$
    where $\Delta_n = \mathbf{p}_{(x,y)} - \boldsymbol{\mu}'_n$ is the vector from the 2D mean to the pixel center [S2, unofficial].
    Using the identity $\frac{\partial \mathbf{Y}}{\partial x} = -\mathbf{Y} \frac{\partial \boldsymbol{\Sigma}'_n}{\partial x} \mathbf{Y}$, compute the gradient with respect to $\boldsymbol{\Sigma}'_n$:
    $$\frac{\partial \sigma_n}{\partial \boldsymbol{\Sigma}'_n} = -\frac{1}{2} \boldsymbol{\Sigma}'_n^{-1} \Delta_n \Delta_n^\top \boldsymbol{\Sigma}'_n^{-1} \quad (31)$$
4.  **2D Mean Gradient:** Compute the gradient of $\sigma_n$ with respect to the 2D mean $\boldsymbol{\mu}'_n$:
    $$\frac{\partial \sigma_n}{\partial \boldsymbol{\mu}'_n} = \boldsymbol{\Sigma}'_n^{-1} \Delta_n \quad (27)$$
5.  **Projection Gradient (3D to 2D):** Propagate gradients from 2D space ($\boldsymbol{\mu}', \boldsymbol{\Sigma}'$) to 3D camera coordinates $\mathbf{t}$ and view matrix $\mathbf{T}_{cw}$ [S2, unofficial].
    *   **2D Mean Contribution to $\mathbf{t}$:**
        $$\frac{\partial \mathcal{L}_{\mu'}}{\partial \mathbf{t}} = \frac{1}{2} \mathbf{P}^\top \begin{bmatrix} w/t_w & 0 & 0 & -w \cdot t_x / t_w^2 \\ 0 & h/t_w & 0 & -w \cdot t_y / t_w^2 \end{bmatrix}^\top \frac{\partial \mathcal{L}}{\partial \boldsymbol{\mu}'} \quad (33)$$
        where $\mathbf{P}$ is the projection matrix, $w, h$ are image dimensions, and $t_w$ is the homogeneous coordinate [S2, unofficial].
    *   **2D Covariance Contribution to $\mathbf{t}$:** Compute Jacobians of the projection matrix $\mathbf{J}$ with respect to camera coordinates $t_x, t_y, t_z, t_u$ (Equations 37, 38 in [S2, unofficial]) and aggregate with $\frac{\partial \mathcal{L}}{\partial \boldsymbol{\Sigma}'}$ [S2, unofficial].
    *   **View Matrix and 3D Mean:**
        $$\frac{\partial \mathcal{L}}{\partial \mathbf{T}_{cw}} = \frac{\partial \mathcal{L}}{\partial \mathbf{t}} \mathbf{q}^\top$$
        $$\frac{\partial \mathcal{L}}{\partial \boldsymbol{\mu}} = \mathbf{R}_{cw}^\top \left[ \frac{\partial \mathcal{L}}{\partial t_x} \quad \frac{\partial \mathcal{L}}{\partial t_y} \quad \frac{\partial \mathcal{L}}{\partial t_z} \right]^\top \quad (40)$$
        where $\mathbf{q} = [\boldsymbol{\mu}, 1]^\top$ and $\mathbf{R}_{cw}$ is the rotation component of $\mathbf{T}_{cw}$ [S2, unofficial].
6.  **Scale and Rotation Gradients:** Decompose the 3D covariance $\boldsymbol{\Sigma} = \mathbf{M}\mathbf{M}^\top$ into rotation $\mathbf{R}$ and scale $\mathbf{S}$ [S2, unofficial].
    *   Compute gradients with respect to $\mathbf{R}$ and $\mathbf{S}$ using:
        $$\frac{\partial \mathcal{L}}{\partial \mathbf{R}} = \frac{\partial \mathcal{L}}{\partial \mathbf{M}} \mathbf{S}^\top, \quad \frac{\partial \mathcal{L}}{\partial \mathbf{S}} = \mathbf{R}^\top \frac{\partial \mathcal{L}}{\partial \mathbf{M}} \quad (44)$$
    *   Compute Jacobians of $\mathbf{R}$ with respect to quaternion components $w, x, y, z$ (Equations 45, 46 in [S2, unofficial]) to get gradients for rotations [S2, unofficial].
7.  **Execution:** Call the C++ backward kernel `_C.rasterize_gaussians_backward(*args)` with the packed arguments [S1, unofficial]. Return the tuple of computed gradients [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def backward_pass(grad_pixel_colors, saved_tensors):
    """
    Computes gradients for Gaussian Splatting parameters.
    grad_pixel_colors: Tensor of shape (H, W, 3) containing dL/dC_i.
    saved_tensors: Dict containing forward pass intermediates.
    """
    # 1. Restore state
    num_rendered = saved_tensors['num_rendered']
    num_buckets = saved_tensors['num_buckets']
    raster_settings = saved_tensors['raster_settings']
    
    # Extract saved tensors
    colors_precomp = saved_tensors['colors_precomp']
    means3D = saved_tensors['means3D']
    scales = saved_tensors['scales']
    rotations = saved_tensors['rotations']
    cov3Ds_precomp = saved_tensors['cov3Ds_precomp']
    radii = saved_tensors['radii']
    dc = saved_tensors['dc']
    sh = saved_tensors['sh']
    geomBuffer = saved_tensors['geomBuffer']
    binningBuffer = saved_tensors['binningBuffer']
    imgBuffer = saved_tensors['imgBuffer']
    sampleBuffer = saved_tensors['sampleBuffer']
    
    # 2. Pack args for C++ kernel
    args = [
        grad_pixel_colors,
        num_rendered,
        num_buckets,
        raster_settings,
        colors_precomp,
        means3D,
        scales,
        rotations,
        cov3Ds_precomp,
        radii,
        dc,
        sh,
        geomBuffer,
        binningBuffer,
        imgBuffer,
        sampleBuffer
    ]
    
    # 3. Call C++ backward kernel
    # The kernel internally implements the back-to-front iteration,
    # transmittance/accumulation updates, and analytic gradient derivations.
    grad_means3D, grad_means2D, grad_dc, grad_sh, grad_colors_precomp, \
    grad_opacities, grad_scales, grad_rotations, grad_cov3Ds_precomp, _, _, _ = \
        _C.rasterize_gaussians_backward(*args)
        
    # 4. Return gradients
    return (grad_means3D, grad_means2D, grad_dc, grad_sh, grad_colors_precomp,
            grad_opacities, grad_scales, grad_rotations, grad_cov3Ds_precomp,
            None, None, None)
```

**Edge Cases and Pitfalls:**
*   **Precision:** The computation involves matrix inverses ($\boldsymbol{\Sigma}'_n^{-1}$) and divisions by $(1-\alpha_n)$, which can be numerically unstable if $\alpha_n \approx 1$ [S2, unofficial].
*   **Boundaries:** Gradients for Gaussians outside the viewing frustum or with zero radius are typically zero or undefined, requiring careful handling in the kernel [S4, unofficial].
*   **Degenerate Input:** If a Gaussian has a singular covariance matrix, the inverse in Equation (31) fails [S2, unofficial].

## Subtopics

### Skip Invisible Gaussian Updates
An optional extension that skips parameter updates for Gaussians that are not visible in the current view during the backward pass, based on the idea by Mallick et al. [S4, unofficial]. It identifies invisible Gaussians and omits their parameter update computation [S4, unofficial]. This speeds up training and reduces VRAM usage but slightly degrades quality and can cause inconsistencies and performance regressions compared to the original 3DGS implementation [S4, unofficial]. On Mip-NeRF360 outdoor scenes (RTX 4090), it reduces training time from 5m31s to 3m03s, VRAM from 6.0GiB to 5.1GiB, and Gaussian count from 3.87M to 3.34M, while PSNR drops from 24.72 to 24.59 [S4, unofficial].

### Gradient Derivation for Masked Gaussian Attributes
Analytical derivation of gradients for Gaussian attributes (e.g., mean $\boldsymbol{\mu}$) during rendering, accounting for the multiplicative effect of the selection mask and 2D projection coefficients [S5]. It computes the gradient for $\mu_i$ using the derived formula, substituting the respective attribute for other parameters [S5]. The effective opacity is $\hat{o}_i = o_i \cdot \hat{M}_i$ [S5]. The loss is $L_s = |I_{es} - I_{GT}|$ [S5]. The gradient is:
$$\frac{\partial L_s}{\partial \mu_i} = G_2 \cdot \left( \frac{\partial L_s}{\partial \hat{o}_i} \cdot o_i \cdot \hat{M}_i \right) + o_i \cdot \hat{M}_i \cdot \left( \frac{\partial L_s}{\partial G_2} \cdot \frac{\partial G_2}{\partial \mu_i} \right)$$
where $G_2(i, p)$ is the effect coefficient of the 2D projection of Gaussian $i$ onto pixel $p$ [S5].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of Gaussians and pixels, dominated by the C++ kernel execution [S1, unofficial].
*   **Memory Complexity:** Requires saving forward pass intermediates (transmittance, accumulated color) for each Gaussian-pixel interaction, increasing VRAM usage [S2, unofficial].
*   **Measured Performance:** On an RTX 4090, the standard backward pass is the bottleneck for training speed [S4, unofficial]. Skipping invisible Gaussians reduces training time by ~45% and VRAM by ~15% [S4, unofficial].
*   **Strengths:** Provides exact analytic gradients, enabling stable and efficient optimization of Gaussian parameters [S2, unofficial].
*   **Limitations:** High VRAM usage due to saved tensors; potential numerical instability near $\alpha=1$ [S2, unofficial]. Skipping updates can degrade quality [S4, unofficial].
*   **When to use:** When training 3D Gaussian Splatting models and gradients are needed for optimization [S2, unofficial].
*   **When not to use:** When memory is severely constrained and quality degradation from skipping updates is unacceptable [S4, unofficial].

## Background
Originated with the 3D Gaussian Splatting paper by Kerbl et al. (2023), which introduced the differentiable rasterization pipeline [S2, unofficial]. The backward pass implementation was detailed in the appendix of the gsplat library documentation and subsequent analyses [S2, S4, unofficial]. It is a core component of the training loop for Gaussian Avatar Reconstruction and other 3DGS-based methods [S2, unofficial].

## Sources
[S1] Ingyu-Jeong/EGGS, https://github.com/Ingyu-Jeong/EGGS, fetched 2026-10-26 (unofficial)
[S2] gsplat: An Open-Source Library for Gaussian Splatting, https://doi.org/10.48550/arXiv.2409.06765, fetched 2026-10-29 (unofficial)
[S3] Lightplane: Highly-Scalable Components for Neural 3D Fields, https://doi.org/10.1109/3dv66043.2025.00058, fetched 2026-10-29
[S4] Faster-GS: Analyzing and Improving Gaussian Splatting Optimization, https://arxiv.org/abs/2602.09999v1, fetched 2026-10-26 (unofficial)
[S5] FlexGS: Train Once, Deploy Everywhere with Many-in-One Flexible 3D Gaussian Splatting, https://doi.org/10.1109/cvpr52734.2025.01523, fetched 2026-10-29
