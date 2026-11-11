# Adaptive Gaussian Refinement

Also called: 7DGS Adaptive Refinement, Residual Gaussian Update

## Overview
Adaptive Gaussian Refinement is a technique that dynamically adjusts Gaussian parameters using neural network-predicted residuals to improve the modeling of complex non-rigid deformations and time-varying appearance [S1]. It addresses the limitation of static covariance matrices in representing dynamic behaviors by applying residual corrections computed by lightweight neural networks [S1]. Unlike standard densification strategies that modify the set of primitives via splitting or cloning, this method refines the existing parameters of each Gaussian to better fit temporal and angular variations [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   Base Gaussian parameters: spatial mean $\mu_p$, temporal coordinate $\mu_t$, directional mean $\mu_d$, and the vectorized lower-triangular elements of the 7D covariance matrix $l$ [S1].
    *   Observed time $t$ [S1].
    *   View direction $d$ (used in subsequent conditional slicing, but the refinement itself depends on the Gaussian's intrinsic parameters and time) [S1].
*   **Outputs:**
    *   Refined parameters: $\hat{\mu}_p, \hat{\mu}_t, \hat{\mu}_d, \hat{l}$ [S1].

**Data Structures:**
*   **Feature Vector $f$:** A concatenated vector containing $\mu_p$, $\mu_t$, $\mu_d$, and the temporal encoding $\gamma(t)$ [S1].
*   **MLP Architecture:** Small two-layer multilayer perceptrons (MLPs) with architecture $C_{in} \times 64 \times C_{out}$ [S1]. The input dimension $C_{in}$ is the size of $f$. The output dimension $C_{out}$ corresponds to the dimension of the specific parameter being refined (e.g., 3 for means, 28 for the lower-triangular covariance vector) [S1].

**Steps:**
1.  **Construct Feature Vector:** Concatenate the spatial mean $\mu_p$, temporal coordinate $\mu_t$, directional mean $\mu_d$, and high-frequency temporal encoding $\gamma(t)$ to form the feature vector $f$:
    $$ f = \mu_p \oplus \mu_t \oplus \mu_d \oplus \gamma(t) $$
    where $\oplus$ denotes vector concatenation [S1].

2.  **Compute Temporal Encoding:** Calculate $\gamma(t)$ using positional encoding with $K=10$ frequencies:
    $$ \gamma(t) = [\sin(2^0 \pi t), \cos(2^0 \pi t), \dots, \sin(2^{K-1} \pi t), \cos(2^{K-1} \pi t)] $$
    with $K=10$ [S1].

3.  **Predict Residuals:** Pass $f$ through separate MLPs $\phi_p, \phi_t, \phi_d, \phi_l$ to predict residual adjustments for the spatial mean, temporal coordinate, directional mean, and covariance, respectively [S1].

4.  **Apply Residuals:** Add the predicted residuals to the original parameters to obtain the refined estimates:
    $$ \hat{\mu}_p = \mu_p + \phi_p(f) $$
    $$ \hat{\mu}_t = \mu_t + \phi_t(f) $$
    $$ \hat{\mu}_d = \mu_d + \phi_d(f) $$
    $$ \hat{l} = l + \phi_l(f) $$
    where $l$ represents the vectorized lower-triangular elements of the 7D covariance matrix [S1].

5.  **Reconstruct Covariance (Implicit):** The refined covariance matrix $\hat{\Sigma}$ is reconstructed from $\hat{l}$ as $\hat{\Sigma} = \hat{L}\hat{L}^\top$, where $\hat{L}$ is the lower-triangular matrix formed from $\hat{l}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def temporal_encoding(t, K=10):
    """
    Computes the high-frequency temporal encoding gamma(t).
    """
    frequencies = np.power(2, np.arange(K))
    angles = frequencies * np.pi * t
    sin_vals = np.sin(angles)
    cos_vals = np.cos(angles)
    return np.concatenate([sin_vals, cos_vals])

def adaptive_gaussian_refinement(mu_p, mu_t, mu_d, l, t, mlps):
    """
    Applies adaptive Gaussian refinement to update Gaussian parameters.
    
    Args:
        mu_p: Spatial mean (3D vector)
        mu_t: Temporal coordinate (scalar or 1D vector)
        mu_d: Directional mean (3D vector)
        l: Vectorized lower-triangular elements of the 7D covariance matrix
        t: Observed time
        mlps: Dictionary containing MLPs phi_p, phi_t, phi_d, phi_l
        
    Returns:
        hat_mu_p, hat_mu_t, hat_mu_d, hat_l: Refined parameters
    """
    # Step 1: Compute temporal encoding
    gamma_t = temporal_encoding(t, K=10)
    
    # Step 2: Construct feature vector f
    # Note: mu_t and mu_d dimensions must be compatible for concatenation.
    # Assuming mu_t is scalar, it might need broadcasting or specific handling.
    # Based on source, f = mu_p || mu_t || mu_d || gamma(t)
    f = np.concatenate([mu_p, [mu_t], mu_d, gamma_t])
    
    # Step 3: Predict residuals using MLPs
    # Each MLP is a 2-layer network: C_in -> 64 -> C_out
    # phi_p outputs residual for mu_p (dim 3)
    # phi_t outputs residual for mu_t (dim 1)
    # phi_d outputs residual for mu_d (dim 3)
    # phi_l outputs residual for l (dim 28 for 7D covariance lower triangle)
    
    res_p = mlps['phi_p'](f)
    res_t = mlps['phi_t'](f)
    res_d = mlps['phi_d'](f)
    res_l = mlps['phi_l'](f)
    
    # Step 4: Apply residuals
    hat_mu_p = mu_p + res_p
    hat_mu_t = mu_t + res_t
    hat_mu_d = mu_d + res_d
    hat_l = l + res_l
    
    return hat_mu_p, hat_mu_t, hat_mu_d, hat_l
```

**Parameters:**
*   $K=10$: Number of frequencies in the temporal encoding [S1].
*   MLP Hidden Size: 64 [S1].
*   MLP Layers: 2 [S1].

**Edge Cases and Pitfalls:**
*   The refinement module is applied *before* the conditional slicing step [S1].
*   The source does not specify how the MLPs are trained or initialized, only their architecture [S1].
*   The dimensionality of $\mu_t$ is not explicitly defined beyond being part of the concatenation; it is treated as a coordinate [S1].

## Operating characteristics
*   **Complexity:** The computational overhead is determined by the forward pass of four small MLPs per Gaussian per time step. The MLPs are described as "lightweight" [S1].
*   **Memory:** Requires storing the parameters of the MLPs ($C_{in} \times 64 + 64 \times C_{out}$ per parameter type) in addition to the Gaussian parameters [S1].
*   **Strengths:** Enables accurate modeling of complex non-rigid deformations and time-varying appearance which static Gaussians cannot capture [S1].
*   **Limitations:** Adds computational complexity compared to static 3DGS. The source does not provide specific performance benchmarks for the refinement step itself, only overall method performance [S1].

## Background
Adaptive Gaussian Refinement was introduced in the context of 7DGS (Unified Spatial-Temporal-Angular Gaussian Splatting) [S1]. It was developed to address the limitation that the intrinsic shape of each Gaussian, determined by its covariance, remains static over time, which hinders representing complex dynamic behaviors such as non-rigid deformations or motion-induced shape changes [S1]. The method was evaluated on synthetic monocular videos (D-NeRF), in-the-wild multi-view videos (Technicolor), and a custom dataset (7DGS-PBR) [S1].

## Sources
[S1] 7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting, https://arxiv.org/abs/2503.07946v1, fetched 2026-10-26
