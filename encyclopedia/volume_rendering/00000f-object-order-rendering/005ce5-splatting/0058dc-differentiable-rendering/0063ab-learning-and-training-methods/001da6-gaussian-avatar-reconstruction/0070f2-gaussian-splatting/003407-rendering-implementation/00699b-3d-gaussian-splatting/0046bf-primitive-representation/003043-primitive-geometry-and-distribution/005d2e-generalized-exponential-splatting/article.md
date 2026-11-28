# Generalized Exponential Splatting


## Overview
Generalized Exponential Splatting (GES) is a 3D scene representation method that replaces the Gaussian basis functions used in 3D Gaussian Splatting with Generalized Exponential Functions (GEF) [S1]. It introduces a learnable shape parameter $\beta$ for each splat to better model signals with sharp edges and high-frequency content, reducing the number of particles required and thus memory footprint while maintaining competitive rendering quality [S1][S2, S3, unofficial]. Unlike standard Gaussian Splatting, which acts as a low-pass filter and requires many small primitives to represent abrupt discontinuities, GES adapts the kernel shape to the local signal characteristics, allowing for more efficient representation of high-bandwidth signals [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of initial 3D primitives (position, scale, rotation, opacity, appearance) derived from Structure-from-Motion (SfM) or a sparse point cloud, and a set of 2D images with camera poses [S1].
*   **Output:** A radiance field representation optimized via gradient-based descent, producing novel view synthesis images [S1].

**Data Structures:**
Each primitive $i$ stores the following parameters:
*   $\boldsymbol{\mu}_i \in \mathbb{R}^3$: Location parameter (mean position) [S1].
*   $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix defining shape and orientation in world space [G]. Parameterized as $\Sigma^w = R S S^T R^T$, where $R$ is rotation and $S$ is scaling [G].
*   $\beta_i \in (0, \infty)$: Learnable shape parameter controlling the exponentiation of the function [S1].
*   $\alpha_i \in \mathbb{R}^+$: Scale parameter, analogous to variance/scale in Gaussian Splatting [S1].
*   $A_i \in \mathbb{R}^+$: Positive amplitude parameter, analogous to opacity/color intensity scaling [S1].
*   Appearance attributes (e.g., Spherical Harmonics coefficients) [G].

**Rendering Pipeline:**
1.  **Projection:** Transform the 3D covariance $\Sigma^w$ to 2D screen space using the view matrix $W$ and the Jacobian of the perspective projection $J$: $\Sigma = J W \Sigma^w W^T J^T$ [G].
2.  **Effective Variance Calculation:** The effective projected variance $\sigma^2$ is determined by the trace of the 2D covariance matrix, adjusted for the shape parameter $\beta$. The specific normalization ensures the integral of the density is preserved [S1].
3.  **Splatting:** Project each primitive to a 2D ellipse. The density contribution of primitive $i$ at screen coordinate $\mathbf{x}$ is given by the Generalized Exponential Function:
    $$
    f(\mathbf{x}; \boldsymbol{\mu}, \boldsymbol{\Sigma}, \beta, A) = A \exp \left( - \left( \frac{\|\mathbf{x} - \boldsymbol{\mu}\|_{\Sigma^{-1}}}{\alpha} \right)^\beta \right)
    $$
    where $\|\cdot\|_{\Sigma^{-1}}$ denotes the Mahalanobis distance with respect to the inverse covariance matrix [S1]. Note: In 1D simulations, this simplifies to $f(x|\mu, \alpha, \beta, A) = A \exp \left( - \left( \frac{|x - \mu|}{\alpha} \right)^\beta \right)$ [S1].
4.  **Compositing:** Perform alpha compositing of the splats in screen-space order to produce the final color and opacity [G].

**Training and Optimization:**
1.  **Initialization:** Initialize splats with parameters including a positive $\beta$ value. The sources specify that for implementation purposes, $\beta$ is initialized to 0 (which corresponds to a shift of -2 in the exponent calculation relative to the standard Gaussian baseline, though the text notes this "should not have any effect on the optimization" due to the learnable nature of $\alpha$ and $\beta$) [S1].
2.  **Gradient Descent:** Optimize $\beta$ along with $\boldsymbol{\mu}$, $\boldsymbol{\Sigma}$ (via $R, S$), $A$, and appearance attributes using gradient-based optimizers (e.g., Adam) [S1].
3.  **Densification and Pruning:** Follow the standard 3D Gaussian Splatting protocol:
    *   Densification starts at iteration 500 and ends at 15,000 [S1].
    *   Cloning/splitting based on gradient magnitude threshold (0.0003) [S1].
    *   Pruning based on opacity and shape thresholds (0.005) [S1].
    *   Intervals: Densification every 100 iterations, opacity reset every 3000, shape reset every 1000, shape pruning every 100 [S1].
4.  **Loss Function:** Standard photometric loss (L1/SSIM) plus a specialized frequency-modulated image loss to encourage high-frequency detail recovery [S1]. The frequency loss involves an Image Laplacian. If the variance $\sigma^2$ is large, the ground truth image is downsampled by a factor `scaleim,freq` (0.2) and the mask $M_\omega$ is upsampled similarly before calculating the loss [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Generalized Exponential Function (1D for clarity, extends to 2D via Mahalanobis distance)
def generalized_exponential(x, mu, alpha, beta, A):
    """
    Computes the GEF value.
    x: input coordinate
    mu: location parameter
    alpha: scale parameter
    beta: shape parameter (learned)
    A: amplitude parameter
    """
    # Ensure beta is positive to avoid numerical issues with negative bases
    beta = max(beta, 1e-6)
    diff = abs(x - mu)
    # The formula from [S1]: A * exp(-(|x - mu| / alpha)^beta)
    # Note: The text mentions shifting functions by -2 for implementation, 
    # but the mathematical definition is as follows.
    exponent = (diff / alpha) ** beta
    return A * exp(-exponent)

# 3D Splatting Step (Conceptual)
def render_splat(screen_x, primitive):
    """
    Renders a single primitive at screen coordinate screen_x.
    primitive contains: mu_3d, Sigma_3d, beta, alpha, A, appearance
    """
    # 1. Project 3D covariance to 2D
    # Sigma_2d = J * W * Sigma_3d * W^T * J^T
    # This yields a 2x2 matrix Sigma_2d
    Sigma_2d = project_covariance(primitive.Sigma_3d, primitive.camera)
    
    # 2. Compute Mahalanobis distance squared
    # d2 = (x - mu_2d)^T * Sigma_2d^-1 * (x - mu_2d)
    diff = screen_x - primitive.mu_2d
    d2 = dot(diff, solve(Sigma_2d, diff))
    
    # 3. Compute effective distance for GEF
    # The text implies the radial distance is scaled by alpha.
    # In 1D, |x-mu|/alpha. In 2D, we use sqrt(d2) / alpha.
    dist = sqrt(d2) / primitive.alpha
    
    # 4. Apply GEF
    # beta is learned per primitive
    val = primitive.A * exp(-(dist ** primitive.beta))
    
    return val, primitive.appearance

# Training Loop Parameters [S1]
ITERATIONS = 40000
LR_POS_INIT = 0.00016
LR_POS_FINAL = 0.0000016
LR_DELAY_MULT = 0.01
LR_POS_MAX_STEPS = 30000
LR_FEATURE = 0.0025
LR_OPACITY = 0.05
LR_SHAPE = 0.001
LR_ROTATION = 0.001
LR_SCALING = 0.005
PERCENT_DENSE = 0.01
PRUNING_THRESHOLD = 0.005
DENSIFY_START = 500
DENSIFY_END = 15000
GRADIENT_THRESHOLD = 0.0003
SCALEIM_FREQ = 0.2 # Downsample factor for frequency loss
LAMBDA_OMEGA = 0.5 # Weight for image Laplacian loss
```

**Edge Cases and Pitfalls:**
*   **Numerical Instability:** Optimization can fail for certain signal types and component counts due to numerical instability in both Gaussian and GEF mixtures [S1].
*   **Large Variance:** When $\sigma^2$ is large, the Difference-of-Gaussians (DoG) term in the frequency loss becomes very large. This requires downsampling the ground truth image and upsampling the mask to prevent overflow or instability [S1].
*   **Approximation Error:** Due to PDF normalization constraints and the approximation of variance scaling based on $\beta$, there is an inherent energy error. In the worst-case scenario (square larger than circle or vice versa), the error ratio $\eta$ is bounded by $0.2732 \le \eta \le 0.3634$ [S1]. This implies up to 36.34% energy error in lost energy of rays passing through splatting components in worst-case geometric alignments, though practical error is much smaller due to many small components [S1].

## Operating characteristics
*   **Memory:** GES achieves a reduction of more than 50% in memory requirement compared to Gaussian Splatting [S1].
*   **Speed:** GES increases rendering speed by up to 39% compared to Gaussian Splatting [S1].
*   **Efficiency:** GES requires far fewer particles to represent a scene, particularly for signals with sharp edges (e.g., squares, triangles) [S1]. For example, fitting a square signal may require 2 GEFs vs 5 Gaussians [S1].
*   **Limitations:** The GEF involves complex integrals of exponentials without closed form depending on $\beta$, making analytical study difficult [S1]. Numerical instability remains a risk during optimization [S1].

## Background
*   **Origin:** GES was introduced in the paper "GES: Generalized Exponential Splatting for Efficient Radiance Field Rendering" [S1].
*   **Development:** It was developed to address the low-pass filtering limitation of standard 3D Gaussian Splatting, which struggles with high-bandwidth signals like sharp edges [S1].
*   **Usage:** GES is used in modern 3D generation pipelines using score distillation sampling from 2D text-to-image models (e.g., DreamGaussian setup), acting as a plug-and-play replacement for Gaussian Splatting [S1][S2, unofficial]. It is also cited in surveys of Gaussian Splatting variants [S2, S3, unofficial].

## Sources
[S1] GES: Generalized Exponential Splatting for Efficient Radiance Field Rendering, https://doi.org/10.1109/cvpr52733.2024.01873, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
