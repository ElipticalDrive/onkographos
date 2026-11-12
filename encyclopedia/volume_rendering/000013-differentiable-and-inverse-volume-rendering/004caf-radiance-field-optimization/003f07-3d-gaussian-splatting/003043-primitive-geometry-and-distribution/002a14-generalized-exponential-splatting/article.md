# Generalized Exponential Splatting
Also called: GES

## Overview
Generalized Exponential Splatting (GES) is a 3D scene representation method that replaces the Gaussian basis functions of 3D Gaussian Splatting with Generalized Exponential Functions (GEF) [S1]. It introduces a learnable shape parameter $\beta$ for each primitive to better model signals with sharp edges and high-frequency content, reducing the number of particles required and thus memory footprint while maintaining competitive rendering quality [S1][S2, unofficial]. Unlike standard Gaussian Splatting, which assumes low-pass characteristics, GES allows primitives to adapt their shape to fit high-bandwidth signals like squares and triangles more efficiently [S1].

## How it works
### Inputs and Outputs
**Inputs:**
- A set of $N$ primitives, each defined by location $\boldsymbol{\mu}$, scale $\alpha$, amplitude $A$, and shape parameter $\beta$ [S1].
- Target images for optimization [S1].

**Outputs:**
- Rendered 2D images from novel viewpoints [S1].
- Optimized primitive parameters ($\boldsymbol{\mu}, \alpha, A, \beta$) [S1].

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

For 3D rendering, the GEF is extended to a multivariate form [S1]. Let $\mathbf{x}$ be a point in 3D space, $\boldsymbol{\mu}$ be the mean position, and $\boldsymbol{\Sigma}$ be the positive definite covariance matrix [G]. The 3D GEF is defined as [S1]:
$$ L(\mathbf{x}; \boldsymbol{\mu}, \boldsymbol{\Sigma}, \beta) = \exp \left\{ -\frac{1}{2} \left( (\mathbf{x} - \boldsymbol{\mu})^T \boldsymbol{\Sigma}^{-1} (\mathbf{x} - \boldsymbol{\mu}) \right)^{1/\beta} \right\} $$
The rendering process projects the 3D primitives to 2D screen space [G]. The 2D covariance $\boldsymbol{\Sigma}'_i$ and mean $\boldsymbol{\mu}'_i$ are computed as described in the parent article [G]. The 2D density at pixel $u$ is evaluated as [S1]:
$$ T_i(u) = \exp \left( -\frac{1}{2} (u - \boldsymbol{\mu}'_i)^T (\boldsymbol{\Sigma}'_i)^{-1} (u - \boldsymbol{\mu}'_i) \right)^{1/\beta_i} $$
The effective opacity is $A_i T_i(u)$, and colors are composited using alpha blending [G].

### Parameters and Optimization
**Initialization:**
-   $\beta$ is initialized to 0, which corresponds to an internal offset; the actual shape parameter used in rendering is $\beta_{\text{actual}} = \beta_{\text{stored}} + 2$ [S1].
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
-   SSIM loss with weight $\lambda_{ssim} = 0.2$ [S1].
-   Specialized frequency-modulated image loss (Image Laplacian) with weight $\lambda_\omega = 0.5$ [S1].
-   For the DoG term in the loss, if $\sigma^2$ is large, the ground truth image is downsampled by a factor `scaleim,freq` (0.2) and the mask $M_\omega$ is upsampled similarly before calculating the loss [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_primitives(N):
    primitives = []
    for _ in range(N):
        p = {
            'mu': random_3d_position(),
            'Sigma': random_positive_definite_matrix(),
            'A': random_positive_scalar(),
            'beta': 0.0,
            'color': random_color()
        }
        primitives.append(p)
    return primitives

def render(primitives, camera):
    screen_primitives = []
    for p in primitives:
        beta_eff = p['beta'] + 2.0
        mu_2d = project_3d_to_2d(p['mu'], camera)
        Sigma_2d = project_covariance(p['Sigma'], camera)
        screen_primitives.append({
            'mu': mu_2d,
            'Sigma': Sigma_2d,
            'A': p['A'],
            'beta': beta_eff,
            'color': p['color'],
            'depth': p['mu'].z
        })
    
    screen_primitives.sort(key=lambda x: x['depth'])
    
    height, width = camera.resolution
    image = [[0.0, 0.0, 0.0] for _ in range(height)]
    alpha_accum = [[0.0 for _ in range(width)] for _ in range(height)]
    
    for prim in screen_primitives:
        bbox = compute_bounding_box(prim['mu'], prim['Sigma'])
        for y in range(max(0, bbox.y_min), min(height, bbox.y_max)):
            for x in range(max(0, bbox.x_min), min(width, bbox.x_max)):
                dist_sq = (vector_sub((x, y), prim['mu'])).T @ inv(prim['Sigma']) @ vector_sub((x, y), prim['mu'])
                T = math.exp(-0.5 * dist_sq) ** (1.0 / prim['beta'])
                alpha = prim['A'] * T
                
                if alpha_accum[y][x] >= 1.0:
                    continue
                    
                c = prim['color']
                image[y][x] = [
                    image[y][x][0] + c[0] * alpha * (1.0 - alpha_accum[y][x]),
                    image[y][x][1] + c[1] * alpha * (1.0 - alpha_accum[y][x]),
                    image[y][x][2] + c[2] * alpha * (1.0 - alpha_accum[y][x])
                ]
                alpha_accum[y][x] += alpha * (1.0 - alpha_accum[y][x])
                
    return image

def optimize(primitives, images, cameras, iterations=40000):
    for it in range(iterations):
        cam, gt_image = random_sample(cameras, images)
        pred_image = render(primitives, cam)
        loss = compute_loss(pred_image, gt_image)
        gradients = compute_gradients(loss, primitives)
        update_parameters(primitives, gradients, learning_rates)
        
        if 500 <= it <= 15000 and it % 100 == 0:
            densify_primitives(primitives, gradients, threshold=0.0003)
        if it % 100 == 0:
            prune_primitives(primitives, opacity_threshold=0.005, shape_threshold=0.005)
        if it % 3000 == 0:
            reset_opacity(primitives)
        if it % 1000 == 0:
            reset_shape(primitives)
```

### Edge Cases and Pitfalls
-   **Numerical Instability**: Optimization can fail for certain signal types and component counts due to numerical instability in both Gaussian and GEF mixtures [S1].
-   **Analytical Difficulty**: The GEF involves complex integrals of exponentials without closed form depending on $\beta$, making analytical study difficult [S1].
-   **Approximation Error**: In the worst-case scenario (square larger than circle), the GES approximation results in a 36.34% energy error in the lost energy of all rays passing through all splatting components [S1]. The error ratio $\eta$ is bounded by $0.2732 \le \eta \le 0.3634$ [S1].

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
