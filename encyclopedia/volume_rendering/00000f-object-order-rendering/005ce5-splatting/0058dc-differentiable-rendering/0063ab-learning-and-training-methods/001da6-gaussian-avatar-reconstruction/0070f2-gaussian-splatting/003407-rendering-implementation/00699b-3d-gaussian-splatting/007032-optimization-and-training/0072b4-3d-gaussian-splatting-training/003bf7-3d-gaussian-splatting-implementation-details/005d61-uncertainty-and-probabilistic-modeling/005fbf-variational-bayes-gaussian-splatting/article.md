# Variational Bayes Gaussian Splatting


## Overview
Variational Bayes Gaussian Splatting (VBGS) is a training method for 3D Gaussian Splatting that frames the optimization of Gaussian mixture model parameters as a variational inference problem rather than a gradient-based maximum likelihood estimation [S1, unofficial]. Unlike standard 3D Gaussian Splatting, which relies on backpropagation through a differentiable renderer and suffers from catastrophic forgetting in continual learning settings, VBGS derives closed-form update rules using Coordinate Ascent Variational Inference (CAVI) [S1, unofficial]. This allows for efficient, continual learning from sequentially streamed 2D and 3D data without the need for replay buffers [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A stream of observed data points $D = \{(s_n, c_n)\}_{n=1}^N$, where $s_n$ is the spatial coordinate (2D pixel location for images, 3D Cartesian for point clouds) and $c_n$ is the color (RGB). The method requires RGBD data [S1, unofficial].
*   **Outputs:** The variational posterior parameters $q(\mu_s, \Sigma_s, \mu_c, \Sigma_c, \pi)$ for $K$ Gaussian components.

**Data Structures:**
Each Gaussian component $k$ stores 29 parameters representing the Natural-Inverse-Wishart (NIW) likelihood for color and position, and the Dirichlet prior for assignments [S1, unofficial]. This is in contrast to the 14 parameters (mean, scale, rotation, opacity, SH coefficients) used in the gradient baseline [S1, unofficial].

The parameters per component are structured as follows:
1.  **Spatial NIW Parameters ($q(\mu_{k,s}, \Sigma_{k,s})$):**
    *   $\eta_{0,s} = (\kappa_{0,s} \cdot m_{0,s}, V_{0,s} + \kappa_{0,s} \cdot m_{0,s} \cdot m_{0,s}^T)$
    *   $\nu_{0,s} = (\kappa_{0,s}, n_{0,s} + D_s + 1)$
    *   Where $m_{0,s}$ is the mean, $\kappa_{0,s}$ is the concentration, $n_{0,s}$ is degrees of freedom, $V_{0,s}$ is the inverse scale matrix, and $D_s$ is the dimensionality of space ($D_s=2$ or $3$) [S1, unofficial].
2.  **Color NIW Parameters ($q(\mu_{k,c}, \Sigma_{k,c})$):**
    *   $\eta_{0,c} = (\kappa_{0,c} \cdot m_{0,c}, V_{0,c} + \kappa_{0,c} \cdot m_{0,c} \cdot m_{0,c}^T)$
    *   $\nu_{0,c} = (\kappa_{0,c}, n_{0,c} + D_c + 1)$
    *   The covariance $\Sigma_{k,c}$ is fixed using a Delta distribution $q(\Sigma_{k,c}) = \text{Delta}(\epsilon I)$ to prevent color blending [S1, unofficial]. Thus, $n_{k,c}$ and $V_{k,c}$ are kept fixed at prior values [S1, unofficial].
3.  **Dirichlet Parameters ($q(\pi)$):**
    *   $\eta_{0,z} = \alpha$ (concentration parameters) [S1, unofficial].

**Algorithm Steps:**
The training uses Coordinate Ascent Variational Inference (CAVI) to maximize the Evidence Lower Bound (ELBO) [S1, unofficial]:
$$ \mathcal{L} = D_{KL}[q(z, \mu_s, \Sigma_s, \mu_c, \Sigma_c, \pi) || p(z, \mu_s, \Sigma_s, \mu_c, \Sigma_c, \pi | s, c)] $$

The algorithm consists of two main steps repeated for each batch or stream of data:

1.  **Assignment Step (E-step equivalent):**
    Compute the assignment log-probability $\log \gamma_n$ for each data point $n$ to each component $k$:
    $$ \log \hat{\gamma}_n = \sum_{k=1}^K \left( E_{q(\mu_{k,s}, \Sigma_{k,s})}[\log p(s_n | \mu_{k,s}, \Sigma_{k,s})] + E_{q(\mu_{k,c}, \Sigma_{k,c})}[\log p(c_n | \mu_{k,c}, \Sigma_{k,c})] + E_{q(\pi)}[\log p(z_n=k)] \right) $$
    Normalize to get probabilities:
    $$ \gamma_{n,k} = \frac{\exp(\log \hat{\gamma}_{n,k})}{\sum_{j=1}^K \exp(\log \hat{\gamma}_{n,j})} $$
    *Note: The sources define $\log \gamma_n$ generally, but the summation over $k$ is implicit in the normalization step described in the notes [S1, unofficial].*

2.  **Parameter Update Step (M-step equivalent):**
    Update the natural parameters of the variational posteriors using the sufficient statistics of the data weighted by the assignments $\gamma_{n,k}$ [S1, unofficial]:
    $$ \eta_{k} = \eta_{0,k} + \sum_{n=1}^N \gamma_{n,k} T(x_n) $$
    $$ \nu_{k} = \nu_{0,k} + \sum_{n=1}^N \gamma_{n,k} $$
    Where $T(x_n)$ are the sufficient statistics: $T(s_n) = (s_n, s_n s_n^T)$ and $T(c_n) = (c_n, c_n c_n^T)$ [S1, unofficial].

3.  **Continual Learning & Reassignment:**
    *   **Order Invariance:** Assignments $q(z)$ are always computed with respect to the *initial* posterior parameters. This ensures that updates are identical to batch processing and avoids catastrophic forgetting [S1, unofficial].
    *   **Reassignment:** To handle unused components (where concentration parameters $\alpha_k$ equal their prior value), replace the means of $n$ components with the location and color of data points sampled proportional to the negative ELBO. $n$ is a fraction (e.g., 0.01 for Replica, 0.05 for Habitat) of the unused components [S1, unofficial]. After reassignment, run CAVI [S1, unofficial].

**Rendering:**
VBGS does not optimize for image reconstruction loss, so standard alpha blending is not sensible. All Gaussians are set to be opaque by setting their alpha value to 1 [S1, unofficial]. 3D rendering leverages the 3D renderer from Kerbl et al. [2023], projecting 3D Gaussians to the image plane and combining color values via alpha blending along a casted ray, but with fixed opacity [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def vbgs_update(data_points, K, prior_params, reassign_fraction=0.0):
    """
    Perform one CAVI update step for VBGS.
    
    Args:
        data_points: List of (s, c) tuples. s is spatial coord, c is color.
        K: Number of Gaussian components.
        prior_params: Dictionary containing initial natural parameters 
                      (eta_s, nu_s, eta_c, nu_c, alpha) for all K components.
        reassign_fraction: Fraction of unused components to reassign.
        
    Returns:
        Updated variational parameters.
    """
    N = len(data_points)
    
    # 1. Compute Assignments
    # Initialize log probabilities
    log_gamma = np.zeros((N, K))
    
    for k in range(K):
        # Extract current variational parameters for component k
        # Note: In continual learning, these are the INITIAL parameters
        mu_s_k, Sigma_s_k = get_mean_cov_from_niw(prior_params['eta_s'][k], prior_params['nu_s'][k])
        mu_c_k = prior_params['eta_c'][k][0] # Mean of color
        # Sigma_c is fixed Delta distribution, so we use epsilon * I
        # E[log p(c|mu, Sigma)] for Delta distribution simplifies to log p(c|mu, epsilon*I)
        
        # Compute log likelihoods for spatial and color
        # p(s|mu, Sigma) ~ MVN
        # p(c|mu, Sigma) ~ MVN with fixed covariance
        log_p_s = mvn_log_prob(data_points, mu_s_k, Sigma_s_k)
        log_p_c = mvn_log_prob_color(data_points, mu_c_k, epsilon=1e-3) # epsilon from notes
        
        # Log prior for component assignment (Dirichlet)
        log_p_z = np.log(prior_params['alpha'][k])
        
        log_gamma[:, k] = log_p_s + log_p_c + log_p_z
        
    # Normalize
    log_gamma_max = np.max(log_gamma, axis=1, keepdims=True)
    log_gamma -= log_gamma_max # Stability
    gamma_hat = np.exp(log_gamma)
    gamma = gamma_hat / np.sum(gamma_hat, axis=1, keepdims=True)
    
    # 2. Update Parameters
    updated_params = copy.deepcopy(prior_params)
    
    for k in range(K):
        # Sufficient statistics
        # T(s) = (s, s s^T)
        # T(c) = (c, c c^T)
        
        # Update eta_s and nu_s
        # eta_s = (kappa * m, V + kappa * m m^T)
        # nu_s = (kappa, n + D + 1)
        
        # Accumulate weighted stats
        weighted_s = np.sum(gamma[:, k:k+1] * np.array([p[0] for p in data_points]), axis=0)
        weighted_s_outer = np.sum(gamma[:, k:k+1] * np.array([[p[0] @ p[0].T for p in data_points]]), axis=0) # Shape (N, D, D)
        
        # Update natural parameters
        # eta_new = eta_0 + sum(gamma * T(x))
        # nu_new = nu_0 + sum(gamma)
        
        # For NIW, the update is additive on the natural parameters
        # eta_s_k = eta_s_0_k + sum_n gamma_nk * (s_n, s_n s_n^T)
        # nu_s_k = nu_s_0_k + sum_n gamma_nk
        
        # Note: The source defines eta and nu as vectors/tuples. 
        # Implementation requires unpacking eta_s into (kappa*m, V+kappa*m*m^T)
        # and updating them.
        
        # Simplified update logic based on source Eq 21-22:
        # eta_k = eta_0_k + sum(gamma * T(x))
        # nu_k = nu_0_k + sum(gamma)
        
        # We must unpack the NIW natural parameters to update them correctly.
        # eta_s has 2 parts: kappa*m and V+kappa*m*m^T.
        # nu_s has 2 parts: kappa and n+D+1.
        
        # ... (Detailed NIW update logic omitted for brevity, follows standard NIW conjugacy) ...
        
        # Update Color Parameters
        # Sigma_c is fixed, so only mu_c updates.
        # mu_c update depends on weighted sum of c.
        
        # Update Dirichlet Alpha
        # alpha_new = alpha_0 + sum(gamma)
        updated_params['alpha'][k] = prior_params['alpha'][k] + np.sum(gamma[:, k])

    # 3. Reassignment (Continual Learning)
    if reassign_fraction > 0:
        unused_components = find_unused_components(updated_params['alpha'], prior_params['alpha'])
        n_reassign = int(reassign_fraction * len(unused_components))
        
        if n_reassign > 0:
            # Sample data points proportional to negative ELBO
            # ELBO contribution per data point is roughly log_gamma
            # Negative ELBO implies high error -> sample these
            error_scores = -np.mean(log_gamma, axis=1) # Simplified heuristic
            indices = sample_indices(data_points, error_scores, n_reassign)
            
            for i in range(n_reassign):
                comp_idx = unused_components[i]
                point = data_points[indices[i]]
                
                # Reset component to prior, then set mean to data point
                # Or simply replace mean in the variational posterior
                # Source: "replace means of n components ... with location and color"
                updated_params['eta_s'][comp_idx] = set_mean_in_niw_eta(updated_params['eta_s'][comp_idx], point[0])
                updated_params['eta_c'][comp_idx] = set_mean_in_niw_eta(updated_params['eta_c'][comp_idx], point[1])
                
    return updated_params

def mvn_log_prob(data, mu, Sigma):
    # Compute log probability of data points under MVN(mu, Sigma)
    # Returns array of shape (N,)
    pass

def mvn_log_prob_color(data, mu, epsilon):
    # Compute log probability under MVN(mu, epsilon*I)
    pass

def get_mean_cov_from_niw(eta, nu):
    # Unpack NIW natural parameters to mean and covariance
    pass

def find_unused_components(alpha, alpha_prior):
    # Components where alpha == alpha_prior
    pass

def sample_indices(data, scores, n):
    # Sample n indices based on scores
    pass
```

## Operating characteristics
*   **Time Complexity:** The assignment step has complexity $O(nm)$, where $n$ is the number of data points and $m$ is the number of Gaussian components. The parameter update step has complexity $O(n)$ [S1, unofficial].
*   **Memory Complexity:** Memory requirements are increased by a factor of 2 compared to the gradient baseline (3DGS) for the same number of Gaussians, due to storing the distribution parameters (29 per component vs 14) [S1, unofficial].
*   **Performance:** On the Tiny ImageNet validation set, VBGS wall-clock time is $0.03 \pm 0.03$ s compared to Gradient $0.05 \pm 0.02$ s (NVIDIA GeForce RTX 4090) [S1, unofficial]. In 3D continual learning on the Blender dataset, VBGS (Random Init) average reconstruction error is $11.19 \pm 3.53$ dB, while Gradient (Random Init) is $21.26 \pm 1.76$ dB [S1, unofficial].
*   **Strengths:** Enables continual learning without catastrophic forgetting; no replay buffers needed; closed-form updates are faster than accumulated gradient steps when data fits in memory [S1, unofficial].
*   **Limitations:** Relies on RGBD data rather than RGB-only; memory usage is higher; in large-scale 3D datasets, batched processing is necessary, which reduces the speed advantage over gradient methods [S1, unofficial].

## Background
VBGS was proposed to address the limitation of 3D Gaussian Splatting (3DGS), which uses backpropagation through a differentiable renderer and struggles with catastrophic forgetting in continuous data streams [S1, unofficial]. It leverages the conjugacy properties of multivariate Gaussians to derive closed-form variational update rules [S1, unofficial]. The method is benchmarked on 2D image (TinyImageNet) and 3D point cloud (Blender 3D models, Habitat scenes, Replica, TUM) datasets [S1, unofficial].

## Sources
[S1] Variational Bayes Gaussian Splatting, https://arxiv.org/abs/2410.03592v2, fetched 2026-10-26 (unofficial)
