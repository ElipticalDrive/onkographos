# Gaussian Splitting and Pruning

Also called: Deterministic splitting, Splitting Gaussians

## Overview
Gaussian Splitting and Pruning is a deterministic density control method for 3D Gaussian Splatting that modifies the number of Gaussians in a reconstruction by splitting those with high reconstruction responsibility and removing those with minimal contribution, based on statistical criteria [S1, unofficial]. It differs from heuristic merging/splitting in original 3DGS [33] by using explicit loss-based statistics and half-normal distribution properties to ensure deterministic refinement [S1, unofficial]. It addresses the inefficiencies of simple pruning, which leaves overlapping and under-reconstructed regions, by actively refining high-error Gaussians rather than just removing low-contribution ones [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussians $\{G_i\}$ with means $\mu_i$, covariances $\Sigma_i$, opacities/weights $\lambda_i$, and colors $c_i$; a set of training images with camera poses; a loss function $\mathcal{L}$ (e.g., MSE or LPIPS).
*   **Outputs:** An updated set of 3D Gaussians with modified counts, positions, covariances, weights, and colors.

**Data Structures:**
*   **Gaussian Attributes:** Each Gaussian $i$ stores:
    *   Mean position $\mu_i \in \mathbb{R}^3$.
    *   Covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$ (often stored as quaternion $q_i$ and scale vector $s_i$).
    *   Weight/Opacity $\lambda_i \in \mathbb{R}$ (or log-weight $\log \lambda_i$).
    *   Color $c_i \in \mathbb{R}^3$ (or inverse-sigmoid color $\sigma^{-1}(c_i)$).
    *   Accumulated loss $\bar{l}_i \in \mathbb{R}$ (computed during the sampling phase).
    *   Accumulated importance score $I_i \in \mathbb{R}$ (used in variants).

**Steps:**

1.  **Initial Optimization:** Train the initial 3D Gaussian model until convergence [S1, unofficial].
2.  **Pruning (Optional/Variant):** Calculate importance scores and remove low-contribution Gaussians [S2].
    *   For each Gaussian $G_i$, compute importance score $I_i = \sum_{j=1}^{K} w_{ij}$, where $w_{ij}$ is the blending weight of $G_i$ for ray $j$, and $K$ is the total number of rays intersected with $G_i$ [S2].
    *   Remove Gaussians where $I_i$ is below a threshold [S2].
3.  **Statistical Analysis for Splitting:**
    *   Compute the mean $\mu_\lambda$ and standard deviation $\sigma_\lambda$ of the weights $\lambda_i$ assigned to each Gaussian [S1, unofficial].
    *   Remove Gaussians satisfying $\lambda_i \leq \mu_\lambda - z_\lambda \sigma_\lambda$, where $z_\lambda$ is a z-score typically set to 2 [S1, unofficial].
    *   Take a random batch of rays (typically ~5% of the dataset) [S1, unofficial].
    *   For each ray in the batch, compute the per-ray loss $L_j$ and the associated set of computed weights $w_i$ for all intersecting Gaussians [S1, unofficial].
    *   Over the batch of $M$ rays, estimate the average loss associated with each Gaussian $i$ as:
        $$ \bar{l}_i = \frac{1}{M} \sum_{j=1}^M L_j \cdot w_i $$
    *   Compute the mean $\mu_{\bar{l}}$ and standard deviation $\sigma_{\bar{l}}$ of these average losses $\bar{l}_i$ [S1, unofficial].
4.  **Splitting:**
    *   Identify Gaussians to split where $\bar{l}_i \geq \mu_{\bar{l}} + z_{\bar{l}} \sigma_{\bar{l}}$, where $z_{\bar{l}}$ is a z-score typically set to 1 [S1, unofficial].
    *   For each Gaussian $G_i$ to be split, compute the eigenvalues and eigenvectors of its covariance matrix $\Sigma_i$. Let $\sigma_1$ be the dominant eigenvalue and $\vec{v}_1$ be the corresponding dominant eigenvector [S1, unofficial].
    *   Generate two new Gaussians, $G_a$ and $G_b$, with the following properties:
        *   **Means:** Shift the initial mean $\mu_i$ in opposite directions along the dominant eigenvector:
            $$ \mu_{a,b} = \mu_i \pm \sigma_1 \vec{v}_1 \sqrt{\frac{2}{\pi}} $$
            The scaling factor $\sqrt{\frac{2}{\pi}} \approx 0.8$ is derived from the half-normal distribution [S1, unofficial].
        *   **Covariances:** Set identical covariance matrices for $G_a$ and $G_b$, reconstructed from the initial eigenvalues and eigenvectors, but with the dominant eigenvalue scaled:
            $$ \sigma_{1_{a,b}} = \sigma_1 \sqrt{1 - \frac{2}{\pi}} $$
            The scaling factor $\sqrt{1 - \frac{2}{\pi}} \approx 0.6$ is derived from the half-normal distribution [S1, unofficial]. The other eigenvalues remain unchanged.
        *   **Weights:** Replicate the initial weight $\lambda_i$ with added noise in the log-weight space:
            $$ \log(\lambda_{a,b}) = \log(\lambda_i) + \mathcal{N}(0, \epsilon_w) $$
            where $\epsilon_w = 0.1$ [S1, unofficial].
        *   **Colors:** Replicate the initial color $c_i$ with added noise in the inverse-sigmoid color space:
            $$ \sigma^{-1}(c_{a,b}) = \sigma^{-1}(c_i) + \mathcal{N}(0, \epsilon_c) $$
            where $\epsilon_c = 0.1$ [S1, unofficial].
5.  **Re-optimization:** Repeat the optimization process (steps 1-4) iteratively to refine the reconstruction [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def gaussian_splitting_and_pruning(gaussians, rays, loss_fn, z_lambda=2.0, z_lbar=1.0, epsilon_w=0.1, epsilon_c=0.1):
    """
    Performs deterministic splitting and pruning of 3D Gaussians.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
            - mu: mean position (3,)
            - cov: covariance matrix (3,3)
            - lambda: weight/opacity (scalar)
            - c: color (3,)
        rays: List of ray objects with attributes:
            - origin: (3,)
            - direction: (3,)
            - gt_color: ground truth color (3,)
        loss_fn: Function to compute per-ray loss.
        z_lambda: Z-score threshold for pruning based on weight.
        z_lbar: Z-score threshold for splitting based on average loss.
        epsilon_w: Noise scale for log-weights.
        epsilon_c: Noise scale for inverse-sigmoid colors.
        
    Returns:
        Updated list of Gaussians.
    """
    
    # Step 1: Pruning based on weight statistics
    weights = np.array([g.lambda_val for g in gaussians])
    mu_lambda = np.mean(weights)
    sigma_lambda = np.std(weights)
    
    # Remove Gaussians with low weights
    gaussians = [g for g in gaussians if g.lambda_val > mu_lambda - z_lambda * sigma_lambda]
    
    # Step 2: Compute average loss per Gaussian
    # Sample a batch of rays (e.g., 5% of dataset)
    batch_size = int(len(rays) * 0.05)
    ray_batch = np.random.choice(rays, batch_size, replace=False)
    
    # Initialize average loss accumulator
    avg_losses = np.zeros(len(gaussians))
    
    for ray in ray_batch:
        # Compute per-ray loss
        L_j = loss_fn(ray)
        
        # Compute blending weights w_i for each Gaussian intersecting this ray
        # This requires rendering the ray with the current Gaussian set
        # Assume a function get_blending_weights(ray, gaussians) returns list of weights
        blending_weights = get_blending_weights(ray, gaussians)
        
        # Accumulate weighted loss for each Gaussian
        for i, w_i in enumerate(blending_weights):
            avg_losses[i] += L_j * w_i
            
    # Average over the batch
    avg_losses /= batch_size
    
    # Step 3: Splitting based on average loss statistics
    mu_lbar = np.mean(avg_losses)
    sigma_lbar = np.std(avg_losses)
    
    split_indices = [i for i, l in enumerate(avg_losses) if l >= mu_lbar + z_lbar * sigma_lbar]
    
    new_gaussians = []
    for i, g in enumerate(gaussians):
        if i in split_indices:
            # Split Gaussian g into two new Gaussians
            # Compute eigenvalues and eigenvectors of covariance matrix
            eigenvalues, eigenvectors = np.linalg.eigh(g.cov)
            
            # Find dominant eigenvalue and eigenvector
            # eigenvalues are sorted in ascending order, so the last one is dominant
            sigma_1 = eigenvalues[-1]
            v_1 = eigenvectors[:, -1]
            
            # Scaling factors from half-normal distribution
            scale_mean = np.sqrt(2.0 / np.pi)
            scale_cov = np.sqrt(1.0 - 2.0 / np.pi)
            
            # Generate new means
            mu_a = g.mu + sigma_1 * v_1 * scale_mean
            mu_b = g.mu - sigma_1 * v_1 * scale_mean
            
            # Generate new covariances
            # Replace dominant eigenvalue with scaled version
            new_eigenvalues = eigenvalues.copy()
            new_eigenvalues[-1] = sigma_1 * scale_cov
            
            # Reconstruct covariance matrix
            cov_a = eigenvectors @ np.diag(new_eigenvalues) @ eigenvectors.T
            cov_b = cov_a.copy()
            
            # Generate new weights with noise in log-space
            log_lambda_a = np.log(g.lambda_val) + np.random.normal(0, epsilon_w)
            log_lambda_b = np.log(g.lambda_val) + np.random.normal(0, epsilon_w)
            lambda_a = np.exp(log_lambda_a)
            lambda_b = np.exp(log_lambda_b)
            
            # Generate new colors with noise in inverse-sigmoid space
            # Assume inverse_sigmoid and sigmoid functions are defined
            inv_c_a = inverse_sigmoid(g.c) + np.random.normal(0, epsilon_c, size=3)
            inv_c_b = inverse_sigmoid(g.c) + np.random.normal(0, epsilon_c, size=3)
            c_a = sigmoid(inv_c_a)
            c_b = sigmoid(inv_c_b)
            
            # Create new Gaussian objects
            g_a = Gaussian(mu=mu_a, cov=cov_a, lambda_val=lambda_a, c=c_a)
            g_b = Gaussian(mu=mu_b, cov=cov_b, lambda_val=lambda_b, c=c_b)
            
            new_gaussians.append(g_a)
            new_gaussians.append(g_b)
        else:
            new_gaussians.append(g)
            
    return new_gaussians

def get_blending_weights(ray, gaussians):
    """
    Computes the blending weight w_i for each Gaussian for a given ray.
    This is a placeholder for the actual differentiable volume rendering logic.
    """
    weights = []
    for g in gaussians:
        # Compute contribution of Gaussian g to ray
        # This involves projecting the Gaussian to 2D, computing opacity, etc.
        # For simplicity, assume a function gaussian_contribution exists
        w = gaussian_contribution(ray, g)
        weights.append(w)
    return weights

def gaussian_contribution(ray, gaussian):
    """
    Placeholder for computing the blending weight of a Gaussian for a ray.
    """
    # Implementation details depend on the specific 3DGS rendering pipeline
    # This typically involves projecting the 3D Gaussian to 2D screen space
    # and computing the alpha value at the ray intersection point.
    pass

def inverse_sigmoid(x):
    return np.log(x / (1 - x))

def sigmoid(x):
    return 1 / (1 + np.exp(-x))
```

**Parameters:**
*   $z_\lambda$: Z-score for pruning threshold, typically 2 [S1, unofficial].
*   $z_{\bar{l}}$: Z-score for splitting threshold, typically 1 [S1, unofficial].
*   $\epsilon_w$: Noise scale for log-weights, typically 0.1 [S1, unofficial].
*   $\epsilon_c$: Noise scale for inverse-sigmoid colors, typically 0.1 [S1, unofficial].
*   Batch size for loss estimation: Typically 5% of the dataset [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The use of inverse-sigmoid color space and log-weight space is critical to avoid coupled gradients and ensure stable optimization [S1, unofficial].
*   **Boundaries:** The splitting process assumes the dominant eigenvector direction is meaningful for splitting. In degenerate cases (e.g., spherical Gaussians), the choice of eigenvector may be arbitrary, but the scaling factors still apply [S1, unofficial].
*   **Degenerate Input:** If all Gaussians have similar weights or losses, the statistical thresholds may not trigger any splits or prunes, leading to no change in density [S1, unofficial].

## Subtopics

### [Octree refinement](8792-octree-refinement.md)
Octree refinement is a spatial subdivision technique used to reorder Gaussian splats for progressive rendering, ensuring that sparsely captured areas of a scene are reconstructed earlier than they would be by global contribution metrics alone. It differs from simple global sorting by enforcing a minimum representation per spatial region, thereby improving background quality at the cost of slight foreground degradation.

### [Blur Split](9237-blur-split.md)
Blur Split is a densification technique for 3D Gaussian Splatting that addresses "under-reconstruction" and blur artifacts in regions with smooth color transitions where gradient-based methods fail. It identifies Gaussians with excessively large maximum contribution areas in screen space and splits them to improve local detail. Unlike standard pruning which removes large Gaussians, Blur Split actively refines them to resolve blurry regions.

### Gaussian Pruning and Sampling
Gaussian Pruning and Sampling is a straightforward approach to reduce the number of Gaussians by calculating an importance score based on accumulated blending weights and removing those below a threshold [S2]. It calculates importance score $I_i = \sum_{j=1}^{K} w_{ij}$ for each Gaussian $G_i$, where $w_{ij}$ is the blending weight of $G_i$ for ray $j$, and $K$ is the total number of rays intersected with $G_i$ [S2]. It prunes Gaussians where $I_i$ is lower than a certain threshold and then optimizes the representation until convergence [S2]. A pitfall is that pruning alone fails to address 'overlapping' and 'under-reconstruction' phenomena, leaving inefficient spatial distribution [S2]. In experiments, pruning reduced Gaussians from 6.1 million to 0.6 million, but degraded PSNR from 25.2 dB to 24.9 dB [S2].

## Operating characteristics
*   **Time Complexity:** The splitting and pruning process involves computing eigenvalues/eigenvectors for each Gaussian to be split, which is $O(1)$ per Gaussian (since covariance is $3 \times 3$). The loss estimation requires a forward pass over a batch of rays, which is $O(M \cdot N)$ where $M$ is the batch size and $N$ is the number of Gaussians. The overall complexity per iteration is dominated by the rendering step.
*   **Memory Complexity:** The method increases the number of Gaussians, so memory usage scales with the final number of Gaussians. The statistical analysis requires storing average losses for each Gaussian, which is $O(N)$.
*   **Measured Performance:** The sources do not provide specific performance metrics for the splitting and pruning method itself, other than noting it allows for increasing detail in an iterative way [S1, unofficial]. The pruning variant reduced the number of Gaussians from 6.1 million to 0.6 million but degraded PSNR from 25.2 dB to 24.9 dB [S2].
*   **Strengths:** Deterministic and statistically grounded; addresses under-reconstruction by splitting high-error Gaussians; avoids coupled gradients by using noise in unconstrained parameterization space [S1, unofficial].
*   **Limitations:** Pruning alone is insufficient and can degrade quality [S2]; the method relies on the assumption that the dominant eigenvector direction is the best direction for splitting, which may not always be optimal for complex geometries [S1, unofficial].
*   **When to use:** When the initial reconstruction has regions of high error or under-reconstruction that cannot be fixed by simple optimization [S1, unofficial].
*   **When not to use:** When computational resources are limited and a simpler pruning strategy is acceptable, despite potential quality degradation [S2].

## Background
Gaussian Splitting and Pruning was introduced in "Flexible Techniques for Differentiable Rendering with 3D Gaussians" [S1, unofficial] as a deterministic alternative to the heuristic merging and splitting techniques in original 3DGS [33]. It builds on the properties of the half-normal distribution to ensure that split Gaussians maintain the statistical properties of the original distribution [S1, unofficial]. The method is part of the broader effort to improve the efficiency and quality of 3D Gaussian Splatting by controlling the density of Gaussians based on reconstruction error [S1, unofficial].

## Sources
[S1] Flexible Techniques for Differentiable Rendering with 3D Gaussians, https://doi.org/10.48550/arxiv.2308.14737, fetched 2026-10-26 (unofficial)
[S2] Mini-Splatting: Representing Scenes with a Constrained Number of Gaussians, https://doi.org/10.1007/978-3-031-72980-5_10, fetched 2026-10-26
