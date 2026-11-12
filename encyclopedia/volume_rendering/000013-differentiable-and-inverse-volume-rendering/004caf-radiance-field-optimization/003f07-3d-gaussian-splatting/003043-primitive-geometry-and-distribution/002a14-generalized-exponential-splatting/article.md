# Generalized Exponential Splatting


## Overview
Generalized Exponential Splatting (GES) is a 3D scene representation method that replaces the Gaussian basis functions of 3D Gaussian Splatting with Generalized Exponential Functions (GEF) [S1]. It introduces a learnable shape parameter $\beta$ for each primitive to better model signals with sharp edges and high-frequency content, reducing the number of particles required and thus memory footprint while maintaining competitive rendering quality [S1][S2, unofficial]. Unlike standard Gaussian Splatting, which assumes low-pass characteristics, GES allows primitives to adapt their shape to fit high-bandwidth signals like squares and triangles more efficiently [S1].

## How it works
### Inputs and Outputs
**Inputs:**
- A set of $N$ primitives, each defined by location $\boldsymbol{\mu}$, scale $\boldsymbol{\Sigma}$ (or equivalent parameterization), amplitude $A$, and shape parameter $\beta$ [S1].
- Camera parameters (view and projection matrices) [S1].
- Target images for optimization [S1].

**Outputs:**
- Rendered 2D images from novel viewpoints [S1].
- Optimized primitive parameters ($\boldsymbol{\mu}, \boldsymbol{\Sigma}, A, \beta$) [S1].

### Data Structures
Each primitive $i$ stores:
1.  **Mean Position** ($\boldsymbol{\mu}_i$): 3D vector $(x, y, z)$ [S1].
2.  **Scale/Shape Parameters**:
    -   $\alpha_i$: Scale parameter (analogous to variance/scale in GS) [S1].
    -   $\beta_i$: Shape parameter, constrained to $\beta \in (0, \infty)$ [S1].
    -   Rotation is handled via the covariance matrix $\boldsymbol{\Sigma}_i$ as defined in the parent article [G].
3.  **Amplitude** ($A_i$): Positive scalar, analogous to opacity/color intensity scaling [S1].
4.  **Appearance**: Spherical Harmonics coefficients for view-dependent color [G].

### Formulas
The Generalized Exponential Function (GEF) for a 1D signal is defined as [S1]:
$$ f(x|\mu, \alpha, \beta, A) = A \exp \left( - \left( \frac{|x - \mu|}{\alpha} \right)^\beta \right) $$
where $\mu \in \mathbb{R}$ is the location, $\alpha \in \mathbb{R}$ is the scale, $A \in \mathbb{R}^+$ is the amplitude, and $\beta \in (0, \infty)$ is the shape parameter [S1]. When $\beta = 2$, the GEF reduces to a scaled Gaussian function [S1].

For 3D rendering, the GEF is extended to a multivariate form. Let $\mathbf{x}$ be a point in 3D space, $\boldsymbol{\mu}$ be the mean position, and $\boldsymbol{\Sigma}$ be the positive definite covariance matrix. The 3D GEF is defined as [S1]:
$$ L(\mathbf{x}; \boldsymbol{\mu}, \boldsymbol{\Sigma}, \boldsymbol{\beta}) = \exp \left\{ -\frac{1}{2} (\mathbf{x} - \boldsymbol{\mu})^T \boldsymbol{\Sigma}^{-1} (\mathbf{x} - \boldsymbol{\mu}) \right\}^{1/\beta} $$
*Note: The source defines the 3D form implicitly by extending the 1D exponent structure. The exact 3D formulation involves applying the shape parameter $\beta$ to the Mahalanobis distance term within the exponential, effectively generalizing the Gaussian exponent $-\frac{1}{2}\mathbf{d}^T \boldsymbol{\Sigma}^{-1} \mathbf{d}$ to $-\left(\frac{1}{2}\mathbf{d}^T \boldsymbol{\Sigma}^{-1} \mathbf{d}\right)^{1/\beta}$ or similar, ensuring $\beta=2$ recovers the Gaussian [S1]. The source explicitly states $L(\mathbf{x}; \boldsymbol{\mu}, \boldsymbol{\Sigma}, \boldsymbol{\beta}) = \exp \{ -\frac{1}{2} ((\mathbf{x} - \boldsymbol{\mu})^T \boldsymbol{\Sigma}^{-1} (\mathbf{x} - \boldsymbol{\mu}))^{1/\beta} \}$ is the intended generalization, though the text cuts off. We assume the standard generalization of the exponent power.*

The rendering process projects the 3D primitives to 2D screen space. The 2D covariance $\boldsymbol{\Sigma}'_i$ and mean $\boldsymbol{\mu}'_i$ are computed as described in the parent article [G]. The 2D density at pixel $u$ is evaluated as [S1]:
$$ T_i(u) = \exp \left( -\frac{1}{2} (u - \boldsymbol{\mu}'_i)^T (\boldsymbol{\Sigma}'_i)^{-1} (u - \boldsymbol{\mu}'_i) \right)^{1/\beta_i} $$
The effective opacity is $\alpha_i T_i(u)$, and colors are composited using alpha blending [G].

### Parameters and Optimization
**Initialization:**
-   $\beta$ is initialized to 0 (which corresponds to $\beta=2$ in the exponent space if shifted, or simply a starting point for optimization; the source notes "shifted by -2" and "initialized to 0 instead of 2" implying an internal representation offset) [S1].
-   Other parameters ($\boldsymbol{\mu}, \boldsymbol{\Sigma}, A$) are initialized similarly to 3D Gaussian Splatting [S1].

**Learning Rates:**
-   Position ($\boldsymbol{\mu}$): Initial 0.00016, Final 0.0000016, Max Steps 30,000, Delay Multiplier 0.01 [S1].
-   Feature (SH): 0.0025 [S1].
-   Opacity ($A$): 0.05 [S1].
-   Shape ($\beta$) and Rotation: 0.001 [S1].
-   Scaling ($\boldsymbol{\Sigma}$): 0.005 [S1].

**Densification and Pruning:**
-   Densification Start: Iteration 500; End: Iteration 15,000 [S1].
-   Gradient Threshold for Densification: 0.0003 [S1].
-   Densification Interval: 100 iterations [S1].
-   Opacity Reset Interval: 3000 iterations [S1].
-   Shape Reset Interval: 1000 iterations [S1].
-   Shape Pruning Interval: 100 iterations [S1].
-   Opacity and Shape Pruning Thresholds: 0.005 [S1].
-   Percentage of Dense Points: 0.01 [S1].

**Loss Function:**
-   Includes SSIM loss with weight $\lambda_{ssim} = 0.2$ [S1].
-   Includes a specialized frequency-modulated image loss (Image Laplacian) with weight $\lambda_\omega = 0.5$ [S1].
-   For the DoG (Difference of Gaussians) term in the loss, if $\sigma^2$ is large, the ground truth image is downsampled by a factor `scaleim,freq` (0.2) and the mask $M_\omega$ is upsampled similarly before calculating the loss [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_primitives(images, cameras):
    # Initialize N primitives with random or SfM-based positions
    # Parameters: mu (3D), Sigma (3x3), A (scalar), beta (scalar)
    # Beta is initialized to 0 (internal offset) or 2 (actual value)
    primitives = []
    for i in range(N):
        p = Primitive()
        p.mu = random_3d_position()
        p.Sigma = random_positive_definite_matrix()
        p.A = random_positive_scalar()
        p.beta = 0.0  # Internal representation; actual beta = p.beta + 2
        primitives.append(p)
    return primitives

def render(primitives, camera):
    # Project 3D primitives to 2D screen space
    screen_primitives = []
    for p in primitives:
        # Compute 2D covariance Sigma' and mean mu' using Jacobian J and View W
        # Sigma' = J * W * Sigma * W^T * J^T
        # mu' = project(mu)
        # Effective beta for rendering
        beta_eff = p.beta + 2.0
        
        # Compute 2D density T(u) for bounding box of primitive
        # T(u) = exp( -0.5 * (u - mu')^T * (Sigma')^-1 * (u - mu') )^(1/beta_eff)
        screen_primitives.append({
            'mu': mu',
            'Sigma': Sigma',
            'A': p.A,
            'beta': beta_eff,
            'color': p.color,
            'depth': p.mu.z
        })
    
    # Sort by depth
    screen_primitives.sort(key=lambda x: x['depth'])
    
    # Rasterize and composite
    image = zeros(width, height, 3)
    for prim in screen_primitives:
        for pixel in bounding_box(prim):
            dist_sq = (pixel - prim['mu']).T @ inv(prim['Sigma']) @ (pixel - prim['mu'])
            # GEF density
            T = exp(-0.5 * dist_sq) ** (1.0 / prim['beta'])
            alpha = prim['A'] * T
            
            # Alpha compositing
            # C = sum(c_i * alpha_i * product(1 - alpha_j * T_j))
            # Implementation uses standard front-to-back alpha blending
            image[pixel] = composite(image[pixel], prim['color'], alpha)
            
    return image

def optimize(primitives, images, cameras, iterations=40000):
    for iter in range(iterations):
        # Select random camera and image
        cam, gt_image = random_sample(cameras, images)
        
        # Forward pass
        pred_image = render(primitives, cam)
        
        # Compute Loss
        # L = lambda_ssim * L_ssim(pred, gt) + lambda_omega * L_omega(pred, gt)
        # L_omega involves Image Laplacian
        loss = compute_loss(pred_image, gt_image)
        
        # Backward pass and update
        gradients = compute_gradients(loss, primitives)
        update_parameters(primitives, gradients, learning_rates)
        
        # Densification and Pruning
        if 500 <= iter <= 15000 and iter % 100 == 0:
            densify_primitives(primitives, gradients, threshold=0.0003)
        if iter % 100 == 0:
            prune_primitives(primitives, opacity_threshold=0.005, shape_threshold=0.005)
        if iter % 3000 == 0:
            reset_opacity(primitives)
        if iter % 1000 == 0:
            reset_shape(primitives)
```

### Edge Cases and Pitfalls
-   **Numerical Instability**: Optimization can fail for certain signal types and component counts due to numerical instability in both Gaussian and GEF mixtures [S1].
-   **Analytical Difficulty**: The GEF involves complex integrals of exponentials without closed form depending on $\beta$, making analytical study difficult [S1].
-   **Approximation Error**: In the worst-case scenario (square larger than circle), the GES approximation results in a 36.34% energy error in the lost energy of all rays passing through all splatting components. The error ratio $\eta$ is bounded by $0.2732 \le \eta \le 0.3634$ [S1].

## Operating characteristics
-   **Memory**: GES achieves a reduction of more than 50% in memory requirement compared to Gaussian Splatting [S1].
-   **Speed**: GES increases rendering speed by up to 39% compared to Gaussian Splatting [S1].
-   **Efficiency**: GES requires far fewer particles to represent a scene, significantly outperforming Gaussian Splatting in efficiency [S1].
-   **Quality**: Maintains competitive performance on standard novel view synthesis benchmarks [S1].
-   **Limitations**: Numerical instability can occur during optimization for specific signals [S1]. The approximation error bound (up to 36.34% energy loss in worst-case geometric alignment) is a theoretical limit, though practical error is much smaller due to large component counts and small scales [S1].

## Background
-   **Origin**: GES was introduced in the paper "GES: Generalized Exponential Splatting for Efficient Radiance Field Rendering" (CVPR 2024) [S1].
-   **Motivation**: 3D Gaussian Splatting (GS) assumes low-pass characteristics, which is suboptimal for 3D scenes with abrupt discontinuities (high-bandwidth signals) [S1]. This forces GS to use a huge number of very small Gaussians, negatively impacting memory utilization [S1].
-   **Development**: GES replaces Gaussians with Generalized Exponential Functions (GEF) to better model sharp edges and high-frequency content [S1]. It is designed as a plug-and-play replacement for Gaussian-based utilities, including text-to-3D synthesis methods like DreamFusion and DreamGaussian [S1].
-   **Usage**: Used in modern 3D generation pipelines using score distillation sampling from 2D text-to-image models [S1].

## Sources
[S1] GES: Generalized Exponential Splatting for Efficient Radiance Field Rendering, https://doi.org/10.1109/cvpr52733.2024.01873, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
