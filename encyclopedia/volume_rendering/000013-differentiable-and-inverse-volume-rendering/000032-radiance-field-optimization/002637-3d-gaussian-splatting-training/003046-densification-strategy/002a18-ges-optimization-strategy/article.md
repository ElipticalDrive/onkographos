# GES Optimization Strategy

Also called: GES Training, Generalized Exponential Splatting Optimization

## Overview
The GES Optimization Strategy is a training methodology for Generalized Exponential Splatting (GES) that controls shape density by selectively pruning splats based on shape attributes, thereby eliminating the need for a variable density mechanism [S1]. It differs from standard 3D Gaussian Splatting by optimizing a shape parameter $\beta$ and employing a frequency-modulated loss to enhance edge reconstruction [S1]. It builds on [Densification Strategy](6234-densification-strategy.md) but replaces the gradient-based densification triggers with shape-attribute-based pruning and periodic shape resets [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Initial Gaussian primitives derived from Structure-from-Motion (SfM) points, including position $x$, covariance $\Sigma$, opacity $\kappa$, spherical harmonics coefficients, and shape parameter $\beta$. Training images and camera poses.
*   **Outputs:** Optimized Gaussian primitives with refined $x, \Sigma, \kappa, \beta$, and color coefficients.

**Data Structures:**
*   **Gaussian Primitives:** Each primitive $\mathcal{G}_i$ stores:
    *   Mean position $x \in \mathbb{R}^3$
    *   Covariance matrix $\Sigma \in \mathbb{R}^{3 \times 3}$ (or its rotation/scale decomposition)
    *   Opacity $\kappa \in \mathbb{R}$
    *   Shape parameter $\beta \in \mathbb{R}$ (initialized to 2)
    *   Spherical Harmonics coefficients for color
*   **Buffers:** GPU buffers for parameters, updated via SGD. Capacity management follows the parent's shared principles [S1].

**Loss Function:**
The total loss $\mathcal{L}$ is a weighted sum of L1 loss, Structural Similarity (SSIM) loss, and a frequency-modulated loss $\mathcal{L}_{\omega}$ [S1]:
$$
\mathcal{L} = \lambda_{L1}\mathcal{L}_1 + \lambda_{\text{ssim}}\mathcal{L}_{\text{ssim}} + \lambda_{\omega}\mathcal{L}_{\omega}
$$
where $\lambda_{L1}$, $\lambda_{\text{ssim}}$, and $\lambda_{\omega}$ are weighting coefficients. The frequency-modulated loss $\mathcal{L}_{\omega}$ uses masks $M_\omega$ to focus on specific frequency bands [S1].

**Parameters:**
*   Total iterations: 40,000 [S1].
*   Density gradient threshold: 0.0003 [S1].
*   Learning rate for $\beta$: 0.0015 [S1].
*   Shape reset interval: 1000 iterations [S1].
*   Shape pruning interval: 100 iterations [S1].
*   Pruning threshold: 0.5 [S1].
*   Shape strength parameter: 0.1 [S1].
*   Image Laplacian scale factor: 0.2 [S1].
*   $\lambda_{\omega}$ coefficient: 0.5 [S1].

**Algorithm Steps:**
1.  **Initialization:** Initialize $x$ and $\Sigma$ from SfM points. Initialize all $\beta = 2$ (pure Gaussian splats) [S1].
2.  **Iteration Loop (1 to 40,000):**
    a.  **Frequency Schedule:** Compute target normalized frequency $\omega$ for the current iteration $t$:
        $$ \omega = \frac{t}{\text{total iterations}} $$
        Generate frequency masks $M_\omega$ based on $\omega$. The masks cover the image such that $1 - M_\omega$ corresponds to $50\% < \omega \le 100\%$, ensuring coverage while focusing on details as optimization progresses [S1].
    b.  **Rendering and Loss:** Render the scene. Compute $\mathcal{L}_1$, $\mathcal{L}_{\text{ssim}}$, and $\mathcal{L}_{\omega}$ using the current masks. Compute total loss $\mathcal{L}$ [S1].
    c.  **Optimization:** Perform SGD update for $x, \kappa, \Sigma$, spherical harmonics, and $\beta$ using the gradient of $\mathcal{L}$ [S1].
    d.  **Shape Reset:** If $t \pmod{1000} == 0$, reset shape parameters $\beta$ [S1].
    e.  **Shape Pruning:** If $t \pmod{100} == 0$, evaluate shape attributes for pruning [S1].
        i.  Compute a shape metric for each primitive.
        ii. If the metric exceeds the pruning threshold (0.5) or relates to the density gradient threshold (0.0003), prune the primitive [S1].
        iii. Remove pruned primitives from the buffer [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ges_optimization(sfm_points, images, cameras, total_iterations=40000):
    # Initialize Gaussians from SfM
    gaussians = []
    for point in sfm_points:
        g = Gaussian()
        g.x = point.position
        g.Sigma = point.covariance
        g.kappa = initial_opacity # Not given in sources, assume standard init [G]
        g.beta = 2.0 # Pure Gaussian splats
        g.sh_coeffs = initial_sh # Not given in sources [G]
        gaussians.append(g)
    
    # Hyperparameters
    lr_beta = 0.0015
    shape_reset_interval = 1000
    shape_pruning_interval = 100
    pruning_threshold = 0.5
    density_gradient_threshold = 0.0003
    lambda_omega = 0.5
    laplacian_scale = 0.2
    
    for t in range(1, total_iterations + 1):
        # 1. Frequency Schedule
        omega = t / total_iterations
        masks = generate_frequency_masks(omega, laplacian_scale)
        
        # 2. Render and Compute Loss
        rendered_image = render(gaussians, cameras)
        target_image = get_target_image(t) # Assuming image selection logic [G]
        
        loss_l1 = compute_l1_loss(rendered_image, target_image)
        loss_ssim = compute_ssim_loss(rendered_image, target_image)
        loss_omega = compute_frequency_loss(rendered_image, target_image, masks)
        
        total_loss = lambda_l1 * loss_l1 + lambda_ssim * loss_ssim + lambda_omega * loss_omega
        
        # 3. Optimization Step
        gradients = compute_gradients(total_loss, gaussians)
        update_gaussians(gaussians, gradients, lr_beta)
        
        # 4. Shape Reset
        if t % shape_reset_interval == 0:
            reset_shape_parameters(gaussians)
            
        # 5. Shape Pruning
        if t % shape_pruning_interval == 0:
            prune_gaussians(gaussians, pruning_threshold, density_gradient_threshold)
            
    return gaussians

def generate_frequency_masks(omega, laplacian_scale):
    # Generate masks M_omega based on omega and laplacian scale
    # Masks ensure coverage and focus on details as omega increases
    # Implementation details for mask generation not fully specified in sources [G]
    pass

def reset_shape_parameters(gaussians):
    # Reset beta values
    for g in gaussians:
        g.beta = 2.0

def prune_gaussians(gaussians, threshold, gradient_threshold):
    # Evaluate shape attributes and prune
    # Specific metric formula not given, but uses threshold and gradient threshold [G]
    to_remove = []
    for g in gaussians:
        shape_metric = compute_shape_metric(g) # Placeholder for specific metric [G]
        if shape_metric > threshold or compute_density_gradient(g) > gradient_threshold:
            to_remove.append(g)
    gaussians = [g for g in gaussians if g not in to_remove]
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of Gaussians and iterations. The frequency mask generation adds overhead per step.
*   **Memory Complexity:** Linear with respect to the number of Gaussians. Buffer management follows the parent's shared principles [S1].
*   **Performance:** Evaluated on an A6000 GPU [S1]. Achieves a balance between high fidelity and efficiency in novel view synthesis [S1].
*   **Strengths:** Eliminates the need for variable density mechanisms by using shape-based pruning. Frequency-modulated loss enhances edge reconstruction [S1].
*   **Limitations:** Requires careful tuning of frequency masks and pruning thresholds. The specific shape metric for pruning is not fully detailed in the sources [S1].

## Background
The GES Optimization Strategy was introduced in the paper "GES: Generalized Exponential Splatting for Efficient Radiance Field Rendering" [S1]. It was developed to improve the efficiency and fidelity of radiance field rendering by optimizing the shape parameter $\beta$ and using frequency-modulated losses. The method was evaluated on novel view synthesis tasks, demonstrating competitive performance against state-of-the-art techniques [S1].

## Sources
[S1] GES: Generalized Exponential Splatting for Efficient Radiance Field Rendering, https://doi.org/10.1109/cvpr52733.2024.01873, fetched 2026-10-26
