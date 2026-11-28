# Analytic-Splatting

Also called: Analytic Integration for 3D Gaussian Splatting

## Overview
Analytic-Splatting is a rendering method that replaces the point-sampling of 2D Gaussians in 3D Gaussian Splatting (3DGS) with an analytical approximation of the integral of the Gaussian signal over the pixel window area to reduce aliasing artifacts during zooming or resolution changes [S2]. Unlike [3D Gaussian Splatting](2812-3d-gaussian-splatting.md) which treats pixels as isolated points, or [Mip-Splatting](3410-3d-gaussian-splatting-mvp.md) which uses a hybrid filtering mechanism that suppresses high-frequency components, Analytic-Splatting preserves high-quality details by analytically integrating the signal using a conditioned logistic function approximation of the Gaussian CDF [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A list of $N$ 3D Gaussians, each defined by a 3D mean $\mathbf{m}$, a 3D covariance matrix $\mathbf{\Sigma}$ (or its Cholesky factor), opacity $\alpha$, and color $\mathbf{c}$ [S2]. A camera view (projection matrix) and pixel coordinates $\mathbf{u} = [u_x, u_y]^\top$ [S2].
*   **Output:** The final color $\mathbf{C}(\mathbf{u})$ for each pixel [S2].

**Data Structures:**
The method operates on the projected 2D Gaussian parameters derived from the 3D input [S2].
*   $\hat{\bm{\mu}} \in \mathbb{R}^2$: The projected 2D mean [S2].
*   $\hat{\bm{\Sigma}} \in \mathbb{R}^{2\times2}$: The projected 2D covariance matrix [S2].
*   $\hat{\bm{\Sigma}}^{-1} = \begin{bmatrix} a & b \\ b & c \end{bmatrix}$: The inverse covariance matrix, decomposed into coefficients:
    $$ a = \frac{\hat{\Sigma}_{22}}{|\hat{\bm{\Sigma}}|}, \quad b = -\frac{\hat{\Sigma}_{12}}{|\hat{\bm{\Sigma}}|}, \quad c = \frac{\hat{\Sigma}_{11}}{|\hat{\bm{\Sigma}}|} $$
    [S2]
*   $\{\lambda_1, \lambda_2\}$: Eigenvalues of $\hat{\bm{\Sigma}}$ [S2].
*   $\{\mathbf{v}_1, \mathbf{v}_2\}$: Corresponding eigenvectors, forming the matrix $V = \begin{bmatrix} \mathbf{v}_1 & \mathbf{v}_2 \end{bmatrix}$ [S2].
*   $\sigma_1 = \sqrt{\lambda_1}, \sigma_2 = \sqrt{\lambda_2}$: Standard deviations along the principal axes [S2].

**Numbered Steps:**

1.  **Project 2D Gaussian:** Compute the projected 2D mean $\hat{\bm{\mu}}$ and covariance $\hat{\bm{\Sigma}}$ from the 3D Gaussian parameters using the affine approximation of the perspective projection [S2]. Compute the inverse $\hat{\bm{\Sigma}}^{-1}$ and its coefficients $a, b, c$ [S2].

2.  **Eigenvalue Decomposition:** Perform eigenvalue decomposition on $\hat{\bm{\Sigma}}$ to obtain eigenvalues $\{\lambda_1, \lambda_2\}$ and eigenvectors $\{\mathbf{v}_1, \mathbf{v}_2\}$ [S2].

3.  **Transform Pixel Center:** Transform the pixel center $\mathbf{u}$ into the diagonalized coordinate system defined by the eigenvectors [S2]. Let $\tilde{\mathbf{u}} = [\tilde{u}_x, \tilde{u}_y]^\top$ be the transformed coordinates:
    $$ \tilde{\mathbf{u}} = V^\top (\mathbf{u} - \hat{\bm{\mu}}) $$
    Component-wise:
    $$ \tilde{u}_x = v_{1x} (u_x - \hat{\mu}_x) + v_{1y} (u_y - \hat{\mu}_y) $$
    $$ \tilde{u}_y = v_{2x} (u_x - \hat{\mu}_x) + v_{2y} (u_y - \hat{\mu}_y) $$
    [S2]

4.  **Compute 1D Logistic Approximation:** Define the standard logistic approximation function $S(x)$ for the Gaussian CDF:
    $$ S(x) = \frac{1}{1 + \exp(-1.6 \cdot x - 0.07 \cdot x^3)} $$
    [S2]
    Define the scaled logistic function for standard deviation $\sigma$:
    $$ S_\sigma(x) = S\left(\frac{x}{\sigma}\right) = \frac{1}{1 + \exp\left(-1.6 \cdot \frac{x}{\sigma} - 0.07 \cdot \left(\frac{x}{\sigma}\right)^3\right)} $$
    [S2]
    Compute the 1D integral approximations for the x and y axes over the unit pixel window centered at $\tilde{u}_x$ and $\tilde{u}_y$:
    $$ \mathcal{I}_{\sigma_1} = \sigma_1 \left[ S_{\sigma_1}\left(\tilde{u}_x + \frac{1}{2}\right) - S_{\sigma_1}\left(\tilde{u}_x - \frac{1}{2}\right) \right] $$
    $$ \mathcal{I}_{\sigma_2} = \sigma_2 \left[ S_{\sigma_2}\left(\tilde{u}_y + \frac{1}{2}\right) - S_{\sigma_2}\left(\tilde{u}_y - \frac{1}{2}\right) \right] $$
    [S2]

5.  **Compute 2D Intensity Integral:** Calculate the analytically approximated 2D Gaussian integral $\mathcal{I}_g^\text{2D}(\mathbf{u})$ by combining the 1D integrals:
    $$ \mathcal{I}_g^\text{2D}(\mathbf{u}) = 2\pi \mathcal{I}_{\sigma_1} \mathcal{I}_{\sigma_2} $$
    [S2]
    This replaces the point-sampled Gaussian value $g^{2D}(\mathbf{u}) = \exp\left(-\frac{1}{2}(\mathbf{u}-\hat{\bm{\mu}})^\top \hat{\bm{\Sigma}}^{-1}(\mathbf{u}-\hat{\bm{\mu}})\right)$ [S2].

6.  **Volume Rendering:** Compute the final color $\mathbf{C}(\mathbf{u})$ using alpha blending, where the transmittance $T_i$ and the contribution of each Gaussian $i$ use the integral $\mathcal{I}_{g-i}^\text{2D}$ instead of the point sample:
    $$ \mathbf{C}(\mathbf{u}) = \sum_{i\in N} T_i \mathcal{I}_{g-i}^\text{2D}(\mathbf{u} | \hat{\bm{\mu}}_i, \hat{\bm{\Sigma}}_i) \alpha_i \mathbf{c}_i $$
    $$ T_i = \prod_{j=1}^{i-1} \left(1 - \mathcal{I}_{g-j}^\text{2D}(\mathbf{u} | \hat{\bm{\mu}}_j, \hat{\bm{\Sigma}}_j) \alpha_j\right) $$
    [S2]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def logistic_cdf_approx(x):
    """
    Approximates the CDF of a standard Gaussian using a logistic function.
    S(x) = 1 / (1 + exp(-1.6*x - 0.07*x^3))
    """
    exponent = -1.6 * x - 0.07 * (x ** 3)
    return 1.0 / (1.0 + math.exp(exponent))

def scaled_logistic_cdf_approx(x, sigma):
    """
    Approximates the CDF of a Gaussian with standard deviation sigma.
    S_sigma(x) = S(x / sigma)
    """
    return logistic_cdf_approx(x / sigma)

def compute_analytic_splat_intensity(u_x, u_y, mu_x, mu_y, v1_x, v1_y, v2_x, v2_y, lambda1, lambda2):
    """
    Computes the 2D integral approximation I_g^2D for a single Gaussian.
    
    Args:
        u_x, u_y: Pixel center coordinates.
        mu_x, mu_y: Projected 2D mean of the Gaussian.
        v1_x, v1_y: Components of the first eigenvector.
        v2_x, v2_y: Components of the second eigenvector.
        lambda1, lambda2: Eigenvalues of the projected covariance matrix.
        
    Returns:
        The 2D intensity integral value.
    """
    # Step 3: Transform pixel center to diagonalized coordinates
    # u_tilde = V^T * (u - mu)
    delta_x = u_x - mu_x
    delta_y = u_y - mu_y
    
    u_tilde_x = v1_x * delta_x + v1_y * delta_y
    u_tilde_y = v2_x * delta_x + v2_y * delta_y
    
    # Step 2/4: Standard deviations
    sigma1 = math.sqrt(lambda1)
    sigma2 = math.sqrt(lambda2)
    
    # Step 4: Compute 1D integral approximations
    # I_sigma = sigma * [S_sigma(u + 0.5) - S_sigma(u - 0.5)]
    
    # X-axis integral
    s1_plus = scaled_logistic_cdf_approx(u_tilde_x + 0.5, sigma1)
    s1_minus = scaled_logistic_cdf_approx(u_tilde_x - 0.5, sigma1)
    i_sigma1 = sigma1 * (s1_plus - s1_minus)
    
    # Y-axis integral
    s2_plus = scaled_logistic_cdf_approx(u_tilde_y + 0.5, sigma2)
    s2_minus = scaled_logistic_cdf_approx(u_tilde_y - 0.5, sigma2)
    i_sigma2 = sigma2 * (s2_plus - s2_minus)
    
    # Step 5: Compute 2D intensity integral
    # I_g^2D = 2 * pi * I_sigma1 * I_sigma2
    intensity = 2.0 * math.pi * i_sigma1 * i_sigma2
    
    return intensity

def render_pixel_analytic(pixels, gaussians):
    """
    Renders a single pixel using Analytic-Splatting.
    
    Args:
        pixels: List of pixel centers (u_x, u_y) to render.
        gaussians: List of Gaussian objects. Each object must have:
                   - mu_x, mu_y (projected 2D mean)
                   - v1_x, v1_y, v2_x, v2_y (eigenvectors of projected covariance)
                   - lambda1, lambda2 (eigenvalues of projected covariance)
                   - alpha (opacity)
                   - c_r, c_g, c_b (color components)
                   
    Returns:
        Dictionary mapping pixel coordinates to final color (r, g, b).
    """
    results = {}
    
    for (u_x, u_y) in pixels:
        # Assumes gaussians are pre-sorted by depth for correct alpha blending
        t = 1.0  # Transmittance
        color_r = 0.0
        color_g = 0.0
        color_b = 0.0
        
        for g in gaussians:
            # Compute analytic intensity
            intensity = compute_analytic_splat_intensity(
                u_x, u_y,
                g.mu_x, g.mu_y,
                g.v1_x, g.v1_y,
                g.v2_x, g.v2_y,
                g.lambda1, g.lambda2
            )
            
            # Contribution
            contribution_r = t * intensity * g.alpha * g.c_r
            contribution_g = t * intensity * g.alpha * g.c_g
            contribution_b = t * intensity * g.alpha * g.c_b
            
            # Accumulate color
            color_r += contribution_r
            color_g += contribution_g
            color_b += contribution_b
            
            # Update transmittance
            t *= (1.0 - intensity * g.alpha)
            
            # Early exit if transmittance is negligible
            if t < 1e-4:
                break
        
        results[(u_x, u_y)] = (color_r, color_g, color_b)
        
    return results
```

**Parameters:**
*   Logistic linear coefficient: $1.6$ [S2].
*   Logistic cubic coefficient: $0.07$ [S2].
*   Pixel window size: Assumed to be $1 \times 1$ centered at the pixel coordinate [S2].

**Edge Cases and Pitfalls:**
*   **Correlation Term:** The 2D integral of a Gaussian with a non-diagonal covariance matrix contains an intractable correlation term $b(x-\hat{\mu}_x)(y-\hat{\mu}_y)$ [S2]. Analytic-Splatting resolves this by diagonalizing the covariance matrix and rotating the integration domain to align with the eigenvectors, approximating the 2D integral as a product of two independent 1D integrals [S2].
*   **Small Standard Deviation:** Unlike Mip-Splatting which convolves with a Gaussian kernel (approximating the integral as $g \circledast g_w$), Analytic-Splatting calculates the integral analytically [S2]. This avoids large gaps for high-frequency components (small $\sigma$) where convolution-based methods might fail [S2].
*   **Precision:** The logistic approximation is differentiable, allowing for gradient computation [S2]. However, extreme values of $x/\sigma$ in the exponential function may lead to overflow or underflow; standard numerical stability techniques (e.g., clamping) should be applied [S2].

## Subtopics

### [Analytic-Splatting Gradient Computation](5597-analytic-splatting-gradient-computation.md)
This method computes the exact gradient of the analytically integrated 2D Gaussian splat intensity with respect to the elements of the 2D covariance matrix ($\hat{\Sigma}$). It solves the problem of differentiating the analytic approximation used in [Analytic-Splatting](8084-analytic-splatting.md), which integrates the Gaussian density over the pixel area to reduce aliasing. Unlike point-sampling methods, this gradient accounts for the boundary effects of the pixel window by propagating derivatives through the eigen-decomposition of the covariance matrix.

## Operating characteristics
*   **Time Complexity:** The method replaces the exponential function evaluation in standard 3DGS with logistic function evaluations and eigen-decomposition [S2]. The logistic function involves exponentials and cubic terms, which are computationally more expensive than a single exponential but cheaper than Monte Carlo super-sampling [S2].
*   **Memory Complexity:** No additional memory is required beyond the standard 3DGS parameters, as the integral is computed analytically on the fly [S2].
*   **Performance:** Analytic-Splatting provides better anti-aliasing capability than standard 3DGS and Mip-Splatting, preserving high-quality details and fidelity, especially during zooming or resolution changes [S2]. It avoids the over-smoothing caused by Mip-Splatting's low-pass filtering [S2].
*   **Strengths:** Analytical solution avoids sampling noise; handles varying resolutions effectively; preserves high-frequency details better than convolution-based methods [S2].
*   **Limitations:** The eigen-decomposition and logistic function evaluation add computational overhead compared to the simple point-sampling of 3DGS [S2].

## Background
Analytic-Splatting was introduced by Zhihao Liang et al. in the paper "Analytic-Splatting: Anti-Aliased 3D Gaussian Splatting via Analytic Integration" (arXiv:2403.11056v2, 2024) [S2]. It addresses the aliasing artifacts in 3D Gaussian Splatting caused by treating pixels as isolated points, which leads to severe blurring or jaggies when rendering at varying resolutions [S2]. The method uses a conditioned logistic function as an analytic approximation of the cumulative distribution function (CDF) of the Gaussian signal [S2].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Analytic-Splatting: Anti-Aliased 3D Gaussian Splatting via Analytic Integration, https://doi.org/10.1007/978-3-031-72643-9_17, fetched 2026-10-26
[S3] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
