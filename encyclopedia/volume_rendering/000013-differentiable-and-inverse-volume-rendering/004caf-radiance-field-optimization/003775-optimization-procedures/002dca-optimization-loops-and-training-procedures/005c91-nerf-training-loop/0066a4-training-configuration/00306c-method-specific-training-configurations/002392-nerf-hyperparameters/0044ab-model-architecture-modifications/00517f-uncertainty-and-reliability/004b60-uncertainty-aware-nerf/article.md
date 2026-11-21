# Uncertainty-aware NeRF

Also called: U-NeRF

## Overview
Uncertainty-aware NeRF (U-NeRF) is a neural radiance field variant that renders RGB-D images with explicit uncertainty estimates for both color and depth, addressing artifacts and redundancy in synthetic data for Scene Coordinate Regression (SCR) [S1]. It differs from standard NeRF by modeling rendered pixel values as Gaussian distributions rather than deterministic points, and from NeRF-W by theoretically grounding the variance aggregation in squared weights rather than standard deviations [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera poses, training images (RGB and optionally depth), and a set of rays $\mathcal{R}$ sampled from these images.
*   **Outputs:** For each ray $r$, the method outputs the expected color $\hat{C}(r)$, color variance $\hat{\sigma}_C^2(r)$, expected depth $\hat{z}(r)$, and depth variance $\hat{\sigma}_z^2(r)$ [S1].

**Data Structures:**
The scene is represented by a single Multilayer Perceptron (MLP).
*   **MLP Architecture:** The MLP takes a 3D position $\mathbf{x} \in \mathbb{R}^3$ and viewing direction $\mathbf{d} \in \mathbb{R}^3$, transformed by positional encoding $\phi(\cdot)$, as input.
*   **MLP Output:** The network outputs a tuple $(\tau, \bar{\mathbf{c}}, \bar{\sigma}_c^2)$ for each sampled point along the ray [S1]:
    *   $\tau$: Volume density (scalar).
    *   $\bar{\mathbf{c}}$: View-dependent mean RGB color (vector of size 3).
    *   $\bar{\sigma}_c^2$: Pre-Softplus variance prediction for color (scalar).

**Numbered Steps:**

1.  **Ray Sampling:** For a camera ray $r(t) = \mathbf{o} + t\mathbf{d}$ traversing range $[t_n, t_f]$, partition the range into $N$ bins and sample $N$ points $\mathbf{x}_i$ uniformly within each bin [S1]. Let $t_i$ be the depth (distance from origin) of the $i$-th sample.

2.  **MLP Evaluation:** For each sample $\mathbf{x}_i$ and direction $\mathbf{d}$, compute the MLP output:
    $$ (\tau_i, \bar{\mathbf{c}}_i, \bar{\sigma}_{c_i}^2) = \text{MLP}(\phi(\mathbf{x}_i, \mathbf{d})) $$
    [S1]

3.  **Variance Activation:** Apply the Softplus activation function to ensure the predicted color variance is positive:
    $$ \sigma_{c_i}^2 = \text{Softplus}(\bar{\sigma}_{c_i}^2) $$
    [S1]

4.  **Volume Rendering Weights:** Compute the transmittance $T_i$ and weight $w_i$ for each sample, identical to standard NeRF [S1]:
    $$ \delta_i = t_{i+1} - t_i $$
    $$ T_i = \exp\left(-\sum_{j=1}^{i-1} \tau_j \delta_j\right) $$
    $$ w_i = T_i (1 - \exp(-\tau_i \delta_i)) $$
    [S1]

5.  **Rendered Color Statistics:** Assuming independent Gaussian distributions for sampled points, compute the expected color $\hat{C}(r)$ and its variance $\hat{\sigma}_C^2(r)$ [S1]:
    $$ \hat{C}(r) = \sum_{i=1}^{N} w_i \bar{\mathbf{c}}_i $$
    $$ \hat{\sigma}_C^2(r) = \sum_{i=1}^{N} w_i^2 \sigma_{c_i}^2 $$
    [S1]
    *Note: The rendered color is modeled as $\hat{C} \sim \mathcal{N}(\hat{C}(r), \hat{\sigma}_C^2(r))$ [S1].*

6.  **Rendered Depth Statistics:** Compute the expected depth $\hat{z}(r)$ and its variance $\hat{\sigma}_z^2(r)$ [S1]:
    $$ \hat{z}(r) = \sum_{i=1}^{N} w_i t_i $$
    $$ \hat{\sigma}_z^2(r) = \sum_{i=1}^{N} w_i (t_i - \hat{z}(r))^2 $$
    [S1]
    *Note: The ray termination distribution is assumed Gaussian [S1].*

7.  **Loss Computation:**
    *   **Color Loss ($L_{color}$):** Use the $\zeta$-NLL loss to prevent premature convergence in hard-to-fit regions [S1]:
        $$ L(r) = \frac{\|\mathbf{C}(r) - \hat{C}(r)\|_2^2}{2\hat{\sigma}_C^2(r)} + \log \hat{\sigma}_C(r) $$
        $$ L_{color} = \sum_{r \in \mathcal{R}} \lfloor \hat{\sigma}_C(r) \rfloor^{-2\zeta} L(r) $$
        Where $\lfloor \cdot \rfloor$ denotes the stop-gradient operator, preventing gradients from flowing through the variance term during backpropagation [S1].
    *   **Depth Loss ($L_{depth}$):** Use standard NLL on the depth output [S1]:
        $$ L_{depth} = \sum_{r \in \mathcal{R}} \frac{\|\hat{z}(r) - z(r)\|_2^2}{2\hat{\sigma}_z^2(r)} + \log \hat{\sigma}_z(r) $$
        Where $z(r)$ is the target depth [S1].
    *   **Overall Loss:**
        $$ L_{overall} = L_{color} + \lambda L_{depth} $$
        [S1]

8.  **Optimization:** Minimize $L_{overall}$ using gradient descent. The method typically employs depth-guided sampling to reduce training cost [S1].

**Parameters:**
*   $\zeta$: Interpolation parameter for the color loss weighting. $\zeta=0$ corresponds to standard NLL; $\zeta=1$ corresponds to MSE. It acts as an adaptive learning rate factor [S1].
*   $\lambda$: Weight for the depth loss term in the overall objective [S1].
*   $N$: Number of sampled points along the ray [S1].

**Edge Cases and Pitfalls:**
*   **Premature Convergence:** Standard NLL minimization leads to sub-optimal mean fits because badly-fit regions receive increasingly less weight as their uncertainty grows. This is mitigated by the stop-gradient variance-weighting term $\lfloor \hat{\sigma}_C(r) \rfloor^{-2\zeta}$ [S1].
*   **Theoretical Grounding:** NeRF-W applies volume rendering over $\sigma_C$ (standard deviation) instead of $\sigma_C^2$ (variance). This method asserts that NeRF-W's approach is not theoretically grounded, as the sum of independent Gaussians results in a variance that is the sum of squared weights times variances, not standard deviations [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

def softplus(x):
    return torch.nn.functional.softplus(x)

def volume_rendering_weights(tau, delta):
    """
    Computes transmittance T and weights w for volume rendering.
    tau: [N] volume density at each sample
    delta: [N] distance between samples
    Returns:
        T: [N] transmittance before each sample
        w: [N] weight for each sample
    """
    # T_i = exp(-sum_{j=1}^{i-1} tau_j * delta_j)
    # Accumulate negative optical depth
    neg_optical_depth = torch.cumsum(tau * delta, dim=0)
    # Shift to get T_i (transmittance arriving at sample i)
    # T_0 = 1 (implicitly), T_i for i>0 is exp(-sum_{j=0}^{i-1})
    # Using shift: T[i] = exp(-sum_{j=0}^{i-1} tau[j]*delta[j])
    # torch.roll with fill_value=0.0 for the first element being 1.0 before exp
    # Actually, standard implementation:
    # alpha = 1 - exp(-tau * delta)
    # T = exp(-cumsum(tau * delta[:-1])) # Careful with indices
    
    # Let's follow the formula strictly:
    # T_i = exp(- sum_{j=1}^{i-1} tau_j delta_j)
    # For i=1, sum is empty -> T_1 = 1.
    
    # Compute cumulative sum of tau * delta
    cum_tau_delta = torch.cumsum(tau * delta, dim=0)
    
    # T_i corresponds to exp(-cum_tau_delta[i-1])
    # For i=0 (first sample), T_0 = 1.0
    T = torch.cat([torch.ones_like(tau[:1]), torch.exp(-cum_tau_delta[:-1])])
    
    # w_i = T_i * (1 - exp(-tau_i * delta_i))
    alpha = 1.0 - torch.exp(-tau * delta)
    w = T * alpha
    
    return T, w

def compute_color_uncertainty(c_mean, c_var, w):
    """
    Computes rendered color mean and variance.
    c_mean: [N, 3] mean color at each sample
    c_var: [N] variance at each sample
    w: [N] volume rendering weights
    Returns:
        C_hat: [3] rendered color mean
        sigma_C_sq: [1] rendered color variance
    """
    C_hat = torch.sum(w.unsqueeze(-1) * c_mean, dim=0)
    # Variance is sum of w_i^2 * sigma_c_i^2
    sigma_C_sq = torch.sum(w**2 * c_var)
    return C_hat, sigma_C_sq

def compute_depth_uncertainty(t, w):
    """
    Computes rendered depth mean and variance.
    t: [N] depth (distance) at each sample
    w: [N] volume rendering weights
    Returns:
        z_hat: [1] rendered depth mean
        sigma_z_sq: [1] rendered depth variance
    """
    z_hat = torch.sum(w * t)
    # Variance is sum of w_i * (t_i - z_hat)^2
    sigma_z_sq = torch.sum(w * (t - z_hat)**2)
    return z_hat, sigma_z_sq

def nll_loss(mean, var, target):
    """
    Computes Negative Log-Likelihood loss for Gaussian distribution.
    mean: predicted mean
    var: predicted variance
    target: ground truth
    Returns:
        loss: scalar NLL loss
    """
    # L = ||target - mean||^2 / (2 * var) + log(sqrt(var))
    # Note: log(sigma) = 0.5 * log(var)
    # The source formula uses log(sigma_hat(r)), which is 0.5 * log(var)
    # However, standard NLL for N(mu, sigma^2) is:
    # 0.5 * log(2*pi*sigma^2) + 0.5 * (x-mu)^2 / sigma^2
    # The source formula (2) is: ||C-C_hat||^2 / (2*sigma^2) + log(sigma_hat)
    # This matches 0.5 * (x-mu)^2 / sigma^2 + 0.5 * log(var)
    
    diff_sq = torch.sum((target - mean)**2)
    loss = diff_sq / (2.0 * var) + 0.5 * torch.log(var)
    return loss

def train_step(mlp, rays, targets, zeta, lambda_depth):
    """
    Single training step for U-NeRF.
    mlp: The MLP model
    rays: List of rays, each containing (origin, direction, t_samples)
    targets: Dict with 'color' [N_rays, 3] and 'depth' [N_rays, 1]
    zeta: Hyperparameter for color loss weighting
    lambda_depth: Weight for depth loss
    """
    mlp.train()
    
    total_color_loss = 0.0
    total_depth_loss = 0.0
    
    for ray_idx, (origin, direction, t_samples) in enumerate(rays):
        # 1. Sample points and get positions
        # t_samples is [N]
        positions = origin + direction.unsqueeze(0) * t_samples.unsqueeze(-1) # [N, 3]
        
        # 2. MLP Forward Pass
        # Input: positions and direction
        # Output: tau, c_mean, c_var_pre
        tau, c_mean, c_var_pre = mlp(positions, direction) # Shapes: [N], [N, 3], [N]
        
        # 3. Activate Variance
        c_var = softplus(c_var_pre)
        
        # 4. Compute Weights
        # delta is difference between consecutive t_samples
        delta = torch.diff(t_samples, dim=0)
        # Pad delta to match N? No, w has N elements.
        # Standard NeRF: delta_i = t_{i+1} - t_i. 
        # If we have N samples, we have N-1 intervals? 
        # The source says "partition [tn, tf] into N bins and sample uniformly in each bin".
        # Usually N samples implies N weights. 
        # Let's assume delta is computed such that it aligns with weights.
        # If t_samples has N elements, delta usually has N-1 or N elements depending on boundary.
        # Source Eq 1: w_i = T_i (1 - exp(-tau_i delta_i)).
        # If N samples, we need N deltas. Typically delta_i is the width of the bin containing sample i.
        # Let's assume delta is passed or computed as bin widths.
        # For simplicity in pseudocode, assume delta is available or computed from t_samples boundaries.
        # Here we assume delta is [N].
        
        T, w = volume_rendering_weights(tau, delta)
        
        # 5. Render Color
        C_hat, sigma_C_sq = compute_color_uncertainty(c_mean, c_var, w)
        
        # 6. Render Depth
        z_hat, sigma_z_sq = compute_depth_uncertainty(t_samples, w)
        
        # 7. Compute Losses
        target_color = targets['color'][ray_idx]
        target_depth = targets['depth'][ray_idx]
        
        # Color NLL component
        L_color_nll = nll_loss(C_hat, sigma_C_sq, target_color)
        
        # Apply Zeta-NLL weighting with stop-gradient
        # sigma_C is sqrt(sigma_C_sq)
        sigma_C = torch.sqrt(sigma_C_sq)
        # Stop gradient on sigma_C
        sigma_C_sg = sigma_C.detach()
        
        # Weighting factor: sigma_C^{-2*zeta}
        weight_factor = sigma_C_sg ** (-2 * zeta)
        
        L_color = weight_factor * L_color_nll
        
        # Depth NLL
        L_depth = nll_loss(z_hat, sigma_z_sq, target_depth)
        
        total_color_loss += L_color
        total_depth_loss += L_depth
        
    # Backpropagation
    loss = total_color_loss + lambda_depth * total_depth_loss
    loss.backward()
    
    return loss.item()
```

## Subtopics

### Scene Reliability Annotation
A semi-automatic preprocessing method used in conjunction with NeRF variants to mark background regions, moving occluders, saturated highlights, and strongly reflective areas as low-reliability regions [S2]. These regions are converted into binary masks which are propagated into the cross-view weighting term during optimization, ensuring that inconsistent evidence from these areas contributes less to the reconstruction [S2]. This reduces viewpoint-dependent artifacts and improves spatial geometric fidelity compared to standard NeRF baselines that exclude such masks [S2].

## Operating characteristics
*   **Time Complexity:** Training time is comparable to standard NeRF, though the additional variance predictions add minor computational overhead to the MLP forward pass. Rendering remains efficient as it involves simple weighted sums [S1].
*   **Memory Complexity:** The MLP must store an additional scalar output channel for variance per sample point, increasing model size negligibly [S1].
*   **Strengths:** Provides pixel-level reliability metrics, allowing for the filtering of poorly rendered images and weighting of pixels in downstream tasks like Scene Coordinate Regression [S1]. The $\zeta$-NLL loss prevents the model from ignoring hard-to-fit regions, leading to better mean fits than standard NLL [S1].
*   **Limitations:** Requires careful tuning of $\zeta$ and $\lambda$. The assumption of independent Gaussian distributions for sampled points may not hold in all complex scenes [S1].
*   **When to use:** When uncertainty estimation is required for downstream applications such as visual localization, active view selection, or filtering synthetic training data [S1].

## Background
Uncertainty-aware NeRF (U-NeRF) was introduced to address challenges in Scene Coordinate Regression (SCR), specifically the high cost of obtaining diverse 2D-3D ground truth and the inefficiency of training on large redundant datasets [S1]. By leveraging NeRF to render RGB-D data with uncertainty, the method enables uncertainty-guided novel view selection, allowing the SCR network to be trained with a small portion of the most informative synthetic samples [S1]. The approach builds on the theoretical foundation that the rendered pixel color, as a weighted sum of independent Gaussian distributions, is itself Gaussian [S1].

## Sources
[S1] Leveraging Neural Radiance Fields for Uncertainty-Aware Visual Localization, https://doi.org/10.1109/icra57147.2024.10610126, fetched 2026-10-26
[S2] High fidelity three dimensional reconstruction and interactive visualization of public art sculptures using neural radiance fields, https://link.springer.com/article/10.1007/s44163-026-02441-3, fetched 2026-10-26
