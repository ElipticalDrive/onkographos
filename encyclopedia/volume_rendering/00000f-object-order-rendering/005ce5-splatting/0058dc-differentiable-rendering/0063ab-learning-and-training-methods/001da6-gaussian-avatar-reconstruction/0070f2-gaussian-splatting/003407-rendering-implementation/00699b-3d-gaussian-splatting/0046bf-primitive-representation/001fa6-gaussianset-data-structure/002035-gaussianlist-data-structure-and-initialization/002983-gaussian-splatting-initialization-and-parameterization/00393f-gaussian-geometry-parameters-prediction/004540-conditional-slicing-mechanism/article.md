# Conditional Slicing Mechanism

## Overview
The Conditional Slicing Mechanism conditions a 7-dimensional Gaussian on observed time $t$ and view direction $d$ to yield a conditional 3D Gaussian governing the spatial component [S1]. This operation "slices" the full 7D distribution to integrate temporal dynamics and view-dependent effects into the traditional 3D rendering pipeline without introducing additional temporal dependencies in the spherical harmonics [S1]. It sets itself apart from static Gaussian Splatting variants by explicitly modeling the joint distribution of space, time, and view, allowing the spatial mean and covariance to shift based on the current rendering context [S1].

## How it works
**Inputs:**
*   Base Gaussian parameters: spatial mean $\mu_p$, temporal mean $\mu_t$, directional mean $\mu_d$, base opacity $\alpha$, scaling factors $\lambda_t, \lambda_d$ [S1].
*   Covariance structure: Lower-triangular matrix $L$ such that $\Sigma = LL^\top$, or the partitioned blocks $\Sigma_{(t,d)}$, $\Sigma_{p,(t,d)}$, $\Sigma_p$ [S1].
*   Context variables: Observed time $t$, view direction $d$ [S1].
*   Optional refinement features: Feature vector $f$ (if adaptive refinement is applied) [S1].

**Outputs:**
*   Conditional spatial mean $\mu_{cond}$ [S1].
*   Conditional spatial covariance $\Sigma_{cond}$ [S1].
*   Conditional opacity $\alpha_{cond}$ [S1].
*   Optional: Scale $S$ and Rotation $R$ extracted from $\Sigma_{cond}$ for densification [S1].

**Data Structures:**
The 7D Gaussian is represented by a mean vector $\mu = [\mu_p, \mu_t, \mu_d]$ and a $7 \times 7$ covariance matrix $\Sigma$ [S1]. The matrix is partitioned into blocks corresponding to spatial ($p$), temporal ($t$), and directional ($d$) dimensions:
$$
\Sigma = \begin{bmatrix} \Sigma_p & \Sigma_{p,(t,d)}^\top \\ \Sigma_{p,(t,d)} & \Sigma_{(t,d)} \end{bmatrix}
$$
where $\Sigma_{(t,d)}$ is the $4 \times 4$ block for time and direction:
$$
\Sigma_{(t,d)} = \begin{bmatrix} \Sigma_t & \Sigma_{td} \\ \Sigma_{td}^\top & \Sigma_d \end{bmatrix}
$$
and $\Sigma_{p,(t,d)} = [\Sigma_{pt} \quad \Sigma_{pd}]$ links spatial dimensions with the combined temporal-directional space [S1].

**Steps:**

1.  **Adaptive Gaussian Refinement (Optional but standard in implementation):**
    Compute a feature vector $f$ by concatenating the means and a temporal encoding $\gamma(t)$:
    $$ f = \text{concat}(\mu_p, \mu_t, \mu_d, \gamma(t)) $$
    Apply refinement functions $\phi_p, \phi_t, \phi_d, \phi_l$ to update the parameters:
    $$ \hat{\mu}_p = \mu_p + \phi_p(f), \quad \hat{\mu}_t = \mu_t + \phi_t(f) $$
    $$ \hat{\mu}_d = \mu_d + \phi_d(f), \quad \hat{l} = l + \phi_l(f) $$
    Reconstruct the refined covariance matrix $\hat{\Sigma} = \hat{L}\hat{L}^\top$ [S1].
    *Note: If refinement is skipped, replace $\hat{\mu}$ and $\hat{\Sigma}$ with $\mu$ and $\Sigma$.*

2.  **Partition the Covariance:**
    Partition $\hat{\Sigma}$ into blocks $\hat{\Sigma}_p$, $\hat{\Sigma}_{(t,d)}$, and $\hat{\Sigma}_{p,(t,d)}$ as defined above [S1].

3.  **Compute Conditional Mean:**
    Calculate the conditional spatial mean $\mu_{cond}$ using the standard multivariate Gaussian conditioning formula:
    $$ \mu_{cond} = \hat{\mu}_p + \hat{\Sigma}_{p,(t,d)} \hat{\Sigma}_{(t,d)}^{-1} \begin{bmatrix} t - \hat{\mu}_t \\ d - \hat{\mu}_d \end{bmatrix} $$
    The term $\hat{\Sigma}_{p,(t,d)} \hat{\Sigma}_{(t,d)}^{-1}$ acts as a correction that shifts the spatial mean according to deviations in time and view direction from their expected values [S1].

4.  **Compute Conditional Covariance:**
    Calculate the conditional spatial covariance $\Sigma_{cond}$:
    $$ \Sigma_{cond} = \hat{\Sigma}_p - \hat{\Sigma}_{p,(t,d)} \hat{\Sigma}_{(t,d)}^{-1} \hat{\Sigma}_{p,(t,d)}^\top $$
    This adjusts the spatial uncertainty by removing the variance explained by the temporal and directional components [S1].

5.  **Compute Modulation Factors:**
    Compute the temporal modulation factor $f_{temp}$:
    $$ f_{temp} = \exp\left(-\frac{1}{2} \lambda_t (t - \hat{\mu}_t)^2 \hat{\Sigma}_t^{-1}\right) $$
    Compute the directional modulation factor $f_{dir}$:
    $$ f_{dir} = \exp\left(-\frac{1}{2} \lambda_d (d - \hat{\mu}_d)^\top \hat{\Sigma}_d^{-1} (d - \hat{\mu}_d)\right) $$
    Here, $\lambda_t$ and $\lambda_d$ are positive scalar parameters controlling sensitivity [S1].

6.  **Compute Conditional Opacity:**
    Combine the base opacity $\alpha$ with the modulation factors:
    $$ \alpha_{cond} = \alpha \cdot f_{temp} \cdot f_{dir} $$
    This ensures Gaussians contribute less when the current time or view direction is far from their expected values [S1].

7.  **Extract Scale and Rotation (Optional):**
    For densification steps, perform SVD on $\Sigma_{cond} = U D U^\top$. Extract scale $S = \text{diag}(D)$ and rotation $R = U$. Adjust $R$ to ensure $\det(R) > 0$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def gamma(t):
    # Standard positional encoding for time [G]
    return np.array([np.sin(t), np.cos(t)])

def phi_p(f): return np.zeros_like(mu_p) # MLP from parent model [S1]
def phi_t(f): return np.zeros_like(mu_t) # MLP from parent model [S1]
def phi_d(f): return np.zeros_like(mu_d) # MLP from parent model [S1]
def phi_l(f): return np.zeros_like(L)    # MLP from parent model [S1]

def conditional_slicing(mu_p, mu_t, mu_d, alpha, L, lambda_t, lambda_d, t, d, apply_refinement=True):
    """
    Conditions a 7D Gaussian on time t and view direction d.
    
    Args:
        mu_p: Spatial mean (3,)
        mu_t: Temporal mean (scalar)
        mu_d: Directional mean (3,)
        alpha: Base opacity (scalar)
        L: Lower triangular matrix (7,7) such that Sigma = L @ L.T
        lambda_t: Temporal sensitivity scalar
        lambda_d: Directional sensitivity scalar
        t: Observed time (scalar)
        d: View direction (3,)
        apply_refinement: Boolean flag for adaptive refinement
        
    Returns:
        mu_cond: Conditional spatial mean (3,)
        Sigma_cond: Conditional spatial covariance (3,3)
        alpha_cond: Conditional opacity (scalar)
        S: Scale (3,) - optional, for densification
        R: Rotation (3,3) - optional, for densification
    """
    # Step 1: Adaptive Refinement (if enabled)
    if apply_refinement:
        # Compute feature vector f
        f = np.concatenate([mu_p, [mu_t], mu_d, gamma(t)])
        
        # Apply refinement functions phi
        delta_mu_p = phi_p(f)
        delta_mu_t = phi_t(f)
        delta_mu_d = phi_d(f)
        delta_l = phi_l(f)
        
        mu_p = mu_p + delta_mu_p
        mu_t = mu_t + delta_mu_t
        mu_d = mu_d + delta_mu_d
        L = L + delta_l

    # Reconstruct Sigma from L
    Sigma = L @ L.T
    
    # Step 2: Partition Sigma
    # Dimensions: p=3, t=1, d=3. Total 7.
    Sigma_p = Sigma[:3, :3]
    Sigma_pt = Sigma[:3, 3:4]
    Sigma_pd = Sigma[:3, 4:7]
    Sigma_t = Sigma[3:4, 3:4]
    Sigma_td = Sigma[4:7, 3:4]
    Sigma_d = Sigma[4:7, 4:7]
    
    # Construct blocks
    Sigma_p_td = np.hstack([Sigma_pt, Sigma_pd]) # (3, 4)
    Sigma_td_t = Sigma_t # (1, 1)
    Sigma_td_d = Sigma_td # (3, 1)
    Sigma_d_d = Sigma_d # (3, 3)
    
    # Sigma_(t,d) is (4,4)
    Sigma_td_block = np.block([
        [Sigma_td_t, Sigma_td_d.T],
        [Sigma_td_d, Sigma_d_d]
    ])
    
    # Step 3: Compute Conditional Mean
    diff = np.array([t - mu_t, *(d - mu_d)]) # (4,)
    x = np.linalg.solve(Sigma_td_block, diff.reshape(-1, 1))
    correction = Sigma_p_td @ x # (3, 1)
    mu_cond = mu_p + correction.flatten()
    
    # Step 4: Compute Conditional Covariance
    inv_Sigma_td = np.linalg.inv(Sigma_td_block)
    Sigma_cond = Sigma_p - (Sigma_p_td @ inv_Sigma_td @ Sigma_p_td.T)
    
    # Step 5: Compute Modulation Factors
    diff_t = t - mu_t
    f_temp = np.exp(-0.5 * lambda_t * (diff_t**2) * (1.0 / Sigma_t[0,0]))
    
    diff_d = d - mu_d
    inv_Sigma_d = np.linalg.inv(Sigma_d)
    quadratic_form = diff_d @ inv_Sigma_d @ diff_d
    f_dir = np.exp(-0.5 * lambda_d * quadratic_form)
    
    # Step 6: Conditional Opacity
    alpha_cond = alpha * f_temp * f_dir
    
    # Step 7: Extract S and R (Optional)
    U, D, Vt = np.linalg.svd(Sigma_cond)
    S = D
    R = U
    if np.linalg.det(R) < 0:
        R[:, 0] *= -1
        
    return mu_cond, Sigma_cond, alpha_cond, S, R
```

**Parameters:**
*   $\lambda_t$: Positive scalar controlling temporal modulation sensitivity [S1].
*   $\lambda_d$: Positive scalar controlling directional modulation sensitivity [S1].
*   $\phi_p, \phi_t, \phi_d, \phi_l$: Refinement functions (typically small MLPs) mapping feature $f$ to parameter deltas [S1].

**Edge Cases and Pitfalls:**
*   **Singularity:** $\Sigma_{(t,d)}$ or $\Sigma_d$ may be singular if correlations are perfect or data is insufficient. Use pseudo-inverse or add small jitter to diagonal if inversion fails [G].
*   **Degenerate Input:** If $t$ or $d$ are NaN, the exponential terms become NaN. Guard against invalid inputs [G].
*   **Precision:** The exponential decay can underflow to zero for large deviations, effectively culling the Gaussian. This is intended behavior but may cause artifacts if $\lambda$ is too high [S1].

## Subtopics

### Slice 7DGS to Conditional 3DGS
This is the core algorithmic procedure described in the main method [S1]. It refers to the specific implementation of partitioning the 7D covariance and computing the conditional 3D Gaussian statistics [S1]. It is chosen when one needs to render a specific frame at time $t$ from direction $d$ using the 7DGS representation [S1]. It is distinct from static slicing because it includes the adaptive refinement step before conditioning [S1].

## Operating characteristics
*   **Time Complexity:** Dominated by matrix inversion of $4 \times 4$ ($\Sigma_{(t,d)}$) and $3 \times 3$ ($\Sigma_d$) matrices, which is $O(1)$ constant time per Gaussian [G]. The refinement step involves MLP evaluation, also $O(1)$ [G].
*   **Memory Complexity:** Requires storing the full 7D parameters ($\mu_p, \mu_t, \mu_d, \Sigma$) plus refinement network weights [S1].
*   **Strengths:** Efficiently integrates temporal and view-dependent effects into standard 3D rendering pipelines without modifying the core rasterizer [S1].
*   **Limitations:** The intrinsic shape of the Gaussian (covariance) remains static over time if refinement is not used, which may hinder representing complex non-rigid deformations [S1].
*   **When to use:** When rendering dynamic scenes with view-dependent appearance using Gaussian Splatting [S1].

## Background
The Conditional Slicing Mechanism was introduced in "7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting" [S1]. It conditions a 7-dimensional Gaussian on observed time and view direction to yield a conditional 3D Gaussian governing the spatial component [S1]. The method leverages standard properties of multivariate Gaussian distributions to condition the spatial component on observed time and view direction [S1]. It builds on the Gaussian Geometry Parameters Prediction method [8442-gaussian-geometry-parameters-prediction.md] by providing a mechanism to interpret the 7D parameters during rendering [S1].

## Sources
[S1] 7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting, https://arxiv.org/abs/2503.07946v1, fetched 2026-10-26
