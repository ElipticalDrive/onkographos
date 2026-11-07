# Gaussian Splitting and Pruning

Also called: Deterministic splitting, Splitting Gaussians

## Overview
Gaussian Splitting and Pruning is a deterministic density control method for 3D Gaussian Splatting that dynamically adjusts the number of Gaussian primitives by splitting those with high reconstruction responsibility and removing those with minimal contribution [S1, unofficial]. It differs from heuristic-based densification (e.g., [3D Gaussian Splatting Adaptive Density Control](3526-3d-gaussian-splatting-adaptive-density-control.md)) by relying on statistical criteria derived from per-ray loss and weights, allowing for iterative, detail-oriented refinement after initial convergence [S1, unofficial].

## How it works
**Inputs:**
- A converged set of 3D Gaussians $\mathcal{G} = \{g_i\}$, where each $g_i$ has mean $\mu_i$, covariance $\Sigma_i$ (eigenvalues $\sigma_{1,i}, \sigma_{2,i}, \sigma_{3,i}$ and eigenvectors $\vec{v}_{1,i}, \vec{v}_{2,i}, \vec{v}_{3,i}$), opacity weight $\lambda_i$, and color $c_i$.
- A random batch of rays $\mathcal{R}_{batch}$ (typically ~5% of the training dataset).
- Ground truth colors $C(\mathbf{r})$ for these rays.

**Outputs:**
- Updated set of Gaussians $\mathcal{G}'$ with modified count, positions, covariances, weights, and colors.

**Data Structures:**
- Gaussian attributes are stored in flat arrays or structures of arrays (SoA) for GPU efficiency:
  - `means`: $N \times 3$ float32
  - `covariances`: $N \times 3$ float32 (eigenvalues) and $N \times 3$ float32 (eigenvectors)
  - `log_opacities`: $N$ float32 (stored as $\log(\lambda)$)
  - `sh_coeffs`: $N \times K$ float32 (Spherical Harmonic coefficients)
  - `colors`: $N \times 3$ float32 (stored as inverse-sigmoid values $\sigma^{-1}(c)$)

**Steps:**

1. **Pruning Phase:**
   - Compute the mean $\mu_\lambda$ and standard deviation $\sigma_\lambda$ of the opacity weights $\lambda_i$ across all Gaussians.
   - Remove any Gaussian $g_i$ where:
     $$ \lambda_i \leq \mu_\lambda - z_\lambda \sigma_\lambda $$
     where $z_\lambda$ is a z-score threshold, typically 2 [S1, unofficial].

2. **Loss Estimation:**
   - Render the batch of rays $\mathcal{R}_{batch}$ using the current Gaussian set.
   - Compute the per-ray loss $L_j$ for each ray $\mathbf{r}_j \in \mathcal{R}_{batch}$ (e.g., MSE against ground truth).
   - For each ray, retrieve the set of computed weights $w_{i,j}$ contributed by each Gaussian $g_i$ to that ray.
   - Estimate the average loss associated with each Gaussian $g_i$ over the batch of $M$ rays:
     $$ \bar{l}_i = \frac{1}{M} \sum_{j=1}^M L_j \cdot w_{i,j} $$
   - Compute the mean $\mu_{\bar{l}}$ and standard deviation $\sigma_{\bar{l}}$ of these average losses $\bar{l}_i$ across all Gaussians.

3. **Splitting Phase:**
   - Identify Gaussians to split where:
     $$ \bar{l}_i \geq \mu_{\bar{l}} + z_{\bar{l}} \sigma_{\bar{l}} $$
     where $z_{\bar{l}}$ is a z-score threshold, typically 1 [S1, unofficial].
   - For each Gaussian $g_i$ selected for splitting:
     - Let $\mu$ be the current mean, $\sigma_1$ be the dominant eigenvalue, and $\vec{v}_1$ be the dominant eigenvector of $\Sigma_i$.
     - Create two new Gaussians, $g_a$ and $g_b$.
     - **New Means:** Shift the mean in opposite directions along the dominant eigenvector:
       $$ \mu_{a,b} = \mu \pm \sigma_1 \vec{v}_1 \sqrt{\frac{2}{\pi}} $$
       The scaling factor $\sqrt{\frac{2}{\pi}} \approx 0.8$ is derived from the half-normal distribution [S1, unofficial].
     - **New Covariances:** Set identical covariance matrices for $g_a$ and $g_b$. Reconstruct the covariance from the initial eigenvectors but scale the dominant eigenvalue:
       $$ \sigma_{1_{a,b}} = \sigma_1 \sqrt{1 - \frac{2}{\pi}} $$
       The scaling factor $\sqrt{1 - \frac{2}{\pi}} \approx 0.6$ is derived from the half-normal distribution [S1, unofficial]. The other eigenvalues $\sigma_{2,i}, \sigma_{3,i}$ remain unchanged.
     - **New Opacity:** Replicate the initial weight $\lambda_i$ with added noise in log-space:
       $$ \log(\lambda_{a,b}) = \log(\lambda_i) + \mathcal{N}(0, \epsilon_w) $$
       where $\epsilon_w = 0.1$ [S1, unofficial].
     - **New Color:** Replicate the initial color $c_i$ with added noise in inverse-sigmoid space:
       $$ \sigma^{-1}(c_{a,b}) = \sigma^{-1}(c_i) + \mathcal{N}(0, \epsilon_c) $$
       where $\epsilon_c = 0.1$ [S1, unofficial].
     - Remove the original Gaussian $g_i$ and add $g_a$ and $g_b$ to the set.

4. **Optimization:**
   - Repeat the standard differentiable rendering optimization loop (forward pass, loss computation, backpropagation) with the updated Gaussian set [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def gaussian_splitting_and_pruning(gaussians, rays, gt_colors, z_lambda=2.0, z_l_bar=1.0, epsilon_w=0.1, epsilon_c=0.1):
    """
    gaussians: dict or object containing arrays:
        means: (N, 3)
        eigenvalues: (N, 3)
        eigenvectors: (N, 3, 3)
        log_opacities: (N,)
        inv_sigmoid_colors: (N, 3)
    rays: list of ray objects or tuples (origin, direction)
    gt_colors: (M, 3) ground truth colors for the rays
    """
    N = gaussians['means'].shape[0]
    
    # Step 1: Pruning
    lambdas = np.exp(gaussians['log_opacities'])
    mu_lambda = np.mean(lambdas)
    sigma_lambda = np.std(lambdas)
    prune_mask = lambdas <= (mu_lambda - z_lambda * sigma_lambda)
    
    # Remove pruned Gaussians
    keep_mask = ~prune_mask
    gaussians['means'] = gaussians['means'][keep_mask]
    gaussians['eigenvalues'] = gaussians['eigenvalues'][keep_mask]
    gaussians['eigenvectors'] = gaussians['eigenvectors'][keep_mask]
    gaussians['log_opacities'] = gaussians['log_opacities'][keep_mask]
    gaussians['inv_sigmoid_colors'] = gaussians['inv_sigmoid_colors'][keep_mask]
    
    # Re-index for new Gaussians
    current_gaussians = gaussians
    
    # Step 2: Loss Estimation
    # Render rays and get per-ray loss and per-Gaussian weights
    # Note: This requires a differentiable renderer. 
    # For the purpose of this algorithm description, we assume these values are computed.
    # L_j: per-ray loss (M,)
    # w_i_j: weight of Gaussian i on ray j (N, M)
    
    # Estimate average loss per Gaussian
    # l_bar_i = (1/M) * sum_j(L_j * w_i_j)
    l_bar = np.mean(current_gaussians['weights'] * rays['losses'][np.newaxis, :], axis=1)
    
    mu_l_bar = np.mean(l_bar)
    sigma_l_bar = np.std(l_bar)
    
    # Step 3: Splitting
    split_mask = l_bar >= (mu_l_bar + z_l_bar * sigma_l_bar)
    gaussians_to_split = current_gaussians['means'][split_mask]
    eigenvalues_to_split = current_gaussians['eigenvalues'][split_mask]
    eigenvectors_to_split = current_gaussians['eigenvectors'][split_mask]
    log_opacities_to_split = current_gaussians['log_opacities'][split_mask]
    inv_sigmoid_colors_to_split = current_gaussians['inv_sigmoid_colors'][split_mask]
    
    new_means = []
    new_eigenvalues = []
    new_eigenvectors = []
    new_log_opacities = []
    new_inv_sigmoid_colors = []
    
    sqrt_2_pi = np.sqrt(2 / np.pi)
    sqrt_1_minus_2_pi = np.sqrt(1 - 2 / np.pi)
    
    for i in range(len(gaussians_to_split)):
        mu = gaussians_to_split[i]
        eigs = eigenvalues_to_split[i]
        vecs = eigenvectors_to_split[i]
        log_lambda = log_opacities_to_split[i]
        inv_c = inv_sigmoid_colors_to_split[i]
        
        # Dominant eigenvalue and eigenvector
        sigma_1 = eigs[0] # Assuming sorted descending
        v_1 = vecs[:, 0]
        
        # New means
        shift = sigma_1 * v_1 * sqrt_2_pi
        mu_a = mu + shift
        mu_b = mu - shift
        
        # New eigenvalues
        sigma_1_new = sigma_1 * sqrt_1_minus_2_pi
        eigs_a = np.array([sigma_1_new, eigs[1], eigs[2]])
        eigs_b = np.array([sigma_1_new, eigs[1], eigs[2]])
        
        # New eigenvectors (same)
        vecs_a = vecs
        vecs_b = vecs
        
        # New log opacities with noise
        noise_w = np.random.normal(0, epsilon_w)
        log_lambda_a = log_lambda + noise_w
        log_lambda_b = log_lambda + noise_w
        
        # New colors with noise
        noise_c = np.random.normal(0, epsilon_c, size=3)
        inv_c_a = inv_c + noise_c
        inv_c_b = inv_c + noise_c
        
        new_means.append(mu_a)
        new_means.append(mu_b)
        new_eigenvalues.append(eigs_a)
        new_eigenvalues.append(eigs_b)
        new_eigenvectors.append(vecs_a)
        new_eigenvectors.append(vecs_b)
        new_log_opacities.append(log_lambda_a)
        new_log_opacities.append(log_lambda_b)
        new_inv_sigmoid_colors.append(inv_c_a)
        new_inv_sigmoid_colors.append(inv_c_b)
    
    # Combine kept and new Gaussians
    final_means = np.vstack([current_gaussians['means'][~split_mask], np.array(new_means)])
    final_eigenvalues = np.vstack([current_gaussians['eigenvalues'][~split_mask], np.array(new_eigenvalues)])
    final_eigenvectors = np.vstack([current_gaussians['eigenvectors'][~split_mask], np.array(new_eigenvectors)])
    final_log_opacities = np.concatenate([current_gaussians['log_opacities'][~split_mask], np.array(new_log_opacities)])
    final_inv_sigmoid_colors = np.vstack([current_gaussians['inv_sigmoid_colors'][~split_mask], np.array(new_inv_sigmoid_colors)])
    
    # Update gaussians object
    gaussians['means'] = final_means
    gaussians['eigenvalues'] = final_eigenvalues
    gaussians['eigenvectors'] = final_eigenvectors
    gaussians['log_opacities'] = final_log_opacities
    gaussians['inv_sigmoid_colors'] = final_inv_sigmoid_colors
    
    return gaussians
```

**Edge Cases and Pitfalls:**
- **Precision:** The noise addition ($\epsilon_w, \epsilon_c$) is critical to prevent coupled gradients and ensure the new Gaussians diverge during optimization [S1, unofficial]. Without noise, splitting may not improve reconstruction quality.
- **Boundary Conditions:** The splitting criteria rely on statistical moments. If the distribution of losses or weights is highly skewed or has outliers, the z-score thresholds may need adjustment.
- **Degenerate Input:** If all Gaussians have similar weights or losses, no splitting or pruning may occur, potentially stalling refinement.

## Subtopics

### [Octree refinement](8792-octree-refinement.md)
Octree refinement is a spatial subdivision technique used to reorder Gaussian splats for progressive rendering, ensuring that sparsely captured areas of a scene are reconstructed earlier than they would be by global contribution metrics alone. It differs from simple global sorting by enforcing a minimum representation per spatial region, thereby improving background quality at the cost of slight foreground degradation.

### [Gaussian Pruning and Sampling](9219-gaussian-pruning-and-sampling.md)
Gaussian Pruning and Sampling is a post-processing or intermediate optimization strategy in 3D Gaussian Splatting that reduces the number of Gaussian primitives by removing those with low contribution to the final image. It addresses the inherent redundancy in dense Gaussian representations initialized from point clouds. Unlike [Gaussian Splitting and Pruning](3304-gaussian-splitting-and-pruning.md), which dynamically adjusts density during training based on gradient-based splitting criteria, this method relies on accumulated rendering weights to identify and discard insignificant Gaussians. It is often combined with sampling techniques to mitigate the loss of geometric detail.

### [Blur Split](9237-blur-split.md)
Blur Split is a densification technique for 3D Gaussian Splatting that addresses "under-reconstruction" and blur artifacts in regions with smooth color transitions where gradient-based methods fail. It identifies Gaussians with excessively large maximum contribution areas in screen space and splits them to improve local detail. Unlike standard pruning which removes large Gaussians, Blur Split actively refines them to resolve blurry regions.

## Operating characteristics
- **Time Complexity:** The pruning and splitting steps are $O(N)$, where $N$ is the number of Gaussians. The rendering of the ray batch is $O(M \cdot N)$, where $M$ is the number of rays in the batch. Since this is done periodically, the overhead is manageable.
- **Memory Complexity:** $O(N)$ for storing Gaussian attributes. Splitting increases $N$, so memory usage grows over time.
- **Measured Performance:** The method allows for iterative detail refinement, improving reconstruction quality in complex areas [S1, unofficial].
- **Strengths:** Deterministic and statistically grounded; avoids heuristic tuning; effective at resolving under-reconstructed areas.
- **Limitations:** Requires a batch of rays and loss computation; may increase the number of Gaussians significantly, impacting rendering speed if not pruned effectively.
- **When to use:** When initial convergence leaves visible artifacts or lack of detail in specific regions, and when a more principled approach than heuristic densification is desired.

## Background
Originated in the paper "Flexible Techniques for Differentiable Rendering with 3D Gaussians" (2023) [S1, unofficial]. It was developed as an alternative to the heuristic merging and splitting techniques proposed in the original 3D Gaussian Splatting paper [33], aiming for a more deterministic and statistically robust method to modify the number of Gaussians in the reconstruction.

## Sources
[S1] Flexible Techniques for Differentiable Rendering with 3D Gaussians, https://doi.org/10.48550/arxiv.2308.14737, fetched 2026-10-26 (unofficial)
