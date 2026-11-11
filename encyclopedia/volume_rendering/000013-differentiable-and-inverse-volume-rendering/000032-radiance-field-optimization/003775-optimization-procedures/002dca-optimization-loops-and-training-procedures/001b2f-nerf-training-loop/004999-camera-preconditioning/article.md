# Camera Preconditioning

Also called: Camera Preconditioning for Neural Radiance Fields

## Overview
Camera Preconditioning (CamP) is a technique to normalize and decorrelate camera parameters during Neural Radiance Field (NeRF) optimization by transforming the optimization problem from direct camera parameters $\boldsymbol{\phi}_i$ to latent preconditioned parameters $\tilde{\boldsymbol{\phi}}_i$ [S1]. It addresses ill-conditioning caused by mixed units (e.g., translation in world units vs. focal length in pixels) and spurious correlations between parameters, which naive optimization or simple per-parameter scaling fails to resolve [S1]. Unlike [Adam](12051-camera-pose-and-extrinsic-intrinsic-optimization.md) which normalizes update magnitudes but ignores parameter sensitivity differences, CamP explicitly scales parameters based on their projected impact on the image plane [S1].

## How it works
**Inputs:**
*   A set of $N$ cameras, each with an initial parameter estimate $\boldsymbol{\phi}_i \in \mathbb{R}^k$ (where $k$ is the dimensionality of the parameterization, e.g., 7 for SE3 + focal length).
*   A set of $m$ proxy scene points $\{\mathbf{x}_j\}_{j=1}^m$ sampled within the frustum of each camera. These can be random points or feature points from Structure-from-Motion (SfM) [S1].

**Outputs:**
*   Preconditioner matrices $P_i^{-1} \in \mathbb{R}^{k \times k}$ for each camera $i$.
*   Optimized latent parameters $\tilde{\boldsymbol{\phi}}_i$.

**Data Structures:**
*   **Camera Parameters:** $\boldsymbol{\phi}_i$ is not stored directly for optimization. Instead, $\tilde{\boldsymbol{\phi}}_i$ is stored.
*   **Preconditioner:** $P_i^{-1}$ is a dense $k \times k$ matrix.
*   **Jacobian:** $J_{\Pi} \in \mathbb{R}^{2m \times k}$, where rows correspond to the 2D projection coordinates of the $m$ proxy points, and columns correspond to the $k$ camera parameters.

**Numbered Steps:**

1.  **Proxy Point Sampling:** For each camera $i$, sample $m$ 3D points $\{\mathbf{x}_j\}$ within the camera's frustum. [S1]
2.  **Jacobian Computation:** Compute the Jacobian matrix $J_{\Pi}$ of the projection function $\Pi(\mathbf{x}; \boldsymbol{\phi})$ with respect to the camera parameters $\boldsymbol{\phi}$, evaluated at the initial parameter estimate $\boldsymbol{\phi}_0$. The matrix $J_{\Pi}$ has dimensions $2m \times k$, where each row $j$ contains the partial derivatives $\frac{\partial \Pi(\mathbf{x}_j; \boldsymbol{\phi}_0)}{\partial \boldsymbol{\phi}}$ [S1].
3.  **Covariance Calculation:** Compute the covariance matrix $\Sigma_{\Pi}$ of the projection derivatives:
    $$ \Sigma_{\Pi} = J_{\Pi}^\top J_{\Pi} $$
    This matrix captures the sensitivity of the projected points to changes in each camera parameter and their correlations [S1].
4.  **Preconditioner Derivation:** Compute the preconditioner $P_i^{-1}$ using the dampened Zero Component Analysis (ZCA) whitening transform:
    $$ P_i^{-1} = (\Sigma_{\Pi} + \lambda \text{diag}(\Sigma_{\Pi}) + \mu I)^{-1/2} $$
    where $\lambda$ and $\mu$ are hyperparameters, and $I$ is the identity matrix [S1]. The term $(\cdot)^{-1/2}$ denotes the inverse matrix square root.
5.  **Latent Parameter Initialization:** Initialize the latent parameters $\tilde{\boldsymbol{\phi}}_i$ such that the initial actual parameters are recovered: $\boldsymbol{\phi}_i^{(0)} = P_i^{-1} \tilde{\boldsymbol{\phi}}_i^{(0)}$. Typically, $\tilde{\boldsymbol{\phi}}_i^{(0)} = P_i \boldsymbol{\phi}_i^{(0)}$ [S1].
6.  **Optimization Loop:**
    a. For each training step, retrieve the latent parameters $\tilde{\boldsymbol{\phi}}_i$.
    b. Compute the actual camera parameters: $\boldsymbol{\phi}_i = P_i^{-1} \tilde{\boldsymbol{\phi}}_i$ [S1].
    c. Use $\boldsymbol{\phi}_i$ to render images and compute the loss.
    d. Compute gradients with respect to $\tilde{\boldsymbol{\phi}}_i$ (via backpropagation through the multiplication by $P_i^{-1}$, which is constant during optimization).
    e. Update $\tilde{\boldsymbol{\phi}}_i$ using an optimizer like Adam [S1].

**Formulas:**
*   **Projection Sensitivity:** The effect of parameter perturbations is analyzed via the augmented projection function $\Pi_m(\boldsymbol{\phi}) : \mathbb{R}^k \to \mathbb{R}^{2m}$, concatenating projections of $m$ points [S1].
*   **ZCA Whitening:** The ideal preconditioner satisfies $\Sigma_{\tilde{\Pi}} = I_k$, where $\Sigma_{\tilde{\Pi}} = J_{\tilde{\Pi}}^\top J_{\tilde{\Pi}}$ and $J_{\tilde{\Pi}} = J_{\Pi} P^{-1}$. The solution is $P^{-1} = \Sigma_{\Pi}^{-1/2}$ [S1].
*   **Dampening:** To improve conditioning, especially with lens distortion parameters, dampening is added: $P^{-1} = (\Sigma_{\Pi} + \lambda \text{diag}(\Sigma_{\Pi}) + \mu I)^{-1/2}$ [S1].

**Parameters:**
*   $\lambda$: Hyperparameter for dampening the diagonal of $\Sigma_{\Pi}$. Not given in the sources.
*   $\mu$: Hyperparameter for dampening the entire covariance matrix $\Sigma_{\Pi}$. Not given in the sources.
*   $m$: Number of proxy points. Not given in the sources.
*   Learning Rate: For camera parameters, a logarithmic decay from $10^{-3}$ to $10^{-4}$ is used in the reference implementation [S1].

**Edge Cases and Pitfalls:**
*   **Dynamic Reparameterization:** The preconditioner is computed once at initialization. Dynamically updating it during optimization renders the running averages of gradient moments in optimizers like Adam inaccurate [S1].
*   **Lens Distortion:** Parameters causing small changes to dramatically affect projection can make $\Sigma_{\Pi}$ ill-conditioned. Dampening parameters $\lambda$ and $\mu$ are critical here [S1].
*   **Shared Intrinsics:** If multiple cameras share intrinsic parameters, an additional loss term $\mathcal{L}_{shared}$ minimizing the variance of shared parameters should be added [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_preconditioner(camera_params, proxy_points, lambda_diag=1e-6, mu_identity=1e-6):
    """
    Computes the preconditioner matrix P_inv for a single camera.
    
    Args:
        camera_params: Array of shape (k,), initial camera parameters.
        proxy_points: Array of shape (m, 3), 3D points in world space.
        lambda_diag: Dampening factor for diagonal of covariance.
        mu_identity: Dampening factor for identity matrix.
        
    Returns:
        P_inv: Preconditioner matrix of shape (k, k).
    """
    k = len(camera_params)
    m = len(proxy_points)
    
    # 1. Compute Jacobian J_Pi of shape (2m, k)
    # J_Pi[j, d] = d(Pi(x_j; phi))_d / d(phi)_d
    # Note: In practice, this requires the projection function and its derivative.
    # We assume a function `compute_jacobian` exists that evaluates this numerically or analytically.
    J_Pi = compute_jacobian(camera_params, proxy_points) 
    
    # 2. Compute Covariance Sigma_Pi = J_Pi^T @ J_Pi
    Sigma_Pi = J_Pi.T @ J_Pi
    
    # 3. Apply dampening
    # Add lambda * diag(Sigma_Pi) to the diagonal
    Dampened_Sigma = Sigma_Pi.copy()
    Dampened_Sigma += lambda_diag * np.diag(np.diag(Sigma_Pi))
    
    # Add mu * I
    Dampened_Sigma += mu_identity * np.eye(k)
    
    # 4. Compute Inverse Matrix Square Root
    # P_inv = (Dampened_Sigma)^(-1/2)
    # This is done via eigenvalue decomposition: A = V @ D @ V^T => A^(-1/2) = V @ D^(-1/2) @ V^T
    eigenvalues, eigenvectors = np.linalg.eigh(Dampened_Sigma)
    
    # Avoid division by zero
    eigenvalues = np.maximum(eigenvalues, 1e-10)
    
    # Inverse square root of eigenvalues
    inv_sqrt_eigenvalues = 1.0 / np.sqrt(eigenvalues)
    
    # Reconstruct matrix
    P_inv = eigenvectors @ np.diag(inv_sqrt_eigenvalues) @ eigenvectors.T
    
    return P_inv

def optimize_cameras(cameras_init, proxy_points_sets, num_steps, lr_init=1e-3, lr_end=1e-4):
    """
    Optimizes camera parameters using Camera Preconditioning.
    
    Args:
        cameras_init: List of initial camera parameter arrays, each shape (k,).
        proxy_points_sets: List of proxy point arrays, each shape (m, 3).
        num_steps: Number of optimization steps.
        lr_init: Initial learning rate.
        lr_end: Final learning rate.
        
    Returns:
        optimized_latent_params: List of optimized latent parameter arrays.
    """
    k = len(cameras_init[0])
    n_cameras = len(cameras_init)
    
    # Precompute preconditioners
    P_inv_list = []
    for i in range(n_cameras):
        P_inv = compute_preconditioner(cameras_init[i], proxy_points_sets[i])
        P_inv_list.append(P_inv)
        
    # Initialize latent parameters: phi_tilde = P @ phi
    # P = (P_inv)^-1
    latent_params = []
    for i in range(n_cameras):
        P = np.linalg.inv(P_inv_list[i])
        phi_tilde = P @ cameras_init[i]
        latent_params.append(phi_tilde)
        
    # Adam optimizer state (simplified)
    m_tilde = [np.zeros_like(lp) for lp in latent_params]
    v_tilde = [np.zeros_like(lp) for lp in latent_params]
    beta1, beta2, eps = 0.9, 0.999, 1e-8
    
    for step in range(num_steps):
        # Logarithmic learning rate decay
        lr = lr_init * (lr_end / lr_init) ** (step / num_steps)
        
        for i in range(n_cameras):
            # 1. Compute actual camera parameters
            phi_actual = P_inv_list[i] @ latent_params[i]
            
            # 2. Render and compute loss
            # loss = render_loss(phi_actual, ...)
            # gradients are computed w.r.t phi_actual
            grad_phi_actual = compute_gradients(phi_actual) # Placeholder for actual gradient computation
            
            # 3. Chain rule: d_loss/d_phi_tilde = d_loss/d_phi_actual * d_phi_actual/d_phi_tilde
            # phi_actual = P_inv @ phi_tilde => d_phi_actual/d_phi_tilde = P_inv
            grad_phi_tilde = P_inv_list[i].T @ grad_phi_actual
            
            # 4. Update latent parameters using Adam
            # m_t = beta1 * m_{t-1} + (1 - beta1) * grad
            # v_t = beta2 * v_{t-1} + (1 - beta2) * grad^2
            # phi_t = phi_{t-1} - lr * m_t_hat / (sqrt(v_t_hat) + eps)
            
            m_tilde[i] = beta1 * m_tilde[i] + (1 - beta1) * grad_phi_tilde
            v_tilde[i] = beta2 * v_tilde[i] + (1 - beta2) * grad_phi_tilde**2
            
            m_hat = m_tilde[i] / (1 - beta1**(step + 1))
            v_hat = v_tilde[i] / (1 - beta2**(step + 1))
            
            latent_params[i] -= lr * m_hat / (np.sqrt(v_hat) + eps)
            
    return latent_params
```

## Operating characteristics
*   **Time Complexity:** The preconditioner computation involves Jacobian evaluation ($O(m \cdot k \cdot \text{proj\_cost})$), covariance calculation ($O(m \cdot k^2)$), and matrix square root inversion ($O(k^3)$). Since $k$ is small (typically 7-10) and $m$ is moderate, this is negligible compared to NeRF training. The preconditioner is computed once.
*   **Memory Complexity:** Stores $P_i^{-1}$ for each camera ($O(N \cdot k^2)$). This is minimal.
*   **Performance:** Improves convergence and accuracy in camera-optimizing NeRFs, particularly for challenging parameterizations like SE3 with focal length [S1].
*   **Strengths:** Handles mixed units and correlations effectively. Simple to implement on top of existing NeRF frameworks.
*   **Limitations:** Preconditioner is static; may become suboptimal if camera estimates deviate significantly from initialization. Not suitable for dynamic scene updates without recomputation (which breaks Adam's momentum) [S1].
*   **When to use:** When optimizing camera poses and intrinsics jointly with NeRF parameters, especially when parameters have different scales or units.
*   **When not to use:** When camera poses are fixed and known accurately, or when dynamic updates to the preconditioner are required for non-stationary optimization landscapes [S1].

## Background
Camera Preconditioning was proposed by Kevin Park et al. in 2023 [S1]. It was implemented on top of ZipNeRF [Barron et al. 2023] and uses a coarse-to-fine strategy. The method draws inspiration from numerical methods for ill-conditioned problems, specifically Zero Component Analysis (ZCA) whitening [Kessy et al. 2018]. It addresses issues observed in naive camera parameterizations where perturbations affect projection significantly more along some axes than others, and where parameters are correlated [S1].

## Sources
[S1] CamP: Camera Preconditioning for Neural Radiance Fields, https://doi.org/10.1145/3618321, fetched 2026-10-30
