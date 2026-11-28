# Jacobian-Free Reparameterization

Also called: linearize operator

## Overview
Jacobian-Free Reparameterization is a method to numerically compute the Jacobian matrix $J_f$ of an image formation model $x = f(p)$ by generating random pose perturbations, rendering images, and solving a linear least-squares system [S2]. It bypasses the analytical differentiation of the image formation process, which is often complex due to lens and illumination modeling [S2]. Unlike methods that only track a target pose, this approach estimates the pose by minimizing feature errors using the estimated Jacobian, with learning the inverse Jacobian noted to result in better performance than inverting an estimate of the Jacobian [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A reference pose $p_{ref}$, a set of $N_s$ random pose perturbations $\Delta p_i$, and a feature extraction function $h$ that maps a rendered image to a feature vector $x$.
*   **Output:** The Jacobian matrix $J_f$ (or its inverse) approximating the local gradient of the image formation model [S2].

**Data Structures:**
*   **Feature Vector $x$:** For $k$ key-point features, $x$ is a vector of $2k$ coordinates. In the source example with 4 key-points, $x = (x_1, y_1, \dots, x_4, y_4)$, resulting in a dimension of 8 [S2].
*   **Pose Difference Matrix $B$:** Size $6 \times N_s$. Each column $i$ contains the relative pose difference $\Delta p_i$ composed of rotation and translation components [S2].
*   **Feature Difference Matrix $E$:** Size $2k \times N_s$. Each column $i$ contains the feature error $\Delta x_i$ [S2].

**Steps:**
1.  **Pose Sampling:** Generate $N_s$ random pose perturbations $\Delta p_i$ around the reference pose $p_{ref}$. The rotation component is sampled using Gibbs vectors (uniformly distributed on a spherical surface), and the translation component is sampled randomly [S2].
2.  **Rendering and Feature Extraction:** For each perturbed pose $p_i = p_{ref} + \Delta p_i$, render the image and extract the feature vector $x_i$ [S2].
3.  **Compute Feature Differences:** Calculate the difference between the reference feature vector $x_{ref}$ and the perturbed feature vector $x_i$:
    $$ \Delta x_i = x_{ref} - x_i $$
    This forms the matrix $E = [\Delta x_1, \dots, \Delta x_{N_s}]$ [S2].
4.  **Compute Pose Differences:** Calculate the relative pose difference $\Delta p_i$. Using Gibbs vectors $q$ for attitude, the relative pose difference is:
    $$ \Delta R_i = R_{ref}^T R_i $$
    $$ \Delta t_i = t_{ref} - t_i $$
    $$ \Delta p_i = [\Delta q_i, \Delta t_i]^T $$
    This forms the matrix $B = [\Delta p_1, \dots, \Delta p_{N_s}]$ [S2].
5.  **Solve for Jacobian:** The relationship is linearized as $E \approx J_f B$. Solve for $J_f$ using a least-squares solution:
    $$ J_f = E B^T (B B^T)^{-1} $$
    The dimension of $E$ is $2k \times N_s$, $B$ is $6 \times N_s$, and $J_f$ is $2k \times 6$ [S2].
6.  **Jacobian-Free Application (Autograd):** In frameworks like Jax, the Jacobian-vector product can be computed without explicit matrix construction using the `linearize` operator. To apply the Jacobian to a covariance matrix $\Sigma$, apply the `linearize` operator twice in sequence to $\Sigma$, transposing the dimensions of the covariance matrix after each application [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import jax
import jax.numpy as jnp
import numpy as np

def generate_pose_perturbation(p_ref, n_samples):
    """
    Generate n_samples random pose perturbations.
    p_ref: (6,) vector [q_x, q_y, q_z, t_x, t_y, t_z]
    Returns: perturbations (6, n_samples)
    """
    # Sample Gibbs vectors for rotation (uniform on sphere)
    # Sample random translations
    # Combine into (6, n_samples) matrix
    # Note: Specific sampling details for Gibbs vectors are not fully specified in [S2] beyond "uniformly on spherical surface"
    # Implementation detail [G]: Use rejection sampling or normal distribution normalization for Gibbs vectors.
    perturbations = np.random.randn(6, n_samples) * 0.1 # Placeholder scale
    return perturbations

def render_and_extract_features(pose, feature_extractor):
    """
    Render image at pose and extract features.
    Returns: feature vector (2*k,)
    """
    # Render image
    # Extract features
    # Return flattened feature vector
    pass

def compute_jacobian(p_ref, feature_extractor, n_samples):
    """
    Compute Jacobian J_f via linear least squares.
    """
    # 1. Pose Sampling
    deltas_p = generate_pose_perturbation(p_ref, n_samples)
    
    # 2. Rendering and Feature Extraction
    # Reference features
    x_ref = render_and_extract_features(p_ref, feature_extractor)
    
    E_list = []
    B_list = []
    
    for i in range(n_samples):
        # Perturbed pose
        p_i = p_ref + deltas_p[:, i]
        
        # Render and extract
        x_i = render_and_extract_features(p_i, feature_extractor)
        
        # 3. Feature Differences
        delta_x = x_ref - x_i
        E_list.append(delta_x)
        
        # 4. Pose Differences
        # Assuming p_ref and p_i are [q, t]
        # Compute relative rotation and translation
        # Note: [S2] defines delta_p as [delta_q, delta_t]
        # Implementation detail [G]: Convert quaternions to Gibbs vectors for delta_q
        delta_q = compute_gibbs_difference(p_ref[:3], p_i[:3])
        delta_t = p_ref[3:] - p_i[3:]
        delta_p = np.concatenate([delta_q, delta_t])
        B_list.append(delta_p)
        
    E = np.column_stack(E_list) # (2k, N_s)
    B = np.column_stack(B_list) # (6, N_s)
    
    # 5. Solve Least Squares
    # J_f = E * B^T * (B * B^T)^-1
    # Or use numpy.linalg.lstsq
    J_f, residuals, rank, s = np.linalg.lstsq(B.T, E.T, rcond=None)
    J_f = J_f.T # (2k, 6)
    
    return J_f

def jacobian_free_apply(p_ref, feature_extractor, sigma):
    """
    Apply Jacobian to covariance matrix sigma without explicit J_f construction.
    Uses JAX linearize operator.
    """
    # [S1] Apply linearize operator twice to sigma, transposing dimensions after each application.
    # This is a high-level description as the exact JAX API for 'linearize' on a custom function
    # requires defining the function f(p) -> x.
    
    def f(p):
        x = render_and_extract_features(p, feature_extractor)
        return x
    
    # JAX linearize returns a function that applies the Jacobian
    # J_f @ v
    linearized_f = jax.linearize(f, p_ref)
    
    # Apply to sigma (assuming sigma is a matrix)
    # The source mentions applying it twice to Sigma with transposes.
    # This likely refers to propagating covariance: Sigma_out = J_f @ Sigma_in @ J_f^T
    # However, the text says "applying it twice in sequence to Sigma".
    # This pseudocode captures the intent of avoiding explicit matrix construction.
    
    # Step 1: Apply J_f to Sigma
    # Step 2: Apply J_f^T (or transpose logic) to the result
    # Note: The exact JAX syntax for this specific covariance propagation is not fully detailed in [S1]
    # beyond "applying it twice in sequence to Sigma, with the dimensions of the covariance matrix transposed after each application".
    
    return None # Placeholder for specific JAX implementation details
```

**Parameters:**
*   $N_s$: Number of pose samples. Must be sufficiently large for the least-squares solution to be stable [S2].
*   Perturbation magnitude: Small enough to justify the linear approximation (Taylor series expansion ignoring higher-order terms) [S2].

**Edge Cases and Pitfalls:**
*   **Rank Deficiency:** The matrix $B$ may be rank-deficient if perturbations do not cover all 6 degrees of freedom. Mitigation: Randomly and carefully perturb the pose along all 6 axes [S2].
*   **Feature Tracking Failure:** If tracking of at least four features is unsuccessful, the method re-initializes with conservative pose samples [S2].
*   **Higher-Order Terms:** The method relies on a linear approximation. If perturbations are too large, higher-order terms in the Taylor series expansion become significant, reducing accuracy [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the number of samples $N_s$ and the cost of rendering and feature extraction. Solving the least-squares system is $O(N_s)$ for fixed feature dimensions [S2].
*   **Memory Complexity:** Stores $N_s$ rendered images and feature vectors. The Jacobian matrix size is fixed ($2k \times 6$) [S2].
*   **Strengths:** Bypasses complex analytical differentiation of the image formation model, including lens and illumination effects [S2]. Learning the inverse Jacobian provides better performance than inverting an estimate [S2].
*   **Limitations:** Requires multiple renders per Jacobian estimation step. Sensitive to feature tracking failures [S2].
*   **When to Use:** When analytical gradients are difficult to derive or when the image formation model includes complex non-differentiable components (e.g., specific lens models) [S2].

## Background
The method originates from work on differentiable rendering for pose estimation in proximity operations [S2]. It builds on the idea that online learning to estimate the Jacobian based on visual features can be used for pose estimation rather than just tracking [S2]. The use of the `linearize` operator in JAX for Jacobian-free reparameterization is discussed in the context of Mip-NeRF 360 [S1].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] Differentiable Rendering for Pose Estimation in Proximity Operations, https://core.ac.uk/download/552738213.pdf, fetched 2026-10-30
