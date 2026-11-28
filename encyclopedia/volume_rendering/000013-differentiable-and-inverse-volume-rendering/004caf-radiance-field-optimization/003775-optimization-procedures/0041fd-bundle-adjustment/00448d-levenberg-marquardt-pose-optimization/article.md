# Levenberg-Marquardt Pose Optimization

Also called: LM Pose Estimation, Pose Optimization via Levenberg-Marquardt

## Overview
Levenberg-Marquardt Pose Optimization is an iterative procedure that minimizes the squared error between reference image features and rendered image features by using a numerically estimated Jacobian [S1]. It sets itself apart from its siblings by relying on finite-difference Jacobian approximation via pose perturbation rather than analytical derivatives or derivative-free search [S1].

## How it works
**Inputs:** Reference image $s_{ref}$, initial pose guess $\hat{p}_0$, Levenberg-Marquardt update parameter $\lambda$, convergence threshold $\epsilon$, number of perturbation samples $N_s$ [S1].
**Outputs:** Converged pose estimate $\hat{p}_{final}$ [S1].

**Data Structures:**
*   Pose vector $\hat{p} \in \mathbb{R}^6$: Represents the 6-DoF camera pose (rotation and translation) [S1].
*   Feature vector $\tilde{x}$: Extracted features from the reference image [S1].
*   Jacobian matrix $J_f$: Numerically estimated Jacobian of the feature error with respect to the pose parameters [S1].

**Steps:**
1.  **Initialization:** Extract reference features $\tilde{x}$ from $s_{ref}$ [S1]. Set current pose $\hat{p}_i = \hat{p}_0$ and initialize pose update $\delta p$ to a value exceeding $\epsilon$ [S1].
2.  **Iteration Loop:** While $\|\delta p\| > \epsilon$:
    a.  **Render and Match:** Render the scene at the current pose guess $\hat{p}_i$ [S1]. Extract features from the rendered image and match them with the reference features $\tilde{x}$ to obtain the current feature correspondence $h(s_i, \hat{p}_i)$ [S1].
    b.  **Pose Sampling:** Sample $N_s$ pose parameters around the current guess $\hat{p}_i$ [S1]. The sampling biases random pose samples from the $SE(3)$ group to be within a $\delta$-neighborhood of the reference pose, specifically sampling rotation axes uniformly on a spherical surface [S1].
    c.  **Jacobian Estimation:** Render images at each sampled pose value and match reference features in each rendered image [S1]. Calculate the Jacobian $J_f$ using the feature differences and pose perturbations via finite differences (referred to as online learning in the source) [S1].
    d.  **LM Update:** Calculate the least-squares pose update $\hat{p}_{i+1}$ using the Levenberg-Marquardt formulation [S1].
    e.  **Pose Update:** Update the pose: $\hat{p}_i \leftarrow \hat{p}_{i+1}$ [S1].
3.  **Termination:** Return $\hat{p}_{final} = \hat{p}_i$ when convergence is reached [S1].

**Formulas:**
*   Feature error vector: $e = \tilde{x} - h(s_i, \hat{p}_i)$ [S1].
*   Jacobian estimation (finite difference accumulation): $J_f \approx \frac{1}{N_s} \sum_{k=1}^{N_s} \frac{h(s_i, \hat{p}_k) - h(s_i, \hat{p}_i)}{\hat{p}_k - \hat{p}_i}$ [S1].
*   Levenberg-Marquardt update: $(J_f^T J_f + \lambda I) \delta p = -J_f^T e$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def levenberg_marquardt_pose_optimization(s_ref, p_hat_0, lambda_param, epsilon, N_s):
    x_tilde = extract_features(s_ref)
    p_hat = p_hat_0
    delta_p = np.ones(6) * 10.0
    while np.linalg.norm(delta_p) > epsilon:
        rendered_img = render_scene(p_hat)
        current_features = extract_features(rendered_img)
        matched_features = match_features(current_features, x_tilde)
        e = x_tilde - matched_features
        
        sampled_poses = sample_poses_around(p_hat, N_s)
        J_f = np.zeros((len(x_tilde), 6))
        for k in range(N_s):
            p_sampled = sampled_poses[k]
            img_sampled = render_scene(p_sampled)
            feat_sampled = extract_features(img_sampled)
            matched_sampled = match_features(feat_sampled, x_tilde)
            delta_e = matched_sampled - matched_features
            delta_p_sampled = p_sampled - p_hat
            for j in range(6):
                if abs(delta_p_sampled[j]) > 1e-8:
                    J_f[:, j] += delta_e / delta_p_sampled[j]
        J_f /= N_s
        
        JTJ = J_f.T @ J_f
        lambda_I = lambda_param * np.eye(6)
        A = JTJ + lambda_I
        b = -J_f.T @ e
        delta_p = np.linalg.solve(A, b)
        p_hat = p_hat + delta_p
    return p_hat

def extract_features(img):
    # Standard feature extraction pipeline
    return np.zeros(128)

def render_scene(pose):
    # Differentiable rendering at given pose
    return np.zeros((224, 224, 3))

def match_features(current, reference):
    # Feature matching routine
    return reference.copy()

def sample_poses_around(p_current, N_s):
    sampled = []
    for _ in range(N_s):
        perturbation = np.random.randn(6) * 0.01
        sampled.append(p_current + perturbation)
    return sampled
```

**Parameters:**
*   $\lambda$: Levenberg-Marquardt update parameter [S1].
*   $\epsilon$: Convergence threshold for $\|\delta p\|$ [S1].
*   $N_s$: Number of pose samples for Jacobian estimation [S1].

**Edge Cases and Pitfalls:**
Not given in the sources.

## Subtopics

### [Derivative-free Pose Estimation via Feature Matching](11310-derivative-free-pose-estimation-via-feature-matching.md)
Estimates 6-DoF pose by rendering sampled perturbations and using feature matching for least-squares differential corrections, bypassing analytical Jacobians.

### [Lie Algebra (Tangent Plane) Pose Optimization](14005-lie-algebra-tangent-plane-pose-optimization.md)
Optimizes pose in the tangent space of $SE(3)$ ($\mathfrak{se}(3)$) for smoother convergence, avoiding noisy photometric loss landscapes on the manifold.

### [Shonan Averaging for Pose Synchronization](17128-shonan-averaging-for-pose-synchronization.md)
Finds global alignment of camera poses from noisy relative measurements by relaxing the problem into higher-dimensional special-orthogonal spaces.

## Operating characteristics
Not given in the sources.

## Background
The method originated in the context of differentiable rendering for pose estimation in proximity operations, such as spacecraft approach and asteroid navigation [S1]. It applies an iterative optimization procedure that uses a numerically estimated Jacobian to minimize the squared error between reference image features and rendered image features [S1].

## Sources
[S1] Differentiable Rendering for Pose Estimation in Proximity Operations, https://core.ac.uk/download/552738213.pdf, fetched 2026-10-30
