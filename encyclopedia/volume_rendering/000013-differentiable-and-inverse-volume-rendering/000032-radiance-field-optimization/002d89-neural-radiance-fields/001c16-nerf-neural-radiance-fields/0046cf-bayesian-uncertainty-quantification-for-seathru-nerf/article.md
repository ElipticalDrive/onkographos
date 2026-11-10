# Bayesian Uncertainty Quantification for SeaThru-NeRF

Also called: Fisher Information Approximation for Spatial Uncertainty, Laplace Approximation for NeRF Uncertainty

## Overview
This method quantifies the spatial uncertainty of a trained SeaThru-NeRF model by approximating the posterior distribution of model perturbations using a Laplace approximation centered at the maximum a posteriori (MAP) estimate. It solves the problem of assessing confidence in 3D reconstruction by producing a continuous spatial uncertainty field $U(x)$ that indicates regions where the geometry or radiance is poorly constrained. It differs from standard NeRF variants by explicitly modeling parameter variance via the Hessian of the negative log-likelihood, approximated through Fisher Information, rather than just optimizing for photometric accuracy.

## How it works

### Inputs and Outputs
**Inputs:**
*   A trained SeaThru-NeRF model with base parameters $\phi^*$.
*   A set of $R$ sampled rays $\mathcal{R} = \{r_1, \dots, r_R\}$ from the training dataset.
*   Ground truth colors $C_{ngt}(r)$ for these rays.
*   Grid size dimension $M$ (typically 256).
*   Regularization constant $\lambda$.

**Outputs:**
*   A spatial uncertainty field $U: \mathbb{R}^3 \to \mathbb{R}^+$, represented as a 3D grid of scalar values.

### Data Structures
*   **Perturbation Parameters $\omega$**: A vector where each element corresponds to a vertex in a 3D grid of size $M \times M \times M$. The total dimension is $M^3$.
*   **Covariance Matrix $\Sigma$**: An approximation of the posterior covariance. In implementation, only the diagonal elements are stored due to sparsity assumptions.
*   **Variance Vector $\sigma$**: A vector of length $M^3$ containing the diagonal entries of $\Sigma$. $\sigma = (\sigma_x, \sigma_y, \sigma_z)$ conceptually, but stored as a flat vector corresponding to grid vertices.
*   **Uncertainty Field $U$**: A 3D texture or grid where values are obtained by trilinear interpolation of $\sigma$.

### Algorithm Steps

1.  **Initialization**:
    Set the perturbation parameters $\omega = 0$. At this point, the perturbed model $\tilde{C}_\omega(r)$ equals the base model $C_{\phi^*}(r)$. This minimizes the reconstruction error and serves as the mode of the posterior distribution [S1].

2.  **Define Negative Log-Likelihood**:
    The objective function $h(\omega)$ is defined as the negative log-likelihood of the posterior distribution, combining reconstruction error and a Gaussian prior on $\omega$ [S1]:
    $$
    h(\omega) = \mathbb{E}_n \mathbb{E}_{r \sim I_n} \|\tilde{C}_\omega(r) - C_{ngt}(r)\|_2^2 + \lambda \|\omega\|_2^2
    $$
    where $\tilde{C}_\omega(r)$ is the predicted color with perturbations, $C_{ngt}(r)$ is the ground truth color, and $\lambda \|\omega\|_2^2$ is the regularization term enforcing $\omega \sim \mathcal{N}(0, \lambda^{-1})$ [S1].

3.  **Approximate the Hessian via Fisher Information**:
    Direct computation of the Hessian $H(\omega) = \nabla^2 h(\omega)$ is expensive. Using the Laplace approximation, the posterior covariance is $\Sigma = -H(0)^{-1}$ [S1]. The Hessian is approximated using the Fisher Information Matrix $I(\omega)$:
    $$
    I(\omega) = \mathbb{E}_{(r,y)} \left[ 4\epsilon_\omega(r) J_\omega(r)^\top J_\omega(r) + 2\lambda I \right]
    $$
    where $\epsilon_\omega(r) = \|\tilde{C}_\omega(r) - C_{ngt}(r)\|_2^2$ and $J_\omega(r) = \frac{\partial \tilde{C}_\omega(r)}{\partial \omega}$ is the Jacobian of the predicted color with respect to $\omega$ [S1].

4.  **Simplify Expectation**:
    The conditional expectation $\mathbb{E}_{y|r}[\epsilon_\omega(r)]$ is approximated as $1/2$ based on the likelihood assumption $\mathcal{N}(C_{ngt}, 1/2)$ [S1]. This simplifies the Fisher Information to:
    $$
    I(\omega) = \mathbb{E}_r \left[ 2 J_\omega(r)^\top J_\omega(r) + 2\lambda I \right]
    $$
    Since $H(\omega) \approx -I(\omega)$, we have:
    $$
    H(\omega) \approx - \left( \mathbb{E}_r \left[ 2 J_\omega(r)^\top J_\omega(r) \right] + 2\lambda I \right)
    $$

5.  **Monte Carlo Approximation**:
    Approximate the expectation $\mathbb{E}_r$ by sampling $R$ rays:
    $$
    H(\omega) \approx - \left( \frac{2}{R} \sum_{r \in \mathcal{R}} J_\omega(r)^\top J_\omega(r) + 2\lambda I \right)
    $$
    Evaluated at $\omega=0$ [S1].

6.  **Diagonal Covariance Approximation**:
    Exploiting the sparsity of $H(\omega)$ (due to local vertex influence), approximate $\Sigma$ by considering only the diagonal elements of $-H(0)^{-1}$ [S1]:
    $$
    \Sigma \approx \text{diag} \left( \left( \frac{2}{R} \sum_{r \in \mathcal{R}} J_0(r)^\top J_0(r) + 2\lambda I \right)^{-1} \right)
    $$
    Note: The sources describe computing the inverse of the diagonal approximation or taking the diagonal of the inverse. Specifically, Eq. (44) in [S1] shows:
    $$
    \Sigma \approx \text{diag} \left( \frac{2}{R} \sum_{r \in \mathcal{R}} J_\omega(r)^\top J_\omega(r) + 2\lambda I \right)^{-1}
    $$
    Since the term inside is diagonal-dominant or treated as such, the inverse of the diagonal elements is computed [S1].

7.  **Compute Variance Vector**:
    Extract the marginal variance vector $\sigma$ from the diagonal entries of $\Sigma$. The norm $\|\sigma\|_2$ measures local spatial uncertainty at each grid vertex [S1].

8.  **Construct Uncertainty Field**:
    Define the spatial uncertainty field $U(x)$ by trilinearly interpolating $\sigma$ at grid vertices:
    $$
    U(x) = \text{Trilinear}(x, \sigma)
    $$
    This maps $\mathbb{R}^3 \to \mathbb{R}^+$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_uncertainty_field(base_model, rays, ground_truth_colors, M, lambda_reg, num_samples):
    """
    Computes the spatial uncertainty field for SeaThru-NeRF.
    
    Args:
        base_model: The trained SeaThru-NeRF model (parameters phi*).
        rays: List of R rays, each ray is a function or data structure allowing 
              evaluation of the model and computation of Jacobians.
        ground_truth_colors: Array of shape (R, 3) containing C_ngt(r).
        M: Grid size dimension (e.g., 256).
        lambda_reg: Regularization constant lambda.
        num_samples: Number of rays R to sample for approximation.
        
    Returns:
        uncertainty_field: 3D numpy array of shape (M, M, M) containing U(x).
    """
    
    # 1. Initialize perturbation parameters omega = 0
    # The dimension is M^3. In practice, we don't store the full vector if M is large,
    # but we need to compute the diagonal of the inverse Hessian.
    # We compute the diagonal elements of the Fisher Information matrix directly.
    
    # Initialize accumulator for the diagonal of the sum of Jacobian products
    # Shape: (M^3,)
    fisher_diag_accumulator = np.zeros(M**3)
    
    # Sample R rays
    # Note: In practice, rays are sampled from the training dataset.
    # For implementation, we iterate over a subset of the provided rays.
    sampled_indices = np.random.choice(len(rays), size=num_samples, replace=False)
    sampled_rays = [rays[i] for i in sampled_indices]
    sampled_gt_colors = ground_truth_colors[sampled_indices]
    
    R = len(sampled_rays)
    
    # 2. Compute Jacobian contributions
    # For each ray, compute J_0(r)^T J_0(r) and add to accumulator.
    # J_0(r) is the Jacobian of the predicted color w.r.t omega at omega=0.
    # Since omega corresponds to grid vertices, J is sparse.
    # We only need the diagonal of J^T J, which is sum_k (dC/domega_k)^2 for each output channel?
    # Actually, J is (3, M^3) for RGB output. J^T J is (M^3, M^3).
    # The diagonal element i is sum_{c=1 to 3} (dC_c / domega_i)^2.
    
    for r, y in zip(sampled_rays, sampled_gt_colors):
        # Evaluate model at omega=0 (base model)
        # predicted_color is shape (3,)
        predicted_color = base_model.evaluate(r) 
        
        # Compute Jacobian J_0(r) w.r.t omega
        # This requires backpropagation through the model with respect to the perturbation parameters.
        # In SeaThru-NeRF, perturbations are added to grid vertices.
        # We need the gradient of the output color w.r.t each grid vertex parameter.
        # Let's assume a function get_jacobian_diagonal(r) that returns a vector of shape (M^3,)
        # where element i is sum_c (dC_c / domega_i)^2.
        
        jacobian_sq_diag = compute_jacobian_squared_diagonal(base_model, r)
        
        # Accumulate: 2 * J^T J
        # Eq 42: I(omega) approx (2/R) sum J^T J + 2 lambda I
        # We are accumulating the sum part.
        fisher_diag_accumulator += jacobian_sq_diag
        
    # 3. Compute the diagonal of the Hessian approximation
    # H(0) approx - [ (2/R) * Sum(J^T J) + 2 lambda I ]
    # Let S = (2/R) * Sum(J^T J) + 2 lambda I
    # The diagonal of S is:
    # diag_S = (2/R) * fisher_diag_accumulator + 2 * lambda_reg
    
    diag_S = (2.0 / R) * fisher_diag_accumulator + 2.0 * lambda_reg
    
    # 4. Compute Covariance Sigma diagonal
    # Sigma approx diag(S)^(-1)
    # Element-wise inverse
    sigma_diag = 1.0 / diag_S
    
    # 5. Construct Uncertainty Field
    # sigma_diag is a vector of length M^3.
    # Reshape to 3D grid (M, M, M)
    # Note: The mapping from vector index to 3D coordinates depends on the grid layout (e.g., row-major).
    sigma_grid = sigma_diag.reshape((M, M, M))
    
    # The uncertainty field U(x) is defined by trilinear interpolation of sigma.
    # If we want the field values at the grid vertices, it is just sigma_grid.
    # If we want a continuous field, we use trilinear interpolation.
    # For storage, we return the grid values.
    
    return sigma_grid

def compute_jacobian_squared_diagonal(model, ray):
    """
    Computes the diagonal of J^T J for a single ray.
    J is the Jacobian of the predicted color (3,) w.r.t omega (M^3,).
    Returns a vector of shape (M^3,) where v[i] = sum_c (dC_c / domega_i)^2.
    """
    # This function requires access to the internal perturbation parameters of the model.
    # In PyTorch/TensorFlow, this involves setting up the computation graph.
    
    # 1. Get the perturbation parameters omega (initialized to 0)
    # 2. Forward pass to get predicted color C_tilde_0(r)
    # 3. Backward pass to get gradients dC/domega
    
    # Pseudocode for framework-specific implementation:
    # omega = torch.zeros(M**3, requires_grad=True)
    # C_pred = model(ray, omega) # Shape (3,)
    # loss = C_pred # We want gradients of output w.r.t omega
    # loss.backward(gradient=torch.ones(3), retain_graph=True)
    # grad = omega.grad # Shape (M^3,)
    
    # The diagonal of J^T J is sum over output channels of (grad_channel)^2.
    # If model outputs RGB, we need gradients for R, G, B separately or jointly.
    # If using a single backward pass with gradient=[1,1,1], we get sum of gradients? 
    # No, we need element-wise square of each channel's gradient.
    
    # Correct approach:
    # grads = []
    # for c in range(3):
    #     model.zero_grad()
    #     C_pred = model(ray, omega)
    #     C_pred[c].backward(retain_graph=True)
    #     grads.append(omega.grad.clone())
    # 
    # j_sq_diag = sum(g**2 for g in grads)
    
    # [G] Implementation detail: The Jacobian is sparse. Most entries are zero because
    # a ray only passes through a few voxels. The implementation should only compute
    # gradients for vertices influencing the ray's path to save memory/time.
    
    # Returning placeholder for the actual framework-dependent logic
    raise NotImplementedError("Framework-specific Jacobian computation required")
```

### Parameters
*   **$\omega$**: Perturbation parameters added to the base model. Assumed to follow a Gaussian prior $\omega \sim \mathcal{N}(0, \lambda^{-1})$ [S1].
*   **$\lambda$**: Regularization constant. Set to $\lambda = 10^{-4} / M^3$, where $M$ is the grid size dimension [S1].
*   **$M$**: Grid size dimension. Typical value is 256. Tested values include 16, 32, 64, 128, 256, 512 [S1].
*   **$R$**: Number of rays sampled for the Monte Carlo approximation. In experiments, 1000 iterations were used, but the specific number of rays per iteration is not explicitly defined as a fixed constant other than "sampling R rays" [S1].

### Edge Cases and Pitfalls
*   **Grid Size $M$**:
    *   **Low $M$ (e.g., 16)**: Leads to insufficient uncertainty estimates and overconfidence. Coarse spatial segmentation loses high-frequency details, preventing the model from capturing complex features [S1].
    *   **High $M$ (e.g., 512)**: Diminishes benefits due to substantial increases in computational resources and time. Beyond a certain point, increased precision does not yield corresponding performance gains in uncertainty accuracy [S1].
    *   **Optimal $M$**: 256 provides an optimal balance between performance and efficiency [S1].
*   **Evaluation Point**: Technically, $U(x)$ measures uncertainty at $(I + D_{\omega^*})^{-1}(x)$, not $x$. However, for a trained SeaThru-NeRF where $D_{\omega^*} = 0$, these points are effectively identical [S1].
*   **Model Variants**: SeaThru-NeRF-lite uses a smaller architecture (~7GB VRAM) vs SeaThru-NeRF (~23GB VRAM). While image quality is slightly lower, it still produces clear results [S1].

## Operating characteristics

*   **Time Complexity**: The method requires computing Jacobians for $R$ rays. Jacobian computation via backpropagation is $O(N_{params} \cdot R)$ in the worst case, but sparsity reduces this. The inversion of the diagonal matrix is $O(M^3)$.
*   **Memory Complexity**: Requires storing the grid of size $M^3$ for $\sigma$. For $M=256$, this is $16.7$ million entries. The Hessian approximation is diagonal, so $O(M^3)$ space.
*   **Performance**: The method provides well-calibrated uncertainty estimates, evaluated using Area Under Sparsification Error (AUSE). Lower AUSE indicates better calibration [S1].
*   **Strengths**: Provides spatially varying uncertainty; computationally feasible via diagonal approximation; integrates with existing NeRF training pipelines.
*   **Limitations**: Computationally expensive for large $M$; relies on the validity of the Laplace approximation and Gaussian prior; uncertainty is relative to the reconstruction error, not absolute geometric truth.

## Background

*   **Origin**: Proposed in "Bayesian uncertainty analysis for underwater 3D reconstruction with neural radiance fields" (2024) [S1].
*   **Development**: Builds on the Laplace approximation for posterior inference in neural networks [S1]. Adapts Fisher Information Matrix approximations for uncertainty quantification in volumetric rendering [S1].
*   **Usage**: Applied to underwater 3D reconstruction using SeaThru-NeRF, evaluated on real-world marine datasets (Curasao, IUI3, Panama, Japanese Gardens) and synthetic underwater data [S1].

## Sources

[S1] Bayesian uncertainty analysis for underwater 3D reconstruction with neural radiance fields, https://doi.org/10.1016/j.apm.2024.115806, fetched 2026-10-29
