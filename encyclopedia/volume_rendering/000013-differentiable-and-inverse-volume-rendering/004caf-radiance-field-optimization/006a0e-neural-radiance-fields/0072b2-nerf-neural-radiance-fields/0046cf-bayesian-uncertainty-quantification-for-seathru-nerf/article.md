# Bayesian Uncertainty Quantification for SeaThru-NeRF

Also called: Bayesian Uncertainty Analysis for Underwater NeRF, Fisher Information Approximation for Spatial Uncertainty, Laplace Approximation Uncertainty Rendering, Laplace Approximation for NeRF Uncertainty, Spatial Perturbation Field $D_\omega$

## Overview
This method quantifies the spatial uncertainty of underwater 3D reconstructions represented by Neural Radiance Fields (NeRF) by approximating the posterior distribution of perturbation parameters using the Laplace approximation [S1]. It distinguishes itself from standard NeRF variants by explicitly modeling the sensitivity of the radiance field to coordinate perturbations, rendering the resulting uncertainty as a volumetric field to identify and remove artifacts [S1]. Unlike sibling methods that focus on view synthesis quality or compression, this approach provides a confidence metric for downstream tasks such as underwater navigation and infrastructure inspection [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   A trained SeaThru-NeRF model (or SeaThru-NeRF-lite) with base parameters $\phi^*$ [S1].
*   A set of training rays $R$ sampled from the training dataset [S1].
*   Ground truth colors $C_{ngt}(r)$ for the sampled rays [S1].
*   Grid size dimension $M$ (typically 256) [S1].
*   Regularization constant $\lambda$ [S1].

**Outputs:**
*   A spatial uncertainty field $U(x): \mathbb{R}^3 \to \mathbb{R}^+$ [S1].
*   A thresholded clean image where high-uncertainty regions (artifacts) are removed [S1].

### Data Structures
*   **Covariance Matrix $\Sigma$**: A diagonal matrix approximating the posterior covariance of the perturbation parameters $\omega$ [S1]. The diagonal elements represent the marginal variance $\sigma$ [S1].
*   **Spatial Uncertainty Field $U(x)$**: A volumetric data field rendered from the diagonal elements of $\Sigma$ [S1]. It is stored as an additional color channel alongside the rendered RGB image [S1].
*   **Perturbation Parameters $\omega$**: A vector of parameters added to the base model, assumed to follow a Gaussian prior $\omega \sim \mathcal{N}(0, \lambda^{-1})$ [S1].

### Method Steps

1.  **Initialize Perturbation Parameters**: Set $\omega = 0$ [S1]. At this state, the perturbed model equals the base model ($\tilde{C}_0(r) = C_{\phi^*}(r)$), which minimizes reconstruction error and serves as the mode of the posterior distribution [S1].

2.  **Define Negative Log-Likelihood**: Define the negative log-likelihood of the posterior distribution $p(\omega|I)$ as [S1]:
    $$
    h(\omega) = \mathbb{E}_n \mathbb{E}_{r \sim I_n} \|\tilde{C}_\omega(r) - C_{ngt}(r)\|_2^2 + \lambda \|\omega\|_2^2
    $$
    where $\tilde{C}_\omega(r)$ is the predicted color with perturbations, $C_{ngt}(r)$ is the ground truth color, and $\lambda \|\omega\|_2^2$ is the regularization term [S1].

3.  **Approximate Hessian via Fisher Information**: Directly computing the Hessian $H(\omega)$ of $h(\omega)$ is expensive [S1]. Instead, approximate it using the Fisher Information matrix $I(\omega)$, related by $I(\omega) \approx -H(\omega)$ [S1]. The Fisher information is defined as [S1]:
    $$
    I(\omega) = \mathbb{E}_{(r,y)} \left[ 4\epsilon_\omega(r) J_\omega(r)^\top J_\omega(r) + 2\lambda I \right]
    $$
    where $\epsilon_\omega(r) = \|\tilde{C}_\omega(r) - C_{ngt}(r)\|_2^2$ and $J_\omega(r) = \frac{\partial \tilde{C}_\omega(r)}{\partial \omega}$ is the Jacobian of the predicted color with respect to $\omega$ [S1].

4.  **Simplify Expectation**: Approximate the conditional expectation $\mathbb{E}_{y|r}[\epsilon_\omega(r)]$ as $1/2$ based on the likelihood $\mathcal{N}(C_{ngt}, 1/2)$ [S1]. This simplifies the Fisher Information to [S1]:
    $$
    I(\omega) = \mathbb{E}_r \left[ 2 J_\omega(r)^\top J_\omega(r) + 2\lambda I \right]
    $$

5.  **Estimate Fisher Information**: Approximate the expectation $\mathbb{E}_r$ by sampling $R$ rays [S1]:
    $$
    I(\omega) \approx \frac{2}{R} \sum_{r} J_\omega(r)^\top J_\omega(r) + 2\lambda I
    $$
    Consequently, the Hessian approximation is [S1]:
    $$
    H(\omega) \approx -\left( \frac{2}{R} \sum_{r} J_\omega(r)^\top J_\omega(r) + 2\lambda I \right)
    $$

6.  **Compute Covariance Matrix**: Using the Laplace approximation, the covariance matrix $\Sigma$ is the inverse of the negative Hessian evaluated at the mode $\omega=0$ [S1]:
    $$
    \Sigma = -H(0)^{-1}
    $$
    Exploiting the sparsity of $H(\omega)$ due to local vertex influence, approximate $\Sigma$ by considering only the diagonal elements [S1]:
    $$
    \Sigma \approx \text{diag}\left( \frac{2}{R} \sum_{r} J_\omega(r)^\top J_\omega(r) + 2\lambda I \right)^{-1}
    $$
    Given the definition $\Sigma = -H(0)^{-1}$ and $H(0) \approx -I(0)$, this implies $\Sigma \approx I(0)^{-1}$ [S1]. The inverse is applied element-wise to the diagonal sum [S1].

7.  **Extract Marginal Variance**: Extract the diagonal elements of $\Sigma$ to form the marginal variance vector $\sigma = (\sigma_x, \sigma_y, \sigma_z)$ [S1]. The norm $\|\sigma\|_2$ measures the local spatial uncertainty at each grid vertex [S1].

8.  **Render Uncertainty Field**: Define the spatial uncertainty field $U(x)$ by trilinearly interpolating the variance vector $\sigma$ at grid vertices [S1]:
    $$
    U(x) = \text{Trilinear}(x, \sigma)
    $$
    Strictly, $U(x)$ measures uncertainty at $(1+D_\omega^*)^{-1}(x)$, but for a trained SeaThru-NeRF where $D_\omega^* = 0$, these points are effectively identical [S1].

9.  **Artifact Removal**: Apply a threshold to $U(x)$ to identify regions with high uncertainty (artifacts) and remove them to yield a clean target object image [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_uncertainty_field(base_model, rays, ground_truth_colors, M, lambda_reg):
    """
    Computes the spatial uncertainty field U(x) for SeaThru-NeRF using Laplace approximation.
    
    Args:
        base_model: Trained SeaThru-NeRF network.
        rays: List of R rays sampled from training data.
        ground_truth_colors: Array of shape (R, 3) containing C_ngt(r).
        M: Grid size dimension (e.g., 256).
        lambda_reg: Regularization constant, typically 10^-4 / M^3.
        
    Returns:
        U: Volumetric uncertainty field (trilinearly interpolated from grid vertices).
    """
    R = len(rays)
    num_vertices = M ** 3
    accumulated_jacobian_squares = np.zeros(num_vertices)

    for r_idx in range(R):
        r = rays[r_idx]
        pred_color = base_model(r)
        # Compute Jacobian J_omega(r) = d(pred_color) / d(omega) via backpropagation
        J = compute_jacobian_wrt_perturbations(base_model, r)
        # Accumulate squared norms per parameter: diag(J^T J)_k = sum_i (J_ik)^2
        accumulated_jacobian_squares += np.sum(J ** 2, axis=0)

    # Fisher Information diagonal approximation: I_diag = (2/R) * sum(J^T J)_diag + 2*lambda
    fisher_diag = (2.0 / R) * accumulated_jacobian_squares + 2.0 * lambda_reg

    # Covariance diagonal approximation: Sigma_diag = 1 / I_diag
    sigma_diag = 1.0 / fisher_diag

    # Map scalar variances to grid vertices
    grid_sigma = np.zeros((M, M, M))
    grid_sigma.ravel()[:] = sigma_diag

    # Render uncertainty field via trilinear interpolation
    U = np.zeros((M, M, M))
    for x in range(M):
        for y in range(M):
            for z in range(M):
                U[x, y, z] = trilinear_interpolate(grid_sigma, x, y, z)
    return U

def compute_jacobian_wrt_perturbations(model, ray):
    # [G] Computes d(output)/d(input) via finite differences for self-containment
    num_params = model.num_perturbation_params
    J = np.zeros((3, num_params))
    base_output = model(ray)
    for k in range(num_params):
        perturbed_ray = ray.copy()
        perturbed_ray.perturbation_params[k] += 1e-5
        perturbed_output = model(perturbed_ray)
        J[:, k] = (perturbed_output - base_output) / 1e-5
    return J

def trilinear_interpolate(grid, x, y, z):
    # [G] Standard trilinear interpolation over a 3D grid
    x0, x1 = max(0, x - 1), min(grid.shape[0] - 1, x + 1)
    y0, y1 = max(0, y - 1), min(grid.shape[1] - 1, y + 1)
    z0, z1 = max(0, z - 1), min(grid.shape[2] - 1, z + 1)
    v000 = grid[x0, y0, z0]
    v100 = grid[x1, y0, z0]
    v010 = grid[x0, y1, z0]
    v110 = grid[x1, y1, z0]
    v001 = grid[x0, y0, z1]
    v101 = grid[x1, y0, z1]
    v011 = grid[x0, y1, z1]
    v111 = grid[x1, y1, z1]
    return (v000 + v100 + v010 + v110 + v001 + v101 + v011 + v111) / 8.0
```

### Parameters
*   **$\omega$**: Perturbation parameters added to the base model, assumed to follow a Gaussian prior $\omega \sim \mathcal{N}(0, \lambda^{-1})$ [S1].
*   **$\lambda$**: Regularization constant. In experiments, set to $\lambda = 10^{-4} / M^3$, where $M$ is the grid size dimension [S1].
*   **$M$**: Grid size dimension determining the granularity of spatial segmentation. Typical value 256; tested values include 16, 32, 64, 128, 256, 512 [S1].
*   **Threshold**: A scalar value applied to $U(x)$ to separate artifacts from valid geometry. Value not specified, adjusted incrementally to clean floating objects [S1].

### Edge Cases and Pitfalls
*   **Grid Size Sensitivity**: Low grid sizes (e.g., $M=16$) lead to insufficient uncertainty estimates and overconfidence due to coarse spatial segmentation losing high-frequency details [S1]. High grid sizes (e.g., $M=512$) diminish benefits due to substantial increases in computational resources and time, with optimal balance found at $M=256$ [S1].
*   **Lighting Assumptions**: The method assumes the light source is only natural ambient light; it fails in deep-sea regions where artificial lighting and multiple scattering occur [S1].
*   **Synthetic Data Overfitting**: Synthetic datasets may cause model overfitting due to simplified assumptions, leading to artificially high uncertainty estimation values [S1].
*   **Coordinate Mapping**: $U(x)$ technically measures uncertainty at $(1+D_\omega^*)^{-1}(x)$, not $x$. However, for trained SeaThru-NeRF where $D_\omega^* = 0$, these points are effectively identical [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Memory**: The method requires storing the covariance matrix $\Sigma$ or its diagonal approximation [S1]. The source notes that SeaThru-NeRF (larger model) uses approximately 23GB VRAM, while SeaThru-NeRF-lite uses ~7GB VRAM [S1]. The uncertainty field $U(x)$ is rendered as an additional color channel [S1].
*   **Time**: Computing the Fisher Information requires sampling $R$ rays and computing Jacobians via backpropagation for each ray [S1]. The complexity scales with the number of rays $R$ and the dimension of $\omega$ (grid vertices) [S1]. High grid sizes ($M=512$) significantly increase computational time [S1].

### Measured Performance
*   **Evaluation Metric**: Area Under Sparsification Error (AUSE) [S1]. Lower AUSE indicates better-calibrated uncertainty estimates [S1].
*   **Model Comparison**: SeaThru-NeRF (larger model) provides higher capacity and better generalization for uncertainty estimation compared to SeaThru-NeRF-lite, which struggles with complex data and high precision tasks [S1]. SeaThru-NeRF-lite has slightly lower image quality but excellent clarity [S1].
*   **Dataset Performance**: The method performs well on both synthetic and real datasets, though synthetic datasets may lead to overfitting and higher uncertainty values due to simplified assumptions [S1].

### Strengths and Limitations
*   **Strengths**: Explicitly infers spatial uncertainty, allowing for artifact removal via thresholding [S1]. Benefits downstream tasks like underwater navigation and infrastructure inspection [S1].
*   **Limitations**: Assumes natural ambient light only [S1]. Computationally expensive for high grid sizes [S1]. Synthetic data may skew uncertainty estimates [S1].

### When to Use
*   Use for underwater 3D reconstruction, ocean exploration, underwater reconnaissance, security surveillance, underwater navigation, localization, and infrastructure inspection [S1].
*   Choose SeaThru-NeRF over SeaThru-NeRF-lite when high precision and generalization are required and sufficient VRAM is available [S1].

## Background

### Origin and History
The method was introduced in the paper "Bayesian uncertainty analysis for underwater 3D reconstruction with neural radiance fields" [S1]. It builds upon the SeaThru-NeRF framework, which models underwater scenes with scattering media [S1]. The approach applies the Laplace approximation to quantify the posterior distribution of perturbation parameters, extending standard NeRF optimization to include uncertainty quantification [S1].

### Applications
The method is designed for applications in ocean exploration, underwater navigation, localization, and infrastructure inspection, where knowing the confidence of the reconstruction is critical for decision-making [S1].

## Sources

[S1] Bayesian uncertainty analysis for underwater 3D reconstruction with neural radiance fields, https://doi.org/10.1016/j.apm.2024.115806, fetched 2026-10-29
