# Adaptive Gaussian Refinement

Also called: Residual Correction, 7DGS Refinement Module

## Overview
Adaptive Gaussian Refinement is a technique that dynamically adjusts Gaussian parameters using neural network-predicted residuals to improve the modeling of complex non-rigid deformations and time-varying appearance [S1]. It addresses the limitation of static covariance in standard 3D Gaussian Splatting, which hinders the representation of dynamic behaviors [S1]. It sets itself apart from sibling methods like [Score-based Densification](4705-score-based-densification.md) by using learned residual corrections rather than gradient-based splitting or cloning to adapt the primitive's geometry and appearance.

## How it works
**Inputs:**
*   Base Gaussian parameters: spatial mean $\mu_p$, temporal coordinate $\mu_t$, directional mean $\mu_d$, vectorized lower-triangular elements of the 7D covariance matrix $l$, and base opacity $\alpha$ [S1].
*   Observed time $t$ [S1].
*   View direction $d$ (used in subsequent conditional slicing, but the refinement itself depends on $t$) [S1].

**Outputs:**
*   Refined parameters: $\hat{\mu}_p, \hat{\mu}_t, \hat{\mu}_d, \hat{l}$ [S1].
*   Refined covariance matrix $\hat{\Sigma}$ derived from $\hat{l}$ [S1].

**Data Structures:**
*   Feature vector $f$: Concatenation of $\mu_p, \mu_t, \mu_d$, and temporal encoding $\gamma(t)$ [S1].
*   MLPs: Small two-layer multilayer perceptrons with architecture $C_{in} \times 64 \times C_{out}$ [S1]. There are four separate MLPs: $\phi_p, \phi_t, \phi_d, \phi_l$ [S1].

**Steps:**
1.  **Construct Feature Vector:** Concatenate the spatial mean, temporal coordinate, directional mean, and high-frequency temporal encoding [S1]:
    $$f = \mu_p \oplus \mu_t \oplus \mu_d \oplus \gamma(t)$$
    where $\oplus$ denotes vector concatenation [S1].

2.  **Temporal Encoding:** Compute $\gamma(t)$ using positional encoding with $K=10$ frequencies [S1]:
    $$\gamma(t) = [\sin(2^0 \pi t), \cos(2^0 \pi t), \dots, \sin(2^{K-1} \pi t), \cos(2^{K-1} \pi t)]$$
    with $K=10$ [S1].

3.  **Predict Residuals:** Pass $f$ through the respective MLPs to predict residual adjustments for position, time, direction, and covariance [S1]:
    *   $\delta_p = \phi_p(f)$
    *   $\delta_t = \phi_t(f)$
    *   $\delta_d = \phi_d(f)$
    *   $\delta_l = \phi_l(f)$

4.  **Apply Residuals:** Add the predicted residuals to the original parameters to yield refined estimates [S1]:
    $$\hat{\mu}_p = \mu_p + \delta_p$$
    $$\hat{\mu}_t = \mu_t + \delta_t$$
    $$\hat{\mu}_d = \mu_d + \delta_d$$
    $$\hat{l} = l + \delta_l$$
    where $l$ represents the vectorized lower-triangular elements of the 7D covariance matrix [S1].

5.  **Reconstruct Covariance:** Compute the refined covariance matrix $\hat{\Sigma}$ from $\hat{l}$ [S1]:
    $$\hat{\Sigma} = \hat{L}\hat{L}^\top$$
    where $\hat{L}$ is the lower-triangular matrix reconstructed from $\hat{l}$ [S1].

**Parameters:**
*   $K=10$: Number of frequency bands in temporal encoding [S1].
*   MLP Architecture: $C_{in} \times 64 \times C_{out}$ for each residual predictor [S1].

**Edge Cases and Pitfalls:**
*   The refinement module is applied before the conditional slicing step [S1].
*   The method assumes the base Gaussian parameters are already defined in the 7D space [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def temporal_encoding(t, K=10):
    """
    Computes high-frequency temporal encoding.
    Args:
        t: float, observed time.
        K: int, number of frequency bands (default 10).
    Returns:
        np.array, encoded vector of size 2*K.
    """
    frequencies = 2 ** np.arange(K)
    sin_vals = np.sin(2 * np.pi * frequencies * t)
    cos_vals = np.cos(2 * np.pi * frequencies * t)
    return np.concatenate([sin_vals, cos_vals])

def adaptive_gaussian_refinement(mu_p, mu_t, mu_d, l, t, phi_p, phi_t, phi_d, phi_l):
    """
    Dynamically adjusts Gaussian parameters via neural network-predicted residuals.
    Args:
        mu_p: np.array, spatial mean.
        mu_t: np.array, temporal coordinate.
        mu_d: np.array, directional mean.
        l: np.array, vectorized lower-triangular elements of covariance.
        t: float, observed time.
        phi_p, phi_t, phi_d, phi_l: callable MLPs, each taking feature vector f.
    Returns:
        tuple: (mu_p_hat, mu_t_hat, mu_d_hat, l_hat, Sigma_hat)
    """
    # Step 1: Construct Feature Vector
    gamma_t = temporal_encoding(t, K=10)
    f = np.concatenate([mu_p, mu_t, mu_d, gamma_t])
    
    # Step 2: Predict Residuals
    delta_p = phi_p(f)
    delta_t = phi_t(f)
    delta_d = phi_d(f)
    delta_l = phi_l(f)
    
    # Step 3: Apply Residuals
    mu_p_hat = mu_p + delta_p
    mu_t_hat = mu_t + delta_t
    mu_d_hat = mu_d + delta_d
    l_hat = l + delta_l
    
    # Step 4: Reconstruct Covariance
    # Assuming l_hat contains the lower triangular elements of a 7x7 matrix
    # Reconstruct L matrix from vector l_hat
    L_hat = np.zeros((7, 7))
    idx = 0
    for i in range(7):
        for j in range(i + 1):
            L_hat[i, j] = l_hat[idx]
            idx += 1
            
    Sigma_hat = L_hat @ L_hat.T
    
    return mu_p_hat, mu_t_hat, mu_d_hat, l_hat, Sigma_hat
```

## Subtopics

### Score-based Densification
Uses gradient magnitude to trigger splitting/cloning. Choose when static primitives are insufficient due to resolution limits rather than dynamic deformation. [Link](4705-score-based-densification.md)

### GroupFlow
Uses flow fields for motion. Choose for explicit motion modeling without residual networks. [Link](6351-groupflow.md)

### 3DGS-MCMC
Uses Markov Chain Monte Carlo for densification. Choose for probabilistic sampling-based adaptation. [Link](7852-3dgs-mcmc.md)

### Adaptive Density Control
General term for managing density. This method is a specific implementation using neural residuals. [Link](9188-adaptive-density-control.md)

## Operating characteristics
*   **Time Complexity**: Adds the cost of four small MLP evaluations per Gaussian per frame. The MLPs are described as "lightweight" [S1].
*   **Memory Complexity**: Requires storing the weights of four MLPs ($C_{in} \times 64 \times C_{out}$ each) [S1].
*   **Strengths**: Enables accurate modeling of complex non-rigid deformations and time-varying appearance [S1]. Preserves computational efficiency of 3DGS while incorporating rich representational capacity [S1].
*   **Limitations**: Introduces neural network inference overhead. Requires training the MLPs alongside the Gaussians.
*   **When to use**: For dynamic scenes with non-rigid motion and time-varying appearance [S1].
*   **When not to use**: For static scenes where standard 3DGS is sufficient.

## Background
Adaptive Gaussian Refinement was introduced in the 7DGS paper (Unified Spatial-Temporal-Angular Gaussian Splatting) [S1]. It was developed to address the limitation of static covariance in representing complex dynamic behaviors such as non-rigid deformations or motion-induced shape changes [S1]. The method was evaluated on synthetic monocular videos (D-NeRF) and in-the-wild multi-view videos (Technicolor) [S1].

## Sources
[S1] 7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting, https://arxiv.org/abs/2503.07946v1, fetched 2026-10-26
