# Adaptive Gaussian Refinement

Also called: 7DGS Adaptive Refinement, Residual Gaussian Adjustment

## Overview
Adaptive Gaussian Refinement is a technique that dynamically adjusts Gaussian parameters using neural network-predicted residuals to improve the modeling of complex non-rigid deformations and time-varying appearance [S1]. It addresses the limitation of static covariance matrices in representing dynamic behaviors by allowing spatial position, temporal coordinate, directional mean, and covariance to vary as functions of observed time [S1]. It is distinct from standard densification strategies (like [Adaptive Density Control](9188-adaptive-density-control.md)) because it modifies existing primitives rather than splitting or cloning them, and it is specific to high-dimensional (7D) Gaussian representations that include temporal and angular dimensions [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   Base Gaussian parameters: spatial mean $\mu_p$, temporal coordinate $\mu_t$, directional mean $\mu_d$, vectorized lower-triangular elements of the 7D covariance matrix $l$, and base opacity $\alpha$ [S1].
    *   Observed time $t$ [S1].
    *   View direction $d$ (used in subsequent conditional slicing, but the refinement itself depends on $t$) [S1].
*   **Outputs:**
    *   Refined parameters: $\hat{\mu}_p, \hat{\mu}_t, \hat{\mu}_d, \hat{l}$ [S1].
    *   Refined covariance matrix $\hat{\Sigma}$ derived from $\hat{l}$ [S1].

**Data Structures:**
*   **Feature Vector $f$:** A concatenated vector containing $\mu_p$ (3D), $\mu_t$ (1D), $\mu_d$ (3D), and temporal encoding $\gamma(t)$ (20D). Total dimension: $3+1+3+20 = 27$ [S1].
*   **MLP Architecture:** Small two-layer multilayer perceptrons (MLPs) with architecture $C_{in} \times 64 \times C_{out}$ [S1]. $C_{in}$ is the dimension of $f$ (27). $C_{out}$ matches the dimension of the parameter being predicted (e.g., 3 for means, 28 for 7D covariance lower-triangular elements) [S1].

**Steps:**
1.  **Temporal Encoding:** Compute high-frequency positional encoding for time $t$:
    $$ \gamma(t) = [\sin(2\pi t), \cos(2\pi t), \dots, \sin(2^{K-1}\pi t), \cos(2^{K-1}\pi t)] $$
    where $K=10$ [S1].
2.  **Feature Construction:** Concatenate the spatial mean, temporal coordinate, directional mean, and temporal encoding:
    $$ f = \mu_p \oplus \mu_t \oplus \mu_d \oplus \gamma(t) $$
    where $\oplus$ denotes vector concatenation [S1].
3.  **Residual Prediction:** Pass $f$ through four separate MLPs ($\phi_p, \phi_t, \phi_d, \phi_l$) to predict residual adjustments for position, time, direction, and covariance [S1].
4.  **Parameter Update:** Add the predicted residuals to the original parameters:
    $$ \hat{\mu}_p = \mu_p + \phi_p(f) $$
    $$ \hat{\mu}_t = \mu_t + \phi_t(f) $$
    $$ \hat{\mu}_d = \mu_d + \phi_d(f) $$
    $$ \hat{l} = l + \phi_l(f) $$
    where $l$ represents the vectorized lower-triangular elements of the 7D covariance matrix [S1].
5.  **Covariance Reconstruction:** Reconstruct the refined covariance matrix from the updated lower-triangular elements:
    $$ \hat{\Sigma} = \hat{L}\hat{L}^\top $$
    where $\hat{L}$ is the matrix form of $\hat{l}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_gaussian_refinement(mu_p, mu_t, mu_d, l, t, K=10):
    """
    Dynamically adjusts Gaussian parameters using neural network-predicted residuals.
    
    Args:
        mu_p: Spatial mean (3,)
        mu_t: Temporal coordinate (1,)
        mu_d: Directional mean (3,)
        l: Vectorized lower-triangular elements of 7D covariance (28,)
        t: Observed time (scalar)
        K: Number of frequency bands for positional encoding (int)
        
    Returns:
        mu_p_refined, mu_t_refined, mu_d_refined, l_refined, Sigma_refined
    """
    # Step 1: Temporal Encoding
    # gamma(t) has shape (2*K,)
    frequencies = 2.0 ** np.arange(K) * np.pi
    sin_vals = np.sin(frequencies * t)
    cos_vals = np.cos(frequencies * t)
    gamma_t = np.concatenate([sin_vals, cos_vals])
    
    # Step 2: Feature Construction
    # f has shape (3 + 1 + 3 + 20,) = (27,)
    f = np.concatenate([mu_p, mu_t, mu_d, gamma_t])
    
    # Step 3: Residual Prediction
    # phi_p, phi_t, phi_d, phi_l are MLPs defined elsewhere
    # Each takes f (27,) and outputs residuals for the respective parameter
    residual_p = phi_p(f)  # Shape (3,)
    residual_t = phi_t(f)  # Shape (1,)
    residual_d = phi_d(f)  # Shape (3,)
    residual_l = phi_l(f)  # Shape (28,)
    
    # Step 4: Parameter Update
    mu_p_refined = mu_p + residual_p
    mu_t_refined = mu_t + residual_t
    mu_d_refined = mu_d + residual_d
    l_refined = l + residual_l
    
    # Step 5: Covariance Reconstruction
    # Convert vectorized lower-triangular elements to matrix L
    L_refined = vector_to_matrix(l_refined)
    # Reconstruct covariance matrix
    Sigma_refined = L_refined @ L_refined.T
    
    return mu_p_refined, mu_t_refined, mu_d_refined, l_refined, Sigma_refined

def vector_to_matrix(v):
    """
    Converts a vector of 28 elements into a 7x7 lower-triangular matrix.
    """
    L = np.zeros((7, 7))
    idx = 0
    for i in range(7):
        for j in range(i + 1):
            L[i, j] = v[idx]
            idx += 1
    return L
```

**Parameters:**
*   $K=10$: Number of frequency bands for temporal encoding [S1].
*   MLP Hidden Dimension: 64 [S1].
*   MLP Input Dimension ($C_{in}$): 27 [S1].

**Edge Cases and Pitfalls:**
*   **Covariance Positivity:** The refinement updates the lower-triangular elements $l$. The resulting matrix $\hat{L}$ must remain valid. If residuals are large, $\hat{\Sigma}$ might become non-positive definite. The sources do not specify a projection step to enforce positive definiteness after refinement [S1].
*   **Temporal Consistency:** The method assumes $t$ is normalized or appropriately scaled for the positional encoding. The sources do not specify the range of $t$ [S1].

## Operating characteristics
*   **Time Complexity:** Adds the cost of four small MLP evaluations per Gaussian per frame. The MLPs are $C_{in} \times 64 \times C_{out}$, which is negligible compared to the rendering cost [S1].
*   **Memory Complexity:** Requires storing the weights of four MLPs. The number of parameters is small: $4 \times (27 \times 64 + 64 \times C_{out})$. For $C_{out}=3$ (means) and $C_{out}=28$ (covariance), this is minimal [S1].
*   **Strengths:** Enables modeling of complex non-rigid deformations and time-varying appearance that static Gaussians cannot capture [S1].
*   **Limitations:** Requires training additional neural networks (MLPs) and increases the complexity of the optimization loop. It is specific to 7D Gaussian representations [S1].
*   **When to use:** When rendering dynamic scenes with non-rigid motion and time-varying appearance, and when using a 7D Gaussian representation [S1].
*   **When not to use:** For static scenes or when using standard 3D Gaussian Splatting without temporal/angular dimensions [S1].

## Background
Adaptive Gaussian Refinement was introduced in the 7DGS (Unified Spatial-Temporal-Angular Gaussian Splatting) paper [S1]. It was developed to address the limitation of static covariance in representing complex dynamic behaviors such as non-rigid deformations or motion-induced shape changes [S1]. The method is applied before the conditional slicing step in the 7DGS pipeline [S1].

## Sources
[S1] 7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting, https://arxiv.org/abs/2503.07946v1, fetched 2026-10-26
