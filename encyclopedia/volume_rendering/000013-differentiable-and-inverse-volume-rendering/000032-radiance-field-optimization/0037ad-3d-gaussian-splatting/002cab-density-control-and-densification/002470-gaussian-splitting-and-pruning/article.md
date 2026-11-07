# Gaussian Splitting and Pruning

Also called: Deterministic splitting, Splitting Gaussians

## Overview
Gaussian Splitting and Pruning is a deterministic density control method for 3D Gaussian Splatting that modifies the number of primitives based on statistical criteria derived from reconstruction loss and blending weights [S1, unofficial][S2]. Unlike heuristic gradient-based densification, it operates after initial convergence to iteratively refine detail by splitting high-responsibility Gaussians and removing minimal contributors [S1, unofficial]. It addresses the inefficiency of fixed Gaussian counts by explicitly targeting regions of high error and redundancy [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
- Input: A converged set of 3D Gaussians $\mathcal{G} = \{g_i\}$, where each $g_i$ has position $\mu_i$, covariance $\Sigma_i$ (eigenvalues $\sigma_{k,i}$, eigenvectors $\vec{v}_{k,i}$), opacity weight $\lambda_i$, and color $c_i$.
- Output: A modified set $\mathcal{G}'$ with updated primitives.

**Data Structures:**
- Gaussian primitive $g_i$:
  - $\mu_i \in \mathbb{R}^3$: Mean position.
  - $\Sigma_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix, parameterized by scale and rotation. Eigen-decomposition: $\Sigma_i = V_i \Lambda_i V_i^T$.
  - $\lambda_i \in \mathbb{R}$: Opacity weight (stored as $\log(\lambda_i)$ for optimization).
  - $c_i \in \mathbb{R}^3$: Color (stored as inverse-sigmoid $\sigma^{-1}(c_i)$ for optimization).
  - $\vec{v}_{1,i}$: Dominant eigenvector of $\Sigma_i$.
  - $\sigma_{1,i}$: Dominant eigenvalue of $\Sigma_i$.

**Steps:**

1. **Compute Minimal Contribution Set (Pruning):**
   - Compute the mean $\mu_\lambda$ and standard deviation $\sigma_\lambda$ of the opacity weights $\lambda_i$ across all Gaussians.
   - Identify Gaussians to prune:
     $$ \lambda_i \leq \mu_\lambda - z_\lambda \sigma_\lambda $$
     where $z_\lambda$ is a z-score, typically 2 [S1, unofficial].

2. **Compute Splitting Criteria:**
   - Select a random batch of rays, typically 5% of the dataset [S1, unofficial].
   - For each ray $j$ in the batch ($M$ rays total), compute the per-ray loss $L_j$ (e.g., L1 or SSIM loss) [S1, unofficial].
   - For each ray, obtain the blending weights $w_{i,j}$ for all intersected Gaussians $i$ [S1, unofficial].
   - Estimate the average loss associated with each Gaussian $i$:
     $$ \bar{l}_i = \frac{1}{M} \sum_{j=1}^M L_j \cdot w_{i,j} $$
   - Compute the mean $\mu_{\bar{l}}$ and standard deviation $\sigma_{\bar{l}}$ of $\bar{l}_i$ across all Gaussians [S1, unofficial].
   - Identify Gaussians to split:
     $$ \bar{l}_i \geq \mu_{\bar{l}} + z_{\bar{l}} \sigma_{\bar{l}} $$
     where $z_{\bar{l}}$ is a z-score, typically 1 [S1, unofficial].

3. **Splitting Operation:**
   - For each Gaussian $g_i$ selected for splitting, create two new Gaussians $g_a$ and $g_b$:
     - **Positions:** Shift the mean $\mu_i$ along the dominant eigenvector $\vec{v}_{1,i}$:
       $$ \mu_{a,b} = \mu_i \pm \sigma_{1,i} \vec{v}_{1,i} \sqrt{\frac{2}{\pi}} $$
       The scaling factor $\sqrt{\frac{2}{\pi}} \approx 0.8$ is derived from the half-normal distribution [S1, unofficial].
     - **Covariance:** Set identical covariance matrices for $g_a$ and $g_b$. Reconstruct $\Sigma$ from the original eigenvectors but scale the dominant eigenvalue:
       $$ \sigma_{1_{a,b}} = \sigma_{1,i} \sqrt{1 - \frac{2}{\pi}} $$
       The scaling factor $\sqrt{1 - \frac{2}{\pi}} \approx 0.6$ is derived from the half-normal distribution [S1, unofficial]. Other eigenvalues remain unchanged.
     - **Opacity:** Replicate the initial weight $\lambda_i$ with additive noise in log-space:
       $$ \log(\lambda_{a,b}) = \log(\lambda_i) + \mathcal{N}(0, \epsilon_w) $$
       where $\epsilon_w = 0.1$ [S1, unofficial].
     - **Color:** Replicate the initial color $c_i$ with additive noise in inverse-sigmoid space:
       $$ \sigma^{-1}(c_{a,b}) = \sigma^{-1}(c_i) + \mathcal{N}(0, \epsilon_c) $$
       where $\epsilon_c = 0.1$ [S1, unofficial].

4. **Optimization:**
   - Repeat the optimization process (backpropagation and gradient descent) on the modified set $\mathcal{G}'$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def gaussian_splitting_and_pruning(gaussians, rays_batch, z_lambda=2.0, z_lbar=1.0, epsilon_w=0.1, epsilon_c=0.1):
    """
    Perform deterministic splitting and pruning of 3D Gaussians.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
            - mu: (3,) position
            - sigma1: float, dominant eigenvalue
            - v1: (3,) dominant eigenvector
            - lambda_val: float, opacity weight
            - color: (3,) color
            - eigenvectors: (3,3) matrix of eigenvectors
        rays_batch: List of ray objects with attributes:
            - loss: float, per-ray loss L_j
            - weights: dict mapping gaussian_index to weight w_ij
        z_lambda: float, z-score for pruning
        z_lbar: float, z-score for splitting
        epsilon_w: float, noise std for log-opacity
        epsilon_c: float, noise std for inverse-sigmoid color
        
    Returns:
        new_gaussians: List of Gaussian objects
    """
    new_gaussians = []
    gaussians_to_prune = set()
    gaussians_to_split = []
    
    # Step 1: Pruning based on opacity weights
    lambda_vals = [g.lambda_val for g in gaussians]
    mu_lambda = np.mean(lambda_vals)
    sigma_lambda = np.std(lambda_vals)
    prune_threshold = mu_lambda - z_lambda * sigma_lambda
    
    for i, g in enumerate(gaussians):
        if g.lambda_val <= prune_threshold:
            gaussians_to_prune.add(i)
        else:
            new_gaussians.append(g) # Keep non-pruned Gaussians
            
    # Step 2: Splitting based on average loss
    # Compute average loss per Gaussian
    M = len(rays_batch)
    loss_sums = np.zeros(len(gaussians))
    count_sums = np.zeros(len(gaussians))
    
    for ray in rays_batch:
        L_j = ray.loss
        for idx, w_ij in ray.weights.items():
            loss_sums[idx] += L_j * w_ij
            count_sums[idx] += 1
            
    # Avoid division by zero
    count_sums[count_sums == 0] = 1
    avg_losses = loss_sums / count_sums
    
    mu_lbar = np.mean(avg_losses)
    sigma_lbar = np.std(avg_losses)
    split_threshold = mu_lbar + z_lbar * sigma_lbar
    
    # Identify Gaussians to split (only those not pruned)
    for i, g in enumerate(gaussians):
        if i not in gaussians_to_prune and avg_losses[i] >= split_threshold:
            gaussians_to_split.append((i, g))
            
    # Step 3: Splitting
    sqrt_2_pi = np.sqrt(2 / np.pi)
    sqrt_1_minus_2_pi = np.sqrt(1 - 2 / np.pi)
    
    for i, g in gaussians_to_split:
        # New positions
        shift = g.sigma1 * g.v1 * sqrt_2_pi
        mu_a = g.mu + shift
        mu_b = g.mu - shift
        
        # New covariance: scale dominant eigenvalue
        # Reconstruct covariance from eigenvectors and scaled eigenvalues
        # Original eigenvalues: sigma1, sigma2, sigma3
        # New dominant eigenvalue: sigma1 * sqrt(1 - 2/pi)
        # Other eigenvalues remain same
        new_sigma1 = g.sigma1 * sqrt_1_minus_2_pi
        
        # Assuming eigenvectors are stored in g.eigenvectors columns
        # Sigma_new = V * diag([new_sigma1, sigma2, sigma3]) * V^T
        # For simplicity, we assume the Gaussian object can handle covariance reconstruction
        # or we just store the new parameters. Here we create new Gaussian objects.
        
        # New opacity
        log_lambda_a = np.log(g.lambda_val) + np.random.normal(0, epsilon_w)
        log_lambda_b = np.log(g.lambda_val) + np.random.normal(0, epsilon_w)
        
        # New color
        inv_sigmoid_c = np.arcsinh(g.color) # Assuming sigma^-1 is arcsinh or similar inverse sigmoid
        # Note: The source uses sigma^-1(c). Standard sigmoid inverse is logit. 
        # If color is in [0,1], logit(c) = log(c/(1-c)). 
        # The source says "inverse-sigmoid colors". We assume a standard inverse sigmoid function.
        inv_sigmoid_c_a = inv_sigmoid_c + np.random.normal(0, epsilon_c)
        inv_sigmoid_c_b = inv_sigmoid_c + np.random.normal(0, epsilon_c)
        
        # Create new Gaussian objects
        # Note: The exact structure of Gaussian object is implementation dependent.
        # We assume a constructor that takes mu, sigma1, v1, eigenvectors, lambda_val, color.
        # For covariance, we need to reconstruct it. 
        # Let's assume we store the new parameters directly.
        
        g_a = Gaussian(
            mu=mu_a,
            sigma1=new_sigma1,
            v1=g.v1,
            eigenvectors=g.eigenvectors, # Keep other eigenvectors and eigenvalues same
            lambda_val=np.exp(log_lambda_a),
            color=np.tanh(inv_sigmoid_c_a) # Assuming sigmoid activation
        )
        
        g_b = Gaussian(
            mu=mu_b,
            sigma1=new_sigma1,
            v1=g.v1,
            eigenvectors=g.eigenvectors,
            lambda_val=np.exp(log_lambda_b),
            color=np.tanh(inv_sigmoid_c_b)
        )
        
        new_gaussians.append(g_a)
        new_gaussians.append(g_b)
        
    return new_gaussians

class Gaussian:
    def __init__(self, mu, sigma1, v1, eigenvectors, lambda_val, color):
        self.mu = mu
        self.sigma1 = sigma1
        self.v1 = v1
        self.eigenvectors = eigenvectors
        self.lambda_val = lambda_val
        self.color = color
```

**Parameters:**
- $z_\lambda$: Z-score for pruning threshold, typically 2 [S1, unofficial].
- $z_{\bar{l}}$: Z-score for splitting threshold, typically 1 [S1, unofficial].
- $\epsilon_w$: Noise standard deviation for log-opacity, typically 0.1 [S1, unofficial].
- $\epsilon_c$: Noise standard deviation for inverse-sigmoid color, typically 0.1 [S1, unofficial].
- Batch size: Typically 5% of the dataset [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Coupled Gradients:** Noise is added to unconstrained parameterization space (log-opacity, inverse-sigmoid color) to avoid coupled gradients during further optimization [S1, unofficial].
- **Under-reconstruction:** Pruning alone fails to address 'overlapping' and 'under-reconstruction' phenomena, leaving inefficient spatial distribution [S2]. Splitting addresses this by refining high-error regions.
- **Blur Artifacts:** Standard splitting may not resolve blur artifacts in smooth regions; specialized variants like Blur Split are needed [S1, 9237-blur-split.md, unofficial].

## Subtopics

### [Octree refinement](8792-octree-refinement.md)
Octree refinement is a spatial subdivision technique used to reorder Gaussian splats for progressive rendering, ensuring that sparsely captured areas of a scene are reconstructed earlier than they would be by global contribution metrics alone. It differs from simple global sorting by enforcing a minimum representation per spatial region, thereby improving background quality at the cost of slight foreground degradation.

### [Blur Split](9237-blur-split.md)
Blur Split is a densification technique for 3D Gaussian Splatting that addresses "under-reconstruction" and blur artifacts in regions with smooth color transitions where gradient-based methods fail. It identifies Gaussians with excessively large maximum contribution areas in screen space and splits them to improve local detail. Unlike standard pruning which removes large Gaussians, Blur Split actively refines them to resolve blurry regions.

### Gaussian Pruning and Sampling
Gaussian Pruning and Sampling is a straightforward approach to reduce the number of Gaussians by calculating an importance score based on accumulated blending weights and removing those below a threshold [S2]. It calculates importance score $I_i = \sum_{j=1}^{K} w_{ij}$ for each Gaussian $G_i$, where $w_{ij}$ is the blending weight for ray $j$ and $K$ is the total number of intersected rays [S2]. Pruning reduces Gaussians from 6.1 million to 0.6 million but degrades PSNR from 25.2 dB to 24.9 dB [S2]. It fails to address 'overlapping' and 'under-reconstruction' phenomena [S2].

## Operating characteristics
- **Time Complexity:** The method requires a forward pass to compute losses and weights for a batch of rays, followed by statistical computations (mean, std) over all Gaussians. Splitting and pruning are linear in the number of Gaussians. The iterative nature (split, then optimize) increases total training time compared to single-pass methods.
- **Memory Complexity:** Memory usage increases during splitting as the number of Gaussians grows. Pruning reduces memory usage. The batch of rays requires memory proportional to the batch size.
- **Measured Performance:** Pruning alone can reduce Gaussian count from 6.1 million to 0.6 million but degrades PSNR from 25.2 dB to 24.9 dB [S2].
- **Strengths:** Deterministic and statistically grounded. Addresses redundancy and high-error regions explicitly. Improves detail in converged models.
- **Limitations:** Requires multiple optimization phases. Pruning alone can degrade quality. May not resolve blur artifacts without specialized variants.
- **When to use:** When a converged model needs refinement for higher detail or when reducing Gaussian count is necessary for efficiency, provided quality degradation is acceptable or mitigated by splitting.
- **When not to use:** When real-time training is required, or when blur artifacts are prevalent and not addressed by other means.

## Background
Gaussian Splitting and Pruning was introduced as a deterministic alternative to heuristic densification methods in 3D Gaussian Splatting [S1, unofficial]. It builds on the observation that Gaussians with minimal contribution can be removed and those with high reconstruction responsibility can be split to increase detail [S1, unofficial]. The method uses statistical criteria (z-scores) to determine thresholds for pruning and splitting [S1, unofficial]. It was developed to provide a more controlled and iterative approach to density control [S1, unofficial].

## Sources
[S1] Flexible Techniques for Differentiable Rendering with 3D Gaussians, https://doi.org/10.48550/arxiv.2308.14737, fetched 2026-10-26 (unofficial)
[S2] Mini-Splatting: Representing Scenes with a Constrained Number of Gaussians, https://doi.org/10.1007/978-3-031-72980-5_10, fetched 2026-10-26
