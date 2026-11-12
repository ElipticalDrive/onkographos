# Camera Preconditioning

Also called: Camera Preconditioning for Neural Radiance Fields

## Overview
Camera Preconditioning (CamP) is a technique to normalize and decorrelate camera parameters during Neural Radiance Field (NeRF) optimization by transforming the optimization problem from direct camera parameters $\boldsymbol{\phi}_i$ to latent preconditioned parameters $\tilde{\boldsymbol{\phi}}_i$, using a preconditioner matrix $P_i^{-1}$ derived from the covariance of projection derivatives [S1]. It addresses issues where parameters are correlated or have different units (e.g., translation in world units vs. focal length in pixels), which causes ill-conditioning in naive optimization [S1]. Unlike per-parameter scaling or adaptive optimizers like Adam, CamP decouples highly correlated coefficients by using the full covariance matrix, ensuring that parameter updates are scaled according to their sensitivity to the rendered image [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Initial camera parameters $\boldsymbol{\phi}_i^0$ for each camera $i$; a set of randomly sampled 3D points within each camera's frustum; hyperparameters $\lambda$ and $\mu$.
*   **Outputs:** Preconditioned camera parameters $\boldsymbol{\phi}_i$ used in the NeRF forward pass; latent parameters $\tilde{\boldsymbol{\phi}}_i$ optimized by the gradient descent algorithm.

**Data Structures:**
*   **Camera Parameters ($\boldsymbol{\phi}_i$):** A vector containing the specific parameterization for camera $i$. Common parameterizations include:
    *   **SE3:** 6D screw-axis representation (rotation $\mathbf{r}$, translation $\mathbf{v}$) [S1].
    *   **SE3+Focal+Intrinsics:** SE3 augmented with focal length ($f$), principal point ($u_0, v_0$), and radial distortion ($k_1, k_2$). Focal length residual is multiplicative: $f' = f \exp(\Delta f)$ [S1].
    *   **FocalPose+Intrinsics:** Joint pose and focal length parameterization, modified as a residual with respect to initial estimates, plus principal point and radial distortion [S1].
    *   **SCNeRF:** 6D rotation, residual 3D translation ($t_x, t_y, t_z$), focal lengths ($f_u, f_v$), principal point ($u_0, v_0$), and radial distortion ($k_1, k_2$) [S1].
*   **Preconditioner ($P_i^{-1}$):** A $k \times k$ matrix where $k$ is the dimension of the camera parameter vector. Computed as $P_i^{-1} = (\Sigma_{\Pi} + \lambda \text{diag}(\Sigma_{\Pi}) + \mu I)^{-1/2}$ [S1].
*   **Covariance Matrix ($\Sigma_{\Pi}$):** A $k \times k$ matrix representing the covariance of the derivatives of projected points with respect to camera parameters [S1].

**Numbered Steps:**
1.  **Point Sampling:** For each camera $i$, sample $m$ points within its view frustum.
    *   Sample pixel positions $\mathbf{p} = (u, v)$ uniformly.
    *   Sample normalized ray distances $d \in [0, 1]$ uniformly between normalized near and far planes $d \sim U(n, f)$ [S1].
    *   Unnormalize sampled depths using the power ladder transform of Zip-NeRF, with a curve identical to its ray sampling distribution [S1].
2.  **Compute Jacobian:** Compute the Jacobian matrix $J_{\Pi} \in \mathbb{R}^{2m \times k}$ of the projection function $\Pi(\mathbf{x}; \boldsymbol{\phi})$ with respect to the camera parameters $\boldsymbol{\phi}$ at the current parameter estimate $\boldsymbol{\phi}_0$. The rows correspond to the 2D coordinates of the $m$ projected points [S1].
3.  **Compute Covariance:** Calculate the covariance matrix $\Sigma_{\Pi} = J_{\Pi}^\top J_{\Pi}$ [S1].
4.  **Compute Preconditioner:** Apply dampening to improve conditioning:
    $$ P_i^{-1} = (\Sigma_{\Pi} + \lambda \text{diag}(\Sigma_{\Pi}) + \mu I)^{-1/2} $$
    where $\lambda$ and $\mu$ are dampening hyperparameters, and $I$ is the identity matrix [S1].
5.  **Reformulate Optimization:** Define latent parameters $\tilde{\boldsymbol{\phi}}_i$ such that the actual camera parameters used in the NeRF are $\boldsymbol{\phi}_i = P_i^{-1} \tilde{\boldsymbol{\phi}}_i$ [S1]. This transforms the covariance of the projected points with respect to $\tilde{\boldsymbol{\phi}}_i$ to the identity matrix $I_k$ (ZCA whitening) [S1].
6.  **Optimization Loop:**
    *   For each iteration, compute the loss $\mathcal{L}$ using $\boldsymbol{\phi}_i = P_i^{-1} \tilde{\boldsymbol{\phi}}_i$.
    *   Update $\tilde{\boldsymbol{\phi}}_i$ using an optimizer (e.g., Adam) [S1].
    *   **Shared Intrinsics:** If intrinsic parameters are shared across cameras, add a variance loss term $\mathcal{L}_{\text{shared}}$ to minimize the variance of these shared parameters [S1].
7.  **Static Preconditioner:** The preconditioner $P_i^{-1}$ is computed once at initialization and kept static during optimization to avoid invalidating the running averages of gradient moments in optimizers like Adam [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_preconditioner(camera_params_0, points_3d, hyperparams):
    """
    Computes the preconditioner matrix P_inv for a single camera.
    
    Args:
        camera_params_0: Initial camera parameters vector phi_0 (size k).
        points_3d: Array of 3D points (size m x 3) within the camera frustum.
        hyperparams: Dictionary containing 'lambda' and 'mu'.
        
    Returns:
        P_inv: The preconditioner matrix (size k x k).
    """
    k = len(camera_params_0)
    m = len(points_3d)
    
    # 1. Project points to compute Jacobian
    # J_Pi is (2*m) x k
    J_Pi = compute_jacobian_projection(camera_params_0, points_3d)
    
    # 2. Compute Covariance Sigma_Pi = J^T J
    Sigma_Pi = J_Pi.T @ J_Pi
    
    # 3. Apply dampening
    lambda_val = hyperparams['lambda']
    mu_val = hyperparams['mu']
    
    # Add lambda * diag(Sigma_Pi) and mu * I
    Damped_Sigma = Sigma_Pi + lambda_val * np.diag(np.diag(Sigma_Pi)) + mu_val * np.eye(k)
    
    # 4. Compute Inverse Matrix Square Root
    # P_inv = (Damped_Sigma)^(-1/2)
    # This can be computed via eigendecomposition: 
    # Damped_Sigma = V @ D @ V.T => P_inv = V @ D^(-1/2) @ V.T
    eigenvalues, eigenvectors = np.linalg.eigh(Damped_Sigma)
    # Avoid division by zero or negative eigenvalues due to numerical noise
    eigenvalues = np.maximum(eigenvalues, 1e-10)
    D_inv_sqrt = np.diag(1.0 / np.sqrt(eigenvalues))
    
    P_inv = eigenvectors @ D_inv_sqrt @ eigenvectors.T
    
    return P_inv

def optimize_cameras(cameras_data, points_3d, hyperparams, optimizer_config):
    """
    Main optimization loop for Camera Preconditioning.
    
    Args:
        cameras_data: List of dictionaries, each containing 'params_0' (initial params)
                      and 'P_inv' (precomputed preconditioner).
        points_3d: Dictionary mapping camera index to 3D points.
        hyperparams: Contains 'lambda', 'mu', learning rates, etc.
        optimizer_config: Configuration for the optimizer (e.g., Adam).
        
    Returns:
        optimized_latent_params: List of optimized latent parameter vectors.
    """
    # Initialize latent parameters tilde_phi = 0 (or small random noise)
    # Since phi = P_inv @ tilde_phi, and we want phi to start at phi_0,
    # we must set tilde_phi such that P_inv @ tilde_phi = phi_0.
    # Thus, tilde_phi = P_inv^{-1} @ phi_0.
    # However, the paper states we optimize tilde_phi and map to phi_i for loss.
    # Usually, optimization starts from zero latent parameters if the preconditioner
    # is designed to map the "standard normal" space to the parameter space.
    # But here, the preconditioner is derived from the sensitivity at phi_0.
    # The paper says: "reformulate the naive optimization problem ... into one in which 
    # we instead optimize over a set of latent preconditioned parameters {tilde_phi_i}".
    # And "each time we compute the loss function we multiply each tilde_phi_i by 
    # its corresponding P_i^{-1} matrix to yield camera parameters phi_i".
    # This implies phi_i = P_i^{-1} @ tilde_phi_i.
    # To start at phi_0, we initialize tilde_phi_i = P_i @ phi_0.
    
    optimized_latent_params = []
    
    for cam_data in cameras_data:
        phi_0 = cam_data['params_0']
        P_inv = cam_data['P_inv']
        
        # Compute initial latent parameters
        # P_inv is (Sigma + dampening)^(-1/2). 
        # We need P such that P_inv = P^{-1}. So P = (P_inv)^{-1}.
        P = np.linalg.inv(P_inv)
        tilde_phi_0 = P @ phi_0
        
        # Initialize optimizer state for tilde_phi
        optimizer = AdamOptimizer(tilde_phi_0, **optimizer_config)
        
        # Optimization loop
        for iteration in range(num_iterations):
            # 1. Compute actual camera parameters
            phi_current = P_inv @ optimizer.get_params()
            
            # 2. Compute Loss
            # Render image using phi_current and NeRF
            # Compute loss against ground truth
            loss = compute_loss(phi_current, points_3d[cam_data['id']])
            
            # 3. Backpropagate to get gradient w.r.t tilde_phi
            # Note: The gradient dL/d(tilde_phi) is computed via backprop through 
            # the mapping phi = P_inv @ tilde_phi. Since P_inv is constant,
            # dL/d(tilde_phi) = P_inv^T @ dL/d(phi).
            # However, in automatic differentiation frameworks, this is handled automatically
            # if phi_current is computed as a tensor operation.
            grad_tilde_phi = compute_gradient(loss, optimizer.get_params())
            
            # 4. Update latent parameters
            optimizer.step(grad_tilde_phi)
            
            # 5. Shared Intrinsic Loss (if applicable)
            if cam_data['has_shared_intrinsics']:
                shared_loss = compute_variance_loss(shared_intrinsics)
                optimizer.step(shared_loss)
                
        optimized_latent_params.append(optimizer.get_params())
        
    return optimized_latent_params

def compute_jacobian_projection(params, points_3d):
    """
    Computes the Jacobian of the projection function with respect to camera parameters.
    This is a placeholder for the specific projection function implementation.
    """
    # Implementation depends on the camera model (e.g., pinhole with distortion)
    # Returns a (2*m) x k matrix
    pass
```

**Parameters:**
*   $\lambda$: Dampening parameter for the diagonal of the covariance matrix. Typical value: $10^{-1}$ [S1].
*   $\mu$: Dampening parameter for the identity matrix added to the covariance matrix. Typical value: $10^{-8}$ [S1].
*   Learning Rate for Camera Parameters: Logarithmically decays from $10^{-3}$ to $10^{-4}$ over training [S1].
*   Warmup Multiplier: Cosine-decayed from $10^{-8}$ to $1$ for the first 2500 iterations [S1].
*   Variance Loss Weights (Shared Intrinsics): Focal length $10^{-1}$, Principal Point $10^{-2}$, Radial Distortion $10^{-2}$ [S1].

**Edge Cases and Pitfalls:**
*   **Dynamic Re-parameterization:** Avoid updating the preconditioner during optimization. Dynamic updates render the running averages of gradient moments used in optimizers like Adam inaccurate [S1].
*   **Lens Distillation:** Lens distortion parameters can cause very small changes to dramatically affect point projection, making the covariance matrix ill-conditioned. Dampening parameters $\lambda$ and $\mu$ are especially important in this case [S1].
*   **Local Minima:** Preconditioning does not always prevent falling into local minima (e.g., white floaters in synthetic NeRF dataset due to bad camera poses) [S1].
*   **Procrustes Alignment:** The Procrustes alignment method breaks when focal length changes significantly, so it is omitted in evaluation [S1].

## Operating characteristics
*   **Time Complexity:** The preconditioner computation involves eigendecomposition of a $k \times k$ matrix, where $k$ is the number of camera parameters (typically small, e.g., 10-20). This is negligible compared to the NeRF training time. Training time is approximately 2 hours on 8 NVIDIA V100 GPUs for 25,000 steps [S1].
*   **Memory Complexity:** Requires storing the preconditioner matrix $P_i^{-1}$ for each camera ($O(k^2)$ per camera) and the sampled points. This is minimal compared to the NeRF grid storage.
*   **Measured Performance:** On the perturbed 360 dataset, preconditioning improves PSNR between 4.3 and 6.8 dB for all camera parameterizations [S1]. On the perturbed synthetic dataset, FocalPose+Intrinsics with CamP achieves PSNR 28.29, SSIM 0.930, LPIPS 0.064, and camera metrics $R=0.420^\circ$, $p=0.186$, $f=54.3$ px [S1].
*   **Strengths:** Decorrelates parameter effects and normalizes their scale, leading to improved convergence and accuracy. Handles mixed units and correlated parameters better than per-parameter scaling or Adam [S1].
*   **Limitations:** Does not guarantee escape from local minima. Performance depends on the quality of the initial camera estimates and the sampled points [S1].

## Background
Camera Preconditioning (CamP) was introduced by K. Park et al. in the paper "CamP: Camera Preconditioning for Neural Radiance Fields" (ACM Trans. Graph., Vol. 42, No. 6, Article 208, December 2023) [S1]. It builds on the observation that naive camera parameterizations in NeRF optimization are ill-conditioned due to differences in parameter magnitude (e.g., world units vs. pixels) and spurious correlations. The method draws inspiration from numerical methods, specifically the Zero Component Analysis (ZCA) whitening transform, to design a per-camera preconditioner [S1]. It is implemented on top of ZipNeRF and used on datasets like 360, Perturbed-360, Perturbed-Synthetic, and ARKit [S1].

## Sources

[S1] CamP: Camera Preconditioning for Neural Radiance Fields, https://doi.org/10.1145/3618321, fetched 2026-10-30
