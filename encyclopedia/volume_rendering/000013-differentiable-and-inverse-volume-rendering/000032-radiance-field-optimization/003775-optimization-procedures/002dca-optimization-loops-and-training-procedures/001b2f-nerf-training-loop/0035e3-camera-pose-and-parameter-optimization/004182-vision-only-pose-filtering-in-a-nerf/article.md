# Vision-Only Pose Filtering in a NeRF

Also called: Optimization-based Pose Filter for NeRF, Recursive SE(3) Optimization

## Overview
This method is an optimization-based filtering approach that estimates the 6 Degrees of Freedom (6DoF) pose and velocities of a robot within a Neural Radiance Field (NeRF) using only onboard RGB camera images [S1, unofficial]. It addresses the limitation of single-image pose estimation methods, which are highly sensitive to initialization and fail in low-gradient regions, by treating pose estimation as a recursive Bayesian estimation problem [S1, unofficial]. Unlike standard joint optimization or single-shot NeRF-based localization, this filter balances a NeRF-based photometric loss with a process loss derived from system dynamics, providing robustness and outputting state covariance for downstream robotics tasks [S1, unofficial].

## How it works
The method operates as a recursive filter at each timestep $t$. It takes a noisy image $I_t$, the previous action $u_t$, and the previous state estimate (mean $\mu_{t-1}$ and covariance $\Sigma_{t-1}$) as inputs, and outputs the updated state mean $\mu_t$ and covariance $\Sigma_t$ [S1, unofficial].

### Data Structures and State Representation
The state $\mu_t$ represents the full dynamic state of the robot, including position, orientation, and velocities. The pose $T_t$ is constructed from the position and rotation elements of $\mu_t$ [S1, unofficial]. The covariance $\Sigma_t$ is a matrix representing the uncertainty of the state estimate [S1, unofficial].

### Algorithm Steps

1.  **State Propagation (Prediction):**
    Using the previous state mean $\mu_{t-1}$, the previous action $u_t$, and a dynamics model $f$, compute the predicted mean $\mu_{t|t-1}$ and the Jacobian of the dynamics $A_{t-1}$ [S1, unofficial].
    $$ \mu_{t|t-1} = f(\mu_{t-1}, u_t) $$
    $$ A_{t-1} = \left. \frac{\partial f(x, u_t)}{\partial x} \right|_{x=\mu_{t-1}} $$
    Propagate the covariance using the process noise covariance $Q_{t-1}$ [S1, unofficial]:
    $$ \Sigma_{t|t-1} = A_{t-1} \Sigma_{t-1} A_{t-1}^T + Q_{t-1} $$

2.  **Pixel Selection:**
    Select a subset of pixels $I$ from the current image $I_t$ using an image feature detector (e.g., ORB) to identify points of interest. This biases sampling toward areas with higher gradient information [S1, unofficial].

3.  **Optimization (Update):**
    Minimize the cost function $J(\mu_t)$ with respect to the state $\mu_t$. The cost function combines a photometric loss and a process loss [S1, unofficial]:
    $$ J(\mu_t) = \| C_I(T_t) - I_t(I) \|_{S_t^{-1}}^2 + \| \mu_{t|t-1} - \mu_t \|_{\Sigma_{t|t-1}^{-1}}^2 $$
    Where:
    *   $C_I(T_t)$ are the predicted pixel colors via the projected NeRF scene using the current pose estimate $T_t$ derived from $\mu_t$.
    *   $I_t(I)$ are the observed image pixels at the selected subset $I$.
    *   $S_t$ is the measurement noise covariance.
    *   $\| x \|_M^2 = x^T M x$ denotes the weighted $L_2$ norm.
    *   The first term is the photometric loss.
    *   The second term is the process loss, weighting the deviation from the predicted state $\mu_{t|t-1}$ [S1, unofficial].

    **Optimization Details:**
    *   **Manifold Optimization:** The optimization is performed on the SE(3) manifold. Unlike methods that optimize w.r.t. a reference pose and project once, this method projects back onto the manifold after *every* gradient step (Recursive SE(3)) [S1, unofficial]. This is necessary because the multiplication of skew-symmetric matrices in the exponential map does not commute:
        $$ \exp(\delta_1) \exp(\delta_2) \cdots T \neq \exp(\delta_1 + \delta_2 + \cdots) T $$
        where $T$ is the reference homogeneous transformation matrix [S1, unofficial]. This recursive projection converges quicker and more smoothly on the noisy photometric loss landscape [S1, unofficial].
    *   **Implementation:** The manifold optimization is implemented using the LieTorch library [S1, unofficial].
    *   **Outlier Rejection:** Outlier rejection is performed on the per-pixel loss to reduce variance [S1, unofficial].

4.  **Covariance Update:**
    Compute the posterior covariance $\Sigma_t$ using the Hessian of the loss function at the optimized mean $\mu_t$ [S1, unofficial]:
    $$ \Sigma_t = \left( \left. \frac{\partial^2 J(x)}{\partial x^2} \right|_{x=\mu_t} \right)^{-1} $$

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch
from lietorch import SE3  # Assuming LieTorch-like API for SE3 manifold operations

def vision_only_pose_filter(
    nerf_model, 
    image_t, 
    action_t, 
    prev_mean, 
    prev_cov, 
    dynamics_model, 
    process_noise_cov, 
    measurement_noise_cov,
    feature_detector
):
    """
    Performs one step of the Vision-Only Pose Filter.
    
    Args:
        nerf_model: The pre-trained NeRF model.
        image_t: Current RGB image tensor.
        action_t: Control input/action vector u_t.
        prev_mean: Previous state mean mu_{t-1}.
        prev_cov: Previous state covariance Sigma_{t-1}.
        dynamics_model: Function f(x, u) predicting next state.
        process_noise_cov: Process noise covariance Q_{t-1}.
        measurement_noise_cov: Measurement noise covariance S_t.
        feature_detector: Function to select pixel indices (e.g., ORB).
        
    Returns:
        new_mean: Updated state mean mu_t.
        new_cov: Updated state covariance Sigma_t.
    """
    
    # 1. State Propagation
    # Predict mean
    pred_mean = dynamics_model(prev_mean, action_t)
    
    # Compute Jacobian of dynamics A_{t-1}
    # Note: In practice, this requires automatic differentiation or finite differences
    # around prev_mean.
    A_prev = compute_jacobian(dynamics_model, prev_mean, action_t)
    
    # Propagate covariance
    pred_cov = A_prev @ prev_cov @ A_prev.T + process_noise_cov
    
    # 2. Pixel Selection
    # Select subset of pixels I using feature detector
    pixel_indices = feature_detector(image_t)
    
    # 3. Optimization (Update)
    # Initialize current state estimate for optimization
    # Start from the predicted mean
    current_state = pred_mean.clone()
    
    # Recursive SE(3) Optimization
    # Loop until convergence
    while not converged(current_state, pred_mean):
        # Construct pose T_t from current_state mean
        # Assuming current_state contains position and rotation components
        T_t = construct_SE3_pose(current_state)
        
        # Render predicted pixels using NeRF
        # C_I(T_t) renders the image for the selected pixel_indices
        predicted_pixels = nerf_model.render(T_t, pixel_indices)
        
        # Get observed pixels
        observed_pixels = image_t[pixel_indices]
        
        # Compute Photometric Loss
        # Weighted L2 norm: || x ||_M^2 = x^T M x
        # S_t_inv is the inverse of measurement_noise_cov
        S_t_inv = torch.inverse(measurement_noise_cov)
        photometric_error = predicted_pixels - observed_pixels
        photometric_loss = photometric_error.T @ S_t_inv @ photometric_error
        
        # Compute Process Loss
        # || mu_{t|t-1} - mu_t ||_{Sigma_{t|t-1}^{-1}}^2
        Sigma_pred_inv = torch.inverse(pred_cov)
        state_error = pred_mean - current_state
        process_loss = state_error.T @ Sigma_pred_inv @ state_error
        
        # Total Cost
        cost = photometric_loss + process_loss
        
        # Compute gradients of cost w.r.t. current_state
        # Note: This requires backpropagation through the NeRF and ray generation
        gradients = compute_gradients(cost, current_state)
        
        # Gradient Step
        # Update state in tangent space
        delta = -learning_rate * gradients
        
        # Project back onto SE(3) manifold after every step
        # This is the key difference from single-projection methods
        current_state = project_SE3(current_state, delta)
        
        # Outlier Rejection (Optional/Implementation specific)
        # Filter pixels with high per-pixel loss before next iteration
        # This reduces variance in the gradient computation
        if outlier_rejection_enabled:
            pixel_indices = filter_outliers(pixel_indices, photometric_error)

    # 4. Covariance Update
    # Compute Hessian of J(x) at the optimized mean
    Hessian = compute_hessian(cost, current_state)
    
    # Posterior Covariance
    new_cov = torch.inverse(Hessian)
    
    return current_state, new_cov
```

### Parameters
*   **Feature Detector:** ORB is used to select pixels for photometric evaluation [S1, unofficial].
*   **Measurement Noise Covariance ($S_t$):** Weighting for the photometric loss [S1, unofficial].
*   **Process Noise Covariance ($Q_{t-1}$):** Weighting for the dynamics model uncertainty [S1, unofficial].
*   **Learning Rate:** Step size for the gradient descent on the manifold [G].

### Edge Cases and Pitfalls
*   **SE(3) Manifold Optimization:** Optimizing with respect to a reference pose and projecting once is mathematically distinct from projecting after every step due to the non-commutativity of skew-symmetric matrix multiplication in the exponential map. The recursive projection method is required for smooth convergence on the noisy photometric loss landscape [S1, unofficial].
*   **Low Gradient Regions:** The process loss term is critical for robustness when the robot travels through regions with low photometric gradient information, allowing the filter to rely more on the dynamics model [S1, unofficial].
*   **Initialization:** While more robust than single-shot methods, the filter still relies on the previous state estimate. Poor initial estimates or large disturbances may require careful tuning of the process noise covariance [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The method involves rendering via NeRF (which is computationally expensive) and optimization steps. The complexity depends on the number of pixels sampled (via feature detector) and the number of optimization iterations. The use of feature detectors reduces the number of pixels evaluated compared to full-image rendering [S1, unofficial].
*   **Memory Complexity:** Requires storing the NeRF model, the state mean and covariance, and intermediate rendering buffers. The covariance update requires computing and inverting the Hessian, which scales with the square of the state dimension [S1, unofficial].
*   **Strengths:** Provides robust pose estimation in low-gradient regions by incorporating dynamics. Outputs state covariance, which is useful for downstream robotics algorithms like collision avoidance. Integrates seamlessly into an online replanning loop [S1, unofficial].
*   **Limitations:** Computationally intensive due to NeRF rendering and optimization. Requires a pre-trained NeRF of the environment. Performance depends on the quality of the dynamics model and noise covariances [S1, unofficial].
*   **When to Use:** Online robot navigation where only monocular RGB images are available, and a NeRF representation of the environment is already trained. Particularly useful when traditional SLAM fails due to textureless regions or when covariance estimates are needed for safety-critical planning [S1, unofficial].
*   **When Not to Use:** When real-time performance is critical and NeRF rendering is too slow, or when a ground truth pose or other sensor modalities (LiDAR, IMU) are available and preferred [S1, unofficial].

## Background
This method was introduced in the paper "Vision-Only Robot Navigation in a Neural Radiance World" [S1, unofficial]. It builds upon previous work on single-image camera pose estimation using pre-trained NeRFs [19] (cited in [S1, unofficial]), extending it from a single-shot maximum likelihood estimation to a recursive Bayesian estimation framework. The work demonstrates the method in simulation environments including a jungle gym, a church, and Stonehenge, using both quadrotor and omnidirectional ground robot models [S1, unofficial]. It is implemented using the LieTorch library [S1, unofficial].

## Sources
[S1] Vision-Only Robot Navigation in a Neural Radiance World, https://doi.org/10.1109/lra.2022.3150497, fetched 2026-10-26 (unofficial)
