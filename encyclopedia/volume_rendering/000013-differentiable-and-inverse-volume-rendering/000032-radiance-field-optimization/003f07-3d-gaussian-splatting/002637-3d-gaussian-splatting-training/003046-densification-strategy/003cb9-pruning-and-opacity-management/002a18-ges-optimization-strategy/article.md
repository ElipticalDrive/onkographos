# GES Optimization Strategy

Also called: GES Training, Generalized Exponential Splatting Optimization

## Overview
The GES Optimization Strategy is a training methodology for Generalized Exponential Splats (GES) that controls shape density by selectively pruning splats based on shape attributes, thereby eliminating the need for a variable density mechanism [S1]. It differs from standard 3D Gaussian Splatting training by optimizing a specific shape parameter $\beta$ alongside position, opacity, and covariance, using a frequency-modulated loss to enhance high-frequency detail reconstruction [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Sparse Structure-from-Motion (SfM) points, input images, and hyperparameters.
*   **Outputs:** Optimized set of GES primitives, each defined by position $x$, opacity $\kappa$, covariance $\Sigma$, shape parameter $\beta$, and spherical harmonics coefficients.

**Data Structures:**
Each GES primitive stores:
*   $x \in \mathbb{R}^3$: Position.
*   $\kappa \in \mathbb{R}$: Opacity.
*   $\Sigma \in \mathbb{R}^{3 \times 3}$: Covariance matrix (positive definite).
*   $\beta \in \mathbb{R}$: Shape parameter controlling the exponential decay rate.
*   $c \in \mathbb{R}^k$: Spherical harmonics coefficients for color.

**Initialization:**
1.  Initialize position $x$ and covariance $\Sigma$ from SfM points [S1].
2.  Initialize all shape parameters $\beta = 2$ (representing pure Gaussian splats) [S1].
3.  Initialize opacity $\kappa$ and spherical harmonics coefficients $c$ (standard initialization, details shared with Gaussian Splatting [S1]).

**Loss Function:**
The total loss $\mathcal{L}$ is a weighted sum of L1 loss, Structural Similarity (SSIM) loss, and a frequency-modulated loss $\mathcal{L}_{\omega}$ [S1]:
$$
\mathcal{L} = \lambda_{L1}\mathcal{L}_1 + \lambda_{\text{ssim}}\mathcal{L}_{\text{ssim}} + \lambda_{\omega}\mathcal{L}_{\omega}
$$
where $\lambda_{L1}$, $\lambda_{\text{ssim}}$, and $\lambda_{\omega}$ are weighting coefficients [S1]. The frequency-modulated loss $\mathcal{L}_{\omega}$ utilizes masks $M_\omega$ to focus on specific frequency bands, with a linear schedule for target normalized frequencies $\omega$ [S1].

**Optimization Steps:**
The optimization runs for a fixed number of iterations using Stochastic Gradient Descent (SGD) [S1].

1.  **Render:** Render the current scene from the camera viewpoint using the current GES primitives.
2.  **Compute Loss:** Calculate $\mathcal{L}$ using the rendered image and the ground truth image [S1].
3.  **Backpropagate:** Compute gradients for $x, \kappa, \Sigma, \beta, c$ [S1].
4.  **Update Parameters:** Update parameters using SGD [S1].
5.  **Periodic Shape Reset:** Every `shape_reset_interval` iterations, reset the shape parameters $\beta$ to the initial value (2) [S1].
6.  **Periodic Shape Pruning:** Every `shape_pruning_interval` iterations, prune primitives based on shape attributes [S1].
    *   Calculate a shape-based metric for each primitive.
    *   If the metric exceeds the `pruning_threshold`, remove the primitive [S1].
7.  **Repeat** until `total_iterations` is reached [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_ges(sfm_points, images, total_iterations=40000):
    # Initialize primitives from SfM
    primitives = []
    for point in sfm_points:
        prim = Primitive(
            x=point.position,
            sigma=point.covariance,
            beta=2.0,  # Pure Gaussian initialization
            kappa=init_opacity(),
            sh_coeffs=init_sh()
        )
        primitives.append(prim)

    # Hyperparameters
    lr_beta = 0.0015
    density_gradient_threshold = 0.0003
    shape_reset_interval = 1000
    shape_pruning_interval = 100
    pruning_threshold = 0.5
    shape_strength = 0.1
    image_laplacian_scale = 0.2
    lambda_omega = 0.5

    for iter in range(total_iterations):
        # 1. Render and Compute Loss
        rendered_image = render(primitives, camera)
        target_image = images[iter % len(images)]
        
        # Frequency mask schedule
        omega = iter / total_iterations
        mask = compute_frequency_mask(omega)
        
        loss = compute_loss(rendered_image, target_image, mask, lambda_omega)
        
        # 2. Backpropagate
        gradients = compute_gradients(loss, primitives)
        
        # 3. Update Parameters
        for prim in primitives:
            prim.x -= lr * gradients[prim].x
            prim.kappa -= lr * gradients[prim].kappa
            prim.sigma -= lr * gradients[prim].sigma
            prim.beta -= lr_beta * gradients[prim].beta
            prim.sh_coeffs -= lr * gradients[prim].sh_coeffs
            
        # 4. Periodic Shape Reset
        if iter % shape_reset_interval == 0:
            for prim in primitives:
                prim.beta = 2.0
                
        # 5. Periodic Shape Pruning
        if iter % shape_pruning_interval == 0:
            primitives = prune_by_shape(primitives, pruning_threshold, shape_strength)
            
    return primitives

def prune_by_shape(primitives, threshold, strength):
    active_primitives = []
    for prim in primitives:
        # Calculate shape-based metric (e.g., gradient norm or density contribution)
        # Note: Exact formula for shape metric not provided in sources [S1]
        shape_metric = compute_shape_metric(prim, strength)
        
        if shape_metric < threshold:
            active_primitives.append(prim)
        # Else: primitive is pruned
    return active_primitives
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
*   $\lambda_{\omega}$: 0.5 [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** GPU-powered computation and specialized CUDA kernels are used to accelerate optimization [S1].
*   **Boundaries:** The frequency mask $M_\omega$ ensures coverage of all image parts, with a focus on details as optimization progresses (50% < $\omega \le$ 100%) [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Evaluated on an A6000 GPU [S1]. Achieves a balance between high fidelity and efficiency in novel view synthesis [S1].
*   **Strengths:** Eliminates the need for a variable density mechanism by controlling shape density via pruning [S1]. Enhances edge reconstruction through frequency-modulated loss [S1].
*   **Limitations:** Not given in the sources.
*   **When to use:** When high-fidelity novel view synthesis is required with efficient radiance field rendering [S1].
*   **When not to use:** Not given in the sources.

## Background
The GES Optimization Strategy was introduced in the paper "GES: Generalized Exponential Splatting for Efficient Radiance Field Rendering" [S1]. It builds on the principles of 3D Gaussian Splatting but introduces generalized exponential splats with a shape parameter $\beta$ [S1]. The method was developed to improve the balance between accuracy and computational load in radiance field optimization [S1].

## Sources
[S1] GES: Generalized Exponential Splatting for Efficient Radiance Field Rendering, https://doi.org/10.1109/cvpr52733.2024.01873, fetched 2026-10-26
