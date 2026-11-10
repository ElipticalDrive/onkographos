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
*   **Covariance Matrix $\Sigma$**: A diagonal matrix approximating the posterior covariance of the perturbation parameters $\omega$. The diagonal elements represent the marginal variance $\sigma$ [S1].
*   **Spatial Uncertainty Field $U(x)$**: A volumetric data field rendered from the diagonal elements of $\Sigma$. It is stored as an additional color channel alongside the rendered RGB image [S1].
*   **Perturbation Parameters $\omega$**: A vector of parameters added to the base model, assumed to follow a Gaussian prior $\omega \sim \mathcal{N}(0, \lambda^{-1})$ [S1].

### Method Steps

1.  **Initialize Perturbation Parameters**: Set $\omega = 0$. At this state, the perturbed model equals the base model ($\tilde{C}_0(r) = C_{\phi^*}(r)$), which minimizes reconstruction error and serves as the mode of the posterior distribution [S1].

2.  **Define Negative Log-Likelihood**: Define the negative log-likelihood of the posterior distribution $p(\omega|I)$ as [S1]:
    $$
    h(\omega) = \mathbb{E}_n \mathbb{E}_{r \sim I_n} \|\tilde{C}_\omega(r) - C_{ngt}(r)\|_2^2 + \lambda \|\omega\|_2^2
    $$
    where $\tilde{C}_\omega(r)$ is the predicted color with perturbations, $C_{ngt}(r)$ is the ground truth color, and $\lambda \|\omega\|_2^2$ is the regularization term [S1].

3.  **Approximate Hessian via Fisher Information**: Directly computing the Hessian $H(\omega)$ of $h(\omega)$ is expensive. Instead, approximate it using the Fisher Information matrix $I(\omega)$, related by $I(\omega) \approx -H(\omega)$ [S1]. The Fisher information is defined as [S1]:
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
    \Sigma \approx \text{diag}\left( \left( \frac{2}{R} \sum_{r} J_\omega(r)^\top J_\omega(r) + 2\lambda I \right)^{-1} \right)
    $$
    *Note: The source text presents Eq. (44) as $\Sigma \approx \text{diag}\left( \frac{2}{R} \sum_{r} J_\omega(r)^\top J_\omega(r) + 2\lambda I \right)^{-1}$. Given the definition $\Sigma = -H(0)^{-1}$ and $H(0) \approx -I(0)$, this implies $\Sigma \approx I(0)^{-1}$. The inverse is applied to the sum.*

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
    
    # Step 1: Initialize omega = 0. 
    # The Jacobian J_omega(r) is computed with respect to perturbation parameters omega.
    # In practice, this involves backpropagating through the model with respect to 
    # the input coordinates or specific perturbation parameters if defined.
    # Here we assume J_omega(r) is the sensitivity of the output color to the 
    # perturbation parameters which are initialized to zero.
    
    # Step 2-5: Compute Fisher Information Approximation
    # I(omega) approx (2/R) * sum(J^T J) + 2*lambda*I
    
    # Initialize sum of outer products of Jacobians
    # The dimension of omega corresponds to the number of grid vertices or parameters.
    # For efficiency, we only compute the diagonal elements of the inverse later.
    # However, the source suggests computing the sum of J^T J first.
    
    # Note: The source implies J_omega(r) is the Jacobian of predicted color w.r.t omega.
    # If omega represents coordinate perturbations, J is a matrix of shape (3, dim_omega).
    # We need to accumulate J^T J.
    
    # To avoid storing the full Hessian/Fisher matrix which may be large,
    # we rely on the diagonal approximation.
    # The source states: "approximate Sigma by considering only the diagonal elements of H".
    # And Sigma = -H(0)^-1.
    # If H is approximated as diagonal, H_ii = - (2/R * sum_j (J_ij)^2 + 2*lambda).
    # Then Sigma_ii = 1 / (2/R * sum_j (J_ij)^2 + 2*lambda).
    
    # Let's assume we can compute the squared Jacobian norms per parameter.
    # In a real implementation, one would iterate over rays and accumulate gradients.
    
    # Placeholder for accumulated squared Jacobian norms per parameter dimension
    # dim_omega is the number of perturbation parameters (e.g., grid vertices)
    dim_omega = get_num_perturbation_params(base_model) 
    accumulated_jacobian_norms = np.zeros(dim_omega)
    
    for r in rays:
        # Get predicted color with omega=0 (base model)
        pred_color = base_model(r)
        
        # Compute error epsilon
        # epsilon = ||C_tilde - C_ngt||^2
        # But for Jacobian, we need d(C_tilde)/d(omega)
        # This requires backpropagation through the model w.r.t omega.
        # Since omega=0, we compute the gradient of the output w.r.t the perturbation inputs.
        
        # Compute Jacobian J_omega(r) = d(C_tilde)/d(omega)
        # This is typically done via automatic differentiation.
        J = compute_jacobian_wrt_perturbations(base_model, r) # Shape: (3, dim_omega)
        
        # Accumulate J^T J. We only need the diagonal of the sum for the final inverse.
        # (J^T J)_kk = sum_i (J_ik)^2
        accumulated_jacobian_norms += np.sum(J**2, axis=0)
        
    # Step 5: Finalize Fisher Information Diagonal
    # I_diag_k = (2/R) * accumulated_jacobian_norms_k + 2*lambda
    fisher_diag = (2.0 / R) * accumulated_jacobian_norms + 2.0 * lambda_reg
    
    # Step 6: Compute Covariance Diagonal
    # Sigma_diag = 1 / fisher_diag
    sigma_diag = 1.0 / fisher_diag
    
    # Step 7: Extract Marginal Variance
    # sigma = (sigma_x, sigma_y, sigma_z) is the marginal variance vector.
    # The source says "sigma = (sigma_x, sigma_y, sigma_z) ... norm ||sigma||_2 measures local spatial uncertainty".
    # And "U(x) = Trilinear(x, sigma)".
    # This implies sigma is a vector field at grid vertices.
    # The diagonal elements of Sigma correspond to the variance of each parameter.
    # If each parameter corresponds to a vertex, then sigma_diag is the variance at each vertex.
    
    # Map sigma_diag to grid vertices
    grid_sigma = map_to_grid_vertices(sigma_diag, M)
    
    # Step 8: Render Uncertainty Field
    # U(x) = Trilinear(x, sigma)
    # This creates a continuous field from the discrete vertex values.
    U = create_trilinear_field(grid_sigma)
    
    return U

def compute_jacobian_wrt_perturbations(model, ray):
    """
    Computes the Jacobian of the predicted color with respect to the perturbation parameters omega.
    This requires the model to accept perturbation parameters or for the perturbations 
    to be applied to the input coordinates.
    """
    # Implementation detail: [G]
    # The source mentions "Spatial perturbation field D_omega based on Bayes' rays".
    # And "Perturb input coordinates using D_omega".
    # So J is d(Color)/d(Coordinate_Perturbation).
    # This is computed via backpropagation through the NeRF network.
    pass

def map_to_grid_vertices(variances, M):
    """
    Maps the scalar variances (one per parameter/vertex) to a 3D grid of size MxMxM.
    """
    # Implementation detail: [G]
    # The source states "each entry of the parameter vector omega corresponds to a vertex in the grid".
    # So we reshape or scatter the variances into the grid.
    pass

def create_trilinear_field(grid_sigma):
    """
    Creates a continuous volumetric field U(x) by trilinear interpolation of grid_sigma.
    """
    # Implementation detail: [G]
    pass
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
*   **Memory**: The method requires storing the covariance matrix $\Sigma$ or its diagonal approximation. The source notes that SeaThru-NeRF (larger model) uses approximately 23GB VRAM, while SeaThru-NeRF-lite uses ~7GB VRAM [S1]. The uncertainty field $U(x)$ is rendered as an additional color channel [S1].
*   **Time**: Computing the Fisher Information requires sampling $R$ rays and computing Jacobians via backpropagation for each ray. The complexity scales with the number of rays $R$ and the dimension of $\omega$ (grid vertices). High grid sizes ($M=512$) significantly increase computational time [S1].

### Measured Performance
*   **Evaluation Metric**: Area Under Sparsification Error (AUSE) [S1]. Lower AUSE indicates better-calibrated uncertainty estimates [S1].
*   **Model Comparison**: SeaThru-NeRF (larger model) provides higher capacity and better generalization for uncertainty estimation compared to SeaThru-NeRF-lite, which struggles with complex data and high precision tasks [S1]. SeaThru-NeRF-lite has slightly lower image quality but excellent clarity [S1].
*   **Dataset Performance**: The method performs well on both synthetic and real datasets, though synthetic datasets may lead to overfitting and higher uncertainty values due to simplified assumptions [S1].

### Strengths and Limitations
*   **Strengths**: Explicitly infers spatial uncertainty, allowing for artifact removal via thresholding. Benefits downstream tasks like underwater navigation and infrastructure inspection [S1].
*   **Limitations**: Assumes natural ambient light only. Computationally expensive for high grid sizes. Synthetic data may skew uncertainty estimates [S1].

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
