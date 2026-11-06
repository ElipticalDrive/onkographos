# GES Optimization Strategy
Also called: GES Training, Generalized Exponential Splatting Optimization

## Overview
The GES Optimization Strategy is a training methodology for Generalized Exponential Splatting (GES) that controls shape density by selectively pruning splats based on shape attributes, thereby eliminating the need for a variable density mechanism [S1]. It distinguishes itself from standard 3D Gaussian Splatting training by optimizing the shape parameter $\beta$ alongside position, opacity, and covariance, using a frequency-modulated loss to enhance high-frequency details [S1]. It builds on [3D Gaussian Splatting Training](3761-3d-gaussian-splatting-training.md) but replaces its density-based densification/pruning with shape-based pruning.

## How it works
**Inputs:**
- Sparse Reconstruction (SfM) points for initialization of position $x$ and covariance $\Sigma$ [S1].
- Training images and camera poses [S1].
- Hyperparameters: Total iterations $T=40,000$, density gradient threshold $\tau_{grad}=0.0003$, learning rate for $\beta$ ($\eta_\beta$) $= 0.0015$, shape reset interval $I_{reset}=1000$, shape pruning interval $I_{prune}=100$, pruning threshold $\tau_{prune}=0.5$, shape strength parameter $\alpha_{shape}=0.1$, image Laplacian scale factor $\sigma_{lap}=0.2$ [S1].

**Outputs:**
- Optimized GES parameters: position $x$, opacity $\kappa$, covariance $\Sigma$, shape parameter $\beta$, and spherical harmonics coefficients [S1].

**Data Structures:**
- Each splat stores: $x \in \mathbb{R}^3$, $\kappa \in \mathbb{R}$, $\Sigma \in \mathbb{R}^{3 \times 3}$ (or 6 unique values), $\beta \in \mathbb{R}$, and SH coefficients [S1].
- The covariance $\Sigma$ is derived from the initial SfM points [S1].

**Initialization:**
1. Initialize position $x$ and covariance $\Sigma$ from SfM points [S1].
2. Initialize all shape parameters $\beta = 2$ (corresponding to pure Gaussian splats) [S1].
3. Initialize opacity $\kappa$ and SH coefficients [S1].

**Loss Function:**
The total loss $\mathcal{L}$ is a weighted sum of L1 loss, Structural Similarity (SSIM) loss, and a frequency-modulated loss $\mathcal{L}_\omega$ [S1]:
$$
\mathcal{L} = \lambda_{L1}\mathcal{L}_1 + \lambda_{\text{ssim}}\mathcal{L}_{\text{ssim}} + \lambda_{\omega}\mathcal{L}_{\omega}
$$
where $\lambda_{L1}$ and $\lambda_{\text{ssim}}$ are standard weights, and $\lambda_{\omega} = 0.5$ [S1]. The frequency-modulated loss $\mathcal{L}_\omega$ uses masks $M_\omega$ to focus on specific frequency bands [S1]. The target normalized frequency $\omega$ is determined by a linear schedule:
$$
\omega = \frac{\text{current iteration}}{T}
$$
The masks $M_\omega$ cover the frequency spectrum such that $1 - M_\omega$ corresponds to the range $50\% < \omega \le 100\%$, ensuring all parts of the image are covered while focusing on details as optimization progresses [S1]. The image Laplacian scale factor $\sigma_{lap}=0.2$ is used in the calculation of the frequency loss [S1].

**Optimization Steps:**
1. **Forward Pass:** Render the scene using the current GES parameters [S1].
2. **Loss Calculation:** Compute $\mathcal{L}_1$, $\mathcal{L}_{\text{ssim}}$, and $\mathcal{L}_\omega$ [S1].
3. **Backward Pass:** Compute gradients for $x, \kappa, \Sigma, \beta$, and SH coefficients using stochastic gradient descent (SGD) accelerated by GPU/CUDA kernels [S1].
4. **Parameter Update:** Update parameters using their respective learning rates [S1].
5. **Shape Reset:** Every $I_{reset}=1000$ iterations, reset the shape parameters $\beta$ [S1].
6. **Shape Pruning:** Every $I_{prune}=100$ iterations, evaluate splats for pruning based on shape attributes [S1].
   - Calculate a shape metric for each splat.
   - If the metric exceeds the pruning threshold $\tau_{prune}=0.5$, remove the splat [S1].
7. **Density Gradient Check:** Monitor the density gradient. If it exceeds $\tau_{grad}=0.0003$, trigger densification or other management steps as defined in the parent training loop [S1]. Note: The specific densification mechanism is shared with Gaussian Splatting [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_ges(sfm_points, images, cameras, total_iterations=40000):
    # Initialization
    splats = []
    for point in sfm_points:
        splat = Splat(
            position=point.position,
            covariance=compute_covariance_from_sfm(point),
            beta=2.0,  # Pure Gaussian initialization
            opacity=1.0,
            sh_coefficients=initialize_sh()
        )
        splats.append(splat)

    # Hyperparameters
    eta_beta = 0.0015
    tau_grad = 0.0003
    I_reset = 1000
    I_prune = 100
    tau_prune = 0.5
    alpha_shape = 0.1
    sigma_lap = 0.2
    lambda_omega = 0.5

    for iteration in range(total_iterations):
        # Sample a random image and camera
        image, camera = sample_training_data(images, cameras)

        # Forward pass: Render
        rendered_image = render_ges(splats, camera)

        # Calculate Loss
        # L1 and SSIM are standard
        l1_loss = calculate_l1_loss(rendered_image, image)
        ssim_loss = calculate_ssim_loss(rendered_image, image)

        # Frequency-modulated loss
        # Linear schedule for omega
        omega = iteration / total_iterations
        # Create mask M_omega based on omega and sigma_lap
        mask = create_frequency_mask(omega, sigma_lap)
        # Calculate frequency loss using the mask
        freq_loss = calculate_frequency_loss(rendered_image, image, mask)

        total_loss = l1_loss + ssim_loss + lambda_omega * freq_loss

        # Backward pass
        gradients = compute_gradients(total_loss, splats)

        # Update parameters
        for splat in splats:
            splat.position -= eta_pos * gradients.position
            splat.opacity -= eta_opacity * gradients.opacity
            splat.covariance -= eta_cov * gradients.covariance
            splat.beta -= eta_beta * gradients.beta
            splat.sh_coefficients -= eta_sh * gradients.sh_coefficients

        # Shape Reset
        if iteration % I_reset == 0:
            for splat in splats:
                splat.beta = 2.0  # Reset to pure Gaussian

        # Shape Pruning
        if iteration % I_prune == 0:
            splats_to_remove = []
            for splat in splats:
                shape_metric = calculate_shape_metric(splat)
                if shape_metric > tau_prune:
                    splats_to_remove.append(splat)
            for splat in splats_to_remove:
                splats.remove(splat)

        # Density Gradient Check (Shared with Gaussian Splatting)
        density_gradient = calculate_density_gradient(splats)
        if density_gradient > tau_grad:
            # Trigger densification as per parent training loop
            densify_splats(splats, density_gradient)

    return splats
```

## Operating characteristics
- **Time Complexity:** Dominated by the rendering and gradient computation steps, which are GPU-accelerated [S1]. The pruning and reset operations are periodic and have negligible overhead [S1].
- **Memory Complexity:** Depends on the number of splats. Pruning reduces memory usage over time [S1].
- **Performance:** Evaluated on an A6000 GPU [S1]. Achieves a balance between high fidelity and efficiency in novel view synthesis [S1].
- **Strengths:** Eliminates the need for variable density mechanisms by using shape-based pruning [S1]. Enhances high-frequency details through frequency-modulated loss [S1].
- **Limitations:** Requires careful tuning of hyperparameters such as $\tau_{prune}$ and $\eta_\beta$ [S1].

## Background
The GES Optimization Strategy was introduced in the paper "GES: Generalized Exponential Splatting for Efficient Radiance Field Rendering" [S1]. It extends the 3D Gaussian Splatting framework by introducing Generalized Exponential Splats and a novel optimization approach for controlling shape density [S1].

## Sources
[S1] GES: Generalized Exponential Splatting for Efficient Radiance Field Rendering, https://doi.org/10.1109/cvpr52733.2024.01873, fetched 2026-10-26
