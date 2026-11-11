# Levenberg-Marquardt Pose Optimization

Also called: LM Pose Estimation, Pose Optimization via Levenberg-Marquardt

## Overview
Levenberg-Marquardt (LM) Pose Optimization is an iterative numerical method that estimates the 6-DoF camera pose by minimizing the squared error between features extracted from a reference image and features rendered from a current pose guess [S1]. It distinguishes itself from gradient-based differentiable rendering approaches by relying on a numerically estimated Jacobian via finite-difference pose perturbations rather than analytical gradients, making it robust to complex lens and illumination dependencies [S1].

## How it works
**Inputs:** Reference image $s_{ref}$, initial pose guess $\hat{p}_0$, Levenberg-Marquardt damping parameter $\lambda$, convergence threshold $\epsilon$, number of pose samples $N_s$.
**Outputs:** Converged pose $\hat{p}_{final}$.

**Data Structures:**
*   Pose vector $\hat{p}$: 6-dimensional vector representing rotation and translation.
*   Feature vector $\tilde{x}$: Extracted features from $s_{ref}$.
*   Jacobian $J_f$: Numerically estimated Jacobian of feature errors with respect to pose parameters.

**Steps:**
1.  **Initialization:** Extract reference features $\tilde{x}$ from $s_{ref}$ [S1].
2.  **Iteration:** While the norm of the pose update $\|\delta p\| > \epsilon$:
    a.  **Render and Match:** Render the scene at the current pose guess $\hat{p}_i$. Extract features $h(s_i, \hat{p}_i)$ and match them with $\tilde{x}$ [S1].
    b.  **Pose Sampling:** Sample $N_s$ pose parameters around the current guess $\hat{p}_i$. The rotation axis for perturbations is sampled uniformly on a spherical surface, biasing samples to a $\delta$-neighborhood of the reference pose [S1].
    c.  **Jacobian Estimation:** Render images at the sampled pose values. Match reference features in each rendered image. Calculate the Jacobian $J_f$ using the perturbation method (Eq. 13 in [S1]), which approximates the derivative of the feature error with respect to the pose [S1].
    d.  **Update:** Calculate the least-squares pose update $\hat{p}_{i+1}$ using the Levenberg-Marquardt update rule (Eq. 16 in [S1]). The update $\delta p$ is applied to $\hat{p}_i$ to get $\hat{p}_{i+1}$ [S1].
3.  **Termination:** Return $\hat{p}_{final}$ when $\|\delta p\| \le \epsilon$ [S1].

**Formulas:**
*   **Jacobian Approximation:** The Jacobian $J_f$ is computed via finite differences. For each sampled pose perturbation $\Delta p_k$, the feature error change $\Delta e_k$ is measured. $J_f$ is derived from the relationship between $\Delta p_k$ and $\Delta e_k$ [S1].
*   **LM Update:** The pose update $\delta p$ is calculated by solving the regularized normal equations:
    $$ (J_f^T J_f + \lambda I) \delta p = -J_f^T e $$
    where $e$ is the feature error vector, $J_f$ is the Jacobian, $\lambda$ is the damping parameter, and $I$ is the identity matrix [S1].

**Parameters:**
*   $\lambda$: LM update parameter. Typical values are not specified in the sources.
*   $\epsilon$: Convergence threshold. Not given in the sources.
*   $N_s$: Number of pose samples. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on numerical stability of the finite difference approximation. Small perturbations may lead to noise in $J_f$ [S1].
*   **Boundaries:** The pose sampling is biased to a $\delta$-neighborhood. If the initial guess is far from the true pose, the local approximation may fail [S1].
*   **Degenerate Input:** Feature matching must be robust. If features cannot be matched, the error vector $e$ is undefined [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lm_pose_optimization(s_ref, p_hat_0, lambda_param, epsilon, N_s):
    # Extract reference features
    x_tilde = extract_features(s_ref)
    
    p_hat = p_hat_0
    delta_p = infinity
    
    while norm(delta_p) > epsilon:
        # Render and match at current pose
        rendered_img = render(scene, p_hat)
        h_current = extract_and_match_features(rendered_img, x_tilde)
        
        # Sample pose perturbations
        pose_samples = sample_pose_around(p_hat, N_s)
        
        # Calculate Jacobian via finite differences
        J_f = calculate_jacobian_numerical(pose_samples, x_tilde, lambda_param)
        
        # Calculate error
        e = h_current - x_tilde
        
        # Solve LM update
        # (J_f^T J_f + lambda * I) * delta_p = -J_f^T * e
        A = transpose(J_f) @ J_f + lambda_param * identity_matrix(6)
        b = -transpose(J_f) @ e
        delta_p = solve_linear_system(A, b)
        
        # Update pose
        p_hat = p_hat + delta_p
        
    return p_hat
```

## Subtopics

### [Derivative-free Pose Estimation via Feature Matching](11310-derivative-free-pose-estimation-via-feature-matching.md)
This method estimates the 6-DoF pose of a target from a single reference image by rendering multiple images at sampled pose perturbations, extracting feature correspondences, and computing a least-squares estimate for the pose using differential corrections to an initial guess. Unlike its sibling [Differentiable Rendering for Pose Estimation](11554-differentiable-rendering-for-pose-estimation.md), which explicitly constructs a numerical Jacobian of the image with respect to pose perturbations to drive gradient-based optimization, this method relies on differential corrections to an initial guess via feature matching. It bypasses the analytical description of the image formation model, which is complex due to lens and illumination dependencies, by seeking a local numerical solution in the feature space.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Demonstrated in simulation for spacecraft approach maneuvers toward the ISS and asteroid relative navigation [S1].
*   **Strengths:** Robust to complex lens and illumination dependencies; avoids analytical gradient computation [S1].
*   **Limitations:** Relies on numerical Jacobian estimation, which may be sensitive to noise and perturbation size [S1].
*   **When to Use:** When analytical gradients are difficult to derive or when the image formation model is complex [S1].
*   **When Not to Use:** When high precision and speed are required and analytical gradients are available [S1].

## Background
The method originates from the work on differentiable rendering for pose estimation in proximity operations, specifically applied to spacecraft navigation scenarios [S1]. It builds on the idea that finite difference schemes can effectively correct pose parameters by minimizing feature errors [S1].

## Sources
[S1] Differentiable Rendering for Pose Estimation in Proximity Operations, https://core.ac.uk/download/552738213.pdf, fetched 2026-10-30
