# Variational Bayes Gaussian Splatting


## Overview
Variational Bayes Gaussian Splatting (VBGS) is a training method for 3D Gaussian Splatting that frames optimization as variational inference over the parameters of a Gaussian Mixture Model (GMM), rather than using gradient descent through a differentiable renderer [S1, unofficial]. By leveraging the conjugacy of multivariate Gaussians, VBGS derives closed-form update rules that enable continual learning from sequential data without catastrophic forgetting or the need for replay buffers [S1, unofficial]. Unlike [3D Gaussian Splatting](9128-3d-gaussian-splatting.md) which optimizes maximum likelihood estimates via backpropagation, VBGS maintains a distribution over parameters, making it distinct from sibling methods like [DeSplat](12925-desplat.md) and [SOGS](13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md) that rely on gradient-based or second-order optimization techniques.

## How it works
**Inputs and Outputs:**
- **Inputs:** A sequence of observed data points $D = \{(s_n, c_n)\}_{n=1}^N$, where $s_n$ is the spatial coordinate (2D for images, 3D for point clouds) and $c_n$ is the color (RGB). For 3D rendering, camera parameters are required to project Gaussians to the image plane.
- **Outputs:** The variational posterior parameters $q(\mu_s, \Sigma_s, \mu_c, \Sigma_c, \pi)$ for $K$ Gaussian components, which define the scene representation.

**Data Structures:**
Each of the $K$ components stores 29 parameters representing the Natural-Inverse-Wishart (NIW) likelihood for color and position, and the Dirichlet prior for component assignments [S1, unofficial].
- **Spatial Parameters ($q(\mu_{k,s}, \Sigma_{k,s})$):** NIW distribution defined by natural parameters $\eta_{0,s}$ and $\nu_{0,s}$.
  - $\eta_{0,s} = (\kappa_{0,s} \cdot m_{0,s}, V_{0,s} + \kappa_{0,s} \cdot m_{0,s} \cdot m_{0,s}^T)$
  - $\nu_{0,s} = (\kappa_{0,s}, n_{0,s} + D_s + 1)$
  - Where $m_{0,s}$ is the mean, $\kappa_{0,s}$ is the concentration, $n_{0,s}$ is degrees of freedom, $V_{0,s}$ is the inverse scale matrix, and $D_s$ is dimensionality (2 or 3).
- **Color Parameters ($q(\mu_{k,c}, \Sigma_{k,c})$):**
  - Mean $q(\mu_{k,c})$ is Normal with parameters derived from NIW natural parameters $\eta_{0,c} = (\kappa_{0,c} \cdot m_{0,c}, V_{0,c} + \kappa_{0,c} \cdot m_{0,c} \cdot m_{0,c}^T)$ and $\nu_{0,c} = (\kappa_{0,c}, n_{0,c} + D_c + 1)$.
  - Covariance $q(\Sigma_{k,c})$ is a Delta distribution $\delta(\epsilon I)$, where $\epsilon$ is a fixed hyperparameter ensuring components commit to a specific color [S1, unofficial]. Consequently, $n_{k,c}$ and $V_{k,c}$ are kept fixed.
- **Assignment Parameters ($q(\pi)$):** Dirichlet distribution with natural parameter $\eta_{0,z} = \alpha_0$.
- **Assignments ($q(z_n)$):** Categorical distribution with parameters $\gamma_n$.

**Algorithm Steps:**
The training uses Coordinate Ascent Variational Inference (CAVI) to maximize the Evidence Lower Bound (ELBO) [S1, unofficial].

1. **Initialization:**
   - Initialize prior natural parameters $\eta_0, \nu_0$ for all $K$ components.
   - If using "Data Init", initialize means based on sampled data points; otherwise, use random initialization.
   - Set $\Sigma_{k,c}$ covariance to $\epsilon I$.

2. **E-Step (Compute Assignments):**
   For each data point $n$ and component $k$, compute the unnormalized log-assignment $\hat{\gamma}_{n,k}$:
   $$ \log \hat{\gamma}_{n,k} = \mathbb{E}_{q(\mu_{k,s}, \Sigma_{k,s})}[\log p(s_n | \mu_{k,s}, \Sigma_{k,s})] + \mathbb{E}_{q(\mu_{k,c}, \Sigma_{k,c})}[\log p(c_n | \mu_{k,c}, \Sigma_{k,c})] + \mathbb{E}_{q(\pi)}[\log p(z_n=k)] $$
   Normalize to obtain probabilities:
   $$ \gamma_{n,k} = \frac{\hat{\gamma}_{n,k}}{\sum_{j=1}^K \hat{\gamma}_{n,j}} $$
   *Note:* The expectations for the MVN log-likelihoods under NIW posteriors are computed using standard variational inference identities.

3. **M-Step (Parameter Updates):**
   Update the natural parameters of the posteriors using the sufficient statistics of the data weighted by assignments $\gamma_{n,k}$ [S1, unofficial]:
   $$ \eta_{k} = \eta_{0,k} + \sum_{n=1}^N \gamma_{n,k} T(x_n) $$
   $$ \nu_{k} = \nu_{0,k} + \sum_{n=1}^N \gamma_{n,k} $$
   Where $T(x_n)$ is the sufficient statistic for the data point (e.g., $s_n$ and $s_n s_n^T$ for spatial, $c_n$ and $c_n c_n^T$ for color).
   - For spatial: $\eta_{k,s} \leftarrow \eta_{0,s} + \sum_n \gamma_{n,k} (s_n, s_n s_n^T)$
   - For color: $\eta_{k,c} \leftarrow \eta_{0,c} + \sum_n \gamma_{n,k} (c_n, c_n c_n^T)$
   - For assignments: $\alpha_k \leftarrow \alpha_{0,k} + \sum_n \gamma_{n,k}$

4. **Continual Learning / Reassignment (Optional):**
   In streaming settings, unused components (where $\alpha_k$ equals prior $\alpha_0$) may be reassigned to improve coverage [S1, unofficial].
   - Identify unused components.
   - Sample $n$ components (e.g., 5% of unused) proportional to negative ELBO (poor fit).
   - Replace their means $\mu_{k,s}$ and $\mu_{k,c}$ with the location and color of sampled data points.
   - Reset their natural parameters to priors and run CAVI.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def vbgs_update(data_points, K, priors, epsilon):
    """
    data_points: list of (s, c) tuples, where s is (D_s,) and c is (3,)
    K: number of components
    priors: dict containing initial natural parameters for spatial, color, and assignment
    epsilon: fixed covariance scale for color
    """
    N = len(data_points)
    
    # Initialize variational parameters from priors
    # eta_s, nu_s for spatial NIW
    # eta_c, nu_c for color NIW (covariance fixed)
    # alpha for Dirichlet assignment
    
    eta_s = np.copy(priors['eta_s']) # Shape (K, 2*D_s) or similar packed form
    nu_s = np.copy(priors['nu_s'])   # Shape (K, 2)
    
    eta_c = np.copy(priors['eta_c'])
    nu_c = np.copy(priors['nu_c'])
    
    alpha = np.copy(priors['alpha']) # Shape (K,)
    
    # E-Step: Compute assignments
    # gamma[n, k]
    gamma = np.zeros((N, K))
    
    for n in range(N):
        s_n, c_n = data_points[n]
        log_gamma = np.zeros(K)
        
        for k in range(K):
            # Compute expected log likelihood for spatial
            # E[log N(s_n | mu, Sigma)] under NIW(q(mu, Sigma))
            # This involves terms with trace(V^{-1} * (s_n - m)(s_n - m)^T) etc.
            # Simplified representation:
            log_p_s = expected_log_mvn_spatial(s_n, eta_s[k], nu_s[k])
            
            # Compute expected log likelihood for color
            # Covariance is fixed Delta(epsilon I), so it's a standard MVN log prob
            # with fixed covariance epsilon*I and mean derived from eta_c, nu_c
            mu_c_k = get_mean_from_niw(eta_c[k], nu_c[k])
            log_p_c = log_mvn(c_n, mu_c_k, epsilon * np.eye(3))
            
            # Expected log prior for assignment
            log_p_z = np.log(alpha[k] + 1e-10) # Approximation or exact Dirichlet expectation
            
            log_gamma[k] = log_p_s + log_p_c + log_p_z
            
        # Normalize
        log_gamma -= np.log(np.sum(np.exp(log_gamma)))
        gamma[n, :] = np.exp(log_gamma)
        
    # M-Step: Update parameters
    # Sufficient statistics
    # T_s = [s_n, s_n @ s_n.T]
    # T_c = [c_n, c_n @ c_n.T]
    
    for k in range(K):
        # Spatial Update
        sum_gamma_s = np.sum(gamma[:, k])
        sum_gamma_s_s = np.zeros((D_s, D_s))
        sum_gamma_s_vec = np.zeros(D_s)
        
        for n in range(N):
            w = gamma[n, k]
            s_n = data_points[n][0]
            sum_gamma_s_vec += w * s_n
            sum_gamma_s_s += w * np.outer(s_n, s_n)
            
        # Update natural parameters
        # eta_s = (kappa*m, V + kappa*m*m^T)
        # nu_s = (kappa, n + D + 1)
        # We accumulate sums to prior
        # Note: Direct update of natural parameters requires careful inversion to get m, V if needed for next step expectations
        # However, CAVI updates natural parameters directly.
        
        # Prior natural params
        eta_s_0_k = priors['eta_s'][k]
        nu_s_0_k = priors['nu_s'][k]
        
        # Add sufficient stats weighted by gamma
        # eta_s has two parts: kappa*m and V + kappa*m*m^T
        # nu_s has two parts: kappa and n + D + 1
        
        # Update kappa and n (from nu_s)
        # nu_s[0] is kappa, nu_s[1] is n + D + 1
        new_kappa_s = eta_s_0_k[0] / get_m_from_eta(eta_s_0_k) # This is complex, usually we track m, kappa, V, n directly
        # Simplified: The source says eta_k = eta_0 + sum gamma T(x)
        # We must implement the exact vector addition for the natural parameter space.
        
        # For implementation, it is easier to maintain m, kappa, V, n and convert to eta for storage if needed,
        # but the source specifies natural parameter updates.
        
        # Let's assume we maintain the sufficient statistics sums directly in the natural parameter space
        # eta_s_k = eta_s_0_k + [sum_gamma_s_vec, sum_gamma_s_s]
        # nu_s_k = nu_s_0_k + [sum_gamma_s, sum_gamma_s] # nu_s[1] is n+D+1, so we add sum_gamma to n part?
        # nu_s = (kappa, n + D + 1). The sum of 1s is sum_gamma. So nu_s[1] += sum_gamma.
        
        # Correct update for nu_s:
        nu_s[k, 0] = nu_s_0_k[0] # kappa is part of eta usually? No, nu_s = (kappa, n+D+1) in source notation?
        # Source: nu0,s = (kappa0,s, n0,s + Ds + 1). eta0,s = (kappa0,s * m0,s, V0,s + kappa0,s * m0,s * m0,s^T)
        # So kappa is in BOTH eta and nu? Yes, for NIW.
        
        # Update nu_s
        nu_s[k, 0] += np.sum(gamma[:, k]) # Add to kappa? No, nu_s[0] is kappa.
        # Wait, source eq 22: nu_k = nu_0 + sum gamma.
        # So nu_s[k, 0] += sum_gamma_s
        # nu_s[k, 1] += sum_gamma_s
        
        # Update eta_s
        # eta_s[0] is kappa*m. New kappa is nu_s[k,0]. New m is sum_gamma_s_vec / sum_gamma_s.
        # So new eta_s[0] = nu_s[k,0] * (sum_gamma_s_vec / sum_gamma_s)
        # eta_s[1] is V + kappa*m*m^T.
        # This requires converting back and forth or maintaining sufficient stats.
        
        # To avoid ambiguity, we store the accumulated sufficient statistics in the natural parameter form
        # as defined by the source:
        eta_s[k, 0] = eta_s_0_k[0] + sum_gamma_s_vec
        eta_s[k, 1] = eta_s_0_k[1] + sum_gamma_s_s
        
        # Color Update
        # Similar to spatial, but covariance is fixed.
        sum_gamma_c = np.sum(gamma[:, k])
        sum_gamma_c_c = np.zeros((3, 3))
        sum_gamma_c_vec = np.zeros(3)
        
        for n in range(N):
            w = gamma[n, k]
            c_n = data_points[n][1]
            sum_gamma_c_vec += w * c_n
            sum_gamma_c_c += w * np.outer(c_n, c_n)
            
        eta_c[k, 0] = priors['eta_c'][k, 0] + sum_gamma_c_vec
        eta_c[k, 1] = priors['eta_c'][k, 1] + sum_gamma_c_c
        
        # Assignment Update
        alpha[k] = priors['alpha'][k] + sum_gamma_c # sum_gamma is same for all k per n, but summed over n
        
    return eta_s, nu_s, eta_c, nu_c, alpha

def expected_log_mvn_spatial(s, eta, nu):
    """
    Computes E_{q(mu, Sigma)}[log N(s | mu, Sigma)] for NIW posterior.
    q(mu, Sigma) ~ NIW(m, kappa, V, n)
    """
    # Extract parameters from natural params
    # eta = (kappa*m, V + kappa*m*m^T)
    # nu = (kappa, n + D + 1)
    
    kappa = nu[0]
    n_plus_D_plus_1 = nu[1]
    n = n_plus_D_plus_1 - s.shape[0] - 1
    
    m = eta[0] / kappa
    V_plus_kmm = eta[1]
    V = V_plus_kmm - np.outer(m, m) * kappa
    
    D = s.shape[0]
    
    # Log prob expectation for NIW:
    # E[log p] = -D/2 log(2pi) - 1/2 log|Sigma| - 1/2 (s-m)^T Sigma^{-1} (s-m)
    # Expectations:
    # E[log|Sigma|] = const + sum psi((n+1-i)/2)
    # E[Sigma^{-1}] = n * V^{-1}
    
    # This is a complex formula. For implementation, use standard NIW log-likelihood expectation.
    # Not provided in closed form in source, but implied by "Eq[log p]".
    # Placeholder for standard NIW expectation calculation.
    pass
```

**Parameters:**
- $\epsilon$: Hyperparameter for fixed color covariance (Delta distribution). Ensures color commitment [S1, unofficial].
- $K$: Number of mixture components (e.g., 10K, 100K) [S1, unofficial].
- Reassign fraction: Fraction of unused components to reassign (e.g., 0.01 for Replica, 0.05 for Habitat) [S1, unofficial].
- Learning rate: Not used in VBGS (closed-form), but used for gradient baseline comparison (0.1) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Intractable Posterior:** Exact inference is intractable due to component interactions; mean-field approximation is required [S1, unofficial].
- **Color Blending:** Without fixing color covariance, components may blend colors. Solved by using a Delta distribution for $q(\Sigma_{k,c})$ [S1, unofficial].
- **Alpha Blending:** VBGS does not optimize for image reconstruction loss, so standard alpha blending is not sensible. All Gaussians are set to opaque (alpha=1) [S1, unofficial].
- **Memory:** Memory usage is 2x the gradient baseline due to storing distribution parameters (29 vs 14 per component) [S1, unofficial].
- **Large Scale Data:** If data does not fit in memory, batched processing is required, which reduces the speed advantage over gradient methods [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Model assignment (E-step) is $O(nm)$ where $n$ is data points and $m$ is components. Parameter update (M-step) is $O(n)$ [S1, unofficial].
- **Memory Complexity:** Increases by a factor of 2 compared to the gradient baseline (3DGS) for the same number of Gaussians [S1, unofficial].
- **Performance:** On Tiny ImageNet, VBGS wall-clock time is $0.03 \pm 0.03$ s vs Gradient $0.05 \pm 0.02$ s (RTX 4090) [S1, unofficial]. In 3D continual learning (Blender), VBGS (Random Init) achieves $11.19 \pm 3.53$ dB error vs Gradient $21.26 \pm 1.76$ dB [S1, unofficial].
- **Strengths:** Enables continual learning without catastrophic forgetting; no replay buffers needed; single update step is faster than accumulated gradient steps when data fits in memory [S1, unofficial].
- **Limitations:** Requires RGBD data (not just RGB); memory intensive; speed advantage lost with batched processing for large datasets [S1, unofficial].

## Background
VBGS was proposed to address the limitation of 3D Gaussian Splatting (3DGS) which relies on backpropagation and struggles with catastrophic forgetting in continual learning scenarios [S1, unofficial]. It frames Gaussian splatting training as variational inference, leveraging the conjugacy of multivariate Gaussians to derive closed-form updates [S1, unofficial]. The method is benchmarked on TinyImageNet, Blender 3D models, and Habitat scenes [S1, unofficial].

## Sources
[S1] Variational Bayes Gaussian Splatting, https://arxiv.org/abs/2410.03592v2, fetched 2026-10-26 (unofficial)
