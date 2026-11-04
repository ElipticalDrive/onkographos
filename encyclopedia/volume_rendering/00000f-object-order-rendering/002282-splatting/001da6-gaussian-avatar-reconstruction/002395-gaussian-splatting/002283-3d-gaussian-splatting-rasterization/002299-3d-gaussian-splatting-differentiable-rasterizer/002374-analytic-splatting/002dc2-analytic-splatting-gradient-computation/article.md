# Analytic-Splatting Gradient Computation

Also called: Gradient of 2D Gaussian integral w.r.t covariance matrix, Gradient of Analytic 2D Gaussian Splat

## Overview
This method computes the exact gradient of the analytically integrated 2D Gaussian splat intensity with respect to the elements of the 2D covariance matrix $\hat{\Sigma}$ [S1]. It enables differentiable training of Gaussian splat parameters by providing the necessary backpropagation signals for the covariance, which is critical for optimizing the shape and orientation of the Gaussians in 3D Gaussian Splatting [S1]. It builds on [Analytic-Splatting](3025-analytic-splatting.md) by extending the analytical integration to include the derivative terms required for gradient-based optimization.

## How it works
**Inputs and Outputs:**
- **Input:** The analytically integrated 2D Gaussian intensity $\mathcal{I}^{2D}_g$, the 2D covariance matrix $\hat{\Sigma}$ (symmetric, $2 \times 2$), and the pixel center coordinates $(u_x, u_y)$ relative to the Gaussian center $(\hat{\mu}_x, \hat{\mu}_y)$.
- **Output:** The gradient matrix $\frac{\partial \mathcal{I}^{2D}_g}{\partial \hat{\Sigma}}$ with respect to the independent elements $\hat{\Sigma}_{11}, \hat{\Sigma}_{12}, \hat{\Sigma}_{22}$.

**Data Structures:**
The covariance matrix $\hat{\Sigma}$ is represented by its eigenvalues $\lambda_1, \lambda_2$ and corresponding eigenvectors $\mathbf{v}_1, \mathbf{v}_2$.
- $\lambda_1, \lambda_2 \in \mathbb{R}$: Eigenvalues.
- $\mathbf{v}_1, \mathbf{v}_2 \in \mathbb{R}^2$: Eigenvectors, normalized such that $\|\mathbf{v}_i\|=1$.

**Method Steps:**
The gradient is computed using the chain rule over the eigenvalues and eigenvectors. For each element $\hat{\Sigma}_{ij}$ where $(i,j) \in \{(1,1), (1,2), (2,2)\}$:

1.  **Decompose Covariance:** Compute the eigenvalues $\lambda_1, \lambda_2$ and eigenvectors $\mathbf{v}_1, \mathbf{v}_2$ of $\hat{\Sigma}$.
2.  **Compute Intensity Gradients w.r.t Eigenvalues:** Calculate $\frac{\partial \mathcal{I}^{2D}_g}{\partial \lambda_1}$ and $\frac{\partial \mathcal{I}^{2D}_g}{\partial \lambda_2}$. These are derived from the analytical intensity formula provided in [Analytic-Splatting](3025-analytic-splatting.md).
3.  **Compute Eigenvalue Gradients w.r.t Covariance:** Calculate $\frac{\partial \lambda_1}{\partial \hat{\Sigma}_{ij}}$ and $\frac{\partial \lambda_2}{\partial \hat{\Sigma}_{ij}}$. The source notes refer to external literature [26] for these specific derivatives [S1].
4.  **Compute Intensity Gradients w.r.t Eigenvectors:** Calculate $\frac{\partial \mathcal{I}^{2D}_g}{\partial \mathbf{v}_1}$ and $\frac{\partial \mathcal{I}^{2D}_g}{\partial \mathbf{v}_2}$.
5.  **Compute Eigenvector Gradients w.r.t Covariance:** Calculate $\frac{\partial \mathbf{v}_1}{\partial \hat{\Sigma}_{ij}}$ and $\frac{\partial \mathbf{v}_2}{\partial \hat{\Sigma}_{ij}}$. The source provides a specific closed-form expression for $\frac{\partial \mathbf{v}_2}{\partial \hat{\Sigma}_{ij}}$ [S1]:
    $$
    \frac{\partial \mathbf{v}_2}{\partial \hat{\Sigma}_{ij}} = 2\pi \mathcal{I}_{\sigma 1} \left( \frac{\partial S_{\sigma_2}(\tilde{u}_y + 1/2)}{\partial \tilde{u}_y} - \frac{\partial S_{\sigma_2}(\tilde{u}_y - 1/2)}{\partial \tilde{u}_y} \right) \begin{bmatrix} u_x - \hat{\mu}_x \\ u_y - \hat{\mu}_y \end{bmatrix}
    $$
    where:
    - $\mathcal{I}_{\sigma 1}$ is the intensity component associated with the first principal axis (derived from $\lambda_1$).
    - $S_{\sigma_2}(\cdot)$ is the cumulative distribution function (CDF) of a 1D Gaussian with standard deviation $\sigma_2 = \sqrt{\lambda_2}$ [S1].
    - $\tilde{u}_y$ is the projected coordinate along the second eigenvector $\mathbf{v}_2$.
    - The term $\frac{\partial S_{\sigma_2}(z)}{\partial z}$ is the probability density function (PDF) of the 1D Gaussian: $\frac{1}{\sqrt{2\pi}\sigma_2} e^{-z^2 / (2\sigma_2^2)}$.
    - The gradient for $\mathbf{v}_1$ is computed similarly, swapping indices 1 and 2 [S1].
6.  **Assemble Final Gradient:** Combine the terms using the chain rule:
    $$
    \frac{\partial \mathcal{I}^{2D}_g}{\partial \hat{\Sigma}_{ij}} = \frac{\partial \mathcal{I}^{2D}_g}{\partial \lambda_1} \frac{\partial \lambda_1}{\partial \hat{\Sigma}_{ij}} + \frac{\partial \mathcal{I}^{2D}_g}{\partial \lambda_2} \frac{\partial \lambda_2}{\partial \hat{\Sigma}_{ij}} + \left(\frac{\partial \mathcal{I}^{2D}_g}{\partial \mathbf{v}_1}\right)^\top \frac{\partial \mathbf{v}_1}{\partial \hat{\Sigma}_{ij}} + \left(\frac{\partial \mathcal{I}^{2D}_g}{\partial \mathbf{v}_2}\right)^\top \frac{\partial \mathbf{v}_2}{\partial \hat{\Sigma}_{ij}}
    $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def analytic_splatting_gradient(I_2D_g, sigma, mu, u_x, u_y):
    """
    Computes gradient of analytic 2D Gaussian intensity w.r.t covariance matrix elements.
    
    Args:
        I_2D_g: Analytic intensity value (scalar).
        sigma: 2D covariance matrix Sigma_hat (2x2 symmetric).
        mu: Gaussian center (2,).
        u_x, u_y: Pixel coordinates (scalars).
        
    Returns:
        grad_sigma: Gradient w.r.t Sigma_hat elements (2x2 symmetric, stored as [d11, d12, d22]).
    """
    # 1. Eigen decomposition
    eigenvalues, eigenvectors = np.linalg.eigh(sigma)
    lambda_1, lambda_2 = eigenvalues[0], eigenvalues[1]
    v_1, v_2 = eigenvectors[:, 0], eigenvectors[:, 1]
    
    # 2. Compute intensity gradients w.r.t eigenvalues
    # Note: Exact formulas for dI/dlambda depend on the specific analytic integral definition
    # from the parent article. Here we assume helper functions are available.
    dI_dlambda_1 = get_dI_dlambda_1(I_2D_g, lambda_1, lambda_2, u_x, u_y, mu)
    dI_dlambda_2 = get_dI_dlambda_2(I_2D_g, lambda_1, lambda_2, u_x, u_y, mu)
    
    # 3. Compute eigenvalue gradients w.r.t covariance elements
    # Refer to external literature [26] for exact formulas.
    # dlambda/dSigma_ij
    dlambda_1_dSigma = get_dlambda_dSigma(lambda_1, lambda_2, v_1, v_2)
    dlambda_2_dSigma = get_dlambda_dSigma(lambda_2, lambda_1, v_2, v_1) # Symmetry handling
    
    # 4. Compute intensity gradients w.r.t eigenvectors
    # dI/dv_k is a vector in R^2
    dI_dv_1 = get_dI_dv_1(I_2D_g, lambda_1, lambda_2, v_1, v_2, u_x, u_y, mu)
    dI_dv_2 = get_dI_dv_2(I_2D_g, lambda_1, lambda_2, v_1, v_2, u_x, u_y, mu)
    
    # 5. Compute eigenvector gradients w.r.t covariance elements
    # Using the specific formula from [S1] for dv_2/dSigma
    sigma_2 = np.sqrt(lambda_2)
    sigma_1 = np.sqrt(lambda_1)
    
    # Project pixel offset onto eigenvectors
    offset = np.array([u_x - mu[0], u_y - mu[1]])
    u_tilde_1 = np.dot(offset, v_1)
    u_tilde_2 = np.dot(offset, v_2)
    
    # PDF of 1D Gaussian
    def gaussian_pdf(z, sigma):
        return (1.0 / (np.sqrt(2 * np.pi) * sigma)) * np.exp(-0.5 * (z / sigma)**2)
    
    # Boundary terms for eigenvector gradient
    # S_sigma2 is CDF, derivative is PDF
    term_y_plus = gaussian_pdf(u_tilde_2 + 0.5, sigma_2)
    term_y_minus = gaussian_pdf(u_tilde_2 - 0.5, sigma_2)
    
    # Pre-factor for dv_2/dSigma
    # I_sigma1 is the intensity component along axis 1
    I_sigma1 = get_I_sigma1_component(lambda_1, sigma_1, u_tilde_1) 
    
    prefactor = 2 * np.pi * I_sigma1 * (term_y_plus - term_y_minus)
    direction_vector = offset # [u_x - mu_x, u_y - mu_y]
    
    # dv_2/dSigma_ij is a vector in R^2 scaled by the scalar derivative dSigma_ij/dSigma_ij (which is 1 or 2 for symmetric)
    # The formula in [S1] gives the vector part. We need to map this to specific matrix elements.
    # For a symmetric matrix, dSigma_12 is shared.
    
    # Construct gradients for each element
    # This is a simplified representation; exact mapping requires careful handling of symmetric matrix derivatives
    grad_11 = np.zeros(2)
    grad_12 = np.zeros(2)
    grad_22 = np.zeros(2)
    
    # Example calculation for dv_2/dSigma_11, dv_2/dSigma_12, dv_2/dSigma_22
    # These depend on the specific derivative of eigenvectors w.r.t matrix elements.
    # The source [S1] Eq 32 gives the vector result of the derivative operation.
    # We assume helper functions compute the scalar coefficients for each matrix element.
    
    dv_2_dSigma_11 = get_dv_dSigma_element(v_2, 1, 1) * prefactor * direction_vector
    dv_2_dSigma_12 = get_dv_dSigma_element(v_2, 1, 2) * prefactor * direction_vector
    dv_2_dSigma_22 = get_dv_dSigma_element(v_2, 2, 2) * prefactor * direction_vector
    
    # Similarly for v_1
    # ... (symmetric logic for v_1)
    
    # 6. Assemble final gradient
    # grad_Sigma_ij = dI/dlambda_1 * dlambda_1/dSigma_ij + dI/dlambda_2 * dlambda_2/dSigma_ij + (dI/dv_1)^T * dv_1/dSigma_ij + (dI/dv_2)^T * dv_2/dSigma_ij
    
    # Return gradients for Sigma_11, Sigma_12, Sigma_22
    return [grad_11, grad_12, grad_22]
```

**Parameters:**
- $\sigma_1, \sigma_2$: Standard deviations derived from eigenvalues $\lambda_1, \lambda_2$.
- $\tilde{u}_y$: Projected coordinate along the second eigenvector.
- Typical values: None specified; depends on scene geometry and Gaussian scale.

**Edge Cases and Pitfalls:**
- **Degenerate Eigenvalues:** If $\lambda_1 \approx \lambda_2$, the eigenvectors are not unique, leading to numerical instability in the gradient computation [G]. A small regularization term may be needed.
- **Precision:** The calculation involves exponentials and square roots. Care must be taken to avoid underflow/overflow for very small or large Gaussians [G].
- **Symmetric Matrix Handling:** The covariance matrix is symmetric. The gradient w.r.t $\hat{\Sigma}_{12}$ must account for the fact that changing $\hat{\Sigma}_{12}$ affects both the $(1,2)$ and $(2,1)$ entries. The source formula assumes standard matrix calculus conventions [S1].

## Operating characteristics
- **Time Complexity:** Dominated by eigen decomposition ($O(1)$ for $2 \times 2$ matrix) and evaluation of Gaussian PDFs/CDFs. Constant time per splat per pixel.
- **Memory Complexity:** $O(1)$ additional memory per splat.
- **Strengths:** Provides exact gradients for the analytic integration, enabling stable training of Gaussian shapes and reducing aliasing artifacts compared to point-sampling gradients.
- **Limitations:** More complex to implement than standard rasterization gradients due to the need for eigen-decomposition and specific eigenvector derivative formulas.
- **When to use:** When training 3D Gaussian Splatting models with analytic integration to improve visual quality and reduce aliasing.

## Background
The method originates from the "Analytic-Splatting" paper, which proposes anti-aliased 3D Gaussian Splatting via analytic integration [S1]. The gradient computation is a crucial component for the differentiable rendering pipeline, allowing the optimization of Gaussian parameters (including covariance) via gradient descent. The specific eigenvector gradient formula is derived in the context of this paper [S1].

## Sources
[S1] Analytic-Splatting: Anti-Aliased 3D Gaussian Splatting via Analytic Integration, https://doi.org/10.1007/978-3-031-72643-9_17, fetched 2026-10-26
