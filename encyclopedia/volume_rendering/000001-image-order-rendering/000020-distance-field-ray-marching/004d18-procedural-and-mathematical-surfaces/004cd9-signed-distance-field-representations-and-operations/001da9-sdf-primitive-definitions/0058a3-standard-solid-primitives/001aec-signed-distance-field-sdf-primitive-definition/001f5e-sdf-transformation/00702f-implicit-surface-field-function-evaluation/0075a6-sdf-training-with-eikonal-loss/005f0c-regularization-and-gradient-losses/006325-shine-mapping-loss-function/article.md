# SHINE-Mapping Loss Function

Also called: Incremental Mapping Loss, Projected Signed Distance BCE Loss

## Overview
The SHINE-Mapping Loss Function is a composite regularization and supervision scheme designed for training neural Signed Distance Field (SDF) representations using direct range sensor data (e.g., LiDAR). It addresses the specific challenge of incremental mapping by combining a Binary Cross Entropy (BCE) loss on sigmoid-mapped projected signed distances with Eikonal regularization and a parameter-sensitivity-based forgetting prevention term [S1]. Unlike vision-based methods that rely on volume-rendered depth, this method uses direct geometric projections for more accurate supervision [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A neural network $f_\theta(\mathbf{x})$ predicting an SDF value $s \in \mathbb{R}$ for a query point $\mathbf{x} \in \mathbb{R}^3$; a batch of sampled 3D points $\{\mathbf{x}_i\}$; the corresponding projected signed distances to the surface $d_i$ derived from range sensor readings; and hyperparameters $\sigma, \lambda_e, \lambda_r$.
*   **Output:** A scalar loss value $\mathcal{L}_{incr}$ used for gradient-based optimization of $\theta$.

**Data Structures**
*   $\theta$: Network parameters (weights and biases).
*   $\mathbf{x}_i$: Query point coordinates.
*   $d_i$: Ground truth projected signed distance for $\mathbf{x}_i$.
*   $\theta^*$: Snapshot of network parameters from a previous training step or initialization (used for historical reference).
*   $\Omega_i$: Importance weight for parameter $\theta_i$, derived from the sensitivity of the loss on historical data to changes in $\theta_i$.

**Formulas**
1.  **Sigmoid Mapping:** The projected signed distance $d$ is mapped to $[0, 1]$ using a sigmoid function controlled by noise magnitude $\sigma$ [S1]:
    $$ S(x) = \frac{1}{1 + e^{x/\sigma}} $$
2.  **Supervision Label:** The target label $l_i$ for point $\mathbf{x}_i$ is:
    $$ l_i = S(d_i) $$
3.  **Network Output:** The predicted occupancy indicator $o_i$ is the sigmoid of the raw SDF output:
    $$ o_i = S(f_\theta(\mathbf{x}_i)) $$
4.  **Binary Cross Entropy (BCE) Loss:** Measures the discrepancy between predicted and target occupancy probabilities [S1]:
    $$ \mathcal{L}_{bce} = l_i \cdot \log(o_i) + (1 - l_i) \cdot \log(1 - o_i) $$
5.  **Eikonal Loss:** Enforces the unit gradient magnitude property of SDFs via automatic differentiation [S1]:
    $$ \mathcal{L}_{eikonal} = \left( \|\nabla f_\theta(\mathbf{x}_i)\|_2 - 1 \right)^2 $$
6.  **Forgetting Prevention Regularization ($\mathcal{L}_r$):** Limits updates to parameters that are critical for previous data. It penalizes deviations from previous parameters $\theta^*$ weighted by importance $\Omega_i$ [S1]:
    $$ \mathcal{L}_r = \sum_{i \in A} \Omega_i (\theta_i - \theta_i^*)^2 $$
    Where $A$ is the set of parameters.
7.  **Total Incremental Loss:**
    $$ \mathcal{L}_{incr} = \mathcal{L}_{bce} + \lambda_e \mathcal{L}_{eikonal} + \lambda_r \mathcal{L}_r $$

**Parameters**
*   $\sigma$: Controls the flatness of the sigmoid and indicates measurement noise magnitude [S1]. Not given in the sources.
*   $\lambda_e$: Weight for the Eikonal loss [S1]. Not given in the sources.
*   $\lambda_r$: Weight for the forgetting prevention term, adjusting the degree of keeping historical knowledge [S1]. Not given in the sources.

**Numbered Steps**
1.  Sample a point $\mathbf{x}_i$ along a sensor ray.
2.  Calculate the projected signed distance $d_i$ from $\mathbf{x}_i$ to the beam endpoint (surface intersection).
3.  Compute the supervision label $l_i = S(d_i)$.
4.  Compute the network output $s_i = f_\theta(\mathbf{x}_i)$.
5.  Compute the predicted occupancy $o_i = S(s_i)$.
6.  Compute $\mathcal{L}_{bce}$ using $l_i$ and $o_i$.
7.  Compute the gradient $\nabla f_\theta(\mathbf{x}_i)$ via automatic differentiation.
8.  Compute $\mathcal{L}_{eikonal}$ using the gradient norm.
9.  Compute $\mathcal{L}_r$ using current parameters $\theta$, previous parameters $\theta^*$, and importance weights $\Omega$.
10. Combine terms to form $\mathcal{L}_{incr}$.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def shine_mapping_loss(f_theta, x_batch, d_batch, theta_star, omega, sigma, lambda_e, lambda_r):
    """
    Computes the SHINE-Mapping incremental loss.
    
    Args:
        f_theta: Neural network function f_theta(x) returning scalar SDF.
        x_batch: Tensor of shape (N, 3) containing query points.
        d_batch: Tensor of shape (N,) containing projected signed distances.
        theta_star: Tensor of previous network parameters (same shape as f_theta params).
        omega: Tensor of importance weights for each parameter (same shape as f_theta params).
        sigma: Float, sigmoid temperature/noise magnitude.
        lambda_e: Float, Eikonal loss weight.
        lambda_r: Float, Forgetting prevention weight.
        
    Returns:
        Scalar loss value.
    """
    # 1. Compute Sigmoid function
    def sigmoid(x, s):
        return 1.0 / (1.0 + torch.exp(-x / s)) # Note: Source uses exp(x/s) in denominator, equivalent to 1/(1+exp(x/s))
        # Correction based on source formula S(x) = 1/(1 + e^{x/σ}):
        # Standard sigmoid is 1/(1+e^-x). Source formula is 1/(1+e^{x/σ}).
        # This is equivalent to sigmoid(-x/σ).
    
    def source_sigmoid(x, s):
        return 1.0 / (1.0 + torch.exp(x / s))

    # 2. Compute Supervision Labels
    # l_i = S(d_i)
    l_batch = source_sigmoid(d_batch, sigma)
    
    # 3. Compute Network Outputs
    # s_i = f_theta(x_i)
    s_batch = f_theta(x_batch)
    
    # 4. Compute Predicted Occupancy
    # o_i = S(f_theta(x_i))
    o_batch = source_sigmoid(s_batch, sigma)
    
    # 5. Compute BCE Loss
    # L_bce = l * log(o) + (1 - l) * log(1 - o)
    # Add small epsilon for numerical stability in log
    eps = 1e-7
    o_clipped = torch.clamp(o_batch, eps, 1 - eps)
    l_clipped = torch.clamp(l_batch, eps, 1 - eps)
    
    l_bce = -(l_clipped * torch.log(o_clipped) + (1 - l_clipped) * torch.log(1 - o_clipped))
    l_bce = torch.mean(l_bce)
    
    # 6. Compute Eikonal Loss
    # Requires gradient of f_theta w.r.t x
    x_batch.requires_grad_(True)
    s_batch_grad = f_theta(x_batch)
    grad = torch.autograd.grad(
        s_batch_grad, x_batch, 
        grad_outputs=torch.ones_like(s_batch_grad), 
        create_graph=True
    )[0]
    grad_norm = torch.norm(grad, dim=1)
    l_eikonal = torch.mean((grad_norm - 1.0) ** 2)
    
    # 7. Compute Forgetting Prevention Loss
    # L_r = sum( omega_i * (theta_i - theta_star_i)^2 )
    # Note: omega and theta are flattened or matched in shape
    param_diff = f_theta.parameters() - theta_star
    l_r = torch.sum(omega * (param_diff ** 2))
    
    # 8. Combine Losses
    l_total = l_bce + lambda_e * l_eikonal + lambda_r * l_r
    
    return l_total
```

**Edge Cases and Pitfalls**
*   **Catastrophic Forgetting:** In incremental mapping, updating features with new data focuses only on reducing current loss, ignoring previous areas [S1]. The term $\mathcal{L}_r$ mitigates this by limiting update direction based on parameter sensitivity to previous data [S1].
*   **Precision:** The sigmoid mapping soft-truncates signed distance values, making precise SDF values far from the surface less impactful [S1].
*   **Importance Weights:** The calculation of $\Omega_i$ (parameter importance) is not explicitly detailed in the provided text, but is referenced as being derived from the sensitivity of previous losses [S1]. [G] Typically, this involves computing the Fisher Information Matrix or similar second-order derivatives of the loss with respect to parameters on a replay buffer or previous data distribution.

## Operating characteristics
*   **Time Complexity:** Dominated by the forward pass of $f_\theta$ and the backward pass for automatic differentiation of the gradient $\nabla f_\theta$. The Eikonal term adds a gradient-of-gradient computation cost.
*   **Memory Complexity:** Requires storing the previous parameter snapshot $\theta^*$ and importance weights $\Omega$, increasing memory overhead compared to standard SDF training.
*   **Strengths:** Provides accurate supervision using direct range readings (LiDAR) rather than inferred depth [S1]. Effectively handles incremental mapping by preventing catastrophic forgetting [S1].
*   **Limitations:** Relies on the availability of projected signed distances (requires ray-surface intersection or similar geometric processing). Performance may degrade if $\sigma$ is not tuned to the specific noise level of the sensor [S1].
*   **When to use:** Large-scale 3D mapping using sparse hierarchical implicit neural representations with LiDAR data, specifically in incremental/online settings [S1].

## Background
The SHINE-Mapping Loss Function was introduced in the SHINE-Mapping paper, which proposes a large-scale 3D mapping system using sparse hierarchical implicit neural representations [S1]. It was developed to address the limitations of replay-based methods in large-scale implicit incremental mapping, where storing all previous data is infeasible [S1]. The method builds on the principle that range sensors provide accurate range measurements, allowing for direct supervision signals [S1].

## Sources
[S1] SHINE-Mapping: Large-Scale 3D Mapping Using Sparse Hierarchical Implicit Neural Representations, https://doi.org/10.1109/icra48891.2023.10160907, fetched 2026-10-26
