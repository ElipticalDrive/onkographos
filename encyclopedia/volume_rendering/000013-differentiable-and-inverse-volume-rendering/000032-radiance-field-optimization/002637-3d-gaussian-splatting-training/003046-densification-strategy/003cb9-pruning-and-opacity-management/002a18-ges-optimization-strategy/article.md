# GES Optimization Strategy

Also called: GES Training, Generalized Exponential Splatting Optimization

## Overview
The GES Optimization Strategy is a training methodology for Generalized Exponential Splatting (GES) that controls shape density by selectively pruning splats based on shape attributes, thereby eliminating the need for a variable density mechanism [S1]. It differs from standard 3D Gaussian Splatting by optimizing a shape parameter $\beta$ alongside position, opacity, and covariance, using a frequency-modulated loss to enhance high-frequency detail reconstruction [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of images with known camera poses, initial 3D Gaussians derived from Structure-from-Motion (SfM) points.
*   **Output:** Optimized Gaussian parameters: position $x$, opacity $\kappa$, covariance $\Sigma$, shape parameter $\beta$, and spherical harmonics coefficients.

**Data Structures:**
*   **Gaussian Attributes:** Each Gaussian $\mathcal{G}_i$ stores:
    *   Position $x \in \mathbb{R}^3$
    *   Opacity $\kappa \in [0, 1]$ (or logit space)
    *   Covariance $\Sigma \in \mathbb{R}^{3 \times 3}$ (symmetric positive definite)
    *   Shape parameter $\beta \in \mathbb{R}$ (controls the exponential decay rate)
    *   Spherical Harmonics coefficients for color
*   **Buffers:** GPU buffers store these attributes. Buffer capacity must be sufficient to handle initial Gaussians plus potential splits, though GES relies on pruning rather than aggressive splitting [S1].

**Parameters:**
*   Total iterations: 40,000 [S1].
*   Density gradient threshold: 0.0003 [S1].
*   Learning rate for $\beta$: 0.0015 [S1].
*   Shape reset interval: 1000 iterations [S1].
*   Shape pruning interval: 100 iterations [S1].
*   Pruning threshold: 0.5 [S1].
*   Shape strength parameter: 0.1 [S1].
*   Image Laplacian scale factor: 0.2 [S1].
*   Frequency loss coefficient $\lambda_\omega$: 0.5 [S1].

**Loss Function:**
The total loss $\mathcal{L}$ is a weighted sum of L1 loss, Structural Similarity (SSIM) loss, and a frequency-modulated loss $\mathcal{L}_\omega$:
$$ \mathcal{L} = \lambda_{L1}\mathcal{L}_1 + \lambda_{\text{ssim}}\mathcal{L}_{\text{ssim}} + \lambda_{\omega}\mathcal{L}_{\omega} $$
where $\lambda_{L1}$ and $\lambda_{\text{ssim}}$ are standard weights (typically 0.8 and 0.2 in related works, though not explicitly fixed in the GES text, $\lambda_\omega$ is explicitly 0.5) [S1].

**Frequency-Modulated Loss $\mathcal{L}_\omega$:**
This loss uses masks $M_\omega$ to focus on specific frequency bands. The target normalized frequency $\omega$ is scheduled linearly:
$$ \omega = \frac{\text{current iteration}}{\text{total iterations}} $$
The mask $M_\omega$ covers frequencies $1 - M_\omega$ of $50\% < \omega \le 100\%$, ensuring coverage of all image parts while focusing on details as optimization progresses [S1]. The Image Laplacian scale factor (0.2) is used in the calculation of this loss [S1].

**Algorithm Steps:**
1.  **Initialization:**
    *   Initialize position $x$ and covariance $\Sigma$ from SfM points [S1].
    *   Initialize all shape parameters $\beta = 2$ (representing pure Gaussian splats) [S1].
    *   Initialize opacity $\kappa$ and spherical harmonics coefficients.
2.  **Training Loop (for iteration $t = 1$ to 40,000):**
    *   **Forward Pass:** Render the scene using the current Gaussian set. Compute $\mathcal{L}_1$, $\mathcal{L}_{\text{ssim}}$, and $\mathcal{L}_\omega$.
    *   **Backward Pass:** Compute gradients for $x, \kappa, \Sigma, \beta$, and spherical harmonics coefficients using Stochastic Gradient Descent (SGD) [S1].
    *   **Update:** Update parameters using their respective learning rates.
    *   **Shape Reset:** If $t \pmod{1000} == 0$, reset shape parameters $\beta$ [S1].
    *   **Shape Pruning:** If $t \pmod{100} == 0$, perform pruning based on shape attributes [S1].
        *   Calculate a shape metric for each Gaussian (e.g., gradient norm or density contribution).
        *   Prune Gaussians where the metric exceeds the pruning threshold (0.5) or fails the density gradient threshold (0.0003) [S1].
        *   The "shape strength parameter" (0.1) balances accuracy and computational load during this optimization [S1].
3.  **Edge Cases:**
    *   **DoG Filter Sensitivity:** High-frequency masks may be sensitive to Difference of Gaussians (DoG) filter artifacts [S1].
    *   **Precision:** GPU-powered computation and specialized CUDA kernels are used to accelerate optimization [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_ges(gaussians, images, cameras, total_iterations=40000):
    # Initialization
    for g in gaussians:
        g.x = get_sfm_position(g.id)
        g.Sigma = get_sfm_covariance(g.id)
        g.beta = 2.0  # Pure Gaussian start
        g.opacity = initialize_opacity()
        g.sh_coeffs = initialize_sh()
    
    lr_beta = 0.0015
    shape_reset_interval = 1000
    shape_pruning_interval = 100
    pruning_threshold = 0.5
    density_gradient_threshold = 0.0003
    shape_strength = 0.1
    laplacian_scale = 0.2
    lambda_omega = 0.5
    
    for t in range(1, total_iterations + 1):
        # 1. Forward Pass
        rendered_image = render(gaussians, cameras)
        
        # Calculate Loss Components
        loss_l1 = calculate_l1_loss(rendered_image, target_image)
        loss_ssim = calculate_ssim_loss(rendered_image, target_image)
        
        # Frequency-modulated loss
        omega = t / total_iterations
        mask_M_omega = create_frequency_mask(omega, laplacian_scale)
        loss_omega = calculate_frequency_loss(rendered_image, target_image, mask_M_omega)
        
        total_loss = lambda_l1 * loss_l1 + lambda_ssim * loss_ssim + lambda_omega * loss_omega
        
        # 2. Backward Pass
        gradients = compute_gradients(total_loss, gaussians)
        
        # 3. Update Parameters
        for g in gaussians:
            g.x -= lr_x * gradients[g.id].x
            g.Sigma -= lr_sigma * gradients[g.id].Sigma
            g.opacity -= lr_opacity * gradients[g.id].opacity
            g.beta -= lr_beta * gradients[g.id].beta
            g.sh_coeffs -= lr_sh * gradients[g.id].sh_coeffs
            
        # 4. Periodic Actions
        if t % shape_reset_interval == 0:
            for g in gaussians:
                g.beta = 2.0  # Reset shape parameter
                
        if t % shape_pruning_interval == 0:
            prune_gaussians(gaussians, pruning_threshold, density_gradient_threshold, shape_strength)

def prune_gaussians(gaussians, threshold, grad_threshold, strength):
    to_remove = []
    for g in gaussians:
        # Calculate shape-based metric (e.g., gradient norm or density contribution)
        # The exact metric formula is not fully detailed, but relies on shape attributes
        metric = compute_shape_metric(g, strength)
        
        if metric > threshold or get_gradient_norm(g) > grad_threshold:
            to_remove.append(g.id)
            
    remove_gaussians(gaussians, to_remove)
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of Gaussians and iterations. The frequency-modulated loss adds overhead for mask generation and filtering [S1].
*   **Memory Complexity:** Determined by the number of active Gaussians. Pruning reduces memory usage over time [S1].
*   **Performance:** Evaluated on an A6000 GPU [S1]. Achieves a balance between high fidelity and efficiency in novel view synthesis [S1].
*   **Strengths:** Eliminates the need for complex variable density mechanisms by using shape-based pruning. Enhances high-frequency details via frequency-modulated loss [S1].
*   **Limitations:** Requires careful tuning of frequency loss parameters. Sensitivity to DoG filters at high frequencies [S1].

## Background
The GES Optimization Strategy was introduced in the paper "GES: Generalized Exponential Splatting for Efficient Radiance Field Rendering" [S1]. It builds upon the principles of 3D Gaussian Splatting but introduces a generalized exponential function controlled by $\beta$. The method was developed to improve reconstruction fidelity and computational efficiency, particularly in handling high-frequency details [S1].

## Sources
[S1] GES: Generalized Exponential Splatting for Efficient Radiance Field Rendering, https://doi.org/10.1109/cvpr52733.2024.01873, fetched 2026-10-26
