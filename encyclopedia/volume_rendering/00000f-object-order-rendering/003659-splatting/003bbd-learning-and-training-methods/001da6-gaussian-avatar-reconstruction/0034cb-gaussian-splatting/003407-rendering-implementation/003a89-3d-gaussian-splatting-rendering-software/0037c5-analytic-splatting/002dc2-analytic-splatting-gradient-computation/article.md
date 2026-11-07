# Analytic-Splatting Gradient Computation

Also called: Gradient of 2D Gaussian integral w.r.t covariance matrix, Gradient of Analytic 2D Gaussian Splat

## Overview
This method computes the exact gradient of the analytically integrated 2D Gaussian splat intensity with respect to the elements of the 2D covariance matrix $\hat{\Sigma}$ [S1]. It enables differentiable training of Gaussian splat parameters by providing the necessary gradients for the covariance components, which are otherwise difficult to optimize due to the non-linear projection and integration steps [S1]. Unlike standard rasterizers that treat pixels as point samples, this gradient computation accounts for the analytic integration over the pixel area, addressing aliasing artifacts inherent in point-based splatting [S1]. It serves as the foundational derivative required for the backward pass of Gaussian gradients [Backward Pass for Gaussian Gradients](9003-backward-pass-for-gaussian-gradients.md).

## How it works
**Inputs:**
- The analytically integrated 2D Gaussian intensity $\mathcal{I}^{2D}_g$.
- The 2D covariance matrix $\hat{\Sigma}$, decomposed into eigenvalues $\lambda_1, \lambda_2$ and eigenvectors $\mathbf{v}_1, \mathbf{v}_2$.
- The projected mean $\hat{\mu} = (\hat{\mu}_x, \hat{\mu}_y)$.
- The pixel coordinates $(u_x, u_y)$.
- The standard deviation $\sigma_1$ associated with the first eigenvalue.
- The cumulative distribution function (CDF) $S_{\sigma_2}$ for the second eigenvalue's standard deviation $\sigma_2$.
- The transformed coordinates $\tilde{u}_y$ in the eigenvector basis.

**Outputs:**
- The gradient matrix $\frac{\partial \mathcal{I}^{2D}_g}{\partial \hat{\Sigma}}$, specifically the elements $\frac{\partial \mathcal{I}^{2D}_g}{\partial \hat{\Sigma}_{11}}$, $\frac{\partial \mathcal{I}^{2D}_g}{\partial \hat{\Sigma}_{12}}$, and $\frac{\partial \mathcal{I}^{2D}_g}{\partial \hat{\Sigma}_{22}}$.

**Data Structures:**
- $\hat{\Sigma}$: A $2 \times 2$ symmetric positive-definite matrix. In implementation, store as three floats: $\hat{\Sigma}_{11}, \hat{\Sigma}_{12}, \hat{\Sigma}_{22}$.
- Eigen-decomposition: $\lambda_1, \lambda_2$ (scalars), $\mathbf{v}_1 = [v_{1x}, v_{1y}]^\top$, $\mathbf{v}_2 = [v_{2x}, v_{2y}]^\top$ (vectors).

**Steps:**

1.  **Compute Eigenvalue Gradients:**
    Calculate the partial derivatives of the intensity with respect to the eigenvalues $\frac{\partial \mathcal{I}^{2D}_g}{\partial \lambda_1}$ and $\frac{\partial \mathcal{I}^{2D}_g}{\partial \lambda_2}$. These are derived from the analytic integral formula of the 2D Gaussian.
    Calculate the partial derivatives of the eigenvalues with respect to the covariance matrix elements $\frac{\partial \lambda_k}{\partial \hat{\Sigma}_{ij}}$ for $k \in \{1, 2\}$ and $(i,j) \in \{(1,1), (1,2), (2,2)\}$. Standard matrix calculus formulas for symmetric matrix eigenvalue derivatives apply [S1].

2.  **Compute Eigenvector Gradients:**
    Calculate the partial derivatives of the intensity with respect to the eigenvectors $\frac{\partial \mathcal{I}^{2D}_g}{\partial \mathbf{v}_1}$ and $\frac{\partial \mathcal{I}^{2D}_g}{\partial \mathbf{v}_2}$.
    The gradient with respect to an eigenvector $\mathbf{v}_k$ is given by:
    $$
    \frac{\partial \mathcal{I}^{2D}_g}{\partial \mathbf{v}_k} = 2\pi \mathcal{I}_{\sigma 1} \left( \frac{\partial S_{\sigma_2}(\tilde{u}_y + 1/2)}{\partial \tilde{u}_y} - \frac{\partial S_{\sigma_2}(\tilde{u}_y - 1/2)}{\partial \tilde{u}_y} \right) \begin{bmatrix} u_x - \hat{\mu}_x \\ u_y - \hat{\mu}_y \end{bmatrix}
    $$
    where $\mathcal{I}_{\sigma 1}$ is the intensity component related to $\sigma_1$, and $S_{\sigma_2}$ is the CDF of a 1D Gaussian with standard deviation $\sigma_2$ [S1].
    Calculate the partial derivatives of the eigenvectors with respect to the covariance matrix elements $\frac{\partial \mathbf{v}_k}{\partial \hat{\Sigma}_{ij}}$. These depend on the specific eigen-decomposition algorithm used [S1].

3.  **Apply Chain Rule:**
    For each element $\hat{\Sigma}_{ij}$ where $(i,j) \in \{(1,1), (1,2), (2,2)\}$, compute the total gradient using the chain rule:
    $$
    \frac{\partial \mathcal{I}^{2D}_g}{\partial \hat{\Sigma}_{ij}} = \frac{\partial \mathcal{I}^{2D}_g}{\partial \lambda_1} \frac{\partial \lambda_1}{\partial \hat{\Sigma}_{ij}} + \frac{\partial \mathcal{I}^{2D}_g}{\partial \lambda_2} \frac{\partial \lambda_2}{\partial \hat{\Sigma}_{ij}} + \left(\frac{\partial \mathcal{I}^{2D}_g}{\partial \mathbf{v}_1}\right)^\top \frac{\partial \mathbf{v}_1}{\partial \hat{\Sigma}_{ij}} + \left(\frac{\partial \mathcal{I}^{2D}_g}{\partial \mathbf{v}_2}\right)^\top \frac{\partial \mathbf{v}_2}{\partial \hat{\Sigma}_{ij}}
    $$
    [S1]

**Parameters:**
- $\sigma_1, \sigma_2$: Standard deviations derived from the scale parameters of the 3D Gaussian. Typical values depend on the scene scale and camera distance.
- $\tilde{u}_y$: The y-coordinate of the pixel center in the local coordinate system defined by the eigenvectors.

**Edge Cases and Pitfalls:**
- **Degenerate Covariance:** If the covariance matrix is singular or near-singular, eigenvalue decomposition may become unstable. Guard against zero or negative eigenvalues.
- **Boundary Conditions:** The term involving the CDF derivatives $\frac{\partial S_{\sigma_2}}{\partial \tilde{u}_y}$ represents the boundary contributions. Ensure the CDF and its derivative (PDF) are evaluated correctly at $\tilde{u}_y \pm 1/2$.
- **Precision:** The subtraction of CDF derivatives can lead to cancellation errors if the pixel is far from the Gaussian center. Use high-precision arithmetic if necessary.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_analytic_splatting_gradient(I_2D_g, lambda_1, lambda_2, v_1, v_2, u_x, u_y, mu_hat_x, mu_hat_y, sigma_1, sigma_2, sigma_1_intensity):
    """
    Computes the gradient of the analytic 2D Gaussian intensity w.r.t. the 2D covariance matrix elements.
    
    Args:
        I_2D_g: Scalar, the integrated intensity.
        lambda_1, lambda_2: Scalars, eigenvalues of the 2D covariance matrix.
        v_1, v_2: Vectors [2], eigenvectors of the 2D covariance matrix.
        u_x, u_y: Scalars, pixel coordinates.
        mu_hat_x, mu_hat_y: Scalars, projected mean coordinates.
        sigma_1, sigma_2: Scalars, standard deviations corresponding to eigenvalues.
        sigma_1_intensity: Scalar, intensity component related to sigma_1.
        
    Returns:
        grad_Sigma_11, grad_Sigma_12, grad_Sigma_22: Scalars, gradients w.r.t. covariance matrix elements.
    """
    
    # Step 1: Compute eigenvalue gradients dI/d_lambda
    # Note: The exact formula for dI/d_lambda depends on the specific analytic integral definition.
    # Assuming dI/d_lambda_1 and dI/d_lambda_2 are available from the forward pass or derived analytically.
    dI_d_lambda_1 = ... # Placeholder for specific derivative of intensity w.r.t lambda_1
    dI_d_lambda_2 = ... # Placeholder for specific derivative of intensity w.r.t lambda_2
    
    # Step 2: Compute eigenvector gradients dI/d_v
    # Calculate the boundary term involving the CDF derivative
    # S_sigma_2 is the CDF of N(0, sigma_2^2). Its derivative is the PDF.
    # Let pdf_sigma_2(x) be the PDF of N(0, sigma_2^2).
    # d/dy S_sigma_2(y) = pdf_sigma_2(y)
    
    # The term in the bracket is:
    # d/d(tilde_u_y) [ S_sigma_2(tilde_u_y + 0.5) - S_sigma_2(tilde_u_y - 0.5) ]
    # = pdf_sigma_2(tilde_u_y + 0.5) - pdf_sigma_2(tilde_u_y - 0.5)
    
    # We need tilde_u_y. This is the projection of (u - mu) onto v_2? 
    # Or is it the coordinate in the rotated frame?
    # Based on the formula, it seems tilde_u_y is the y-coordinate in the eigenbasis.
    # Let's assume v_1 and v_2 form the rotation matrix.
    # u_local = R^T * (u - mu)
    # tilde_u_y = u_local[1]
    
    diff_u = [u_x - mu_hat_x, u_y - mu_hat_y]
    # Rotate to eigenbasis
    # v_1 = [v1x, v1y], v_2 = [v2x, v2y]
    # R = [v1x v2x; v1y v2y]
    # R^T = [v1x v1y; v2x v2y]
    # tilde_u_x = v1x * diff_u[0] + v1y * diff_u[1]
    # tilde_u_y = v2x * diff_u[0] + v2y * diff_u[1]
    
    tilde_u_y = v_2[0] * diff_u[0] + v_2[1] * diff_u[1]
    
    # PDF of N(0, sigma_2^2) at x is (1/(sqrt(2*pi)*sigma_2)) * exp(-x^2 / (2*sigma_2^2))
    def pdf_sigma_2(x):
        return (1.0 / (sqrt(2 * pi) * sigma_2)) * exp(-x**2 / (2 * sigma_2**2))
        
    boundary_term = pdf_sigma_2(tilde_u_y + 0.5) - pdf_sigma_2(tilde_u_y - 0.5)
    
    # Gradient w.r.t eigenvector v_k
    # dI/d_v_k = 2 * pi * I_sigma_1 * boundary_term * [u_x - mu_x, u_y - mu_y]^T
    factor = 2 * pi * sigma_1_intensity * boundary_term
    dI_d_v_1 = [factor * diff_u[0], factor * diff_u[1]]
    dI_d_v_2 = [factor * diff_u[0], factor * diff_u[1]]
    
    # Step 3: Compute d_lambda/d_Sigma and d_v/d_Sigma
    # These are standard matrix calculus derivatives.
    # For a 2x2 symmetric matrix Sigma = [[a, b], [b, c]]
    # Eigenvalues lambda = (a+c)/2 +/- sqrt(((a-c)/2)^2 + b^2)
    # Let D = sqrt(((a-c)/2)^2 + b^2)
    # lambda_1 = (a+c)/2 + D
    # lambda_2 = (a+c)/2 - D
    
    # Derivatives of lambda w.r.t a, b, c
    # dD/da = 0.5 * (a-c) / (2*D) = (a-c)/(4*D)
    # dD/db = 2*b / (2*D) = b/D
    # dD/dc = -(a-c)/(4*D)
    
    # d_lambda_1/da = 0.5 + dD/da
    # d_lambda_1/db = dD/db
    # d_lambda_1/dc = 0.5 + dD/dc
    
    # d_lambda_2/da = 0.5 - dD/da
    # d_lambda_2/db = -dD/db
    # d_lambda_2/dc = 0.5 - dD/dc
    
    # Note: The source does not provide explicit formulas for d_lambda/d_Sigma and d_v/d_Sigma,
    # referring to external literature [26]. We must implement these standard derivatives.
    
    # Let a = Sigma_11, b = Sigma_12, c = Sigma_22
    a = Sigma_11
    b = Sigma_12
    c = Sigma_22
    
    D = sqrt(((a-c)/2)**2 + b**2)
    
    if D < 1e-9:
        # Degenerate case, eigenvalues are equal
        # Gradients are undefined or require special handling
        return 0, 0, 0
        
    dD_da = (a-c) / (4*D)
    dD_db = b / D
    dD_dc = -(a-c) / (4*D)
    
    dL1_da = 0.5 + dD_da
    dL1_db = dD_db
    dL1_dc = 0.5 + dD_dc
    
    dL2_da = 0.5 - dD_da
    dL2_db = -dD_db
    dL2_dc = 0.5 - dD_dc
    
    # Derivatives of eigenvectors w.r.t Sigma elements
    # For distinct eigenvalues, the derivative of eigenvector v_k w.r.t Sigma_ij is complex.
    # Standard formula: d v_k / d Sigma_ij = ...
    # This part is referenced as [26] in the source.
    # We will assume helper functions are available or implement standard formulas.
    
    # For simplicity, we denote these as known quantities in the final sum.
    # In a real implementation, these would be computed explicitly.
    
    # dI/d_Sigma_11 = dI/dL1 * dL1/da + dI/dL2 * dL2/da + (dI/dv1)^T * dv1/da + (dI/dv2)^T * dv2/da
    # dI/d_Sigma_12 = dI/dL1 * dL1/db + dI/dL2 * dL2/db + (dI/dv1)^T * dv1/db + (dI/dv2)^T * dv2/db
    # dI/d_Sigma_22 = dI/dL1 * dL1/dc + dI/dL2 * dL2/dc + (dI/dv1)^T * dv1/dc + (dI/dv2)^T * dv2/dc
    
    # Since dI/dv1 and dI/dv2 are the same vector in this specific formulation (based on the source formula),
    # we can simplify the eigenvector part.
    
    # However, the source formula for dI/dv_k is identical for both k=1 and k=2?
    # Let's re-read carefully.
    # "The eigenvector gradients are given by ... in R^2"
    # It seems the gradient w.r.t the eigenvector direction is the same vector, but it is projected onto the derivative of the eigenvector itself.
    
    # We need dv1/da, dv1/db, dv1/dc, etc.
    # This is non-trivial and depends on the specific eigen-decomposition.
    # We will leave these as placeholders for the implementer to fill from standard linear algebra libraries or [26].
    
    # For the purpose of this article, we state the structure.
    
    # Placeholder for eigenvector derivatives
    dv1_da = ... # Compute from eigen-decomposition derivative
    dv1_db = ...
    dv1_dc = ...
    dv2_da = ...
    dv2_db = ...
    dv2_dc = ...
    
    # Dot products
    dot_v1_da = dot(dI_d_v_1, dv1_da) + dot(dI_d_v_2, dv2_da)
    dot_v1_db = dot(dI_d_v_1, dv1_db) + dot(dI_d_v_2, dv2_db)
    dot_v1_dc = dot(dI_d_v_1, dv1_dc) + dot(dI_d_v_2, dv2_dc)
    
    grad_Sigma_11 = dI_d_lambda_1 * dL1_da + dI_d_lambda_2 * dL2_da + dot_v1_da
    grad_Sigma_12 = dI_d_lambda_1 * dL1_db + dI_d_lambda_2 * dL2_db + dot_v1_db
    grad_Sigma_22 = dI_d_lambda_1 * dL1_dc + dI_d_lambda_2 * dL2_dc + dot_v1_dc
    
    return grad_Sigma_11, grad_Sigma_12, grad_Sigma_22
```

## Subtopics

### [Backward Pass for Gaussian Gradients](9003-backward-pass-for-gaussian-gradients.md)
Computes the exact gradient of a scalar loss $L$ with respect to the input 3D Gaussian parameters (color $c$, opacity $o$, mean $\mu$, scale $s$, rotation $q$) by propagating pixel gradients backward through the alpha-compositing rasterizer. It enables differentiable training of Gaussian splat parameters by providing gradients for all learnable variables required for optimization. Unlike the forward pass which composites colors front-to-back, this method propagates gradients back-to-front to correctly account for occlusion and transparency.

## Operating characteristics
- **Time Complexity:** The computation involves eigen-decomposition and several vector operations. It is $O(1)$ per splat per pixel, but the eigen-decomposition adds a constant overhead compared to simple point-sampling gradients.
- **Memory Complexity:** $O(1)$ per splat, storing intermediate gradient values.
- **Strengths:** Provides exact gradients for the analytic integration, leading to better convergence and reduced aliasing artifacts during training.
- **Limitations:** More complex to implement than standard rasterizer gradients due to the need for eigenvalue/eigenvector derivatives.
- **When to use:** When training 3D Gaussian Splatting models with the Analytic-Splatting forward pass to ensure gradient consistency.

## Background
The method originates from the "Analytic-Splatting: Anti-Aliased 3D Gaussian Splatting via Analytic Integration" paper [S1]. It was developed to address the aliasing issues in standard 3D Gaussian Splatting by providing a differentiable analytic integration of the Gaussian over the pixel area. The gradient computation is a key component of this method, enabling end-to-end training.

## Sources
[S1] Analytic-Splatting: Anti-Aliased 3D Gaussian Splatting via Analytic Integration, https://doi.org/10.1007/978-3-031-72643-9_17, fetched 2026-10-26
