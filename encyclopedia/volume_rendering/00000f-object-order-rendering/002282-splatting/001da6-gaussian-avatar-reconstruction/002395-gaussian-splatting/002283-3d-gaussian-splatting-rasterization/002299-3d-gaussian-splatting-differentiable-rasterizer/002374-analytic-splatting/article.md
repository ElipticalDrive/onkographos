# Analytic-Splatting

Also called: Analytic Integration for 3D Gaussian Splatting

## Overview
Analytic-Splatting is a differentiable rasterization method that replaces the point-sampling of 2D Gaussians in 3D Gaussian Splatting (3DGS) with an analytical approximation of the integral of the Gaussian signal over the pixel window area [S1]. It solves the aliasing artifacts (blurring or jaggies) caused by treating pixels as isolated points when the camera zooms or resolution changes [S1]. Unlike [Mip-Splatting](3415-gaussian-splatting-rendering-pipeline.md) which uses a hybrid low-pass filter that suppresses high-frequency components and causes over-smoothing, Analytic-Splatting preserves high-quality details by calculating the integral analytically using a conditioned logistic function approximation of the Gaussian CDF [S1].

## How it works
**Inputs:**
*   Set of $N$ projected 2D Gaussians, each defined by mean $\hat{\bm{\mu}}_i \in \mathbb{R}^2$, covariance $\hat{\bm{\Sigma}}_i \in \mathbb{R}^{2\times2}$, opacity $\alpha_i \in [0,1]$, and color $\mathbf{c}_i \in \mathbb{R}^3$ [S1].
*   Pixel center coordinates $\mathbf{u} = [u_x, u_y]^\top$ [S1].

**Outputs:**
*   Final color $\mathbf{C}(\mathbf{u}) \in \mathbb{R}^3$ [S1].
*   Gradients $\frac{\partial \mathcal{I}_g^{2D}}{\partial \hat{\bm{\mu}}}$, $\frac{\partial \mathcal{I}_g^{2D}}{\partial \lambda}$, $\frac{\partial \mathcal{I}_g^{2D}}{\partial \mathbf{v}}$ for backpropagation [S1].

**Data Structures:**
*   $\hat{\bm{\Sigma}}$: $2 \times 2$ symmetric matrix [S1].
*   $\hat{\bm{\Sigma}}^{-1}$: Inverse covariance matrix, decomposed into coefficients $a, b, c$ [S1].
*   $\lambda_1, \lambda_2$: Eigenvalues of $\hat{\bm{\Sigma}}$ [S1].
*   $\mathbf{v}_1, \mathbf{v}_2$: Eigenvectors of $\hat{\bm{\Sigma}}$, forming matrix $V = [\mathbf{v}_1, \mathbf{v}_2]$ [S1].
*   $\sigma_1 = \sqrt{\lambda_1}, \sigma_2 = \sqrt{\lambda_2}$ [S1].

**Parameters:**
*   Logistic approximation coefficients: linear term $1.6$, cubic term $0.07$ [S1].
*   Derivative coefficient: $0.21$ (derived from the derivative of $-0.07 \cdot x^3$) [S1].

**Step-by-Step Method:**

1.  **Compute Inverse Covariance Coefficients:**
    Calculate the determinant $|\hat{\bm{\Sigma}}| = \hat{\Sigma}_{11}\hat{\Sigma}_{22} - \hat{\Sigma}_{12}^2$ [S1].
    Compute coefficients for $\hat{\bm{\Sigma}}^{-1} = \begin{bmatrix} a & b \\ b & c \end{bmatrix}$:
    $$ a = \frac{\hat{\Sigma}_{22}}{|\hat{\bm{\Sigma}}|}, \quad b = -\frac{\hat{\Sigma}_{12}}{|\hat{\bm{\Sigma}}|}, \quad c = \frac{\hat{\Sigma}_{11}}{|\hat{\bm{\Sigma}}|} $$
    [S1]

2.  **Eigenvalue Decomposition:**
    Perform eigenvalue decomposition on $\hat{\bm{\Sigma}}$ to obtain eigenvalues $\{\lambda_1, \lambda_2\}$ and corresponding eigenvectors $\{\mathbf{v}_1, \mathbf{v}_2\}$ [S1].
    Define standard deviations $\sigma_1 = \sqrt{\lambda_1}$ and $\sigma_2 = \sqrt{\lambda_2}$ [S1].

3.  **Transform Pixel Center:**
    Transform the pixel center $\mathbf{u}$ into the diagonalized coordinate system defined by the eigenvectors [S1]. Let $\tilde{\mathbf{u}} = [\tilde{u}_x, \tilde{u}_y]^\top$ [S1].
    $$ \tilde{\mathbf{u}} = V^\top (\mathbf{u} - \hat{\bm{\mu}}) $$
    Component-wise:
    $$ \tilde{u}_x = v_{1x}(u_x - \hat{\mu}_x) + v_{1y}(u_y - \hat{\mu}_y) $$
    $$ \tilde{u}_y = v_{2x}(u_x - \hat{\mu}_x) + v_{2y}(u_y - \hat{\mu}_y) $$
    [S1]

4.  **Compute 1D Logistic Approximations:**
    Define the conditioned logistic function $S(x)$ for the standard Gaussian CDF ($\sigma=1$):
    $$ S(x) = \frac{1}{1 + \exp(-1.6 \cdot x - 0.07 \cdot x^3)} $$
    [S1]
    
    Define the scaled logistic function for standard deviation $\sigma$:
    $$ S_\sigma(x) = S\left(\frac{x}{\sigma}\right) = \frac{1}{1 + \exp\left(-1.6 \cdot \frac{x}{\sigma} - 0.07 \cdot \left(\frac{x}{\sigma}\right)^3\right)} $$
    [S1]

5.  **Compute 2D Intensity Integral:**
    Approximate the integral of the 2D Gaussian signal over the 1x1 pixel window centered at $\mathbf{u}$ [S1]. The 2D integral $\mathcal{I}_g^{2D}(\mathbf{u})$ is decomposed into the product of two independent 1D integrals along the eigenvectors:
    $$ \mathcal{I}_g^{2D}(\mathbf{u}) = 2\pi \sigma_1 \sigma_2 \left[ S_{\sigma_1}\left(\tilde{u}_x + \frac{1}{2}\right) - S_{\sigma_1}\left(\tilde{u}_x - \frac{1}{2}\right) \right] \left[ S_{\sigma_2}\left(\tilde{u}_y + \frac{1}{2}\right) - S_{\sigma_2}\left(\tilde{u}_y - \frac{1}{2}\right) \right] $$
    [S1]
    
    Let $\mathcal{I}_{\sigma_1} = \sigma_1 \left[ S_{\sigma_1}(\tilde{u}_x + \frac{1}{2}) - S_{\sigma_1}(\tilde{u}_x - \frac{1}{2}) \right]$ and $\mathcal{I}_{\sigma_2} = \sigma_2 \left[ S_{\sigma_2}(\tilde{u}_y + \frac{1}{2}) - S_{\sigma_2}(\tilde{u}_y - \frac{1}{2}) \right]$ [S1]. Then:
    $$ \mathcal{I}_g^{2D}(\mathbf{u}) = 2\pi \mathcal{I}_{\sigma_1} \mathcal{I}_{\sigma_2} $$
    [S1]

6.  **Volume Rendering:**
    Compute the final color using alpha blending, replacing the point sample $g^{2D}$ with the integral $\mathcal{I}_g^{2D}$:
    $$ \mathbf{C}(\mathbf{u}) = \sum_{i \in N} T_i \mathcal{I}_{g-i}^{2D}(\mathbf{u} | \hat{\bm{\mu}}_i, \hat{\bm{\Sigma}}_i) \alpha_i \mathbf{c}_i $$
    $$ T_i = \prod_{j=1}^{i-1} \left( 1 - \mathcal{I}_{g-j}^{2D}(\mathbf{u} | \hat{\bm{\mu}}_j, \hat{\bm{\Sigma}}_j) \alpha_j \right) $$
    [S1]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def logistic_approx(x):
    """Conditioned logistic function S(x) approximating standard Gaussian CDF."""
    exponent = -1.6 * x - 0.07 * (x ** 3)
    return 1.0 / (1.0 + math.exp(exponent))

def scaled_logistic(x, sigma):
    """Scaled logistic function S_sigma(x) for standard deviation sigma."""
    x_scaled = x / sigma
    return logistic_approx(x_scaled)

def compute_intensity_integral(u_x, u_y, mu_x, mu_y, v1_x, v1_y, v2_x, v2_y, sigma1, sigma2):
    """Computes the 2D intensity integral I_g^2D for a single Gaussian."""
    dx = u_x - mu_x
    dy = u_y - mu_y
    
    tilde_u_x = v1_x * dx + v1_y * dy
    tilde_u_y = v2_x * dx + v2_y * dy
    
    s1_plus = scaled_logistic(tilde_u_x + 0.5, sigma1)
    s1_minus = scaled_logistic(tilde_u_x - 0.5, sigma1)
    i_sigma1 = sigma1 * (s1_plus - s1_minus)
    
    s2_plus = scaled_logistic(tilde_u_y + 0.5, sigma2)
    s2_minus = scaled_logistic(tilde_u_y - 0.5, sigma2)
    i_sigma2 = sigma2 * (s2_plus - s2_minus)
    
    intensity = 2.0 * math.pi * i_sigma1 * i_sigma2
    return intensity, tilde_u_x, tilde_u_y, i_sigma1, i_sigma2, s1_plus, s1_minus, s2_plus, s2_minus

def compute_gradients(u_x, u_y, mu_x, mu_y, v1_x, v1_y, v2_x, v2_y, sigma1, sigma2, tilde_u_x, tilde_u_y, i_sigma1, i_sigma2, s1_plus, s1_minus, s2_plus, s2_minus):
    """Computes gradients of I_g^2D w.r.t mean, eigenvalues, and eigenvectors per [S1]."""
    
    def d_s_sigma_dx(x, sigma):
        x_scaled = x / sigma
        s_val = scaled_logistic(x, sigma)
        return ((1.6 + 0.21 * (x_scaled ** 2)) * s_val * (1.0 - s_val)) / sigma

    def d_s_sigma_dsigma(x, sigma):
        x_scaled = x / sigma
        s_val = scaled_logistic(x, sigma)
        return ((1.6 + 0.21 * (x_scaled ** 2)) * s_val * (1.0 - s_val)) * (-x / (sigma ** 2))

    ds1_plus_dx = d_s_sigma_dx(tilde_u_x + 0.5, sigma1)
    ds1_minus_dx = d_s_sigma_dx(tilde_u_x - 0.5, sigma1)
    ds1_plus_dsigma = d_s_sigma_dsigma(tilde_u_x + 0.5, sigma1)
    ds1_minus_dsigma = d_s_sigma_dsigma(tilde_u_x - 0.5, sigma1)
    
    ds2_plus_dy = d_s_sigma_dx(tilde_u_y + 0.5, sigma2)
    ds2_minus_dy = d_s_sigma_dx(tilde_u_y - 0.5, sigma2)
    ds2_plus_dsigma = d_s_sigma_dsigma(tilde_u_y + 0.5, sigma2)
    ds2_minus_dsigma = d_s_sigma_dsigma(tilde_u_y - 0.5, sigma2)
    
    dI_dmu_x = 2.0 * math.pi * (
        i_sigma2 * (ds1_minus_dx - ds1_plus_dx) * v1_x +
        i_sigma1 * (ds2_minus_dy - ds2_plus_dy) * v2_x
    )
    dI_dmu_y = 2.0 * math.pi * (
        i_sigma2 * (ds1_minus_dx - ds1_plus_dx) * v1_y +
        i_sigma1 * (ds2_minus_dy - ds2_plus_dy) * v2_y
    )
    
    dI_dlambda1 = (math.pi * i_sigma2 / sigma1) * (
        (s1_plus - s1_minus) + sigma1 * (ds1_plus_dsigma - ds1_minus_dsigma)
    )
    dI_dlambda2 = (math.pi * i_sigma1 / sigma2) * (
        (s2_plus - s2_minus) + sigma2 * (ds2_plus_dsigma - ds2_minus_dsigma)
    )
    
    diff_u_x = u_x - mu_x
    diff_u_y = u_y - mu_y
    dI_dv1_x = 2.0 * math.pi * i_sigma2 * (ds1_plus_dx - ds1_minus_dx) * diff_u_x
    dI_dv1_y = 2.0 * math.pi * i_sigma2 * (ds1_plus_dx - ds1_minus_dx) * diff_u_y
    dI_dv2_x = 2.0 * math.pi * i_sigma1 * (ds2_plus_dy - ds2_minus_dy) * diff_u_x
    dI_dv2_y = 2.0 * math.pi * i_sigma1 * (ds2_plus_dy - ds2_minus_dy) * diff_u_y
    
    return {
        'dI_dmu': [dI_dmu_x, dI_dmu_y],
        'dI_dlambda1': dI_dlambda1,
        'dI_dlambda2': dI_dlambda2,
        'dI_dv1': [dI_dv1_x, dI_dv1_y],
        'dI_dv2': [dI_dv2_x, dI_dv2_y]
    }
```

**Edge Cases and Pitfalls:**
*   **Correlation Term:** The 2D integral with the correlation term $b(x-\hat{\mu}_x)(y-\hat{\mu}_y)$ is intractable analytically [S1]. The method resolves this by diagonalizing the covariance matrix and rotating the integration domain to align with eigenvectors, approximating the 2D integral as a product of two independent 1D integrals [S1].
*   **Small Standard Deviation:** Unlike Mip-Splatting which convolves with a Gaussian kernel and suppresses high-frequency components causing over-smoothing, Analytic-Splatting calculates the integral analytically to preserve details [S1].
*   **Super Sampling:** Unlike Monte Carlo super sampling, this method uses a closed-form approximation, avoiding the computation burden of increasing sample counts [S1].

## Operating characteristics
*   **Time Complexity:** The method replaces the exponential evaluation of the 2D Gaussian with a few logistic function evaluations and arithmetic operations [S1]. It is comparable in speed to standard 3DGS point sampling but more expensive than simple point sampling due to the eigen-decomposition and logistic calculations [S1]. It is significantly faster than super-sampling methods [S1].
*   **Memory Complexity:** No additional memory is required beyond the standard Gaussian parameters [S1]. The logistic function uses fixed constants [S1].
*   **Strengths:** Removes aliasing artifacts (blurring/jaggies) during zooming or resolution changes [S1]. Preserves high-frequency details better than Mip-Splatting [S1].
*   **Limitations:** Requires eigenvalue decomposition per Gaussian, which can be computationally intensive if not optimized [S1]. The approximation error depends on the validity of the logistic approximation for the specific range of $\sigma$ values encountered [S1].
*   **When to use:** When anti-aliasing is critical and high-frequency detail preservation is desired, especially for dynamic camera movements or varying resolutions [S1].
*   **When not to use:** If computational resources are extremely limited and aliasing is acceptable [S1].

## Background
Analytic-Splatting was introduced by Zhihao Liang, Qi Zhang, Wenbo Hu, Lei Zhu, Ying Feng, and Kui Jia in the paper "Analytic-Splatting: Anti-Aliased 3D Gaussian Splatting via Analytic Integration" (arXiv:2403.11056, 2024) [S1]. It addresses the limitation of standard 3D Gaussian Splatting, which treats pixels as isolated points and suffers from aliasing when the pixel footprint changes due to focal length or camera distance adjustments [S1]. The method builds on the 3DGS framework but replaces the point-sampling shading with an analytical integration of the Gaussian signal over the pixel window area [S1].

## Sources
[S1] Analytic-Splatting: Anti-Aliased 3D Gaussian Splatting via Analytic Integration, https://doi.org/10.1007/978-3-031-72643-9_17, fetched 2026-10-26
