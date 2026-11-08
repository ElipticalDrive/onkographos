# Vision-Only Pose Filtering in a NeRF

Also called: Optimization-based Pose Filter for NeRF, Recursive SE(3) Optimization

## Overview
This method is an optimization-based state estimation filter that estimates the 6-DoF pose and velocities of a robot within a Neural Radiance Field (NeRF) using only onboard monocular RGB images [S1]. It addresses the limitation of single-shot pose estimators by treating the problem as recursive Bayesian estimation, balancing a NeRF-based photometric loss with a dynamics prediction loss to provide robustness in low-gradient regions and output state covariance [S1]. It is distinguished from sibling methods like [NeRF with Unknown Camera Parameters](7405-nerf-with-unknown-camera-parameters.md) by its explicit incorporation of system dynamics and uncertainty propagation, and from [NoPe-NeRF](8705-nope-nerf.md) by its focus on online robot navigation rather than static scene reconstruction with unknown intrinsics.

## How it works
The method operates as a recursive filter at each timestep $t$, updating the robot's state belief based on a new image $I_t$, the previous action $u_t$, and the prior state estimate.

**Inputs and Outputs:**
*   **Inputs:** Current image $I_t$, previous action $u_t$, prior state mean $\mu_{t-1}$, prior state covariance $\Sigma_{t-1}$, pre-trained NeRF model, and a subset of selected pixels $I$ (identified via feature detection).
*   **Outputs:** Updated state mean $\mu_t$, updated state covariance $\Sigma_t$, and the robot pose $T_t$ derived from $\mu_t$.

**Data Structures:**
*   **State Vector $\mu_t$:** Contains the robot's position, orientation (represented in SE(3)), and velocities. The exact dimensionality depends on the specific dynamics model $f$, but typically includes 6-DoF pose and 6-DoF velocity [S1].
*   **Covariance Matrix $\Sigma_t$:** A symmetric positive-definite matrix representing the uncertainty of the state estimate [S1].
*   **Pixel Subset $I$:** A set of 2D image coordinates $(u, v)$ selected using an image feature detector (e.g., ORB) to bias sampling toward areas of high gradient information [S1].

**Numbered Steps:**

1.  **State Propagation (Prediction):**
    Propagate the state mean and covariance forward using the system dynamics model $f$ and the applied action $u_t$ [S1].
    $$ \mu_{t|t-1} = f(\mu_{t-1}, u_t) $$
    $$ A_{t-1} = \left. \frac{\partial f(x, u_t)}{\partial x} \right|_{x=\mu_{t-1}} $$
    $$ \Sigma_{t|t-1} = A_{t-1} \Sigma_{t-1} A_{t-1}^T + Q_{t-1} $$
    where $Q_{t-1}$ is the process noise covariance [S1].

2.  **Pixel Selection:**
    Select a subset of pixels $I$ from the current image $I_t$ using an image feature detector (e.g., ORB) to identify points of interest [S1].

3.  **Optimization (Update):**
    Minimize the cost function $J(\mu_t)$ with respect to the state $\mu_t$ to find the posterior mean $\mu_t$. The cost function combines a photometric loss and a process loss [S1]:
    $$ J(\mu_t) = \| C_I(T_t) - I_t(I) \|_{S_t^{-1}}^2 + \| \mu_{t|t-1} - \mu_t \|_{\Sigma_{t|t-1}^{-1}}^2 $$
    where:
    *   $C_I(T_t)$ represents the predicted pixel values for the selected subset $I$ given the pose $T_t$ (derived from $\mu_t$) and the NeRF.
    *   $I_t(I)$ are the observed pixel values from the image $I_t$ at locations $I$.
    *   $S_t$ is the measurement noise covariance.
    *   The notation $\| x \|_M^2 = x^T M x$ denotes the weighted $L_2$ norm [S1].

4.  **Outlier Rejection:**
    Perform outlier rejection on the per-pixel loss to reduce variance before or during the optimization step [S1].

5.  **Covariance Update:**
    Compute the posterior covariance $\Sigma_t$ using the Hessian of the loss function at the optimized mean [S1]:
    $$ \Sigma_t = \left( \left. \frac{\partial^2 J(x)}{\partial x^2} \right|_{x=\mu_t} \right)^{-1} $$

6.  **Manifold Projection:**
    During the optimization of $\mu_t$, project the pose back onto the SE(3) manifold after every gradient step. This "recursive SE(3)" approach is preferred over optimizing with respect to a reference pose and projecting once, as the multiplication of skew-symmetric matrices in the exponential map does not commute, and recursive projection converges quicker and more smoothly on the noisy photometric loss landscape [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch
from lietorch import SE3 # Assuming LieTorch library for SE3 operations [S1]

def vision_only_pose_filter(
    prev_mu, 
    prev_Sigma, 
    action_u, 
    current_image, 
    nerf_model, 
    pixel_subset_I, 
    Q, 
    S, 
    optimizer_params
):
    """
    Performs one step of the Vision-Only Pose Filter.
    
    Args:
        prev_mu: Previous state mean (numpy/tensor)
        prev_Sigma: Previous state covariance (numpy/tensor)
        action_u: Control action applied at previous step
        current_image: RGB image tensor from current timestep
        nerf_model: Pre-trained NeRF model for rendering
        pixel_subset_I: Indices of pixels to evaluate (e.g., from ORB)
        Q: Process noise covariance
        S: Measurement noise covariance
        optimizer_params: Configuration for the optimizer (e.g., learning rate)
        
    Returns:
        mu_t: Updated state mean
        Sigma_t: Updated state covariance
    """
    
    # Step 1: State Propagation
    # f is the dynamics model function
    mu_pred = dynamics_model(prev_mu, action_u)
    
    # Compute Jacobian A of dynamics model w.r.t state x at mu_pred
    A = compute_jacobian_dynamics(prev_mu, action_u)
    
    # Propagate covariance
    Sigma_pred = A @ prev_Sigma @ A.T + Q
    
    # Step 2: Pixel Selection (Assumed pre-computed or done externally)
    # pixel_subset_I is provided as input
    
    # Step 3 & 6: Optimization with Recursive SE(3) Projection
    # Initialize optimization variable with predicted state
    # Note: mu_t contains pose and velocity. Pose is in SE(3).
    # We optimize in the tangent space or use a manifold optimizer.
    # Here we assume a generic optimizer that handles SE(3) via LieTorch.
    
    # Create a differentiable variable for the state
    # The exact representation depends on the LieTorch implementation details [S1]
    mu_opt = torch.tensor(mu_pred, requires_grad=True, dtype=torch.float32)
    
    # Define the cost function
    def cost_function(mu):
        # Extract pose T_t from state mu
        T_t = extract_pose_from_state(mu)
        
        # Render predicted pixels using NeRF
        # C_I(T_t) renders the image for subset I given pose T_t
        predicted_pixels = nerf_model.render(T_t, pixel_subset_I)
        
        # Get observed pixels
        observed_pixels = current_image[pixel_subset_I]
        
        # Photometric Loss: || C_I(T_t) - I_t(I) ||_{S^{-1}}^2
        # Weighted L2 norm: x^T S^{-1} x
        diff_photo = predicted_pixels - observed_pixels
        # Reshape for matrix multiplication if necessary
        diff_photo_flat = diff_photo.view(-1, 1)
        S_inv = torch.inverse(S)
        photometric_loss = diff_photo_flat.T @ S_inv @ diff_photo_flat
        
        # Process Loss: || mu_pred - mu ||_{Sigma_pred^{-1}}^2
        diff_proc = mu_opt - mu_pred
        diff_proc_flat = diff_proc.view(-1, 1)
        Sigma_pred_inv = torch.inverse(Sigma_pred)
        process_loss = diff_proc_flat.T @ Sigma_pred_inv @ diff_proc_flat
        
        return photometric_loss + process_loss

    # Optimization Loop
    optimizer = torch.optim.Adam([mu_opt], **optimizer_params)
    
    for step in range(max_iterations):
        optimizer.zero_grad()
        loss = cost_function(mu_opt)
        
        # Outlier rejection can be applied here by masking pixels with high residual
        # This is implementation-specific but noted as a step [S1]
        
        loss.backward()
        optimizer.step()
        
        # Recursive SE(3) Projection: Project pose back onto manifold after every step
        # This ensures the pose component of mu_opt remains a valid SE(3) transformation
        # LieTorch handles this projection internally or via a specific method
        mu_opt = project_se3(mu_opt) 
        
    # Step 4: Outlier Rejection (Post-optimization refinement or iterative)
    # The source mentions outlier rejection on per-pixel loss [S1].
    # This can be implemented by re-evaluating residuals and re-running optimization
    # with masked pixels, or using a robust loss function.
    # For simplicity, we assume the optimizer handles robustness or this is a pre-step.
    # If iterative:
    # residuals = compute_residuals(mu_opt, current_image, pixel_subset_I)
    # mask = apply_outlier_rejection(residuals)
    # pixel_subset_I = pixel_subset_I[mask]
    # Re-run optimization with masked pixels...

    # Step 5: Covariance Update
    # Compute Hessian of J(x) at mu_t
    # Since we used an optimizer, we can approximate the Hessian or compute it explicitly
    # The source specifies: Sigma_t = (d^2 J(x) / dx^2 |_{x=mu_t})^{-1} [S1]
    
    # Compute Hessian using autograd or finite differences
    # Note: Computing full Hessian can be expensive. 
    # In practice, the inverse Hessian is often approximated by the optimizer's state (e.g., Adam inverse variance)
    # or computed via Gauss-Newton approximation if the loss is quadratic.
    # Here we follow the source's exact formula.
    
    # To compute Hessian in PyTorch:
    # J = cost_function(mu_opt)
    # H = torch.autograd.functional.hessian(cost_function, mu_opt)
    # Sigma_t = torch.inverse(H)
    
    # However, for large state spaces, this is intractable. 
    # The source implies this is the theoretical definition. 
    # We will write the code assuming a feasible Hessian computation or approximation.
    
    # Approximation using the inverse of the optimizer's accumulated gradients or a diagonal approximation
    # is common, but strictly following the source:
    H = compute_hessian(cost_function, mu_opt)
    Sigma_t = torch.inverse(H)
    
    return mu_opt.detach().numpy(), Sigma_t.detach().numpy()

def project_se3(state):
    """
    Projects the pose component of the state back onto the SE(3) manifold.
    Uses LieTorch's exponential map properties [S1].
    """
    # LieTorch handles SE3 objects. 
    # If optimizing in tangent space, we exponentiate.
    # If optimizing in matrix space, we use SVD or polar decomposition.
    # LieTorch provides a .normalize() or similar method for SE3 objects.
    # Assuming state is a LieTorch SE3 object or can be converted.
    if isinstance(state, torch.Tensor):
        # Convert to SE3 object if needed
        se3_obj = SE3(state)
        se3_obj = se3_obj.normalize() # Project to manifold
        return se3_obj.data
    return state
```

**Parameters:**
*   **Feature Detector:** ORB is used to select pixels for photometric evaluation [S1].
*   **Process Noise Covariance $Q_{t-1}$:** Not given in the sources.
*   **Measurement Noise Covariance $S_t$:** Not given in the sources.
*   **Learning Rate:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **SE(3) Manifold Optimization:** Multiplication of skew-symmetric matrices in the exponential map does not commute. Optimizing with respect to a reference pose and projecting once is mathematically distinct from recursive projection. The recursive SE(3) approach is required for quicker and smoother convergence due to the noisy photometric loss landscape [S1].
*   **Low Gradient Regions:** The method is designed to be robust in regions with low photometric gradient information by relying more on the dynamics model (process loss) [S1].
*   **Implementation:** Optimization on the manifold is implemented using the LieTorch library [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. Depends on the number of pixels in subset $I$, the complexity of the NeRF rendering, and the number of optimization iterations.
*   **Memory Complexity:** Not given in the sources. Depends on the state vector size and the NeRF model size.
*   **Measured Performance:** Demonstrated in simulation with a quadrotor navigating through a jungle gym, the inside of a church, and Stonehenge, and with an omnidirectional ground robot navigating through a church [S1].
*   **Strengths:** Provides robust pose estimation in low-gradient regions; outputs state covariance useful for parallel algorithms like collision avoidance; integrates seamlessly with trajectory planning in an online replanning loop [S1].
*   **Limitations:** Requires a pre-trained NeRF model; computationally intensive due to NeRF rendering and optimization at each timestep; sensitive to the accuracy of the dynamics model $f$ and noise covariances $Q, S$ [S1].
*   **When to Use:** Online robot navigation in environments represented by NeRFs where only monocular vision is available and dynamics are known [S1].
*   **When Not to Use:** When high-frequency updates are not required and offline batch optimization is acceptable; when the environment is not well-represented by a NeRF; when dynamics are unknown or highly uncertain [S1].

## Background
The method was introduced by Mikhalkiewicz et al. in 2022 in the paper "Vision-Only Robot Navigation in a Neural Radiance World" [S1]. It builds upon previous work on single-image camera pose estimation using NeRFs [19] (cited in [S1]) but extends it to recursive Bayesian estimation. The method is part of a larger navigation framework that combines trajectory planning (based on differential flatness) and pose filtering for online replanning [S1]. It was demonstrated in high-fidelity simulation environments including a playground, a church, and Stonehenge [S1].

## Sources
[S1] Vision-Only Robot Navigation in a Neural Radiance World, https://doi.org/10.1109/lra.2022.3150497, fetched 2026-10-26
