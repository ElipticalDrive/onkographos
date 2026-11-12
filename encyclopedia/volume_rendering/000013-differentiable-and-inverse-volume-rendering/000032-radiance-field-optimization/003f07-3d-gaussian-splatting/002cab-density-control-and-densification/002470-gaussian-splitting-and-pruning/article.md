# Gaussian Splitting and Pruning

Also called: Deterministic splitting, Splitting Gaussians

## Overview
Gaussian Splitting and Pruning is a deterministic density control method for 3D Gaussian Splatting that modifies the number of Gaussian primitives based on statistical criteria derived from reconstruction loss and weight contributions [S1, unofficial]. Unlike the heuristic gradient-based densification used in the base 3D Gaussian Splatting method, this approach uses a statistical analysis of per-ray losses and weights to make deterministic decisions about splitting and pruning [S1, unofficial].

## How it works
**Inputs:**
*   A set of 3D Gaussians $G = \{G_i\}_{i=1}^N$, where each $G_i$ has mean $\mu_i$, covariance $\Sigma_i$, opacity weight $\lambda_i$, and color $c_i$ [S1, unofficial].
*   A random batch of rays $\mathcal{R}$ (typically ~5% of the training dataset) [S1, unofficial].
*   Per-ray loss values $L_j$ for each ray $j$ in the batch [S1, unofficial].
*   Per-ray blending weights $w_{i,j}$ for each Gaussian $i$ contributing to ray $j$ [S1, unofficial].
*   Z-score thresholds: $z_\lambda$ (typically 2) for pruning and $z_{\bar{l}}$ (typically 1) for splitting [S1, unofficial].

**Outputs:**
*   A modified set of Gaussians with updated positions, covariances, weights, and colors, where high-loss Gaussians are split and low-weight Gaussians are removed [S1, unofficial].

**Data Structures:**
*   Gaussian properties are stored in arrays or structures containing:
    *   $\mu_i \in \mathbb{R}^3$: Mean position [S1, unofficial].
    *   $\Sigma_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix, represented by eigenvalues $\sigma_k$ and eigenvectors $\vec{v}_k$ [S1, unofficial].
    *   $\lambda_i \in \mathbb{R}$: Opacity weight, stored as log-weight $\log(\lambda_i)$ for optimization [S1, unofficial].
    *   $c_i \in \mathbb{R}^3$: Color, stored as inverse-sigmoid $\sigma^{-1}(c_i)$ for optimization [S1, unofficial].

**Steps:**

1.  **Compute Weight Statistics:**
    Calculate the mean $\mu_\lambda$ and standard deviation $\sigma_\lambda$ of the opacity weights $\lambda_i$ across all Gaussians [S1, unofficial].

2.  **Prune Low-Contribution Gaussians:**
    Remove any Gaussian $G_i$ if its weight satisfies:
    $$ \lambda_i \leq \mu_\lambda - z_\lambda \sigma_\lambda $$
    where $z_\lambda$ is typically 2 [S1, unofficial].

3.  **Compute Average Loss per Gaussian:**
    For a batch of $M$ rays, estimate the average loss $\bar{l}_i$ associated with each Gaussian $G_i$ by summing the product of the per-ray loss $L_j$ and the Gaussian's weight $w_{i,j}$ for all rays $j$, then dividing by $M$:
    $$ \bar{l}_i = \frac{1}{M} \sum_{j=1}^M L_j \cdot w_{i,j} $$
    [S1, unofficial].

4.  **Compute Loss Statistics:**
    Calculate the mean $\mu_{\bar{l}}$ and standard deviation $\sigma_{\bar{l}}$ of the average losses $\bar{l}_i$ across all Gaussians [S1, unofficial].

5.  **Identify Gaussians for Splitting:**
    Select Gaussians $G_i$ for splitting if their average loss satisfies:
    $$ \bar{l}_i \geq \mu_{\bar{l}} + z_{\bar{l}} \sigma_{\bar{l}} $$
    where $z_{\bar{l}}$ is typically 1 [S1, unofficial].

6.  **Split Selected Gaussians:**
    For each selected Gaussian $G_i$ with mean $\mu$, dominant eigenvalue $\sigma_1$, and dominant eigenvector $\vec{v}_1$ of $\Sigma_i$:
    *   **New Means:** Generate two new means $\mu_a$ and $\mu_b$ by shifting the original mean in opposite directions along the dominant eigenvector:
        $$ \mu_{a,b} = \mu \pm \sigma_1 \vec{v}_1 \sqrt{\frac{2}{\pi}} $$
        The scaling factor $\sqrt{\frac{2}{\pi}} \approx 0.8$ is derived from the half-normal distribution [S1, unofficial].
    *   **New Covariances:** Create two new Gaussians with identical covariance matrices, reconstructed from the original eigenvalues and eigenvectors, but with the dominant eigenvalue scaled:
        $$ \sigma_{1_{a,b}} = \sigma_1 \sqrt{1 - \frac{2}{\pi}} $$
        The scaling factor $\sqrt{1 - \frac{2}{\pi}} \approx 0.6$ is derived from the half-normal distribution [S1, unofficial]. The other eigenvalues remain unchanged.
    *   **New Weights:** Replicate the initial weight $\lambda$ and add noise in the unconstrained parameterization space to avoid coupled gradients:
        $$ \log(\lambda_{a,b}) = \log(\lambda) + \mathcal{N}(0, \epsilon_w) $$
        where $\epsilon_w = 0.1$ [S1, unofficial].
    *   **New Colors:** Replicate the initial color $c$ and add noise in the unconstrained parameterization space:
        $$ \sigma^{-1}(c_{a,b}) = \sigma^{-1}(c) + \mathcal{N}(0, \epsilon_c) $$
        where $\epsilon_c = 0.1$ [S1, unofficial].

7.  **Repeat Optimization:**
    After splitting and pruning, repeat the optimization process (gradient descent) to refine the new Gaussians [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class Gaussian:
    def __init__(self, mu, eigvals, eigvecs, log_lambda, inv_sigmoid_c):
        self.mu = mu
        self.eigvals = eigvals
        self.eigvecs = eigvecs
        self.log_lambda = log_lambda
        self.inv_sigmoid_c = inv_sigmoid_c

def gaussian_splitting_and_pruning(gaussians, rays, z_lambda=2.0, z_l_bar=1.0, epsilon_w=0.1, epsilon_c=0.1):
    """
    Perform deterministic splitting and pruning of 3D Gaussians based on statistical criteria.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
            - mu: np.array (3,) mean position
            - eigvals: np.array (3,) eigenvalues of covariance matrix
            - eigvecs: np.array (3,3) eigenvectors of covariance matrix
            - log_lambda: float log of opacity weight
            - inv_sigmoid_c: np.array (3,) inverse-sigmoid of color
        rays: List of ray objects, each with:
            - loss: float per-ray loss
            - weights: dict mapping gaussian_index to float weight w_i,j
        z_lambda: Z-score threshold for pruning
        z_l_bar: Z-score threshold for splitting
        epsilon_w: Noise scale for log-weights
        epsilon_c: Noise scale for inverse-sigmoid colors
        
    Returns:
        new_gaussians: List of Gaussian objects after splitting and pruning
    """
    n_gaussians = len(gaussians)
    
    # Step 1: Compute Weight Statistics
    # Source uses actual weights lambda for thresholding, not log_lambda
    actual_lambdas = np.exp([g.log_lambda for g in gaussians])
    mu_lambda = np.mean(actual_lambdas)
    sigma_lambda = np.std(actual_lambdas)
    
    # Step 2: Prune Low-Contribution Gaussians
    prune_indices = set()
    for i, lam in enumerate(actual_lambdas):
        if lam <= mu_lambda - z_lambda * sigma_lambda:
            prune_indices.add(i)
            
    # Step 3: Compute Average Loss per Gaussian
    loss_sums = np.zeros(n_gaussians)
    M = len(rays)
    
    for ray in rays:
        ray_loss = ray.loss
        for idx, w in ray.weights.items():
            loss_sums[idx] += ray_loss * w
            
    avg_losses = loss_sums / M
    
    # Step 4: Compute Loss Statistics
    mu_l_bar = np.mean(avg_losses)
    sigma_l_bar = np.std(avg_losses)
    
    # Step 5: Identify Gaussians for Splitting
    split_indices = set()
    for i, l_bar in enumerate(avg_losses):
        if l_bar >= mu_l_bar + z_l_bar * sigma_l_bar:
            split_indices.add(i)
            
    # Step 6: Split Selected Gaussians
    new_gaussians = []
    scale_mean = np.sqrt(2.0 / np.pi)
    scale_cov = np.sqrt(1.0 - 2.0 / np.pi)
    
    for i, g in enumerate(gaussians):
        if i in prune_indices:
            continue
            
        if i in split_indices:
            mu = g.mu
            eigvals = g.eigvals
            eigvecs = g.eigvecs
            
            # Dominant eigenvalue and eigenvector
            dominant_idx = np.argmax(eigvals)
            sigma_1 = eigvals[dominant_idx]
            v_1 = eigvecs[:, dominant_idx]
            
            # New means
            mu_a = mu + sigma_1 * v_1 * scale_mean
            mu_b = mu - sigma_1 * v_1 * scale_mean
            
            # New eigenvalues: dominant scaled, others same
            new_eigvals_a = eigvals.copy()
            new_eigvals_a[dominant_idx] = sigma_1 * scale_cov
            new_eigvals_b = new_eigvals_a.copy()
            
            # New weights and colors with noise in unconstrained space
            log_lambda_a = g.log_lambda + np.random.normal(0, epsilon_w)
            log_lambda_b = g.log_lambda + np.random.normal(0, epsilon_w)
            
            inv_sigmoid_c_a = g.inv_sigmoid_c + np.random.normal(0, epsilon_c, size=3)
            inv_sigmoid_c_b = g.inv_sigmoid_c + np.random.normal(0, epsilon_c, size=3)
            
            g_a = Gaussian(mu=mu_a, eigvals=new_eigvals_a, eigvecs=eigvecs, 
                           log_lambda=log_lambda_a, inv_sigmoid_c=inv_sigmoid_c_a)
            g_b = Gaussian(mu=mu_b, eigvals=new_eigvals_b, eigvecs=eigvecs, 
                           log_lambda=log_lambda_b, inv_sigmoid_c=inv_sigmoid_c_b)
                           
            new_gaussians.append(g_a)
            new_gaussians.append(g_b)
        else:
            new_gaussians.append(g)
            
    return new_gaussians
```

## Subtopics

### [Octree refinement](8792-octree-refinement.md)
Octree refinement is a spatial subdivision technique used to reorder Gaussian splats for progressive rendering, ensuring that sparsely captured areas of a scene are reconstructed earlier than they would be by global contribution metrics alone. It differs from simple global sorting by enforcing a minimum representation per spatial region, thereby improving background quality at the cost of slight foreground degradation.

### [Gaussian Pruning and Sampling](9219-gaussian-pruning-and-sampling.md)
Gaussian Pruning and Sampling is a density control strategy that reduces the number of 3D Gaussian primitives in a radiance field by removing those with minimal contribution to the rendered image. It solves the problem of redundant Gaussian representation and inefficient spatial distribution, which leads to high storage costs and slow optimization times. Unlike opacity-based pruning, which can cause noticeable quality drops or require aggressive thresholds, weight-based pruning evaluates the accumulated blending weights of Gaussians across views, balancing rendering quality with computational efficiency.

### [Blur Split](9237-blur-split.md)
Blur Split is a densification technique for 3D Gaussian Splatting that addresses "under-reconstruction" and blur artifacts in regions with smooth color transitions where gradient-based methods fail. It identifies Gaussians with excessively large maximum contribution areas in screen space and splits them to improve local detail. Unlike standard pruning which removes large Gaussians, Blur Split actively refines them to resolve blurry regions.

### Spatial Splitting with Error Moments
Spatial Splitting with Error Moments is a densification technique that splits Gaussians with error above a threshold $\tau_{space}$ by placing a splitting plane through the Gaussian's center, oriented based on the error moments rather than the Gaussian's longest axis [S2]. For each Gaussian $g$ with error $> \tau_{space}$, the splitting plane normal $n$ is calculated as the normalized product of the inverse error covariance matrix $\mathbf{S}_g^c$ and the first moment vector $\mathbf{r}_g$:
$$ n = \frac{(\mathbf{S}_g^c)^{-1} \mathbf{r}_g}{\left\| (\mathbf{S}_g^c)^{-1} \mathbf{r}_g \right\|} $$
[2]. Unlike FCC* 24 and DDL* 25b which use a plane perpendicular to the Gaussian's longest axis, this method uses the error moments to determine orientation to avoid Gaussians becoming too small too quickly [S2].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
Not given in the sources.

**Scaling:**
The method scales with the number of Gaussians and the size of the ray batch [S1, unofficial]. The deterministic nature allows for iterative refinement, potentially requiring multiple passes as the model converges and new high-loss areas are identified [S1, unofficial].

**Strengths and Limitations:**
Strengths include a deterministic approach that avoids the randomness of heuristic methods, allowing for controlled detail increase [S1, unofficial]. Limitations include the need to compute per-ray losses and weights, which adds computational overhead [S1, unofficial]. The method relies on statistical thresholds ($z_\lambda, z_{\bar{l}}$) which may need tuning for different scenes [S1, unofficial].

**When to use it and when not:**
Use when a deterministic, statistically grounded approach to density control is preferred over heuristic methods [S1, unofficial]. Not given in the sources for when not to use.

## Background
Origin: The method was introduced in "Flexible Techniques for Differentiable Rendering with 3D Gaussians" [S1, unofficial]. It provides an alternative to the heuristic merging and splitting techniques explored in the original 3D Gaussian Splatting paper [S1, unofficial]. The splitting is based on the properties of the half-normal distribution [S1, unofficial].

## Sources
[S1] Flexible Techniques for Differentiable Rendering with 3D Gaussians, https://doi.org/10.48550/arxiv.2308.14737, fetched 2026-10-26 (unofficial)
[S2] Adaptive Spatio‐Temporal 3D Gaussian Splatting for Scenes with Oscillatory Motion, https://doi.org/10.1111/cgf.70410, fetched 2026-10-26
