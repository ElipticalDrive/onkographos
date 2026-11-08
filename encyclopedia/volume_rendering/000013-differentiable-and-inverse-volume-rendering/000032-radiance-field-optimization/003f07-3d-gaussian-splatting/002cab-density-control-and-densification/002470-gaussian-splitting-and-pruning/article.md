# Gaussian Splitting and Pruning

Also called: Deterministic splitting, Splitting Gaussians

## Overview
Gaussian Splitting and Pruning is a deterministic densification and regularization technique for 3D Gaussian Splatting that adjusts the number of Gaussian primitives based on statistical criteria derived from reconstruction loss and opacity weights [S1, unofficial]. Unlike the gradient-based heuristic splitting in the parent method [Density Control and Densification](5337-density-control-and-densification.md), which relies on accumulated view-space positional gradients and fixed size thresholds, this method evaluates the actual contribution of each Gaussian to the rendering error and opacity distribution [S1, unofficial]. It is chosen when a more rigorous, loss-driven control of Gaussian density is required to resolve under-reconstructed areas or remove redundant primitives without relying on arbitrary gradient magnitude thresholds.

## How it works
The method operates in two phases: an initial convergence phase followed by iterative statistical splitting and pruning. It is typically applied after the initial model has converged [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   A set of $N$ 3D Gaussians $\{G_i\}$, where each $G_i$ is defined by mean $\mu_i$, covariance $\Sigma_i$ (or scale/rotation), opacity weight $\lambda_i$, and color $c_i$.
    *   A dataset of training views.
    *   A reconstruction loss function $L$ (e.g., L1, SSIM, or combined) [S1, unofficial].
*   **Outputs:**
    *   A modified set of Gaussians with updated positions, covariances, weights, and colors.

### Data Structures
Each Gaussian $G_i$ maintains:
*   $\mu_i \in \mathbb{R}^3$: Mean position.
*   $\Sigma_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix, often parameterized by scale $\sigma$ and rotation $q$.
*   $\lambda_i \in \mathbb{R}$: Opacity weight (often stored as log-weight for optimization stability).
*   $c_i \in \mathbb{R}^3$: Color (often stored as inverse-sigmoid for optimization stability).

### Algorithm Steps

1.  **Initial Convergence:** Train the initial set of Gaussians until the model converges [S1, unofficial].
2.  **Pruning (Low Contribution Removal):**
    *   Compute the mean $\mu_\lambda$ and standard deviation $\sigma_\lambda$ of the opacity weights $\lambda_i$ across all Gaussians.
    *   Remove any Gaussian $G_i$ where:
        $$ \lambda_i \leq \mu_\lambda - z_\lambda \sigma_\lambda $$
    *   Here, $z_\lambda$ is a z-score threshold, typically set to 2 [S1, unofficial].
3.  **Splitting (High Responsibility Refinement):**
    *   **Loss Estimation:**
        *   Select a random batch of rays (typically ~5% of the dataset) [S1, unofficial].
        *   For each ray $j$ in the batch, compute the per-ray loss $L_j$ and the set of rendering weights $w_{i,j}$ for each Gaussian $i$ contributing to that ray.
        *   Over the batch of $M$ rays, estimate the average loss associated with each Gaussian $i$:
            $$ \bar{l}_i = \frac{1}{M} \sum_{j=1}^M L_j \cdot w_{i,j} $$
        *   Compute the mean $\mu_{\bar{l}}$ and standard deviation $\sigma_{\bar{l}}$ of these average losses $\bar{l}_i$.
    *   **Selection:**
        *   Identify Gaussians for splitting where:
            $$ \bar{l}_i \geq \mu_{\bar{l}} + z_{\bar{l}} \sigma_{\bar{l}} $$
        *   Here, $z_{\bar{l}}$ is a z-score threshold, typically set to 1 [S1, unofficial].
    *   **Splitting Execution:**
        *   For each selected Gaussian $G_i$ with mean $\mu$, covariance $\Sigma$ (eigenvalues $\sigma_k$, eigenvectors $\vec{v}_k$), weight $\lambda$, and color $c$:
            *   Identify the dominant eigenvalue $\sigma_1$ and its corresponding eigenvector $\vec{v}_1$ of $\Sigma$ [S1, unofficial].
            *   Generate two new Gaussians $G_a$ and $G_b$.
            *   **New Means:** Shift the original mean in opposite directions along the dominant eigenvector by a factor derived from the half-normal distribution:
                $$ \mu_{a,b} = \mu \pm \sigma_1 \vec{v}_1 \sqrt{\frac{2}{\pi}} $$
                The scaling factor $\sqrt{\frac{2}{\pi}} \approx 0.8$ [S1, unofficial].
            *   **New Covariances:** Assign identical covariance matrices to both new Gaussians. The dominant eigenvalue is scaled down:
                $$ \sigma_{1_{a,b}} = \sigma_1 \sqrt{1 - \frac{2}{\pi}} $$
                The scaling factor $\sqrt{1 - \frac{2}{\pi}} \approx 0.6$ [S1, unofficial]. The other eigenvalues remain unchanged. Reconstruct $\Sigma_{a,b}$ from these eigenvalues and the original eigenvectors.
            *   **New Weights:** Replicate the initial weight $\lambda$ but add noise in the unconstrained log-space to avoid coupled gradients:
                $$ \log(\lambda_{a,b}) = \log(\lambda) + \mathcal{N}(0, \epsilon_w) $$
                Where $\epsilon_w = 0.1$ [S1, unofficial].
            *   **New Colors:** Replicate the initial color $c$ but add noise in the unconstrained inverse-sigmoid space:
                $$ \sigma^{-1}(c_{a,b}) = \sigma^{-1}(c) + \mathcal{N}(0, \epsilon_c) $$
                Where $\epsilon_c = 0.1$ [S1, unofficial].
4.  **Re-optimization:** Repeat the optimization process with the modified set of Gaussians [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def gaussian_splitting_and_pruning(gaussians, rays_batch, loss_fn, z_lambda=2.0, z_lbar=1.0, eps_w=0.1, eps_c=0.1):
    """
    Performs deterministic splitting and pruning of 3D Gaussians based on statistical criteria.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
                   - mu: (3,) mean position
                   - cov: (3,3) covariance matrix
                   - lam: opacity weight
                   - c: color
        rays_batch: List of ray objects, each with:
                    - loss: scalar loss value L_j
                    - weights: dict mapping gaussian_index to weight w_i_j
        loss_fn: Function to compute per-ray loss if not pre-computed.
        z_lambda: Z-score threshold for pruning.
        z_lbar: Z-score threshold for splitting.
        eps_w: Noise scale for log-weights.
        eps_c: Noise scale for inverse-sigmoid colors.
        
    Returns:
        Updated list of Gaussians.
    """
    
    # Step 1: Pruning based on opacity weights
    lams = [g.lam for g in gaussians]
    mu_lam = np.mean(lams)
    sigma_lam = np.std(lams)
    
    # Identify indices to keep
    keep_indices = [i for i, g in enumerate(gaussians) if g.lam > mu_lam - z_lambda * sigma_lam]
    
    # Remove pruned Gaussians
    gaussians = [g for i, g in enumerate(gaussians) if i in keep_indices]
    
    # Step 2: Splitting based on average loss
    # Estimate average loss per Gaussian over the batch
    # Note: rays_batch should contain pre-computed losses and weights for efficiency
    # If not, compute them here. Assuming rays_batch has 'loss' and 'weights' (dict of {g_idx: w})
    
    # Initialize accumulators for average loss
    # We need to map current gaussian indices to their contribution
    # Since we pruned, indices might have shifted. Let's assume we process the current list.
    
    n_gaussians = len(gaussians)
    loss_sums = np.zeros(n_gaussians)
    counts = np.zeros(n_gaussians)
    
    for ray in rays_batch:
        L_j = ray.loss
        weights = ray.weights # dict {original_index: weight}
        
        # Map original indices to current indices if necessary. 
        # For simplicity, assume rays_batch weights are keyed by current index or we handle mapping.
        # In practice, one might store a mapping from original ID to current index.
        # Here, we assume 'weights' keys correspond to the current 'gaussians' list indices.
        # If the source implies weights are from the *current* rendering pass, this holds.
        
        for g_idx, w_ij in weights.items():
            if 0 <= g_idx < n_gaussians:
                loss_sums[g_idx] += L_j * w_ij
                counts[g_idx] += 1
                
    # Compute average loss
    # Handle division by zero for Gaussians not hit by any ray in batch
    avg_losses = np.where(counts > 0, loss_sums / counts, 0.0)
    
    mu_lbar = np.mean(avg_losses)
    sigma_lbar = np.std(avg_losses)
    
    # Identify Gaussians to split
    split_indices = [i for i in range(n_gaussians) if avg_losses[i] >= mu_lbar + z_lbar * sigma_lbar]
    
    new_gaussians = []
    for i, g in enumerate(gaussians):
        if i in split_indices:
            # Perform Split
            mu = g.mu
            cov = g.cov
            
            # Eigen decomposition
            eigenvalues, eigenvectors = np.linalg.eigh(cov)
            # Sort eigenvalues/eigenvectors to find dominant one (largest eigenvalue)
            # eigh returns sorted ascending, so last is dominant
            idx_dominant = np.argmax(eigenvalues)
            sigma_1 = eigenvalues[idx_dominant]
            v_1 = eigenvectors[:, idx_dominant]
            
            # Scaling factors from half-normal distribution
            scale_mean = np.sqrt(2.0 / np.pi)
            scale_cov = np.sqrt(1.0 - 2.0 / np.pi)
            
            # New means
            mu_a = mu + sigma_1 * v_1 * scale_mean
            mu_b = mu - sigma_1 * v_1 * scale_mean
            
            # New covariances
            # Replace dominant eigenvalue
            new_eigenvalues = eigenvalues.copy()
            new_eigenvalues[idx_dominant] = sigma_1 * scale_cov
            
            # Reconstruct covariance
            # cov = V * diag(eigenvalues) * V^T
            cov_a = eigenvectors @ np.diag(new_eigenvalues) @ eigenvectors.T
            cov_b = cov_a # Identical covariance
            
            # New weights and colors with noise
            log_lam = np.log(g.lam)
            log_lam_a = log_lam + np.random.normal(0, eps_w)
            log_lam_b = log_lam + np.random.normal(0, eps_w)
            
            lam_a = np.exp(log_lam_a)
            lam_b = np.exp(log_lam_b)
            
            # Inverse sigmoid for color
            # Assuming g.c is in [0,1] and we need to add noise in inverse-sigmoid space
            # sigma_inv(c) = log(c / (1-c))
            # Add noise
            inv_sig_c = np.log(g.c / (1 - g.c + 1e-7)) # +1e-7 for stability
            inv_sig_c_a = inv_sig_c + np.random.normal(0, eps_c, size=3)
            inv_sig_c_b = inv_sig_c + np.random.normal(0, eps_c, size=3)
            
            # Sigmoid back to [0,1]
            c_a = 1.0 / (1.0 + np.exp(-inv_sig_c_a))
            c_b = 1.0 / (1.0 + np.exp(-inv_sig_c_b))
            
            # Create new Gaussian objects
            # Assuming Gaussian class constructor
            g_a = Gaussian(mu=mu_a, cov=cov_a, lam=lam_a, c=c_a)
            g_b = Gaussian(mu=mu_b, cov=cov_b, lam=lam_b, c=c_b)
            
            new_gaussians.append(g_a)
            new_gaussians.append(g_b)
        else:
            new_gaussians.append(g)
            
    return new_gaussians

class Gaussian:
    def __init__(self, mu, cov, lam, c):
        self.mu = mu
        self.cov = cov
        self.lam = lam
        self.c = c
```

### Parameters and Edge Cases
*   **Z-scores:** $z_\lambda = 2$ and $z_{\bar{l}} = 1$ are typical values [S1, unofficial].
*   **Noise Scale:** $\epsilon_w = 0.1$ and $\epsilon_c = 0.1$ [S1, unofficial].
*   **Batch Size:** ~5% of the dataset for loss estimation [S1, unofficial].
*   **Edge Cases:**
    *   **Zero Count:** If a Gaussian is not hit by any ray in the batch, its average loss is 0. This may prevent it from being split but also protects it from being split due to noise.
    *   **Numerical Stability:** Inverse sigmoid requires care near 0 and 1. Log-weights require positive weights.
    *   **Degenerate Covariance:** If the dominant eigenvalue is near zero, the split direction is undefined. A guard should check for this.

## Subtopics

### [Octree refinement](8792-octree-refinement.md)
Octree refinement is a spatial subdivision technique used to reorder Gaussian splats for progressive rendering, ensuring that sparsely captured areas of the scene are reconstructed earlier than they would be by global contribution metrics alone. It differs from simple global sorting by enforcing a minimum representation per spatial region, thereby improving background quality at the cost of slight foreground degradation.

### [Gaussian Pruning and Sampling](9219-gaussian-pruning-and-sampling.md)
Gaussian Pruning and Sampling is a post-processing or intermediate optimization strategy in 3D Gaussian Splatting that reduces the number of Gaussian primitives by removing those with low contribution to the final image. It addresses the inherent redundancy in dense Gaussian representations initialized from point clouds. Unlike Gaussian Splitting and Pruning, which dynamically adjusts density during training based on gradient-based splitting criteria, this method relies on accumulated rendering weights to identify and discard insignificant Gaussians. It is often combined with sampling techniques to mitigate the loss of geometric detail.

### [Blur Split](9237-blur-split.md)
Blur Split is a densification technique for 3D Gaussian Splatting that addresses "under-reconstruction" and blur artifacts in regions with smooth color transitions where gradient-based methods fail. It identifies Gaussians with excessively large maximum contribution areas in screen space and splits them to improve local detail. Unlike standard pruning which removes large Gaussians, Blur Split actively refines them to resolve blurry regions.

### Spatial Splitting with Error Moments
Spatial Splitting with Error Moments is a densification technique that splits Gaussians with error above a threshold $\tau_{space}$ by placing a splitting plane through the Gaussian's center, oriented based on the error moments rather than the Gaussian's longest axis [S2]. For each Gaussian $g$ with error $> \tau_{space}$, the splitting plane normal $n$ is calculated as the normalized product of the inverse error covariance matrix and the first moment vector:
$$ n = \frac{(\mathbf{S}_g^c)^{-1} \mathbf{r}_g}{\left\| (\mathbf{S}_g^c)^{-1} \mathbf{r}_g \right\|} $$
where $\mathbf{S}_g^c$ is the error covariance matrix and $\mathbf{r}_g$ is the first moment [S2]. Unlike FCC* 24 and DDL* 25b which use a plane perpendicular to the Gaussian's longest axis, this method uses the error moments to determine orientation to avoid Gaussians becoming too small too quickly [S2].

## Operating characteristics
*   **Time Complexity:** The pruning step is $O(N)$ where $N$ is the number of Gaussians. The splitting step requires a forward pass over a batch of rays to compute losses and weights, which is $O(M \cdot K)$ where $M$ is the batch size and $K$ is the average number of Gaussians per ray. The statistical calculations are $O(N)$. The splitting itself is $O(N_{split})$. Overall, it adds a significant overhead per densification step due to the ray batch evaluation.
*   **Memory Complexity:** Requires storing the batch of rays and their associated losses and weights, which is $O(M \cdot K)$. The Gaussian set size grows with splitting, capped by implementation-specific limits.
*   **Performance:** This method provides a more controlled and deterministic increase in detail compared to heuristic gradient-based methods. It is particularly effective in resolving areas with high reconstruction error that may not have high positional gradients. However, it is computationally more expensive per iteration due to the need for loss estimation over a ray batch.
*   **Strengths:** Deterministic, loss-driven, avoids arbitrary thresholds for splitting, handles under-reconstruction well.
*   **Limitations:** Computationally intensive, requires careful tuning of z-scores, may split Gaussians in noisy regions if loss is high due to noise rather than geometry.

## Background
Gaussian Splitting and Pruning was introduced as an alternative to the heuristic splitting techniques in the original 3D Gaussian Splatting paper [33] (referenced in [S1, unofficial]). It was developed to provide a more rigorous, statistical approach to modifying the number of Gaussians in a reconstruction [S1, unofficial]. The method leverages the properties of the half-normal distribution to determine the new positions and covariances of split Gaussians, ensuring that the split is statistically sound [S1, unofficial]. It is cited in the context of flexible techniques for differentiable rendering with 3D Gaussians [S1, unofficial].

## Sources
[S1] Flexible Techniques for Differentiable Rendering with 3D Gaussians, https://doi.org/10.48550/arxiv.2308.14737, fetched 2026-10-26 (unofficial)
[S2] Adaptive Spatio‐Temporal 3D Gaussian Splatting for Scenes with Oscillatory Motion, https://doi.org/10.1111/cgf.70410, fetched 2026-10-26
