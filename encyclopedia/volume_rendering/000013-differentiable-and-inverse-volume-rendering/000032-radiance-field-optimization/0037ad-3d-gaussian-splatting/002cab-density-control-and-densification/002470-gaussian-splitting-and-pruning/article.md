# Gaussian Splitting and Pruning

Also called: Deterministic splitting, Splitting Gaussians

## Overview
Gaussian Splitting and Pruning is a deterministic density control method for 3D Gaussian Splatting that modifies the primitive count based on statistical criteria derived from reconstruction loss and opacity weights [S1, unofficial]. Unlike heuristic approaches that rely on gradient magnitude thresholds, this method explicitly identifies Gaussians with minimal contribution for removal and those with excessive reconstruction responsibility for splitting [S1, unofficial]. It operates by computing global statistics over a random batch of rays to determine z-score thresholds for pruning and splitting, allowing for iterative refinement of detail after initial convergence [S1, unofficial].

## How it works
**Inputs:**
- Set of 3D Gaussians $\mathcal{G} = \{g_i\}$, where each $g_i$ has position $\mu_i$, covariance $\Sigma_i$ (eigenvalues $\sigma_{k,i}$, eigenvectors $\vec{v}_{k,i}$), opacity weight $\lambda_i$, and color $c_i$ [S1, unofficial].
- A random batch of rays $\mathcal{R}_{batch}$, typically comprising ~5% of the dataset [S1, unofficial].
- Per-ray loss values $L_j$ for each ray $j \in \mathcal{R}_{batch}$ [S1, unofficial].
- Per-ray weights $w_{i,j}$ assigned to Gaussian $i$ for ray $j$ [S1, unofficial].

**Outputs:**
- Updated set of 3D Gaussals $\mathcal{G}'$ with modified count, positions, covariances, weights, and colors [S1, unofficial].

**Data Structures:**
- Gaussian primitive $g_i$:
  - $\mu_i \in \mathbb{R}^3$: Mean position.
  - $\Sigma_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix, parameterized by eigenvalues $\sigma_{1,i} \ge \sigma_{2,i} \ge \sigma_{3,i}$ and eigenvectors $\vec{v}_{1,i}, \vec{v}_{2,i}, \vec{v}_{3,i}$.
  - $\lambda_i \in \mathbb{R}$: Opacity weight (log-scaled during optimization).
  - $c_i \in \mathbb{R}^3$: Color (inverse-sigmoid scaled during optimization).

**Steps:**

1. **Compute Pruning Statistics:**
   Calculate the mean $\mu_\lambda$ and standard deviation $\sigma_\lambda$ of the opacity weights $\lambda_i$ across all Gaussians in $\mathcal{G}$ [S1, unofficial].

2. **Prune Gaussians:**
   Remove any Gaussian $g_i$ where:
   $$ \lambda_i \leq \mu_\lambda - z_\lambda \sigma_\lambda $$
   where $z_\lambda$ is a z-score threshold, typically set to 2 [S1, unofficial].

3. **Compute Splitting Statistics:**
   For the batch of $M$ rays $\mathcal{R}_{batch}$, estimate the average loss $\bar{l}_i$ for each Gaussian $g_i$ using the per-ray losses $L_j$ and weights $w_{i,j}$:
   $$ \bar{l}_i = \frac{1}{M} \sum_{j=1}^M L_j \cdot w_{i,j} $$
   Calculate the mean $\mu_{\bar{l}}$ and standard deviation $\sigma_{\bar{l}}$ of these average losses $\bar{l}_i$ across all Gaussians [S1, unofficial].

4. **Identify Gaussians for Splitting:**
   Select Gaussians $g_i$ where:
   $$ \bar{l}_i \geq \mu_{\bar{l}} + z_{\bar{l}} \sigma_{\bar{l}} $$
   where $z_{\bar{l}}$ is a z-score threshold, typically set to 1 [S1, unofficial].

5. **Split Selected Gaussians:**
   For each selected Gaussian $g_i$ with mean $\mu$, dominant eigenvalue $\sigma_1$, and dominant eigenvector $\vec{v}_1$:
   - **New Means:** Generate two new means $\mu_a, \mu_b$ by shifting $\mu$ along $\vec{v}_1$:
     $$ \mu_{a,b} = \mu \pm \sigma_1 \vec{v}_1 \sqrt{\frac{2}{\pi}} $$
     The scaling factor $\sqrt{\frac{2}{\pi}} \approx 0.8$ is derived from the half-normal distribution [S1, unofficial].
   - **New Covariances:** Create two new Gaussians with identical covariance matrices reconstructed from the original eigenvalues and eigenvectors, but with the dominant eigenvalue scaled:
     $$ \sigma_{1_{a,b}} = \sigma_1 \sqrt{1 - \frac{2}{\pi}} $$
     The scaling factor $\sqrt{1 - \frac{2}{\pi}} \approx 0.6$ is derived from the half-normal distribution [S1, unofficial]. The other eigenvalues $\sigma_{2,i}, \sigma_{3,i}$ remain unchanged.
   - **New Weights:** Set the log-weights for the new Gaussians by adding Gaussian noise to the original log-weight:
     $$ \log(\lambda_{a,b}) = \log(\lambda_i) + \mathcal{N}(0, \epsilon_w) $$
     where $\epsilon_w = 0.1$ [S1, unofficial].
   - **New Colors:** Set the inverse-sigmoid colors for the new Gaussians by adding Gaussian noise to the original inverse-sigmoid color:
     $$ \sigma^{-1}(c_{a,b}) = \sigma^{-1}(c_i) + \mathcal{N}(0, \epsilon_c) $$
     where $\epsilon_c = 0.1$ [S1, unofficial].

6. **Replace and Optimize:**
   Remove the original Gaussian $g_i$ and insert the two new Gaussians $g_a, g_b$ into $\mathcal{G}$. Repeat the optimization process [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def gaussian_splitting_and_pruning(G, rays_batch, losses, weights_batch, z_lambda=2.0, z_lbar=1.0, eps_w=0.1, eps_c=0.1):
    """
    G: List of Gaussian objects with attributes: mu, sigma1, v1, lambda, c
    rays_batch: List of ray indices
    losses: Array of per-ray losses L_j
    weights_batch: 2D array of shape (num_gaussians, num_rays) containing w_{i,j}
    """
    num_gaussians = len(G)
    M = len(rays_batch)
    
    # Step 1 & 2: Pruning
    lambdas = np.array([g.lambda_val for g in G])
    mu_lambda = np.mean(lambdas)
    sigma_lambda = np.std(lambdas)
    prune_threshold = mu_lambda - z_lambda * sigma_lambda
    
    # Identify indices to keep (not prune)
    keep_indices = [i for i in range(num_gaussians) if lambdas[i] > prune_threshold]
    
    # Step 3 & 4: Splitting Statistics
    # Compute average loss per Gaussian over the batch
    # weights_batch[i, j] corresponds to w_{i,j} for ray j
    # losses[j] corresponds to L_j
    avg_losses = np.zeros(num_gaussians)
    for i in keep_indices:
        # Sum over rays in the batch
        avg_losses[i] = np.sum(losses * weights_batch[i, :M]) / M
        
    mu_lbar = np.mean(avg_losses)
    sigma_lbar = np.std(avg_losses)
    split_threshold = mu_lbar + z_lbar * sigma_lbar
    
    # Identify Gaussians to split
    split_indices = [i for i in keep_indices if avg_losses[i] >= split_threshold]
    
    new_Gaussians = []
    for i in split_indices:
        g = G[i]
        
        # Step 5: Split
        # New means
        shift = g.sigma1 * g.v1 * np.sqrt(2 / np.pi)
        mu_a = g.mu + shift
        mu_b = g.mu - shift
        
        # New covariances (dominant eigenvalue scaled)
        new_sigma1 = g.sigma1 * np.sqrt(1 - 2 / np.pi)
        
        # New weights (log-space)
        log_lambda_a = np.log(g.lambda_val) + np.random.normal(0, eps_w)
        log_lambda_b = np.log(g.lambda_val) + np.random.normal(0, eps_w)
        
        # New colors (inverse-sigmoid space)
        # Assuming inverse_sigmoid is a function that undoes sigmoid
        inv_c_a = inverse_sigmoid(g.c) + np.random.normal(0, eps_c)
        inv_c_b = inverse_sigmoid(g.c) + np.random.normal(0, eps_c)
        
        # Create new Gaussian objects
        g_a = Gaussian(mu=mu_a, sigma1=new_sigma1, v1=g.v1, sigma2=g.sigma2, v2=g.v2, sigma3=g.sigma3, v3=g.v3, lambda_val=np.exp(log_lambda_a), c=sigmoid(inv_c_a))
        g_b = Gaussian(mu=mu_b, sigma1=new_sigma1, v1=g.v1, sigma2=g.sigma2, v2=g.v2, sigma3=g.sigma3, v3=g.v3, lambda_val=np.exp(log_lambda_b), c=sigmoid(inv_c_b))
        
        new_Gaussians.append(g_a)
        new_Gaussians.append(g_b)
        
    # Construct final list: keep non-split, non-pruned + new split ones
    final_G = []
    for i in range(num_gaussians):
        if i in keep_indices and i not in split_indices:
            final_G.append(G[i])
        elif i in split_indices:
            # Find the corresponding new Gaussians (order matters if multiple splits)
            # In practice, we append them as we iterate
            pass
            
    # Rebuild list properly
    final_G = []
    for i in range(num_gaussians):
        if i in keep_indices and i not in split_indices:
            final_G.append(G[i])
            
    final_G.extend(new_Gaussians)
    
    return final_G

def inverse_sigmoid(x):
    return np.log(x / (1 - x))

def sigmoid(x):
    return 1 / (1 + np.exp(-x))
```

**Edge Cases and Pitfalls:**
- **Coupled Gradients:** Noise must be added in the unconstrained parameterization space (log-weights, inverse-sigmoid colors) to prevent coupled gradients during subsequent optimization [S1, unofficial].
- **Dominant Eigenvalue:** The splitting direction is strictly along the dominant eigenvector $\vec{v}_1$. If the covariance is isotropic or nearly so, the choice of $\vec{v}_1$ may be unstable; however, the method assumes a well-defined dominant direction [S1, unofficial].
- **Batch Size:** The statistical estimates ($\mu_{\bar{l}}, \sigma_{\bar{l}}$) depend on the random batch of rays. A small batch may lead to noisy thresholds. The source suggests ~5% of the dataset [S1, unofficial].

## Subtopics

### [Octree refinement](8792-octree-refinement.md)
Octree refinement is a spatial subdivision technique used to reorder Gaussian splats for progressive rendering, ensuring that sparsely captured areas of a scene are reconstructed earlier than they would be by global contribution metrics alone. It differs from simple global sorting by enforcing a minimum representation per spatial region, thereby improving background quality at the cost of slight foreground degradation.

## Operating characteristics
- **Time Complexity:** The method requires a forward pass to compute losses and weights for a batch of rays, followed by statistical computations over all Gaussians. Splitting doubles the number of primitives for selected Gaussians, increasing the cost of subsequent rendering steps. The statistical computation is $O(N)$ where $N$ is the number of Gaussians [S1, unofficial].
- **Memory Complexity:** Memory usage increases as Gaussians are split. Pruning reduces memory usage for low-contribution Gaussians. The net effect depends on the balance between splitting and pruning [S1, unofficial].
- **Strengths:** Deterministic and statistically grounded, avoiding the heuristic nature of gradient-based splitting. Allows for iterative refinement of detail [S1, unofficial].
- **Limitations:** Requires a full forward pass over a batch of rays to compute losses and weights, which can be computationally expensive. The method is applied after initial convergence, so it does not guide the initial optimization [S1, unofficial].
- **When to use:** When a deterministic, statistically-based approach to density control is preferred over gradient-based heuristics, and when iterative refinement of detail is desired [S1, unofficial].

## Background
The method was introduced in "Flexible Techniques for Differentiable Rendering with 3D Gaussians" (2023) as an alternative to the heuristic splitting techniques in the original 3D Gaussian Splatting paper [S1, unofficial]. It leverages the properties of the half-normal distribution to determine the shift and scale of new Gaussians during splitting [S1, unofficial].

## Sources
[S1] Flexible Techniques for Differentiable Rendering with 3D Gaussians, https://doi.org/10.48550/arxiv.2308.14737, fetched 2026-10-26 (unofficial)
