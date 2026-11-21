# SHINE-Mapping Loss Function

Also called: Incremental Mapping Loss, Projected Signed Distance BCE Loss

## Overview
The SHINE-Mapping Loss Function is a composite objective for training neural Signed Distance Field (SDF) representations using LiDAR range data. It addresses the specific challenge of **incremental mapping**, where a model must update its geometric representation with new sensor data without suffering from **catastrophic forgetting** of previously observed areas [S1]. Unlike vision-based methods that rely on volume-rendered depth, this method uses direct projected signed distances from range sensors for supervision, mapping them to a binary occupancy probability via a sigmoid function and optimizing with Binary Cross Entropy (BCE) [S1]. It distinguishes itself from sibling methods like the [Surface SDF Loss](14235-surface-sdf-loss.md) by explicitly incorporating a regularization term ($L_r$) that constrains parameter updates based on their sensitivity to historical data, enabling scalable, large-scale 3D mapping [S1].

## How it works

### Inputs and Outputs
*   **Inputs**:
    *   Neural network parameters $\theta$ (defining $f_\theta(\mathbf{x})$).
    *   A batch of sampled query points $\mathcal{Q} = \{\mathbf{x}_i\}$, where $\mathbf{x}_i \in \mathbb{R}^3$. These points are sampled along LiDAR rays.
    *   For each $\mathbf{x}_i$, the **projected signed distance** $d_i$ to the surface (distance to the beam endpoint).
    *   Hyperparameters: $\sigma$ (sigmoid sharpness/noise magnitude), $\lambda_e$ (Eikonal weight), $\lambda_r$ (forgetting prevention weight).
    *   Historical parameters $\theta^*$ (snapshot of weights from previous training steps) and the set of previously important parameters $A$ [S1].
*   **Outputs**:
    *   A scalar loss value $\mathcal{L}$ used for backpropagation.
    *   Updated network parameters $\theta$.

### Data Structures
*   **Network Output**: $f_\theta(\mathbf{x}_i)$ returns a scalar SDF value $s_i$.
*   **Supervision Label**: $l_i \in [0, 1]$, derived from $d_i$.
*   **Prediction**: $o_i \in [0, 1]$, derived from $s_i$.
*   **Parameter Mask/Importance**: A set $A$ of indices or a mask indicating parameters critical for previous tasks [S1].

### Formulas and Definitions

1.  **Sigmoid Mapping Function**:
    Maps signed distance $x$ to $[0, 1]$.
    $$ S(x) = \frac{1}{1 + e^{x/\sigma}} $$
    Where $\sigma$ controls the flatness of the function and indicates the magnitude of measurement noise [S1].

2.  **Supervision Label ($l_i$)**:
    $$ l_i = S(d_i) $$
    This represents the target occupancy probability for point $\mathbf{x}_i$ [S1].

3.  **Network Prediction ($o_i$)**:
    $$ o_i = S(f_\theta(\mathbf{x}_i)) $$
    The model's predicted occupancy probability [S1].

4.  **Binary Cross Entropy Loss ($L_{bce}$)**:
    $$ L_{bce} = l_i \cdot \log(o_i) + (1 - l_i) \cdot \log(1 - o_i) $$
    Optimizes the predicted occupancy against the sigmoid-mapped projected signed distance [S1].

5.  **Eikonal Loss ($L_{eikonal}$)**:
    Enforces the unit gradient magnitude constraint typical for SDFs.
    $$ L_{eikonal} = \left( \|\nabla f_\theta(\mathbf{x}_i)\|_2 - 1 \right)^2 $$
    Computed via automatic differentiation [S1].

6.  **Forgetting Prevention Regularization ($L_r$)**:
    Limits updates to parameters that are not critical for previous data.
    $$ L_r = \sum_{i \in A} (\theta_i - \theta^*_i)^2 $$
    Where $A$ is the set of parameters, and $\theta^*$ are the historical weights [S1]. *Note: The source formula uses $\Omega_i$ implicitly as the identity or importance weight for parameters in set $A$.*

7.  **Total Loss ($L_{incr}$)**:
    $$ L_{incr} = L_{bce} + \lambda_e L_{eikonal} + \lambda_r L_r $$
    Used for incremental mapping [S1].

    For batch-based (non-incremental) training, the loss simplifies to:
    $$ L_{batch} = L_{bce} + \lambda_e L_{eikonal} $$

### Parameters
*   $\sigma$: Controls sigmoid flatness. Not given in the sources.
*   $\lambda_e$: Weight for Eikonal loss. Not given in the sources.
*   $\lambda_r$: Weight for forgetting prevention. Not given in the sources.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def sigmoid_sdf(x, sigma):
    """
    Maps signed distance to [0, 1] using sigmoid.
    Args:
        x: Tensor of signed distances.
        sigma: Hyperparameter for flatness/noise magnitude.
    Returns:
        Tensor of values in [0, 1].
    """
    return 1.0 / (1.0 + torch.exp(x / sigma))

def compute_shine_loss(f_theta, points, projected_distances, sigma, lambda_e, lambda_r, theta_star, important_params_mask):
    """
    Computes the SHINE-Mapping loss.
    
    Args:
        f_theta: Neural network function f_theta(x).
        points: Tensor of query points (N, 3).
        projected_distances: Tensor of projected signed distances to surface (N,).
        sigma: Sigmoid sharpness parameter.
        lambda_e: Weight for Eikonal loss.
        lambda_r: Weight for forgetting prevention.
        theta_star: Historical parameters (snapshot) for forgetting prevention.
        important_params_mask: Boolean mask or set of indices for parameters in A.
        
    Returns:
        Scalar loss tensor.
    """
    # 1. Get raw SDF predictions
    s = f_theta(points)  # Shape: (N,)
    
    # 2. Compute supervision labels and predictions via sigmoid
    # l_i = S(d_i)
    labels = sigmoid_sdf(projected_distances, sigma)
    # o_i = S(f_theta(x_i))
    predictions = sigmoid_sdf(s, sigma)
    
    # 3. Binary Cross Entropy Loss
    # L_bce = l * log(o) + (1-l) * log(1-o)
    # Note: torch.nn.BCELoss expects probabilities in [0,1]
    # Adding small epsilon for numerical stability in log
    eps = 1e-7
    predictions_clamped = torch.clamp(predictions, eps, 1 - eps)
    labels_clamped = torch.clamp(labels, eps, 1 - eps)
    
    l_bce = F.binary_cross_entropy(predictions_clamped, labels_clamped)
    
    # 4. Eikonal Loss
    # Requires gradient of f_theta w.r.t points
    points.requires_grad_(True)
    grad_output = torch.ones_like(s)
    gradients = torch.autograd.grad(
        s, points, grad_output, create_graph=True, retain_graph=True
    )[0]
    gradient_magnitude = torch.norm(gradients, dim=1, keepdim=True) # Shape: (N, 1)
    l_eikonal = torch.mean((gradient_magnitude - 1.0) ** 2)
    
    # 5. Forgetting Prevention Regularization (L_r)
    # L_r = sum_{i in A} (theta_i - theta_star_i)^2
    # Assuming theta is a list of tensors or a single concatenated tensor.
    # Here we assume f_theta.parameters() returns an iterable of tensors.
    # We need to align with theta_star which must have the same structure.
    
    l_r = 0.0
    if lambda_r > 0:
        for param, param_star in zip(f_theta.parameters(), theta_star):
            # Apply mask if important_params_mask is provided per parameter
            if important_params_mask is not None:
                # Example: mask is a boolean tensor of same shape as param
                # Only sum squared diff for important params
                diff = param - param_star
                masked_diff = diff * important_params_mask
                l_r += torch.sum(masked_diff ** 2)
            else:
                # If no specific mask, assume all params are in A or mask is implicit
                # Based on source: "sum_{i in A} (theta_i - theta_star_i)^2"
                # If A is all params, just sum all.
                diff = param - param_star
                l_r += torch.sum(diff ** 2)
                
    # 6. Total Loss
    loss = l_bce + lambda_e * l_eikonal + lambda_r * l_r
    
    return loss
```

### Edge Cases and Pitfalls
*   **Catastrophic Forgetting**: In incremental mapping, updating features with new data focuses only on reducing the current loss, ignoring previous areas. This leads to a decline in reconstruction accuracy for previously mapped regions [S1]. The term $L_r$ mitigates this by limiting update directions based on parameter sensitivity [S1].
*   **Precision Far from Surface**: Precise SDF values far from a surface have little impact on geometry. The sigmoid mapping $S(x)$ handles this by soft truncation, giving higher impact to points closer to the endpoint (where $d_i \approx 0$) [S1].
*   **Measurement Noise**: The parameter $\sigma$ indicates the magnitude of measurement noise. Incorrect $\sigma$ may lead to suboptimal flatness in the supervision signal [S1].

## Operating characteristics

*   **Time Complexity**: Dominated by the forward pass of the neural network and the computation of gradients via automatic differentiation for the Eikonal term. The regularization term $L_r$ is $O(|A|)$ where $|A|$ is the number of important parameters.
*   **Memory Complexity**: Requires storing the historical parameters $\theta^*$ for the forgetting prevention term, effectively doubling the memory footprint for parameters during incremental updates [S1].
*   **Strengths**:
    *   Uses direct range readings (projected signed distance) for more accurate supervision compared to volume-rendered depth [S1].
    *   Explicitly handles catastrophic forgetting in incremental mapping scenarios [S1].
    *   Sigmoid mapping provides robust supervision by focusing on surface-proximal points [S1].
*   **Limitations**:
    *   Requires careful tuning of $\lambda_r$ to balance new information retention vs. historical preservation [S1].
    *   Relies on accurate LiDAR range measurements; performance may degrade with noisy or sparse range data if $\sigma$ is not adjusted [S1].
*   **When to Use**: Large-scale 3D mapping using LiDAR where incremental updates are necessary and catastrophic forgetting is a concern [S1].
*   **When Not to Use**: When only visual (RGB) data is available (use vision-based volume rendering approaches) or when offline batch training is feasible and forgetting is not an issue (use $L_{batch}$) [S1].

## Background

The SHINE-Mapping Loss Function was introduced in the paper "SHINE-Mapping: Large-Scale 3D Mapping Using Sparse Hierarchical Implicit Neural Representations" [S1]. It was developed to address the limitations of replay-based methods in large-scale implicit incremental mapping, which are computationally expensive or impractical [S1]. The method builds on the principles of continual learning, specifically regularization-based approaches, to prevent catastrophic forgetting in feature grid-based implicit representations [S1]. It contrasts with vision-based approaches that use volume rendering depth supervision by leveraging the accuracy of LiDAR range sensors [S1].

## Sources

[S1] SHINE-Mapping: Large-Scale 3D Mapping Using Sparse Hierarchical Implicit Neural Representations, https://doi.org/10.1109/icra48891.2023.10160907, fetched 2026-10-26
