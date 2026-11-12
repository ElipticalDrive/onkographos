# GES Optimization Strategy

Also called: GES Training, Generalized Exponential Splatting Optimization

## Overview
The GES Optimization Strategy is a training methodology for Generalized Exponential Splats (GES) that controls shape density by selectively pruning splats based on shape attributes, thereby eliminating the need for a variable density mechanism [S1]. It differs from standard 3D Gaussian Splatting by optimizing a shape parameter $\beta$ alongside position, opacity, and covariance, using a frequency-modulated loss to enhance high-frequency detail reconstruction [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of images with known camera poses; initial 3D points from Structure-from-Motion (SfM).
*   **Output:** Optimized GES primitives, each defined by position $\mathbf{x}$, covariance $\Sigma$, opacity $\kappa$, shape parameter $\beta$, and spherical harmonics coefficients.

**Data Structures:**
Each Gaussian primitive $g_k$ stores:
*   $\mathbf{x}_k \in \mathbb{R}^3$: Position.
*   $\Sigma_k \in \mathbb{R}^{3 \times 3}$: Covariance matrix.
*   $\kappa_k \in \mathbb{R}$: Opacity.
*   $\beta_k \in \mathbb{R}$: Shape parameter.
*   $\mathbf{c}_k$: Spherical harmonics coefficients.

**Initialization:**
1.  Initialize positions $\mathbf{x}$ and covariance $\Sigma$ from SfM points [S1].
2.  Initialize all shape parameters $\beta_k = 2$ (corresponding to pure Gaussian splats) [S1].
3.  Initialize opacity $\kappa$ and spherical harmonics coefficients [S1].

**Loss Function:**
The total loss $\mathcal{L}$ is a weighted sum of L1 loss, Structural Similarity (SSIM) loss, and a frequency-modulated loss $\mathcal{L}_\omega$ [S1]:

$$
\mathcal{L} = \lambda_{L1}\mathcal{L}_1 + \lambda_{\text{ssim}}\mathcal{L}_{\text{ssim}} + \lambda_{\omega}\mathcal{L}_{\omega}
$$

The frequency-modulated loss $\mathcal{L}_\omega$ uses masks $M_\omega$ to focus on specific frequency bands. The target normalized frequency $\omega$ is scheduled linearly over the total iterations [S1]:

$$
\omega = \frac{\text{current iteration}}{\text{total iterations}}
$$

The masks $M_\omega$ cover the range $50\% < \omega \le 100\%$ to ensure all image parts are covered, with higher $\omega$ focusing on details [S1].

**Optimization Steps:**
The optimization runs for a fixed number of iterations. Stochastic Gradient Descent (SGD) is used, accelerated by GPU/CUDA kernels [S1].

1.  **Render:** Project all GES primitives to 2D and render the image.
2.  **Compute Loss:** Calculate $\mathcal{L}_1$, $\mathcal{L}_{\text{ssim}}$, and $\mathcal{L}_\omega$ using the current masks $M_\omega$.
3.  **Backpropagate:** Compute gradients for $\mathbf{x}$, $\Sigma$, $\kappa$, $\beta$, and spherical harmonics coefficients.
4.  **Update Parameters:** Update parameters using SGD with the respective learning rates.
5.  **Periodic Shape Reset:** Every `shape_reset_interval` iterations, reset the shape parameters $\beta$ [S1].
6.  **Periodic Shape Pruning:** Every `shape_pruning_interval` iterations, evaluate the shape attributes of each primitive. If a primitive's shape metric exceeds the `pruning_threshold`, prune it [S1].
7.  **Density Gradient Check:** Monitor the density gradient. If the gradient exceeds the `density_gradient_threshold`, it may trigger further adjustments (though the primary pruning mechanism is shape-based) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_ges(scene_data, total_iterations=40000):
    # Initialize primitives from SfM
    primitives = initialize_from_sfm(scene_data.sfm_points)
    
    # Set initial beta to 2 (pure Gaussian)
    for p in primitives:
        p.beta = 2.0
        
    # Hyperparameters
    lr_beta = 0.0015
    lr_other = ... # Standard learning rates for x, kappa, etc.
    shape_reset_interval = 1000
    shape_pruning_interval = 100
    pruning_threshold = 0.5
    density_gradient_threshold = 0.0003
    lambda_L1 = 1.0
    lambda_ssim = 0.2
    lambda_omega = 0.5
    laplacian_scale = 0.2
    
    for iteration in range(total_iterations):
        # 1. Render
        rendered_image = render(primitives, scene_data.cameras)
        target_image = scene_data.current_view.image
        
        # 2. Compute Loss
        # Calculate L1 and SSIM
        L1 = compute_l1_loss(rendered_image, target_image)
        SSIM = compute_ssim_loss(rendered_image, target_image)
        
        # Calculate frequency-modulated loss
        # Schedule omega linearly
        omega = iteration / total_iterations
        # Create mask M_omega based on omega
        M_omega = create_frequency_mask(omega, laplacian_scale)
        L_omega = compute_frequency_loss(rendered_image, target_image, M_omega)
        
        L = lambda_L1 * L1 + lambda_ssim * SSIM + lambda_omega * L_omega
        
        # 3. Backpropagate
        gradients = compute_gradients(L, primitives)
        
        # 4. Update Parameters
        for p in primitives:
            p.x -= lr_other * gradients[p.id].x
            p.Sigma -= lr_other * gradients[p.id].Sigma
            p.kappa -= lr_other * gradients[p.id].kappa
            p.beta -= lr_beta * gradients[p.id].beta
            p.sh -= lr_other * gradients[p.id].sh
            
        # 5. Periodic Shape Reset
        if iteration % shape_reset_interval == 0:
            for p in primitives:
                p.beta = 2.0 # Reset to pure Gaussian
                
        # 6. Periodic Shape Pruning
        if iteration % shape_pruning_interval == 0:
            primitives_to_remove = []
            for p in primitives:
                # Evaluate shape attribute metric
                # Note: The exact formula for the shape metric is not explicitly 
                # detailed in the source, but it is compared against pruning_threshold.
                shape_metric = evaluate_shape_metric(p) 
                if shape_metric > pruning_threshold:
                    primitives_to_remove.append(p.id)
            
            # Remove primitives
            primitives = [p for p in primitives if p.id not in primitives_to_remove]
            
        # 7. Density Gradient Check (Implicit in pruning logic or separate check)
        # The source mentions a density gradient threshold of 0.0003.
        # This is likely used to determine if densification is needed, 
        # but GES claims to eliminate variable density mechanisms via shape pruning.
        # We assume this threshold is used for the shape pruning decision or 
        # as a guard against excessive density.
        
    return primitives
```

**Parameters:**
*   Total iterations: 40,000 [S1].
*   Density gradient threshold: 0.0003 [S1].
*   Learning rate for $\beta$: 0.0015 [S1].
*   Shape reset interval: 1000 iterations [S1].
*   Shape pruning interval: 100 iterations [S1].
*   Pruning threshold: 0.5 [S1].
*   Shape strength parameter: 0.1 [S1].
*   Image Laplacian scale factor: 0.2 [S1].
*   $\lambda_\omega$: 0.5 [S1].

**Edge Cases and Pitfalls:**
*   **High-Frequency Sensitivity:** The DoG (Difference of Gaussians) filter used in frequency loss may be sensitive to high frequencies, requiring careful mask design [S1].
*   **Shape Parameter Initialization:** Starting with $\beta=2$ ensures pure Gaussian behavior initially, which may need adjustment if the scene requires non-Gaussian shapes early on.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of primitives and iterations. The frequency mask computation adds a small overhead per iteration.
*   **Memory Complexity:** Linear with respect to the number of primitives. Pruning reduces memory footprint over time.
*   **Performance:** Achieves a balance between high fidelity and efficiency in novel view synthesis [S1].
*   **Strengths:** Eliminates the need for complex variable density mechanisms; frequency-modulated loss enhances edge details.
*   **Limitations:** Requires careful tuning of frequency masks and shape pruning thresholds.

## Background
The GES Optimization Strategy was introduced in the paper "GES: Generalized Exponential Splatting for Efficient Radiance Field Rendering" [S1]. It builds upon the principles of 3D Gaussian Splatting but introduces a shape parameter $\beta$ and a frequency-modulated loss to improve rendering quality without the computational overhead of variable density mechanisms.

## Sources
[S1] GES: Generalized Exponential Splatting for Efficient Radiance Field Rendering, https://doi.org/10.1109/cvpr52733.2024.01873, fetched 2026-10-26
