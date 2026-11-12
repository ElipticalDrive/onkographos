# Multi-scale Joint Training Loss

Also called: Coarse-Fine Joint Loss, Joint Loss, Multi-scale Joint Loss

## Overview
A composite loss function used in multi-stage or multi-scale radiance field optimization to balance low-frequency structural consistency with high-frequency detail fidelity. It differs from single-scale losses by simultaneously optimizing outputs from multiple network stages or resolution levels, ensuring that coarse approximations do not hinder fine detail reconstruction. It builds on [Total Loss Function](4177-total-loss-function.md).

## How it works
The method defines a total loss $\mathcal{L}$ as a weighted sum of losses computed at each scale or stage $i$. Two primary formulations exist in the literature, often used in conjunction with dynamic network growth or multi-resolution training.

### Inputs and Outputs
- **Inputs**: Network parameters $\theta$, query coordinates (often passed through all outlets/stages), ground truth images or reference views, and hyperparameters $\alpha, \beta_1, \beta_2, D$.
- **Outputs**: A scalar loss value $\mathcal{L}$ used for gradient backpropagation.

### Data Structures
- **Stages/Layers**: A set of $D$ network stages or resolution levels. In dynamic growth methods, $D$ increases over time [S1].
- **Loss Components**:
  - $\mathcal{L}_{MSE}^i$: Mean Squared Error loss for stage $i$.
  - $\mathcal{L}_{unct}^i$: Uncertainty loss for stage $i$ [S1].
  - $L_{coarse}(\theta)$: Low-frequency consistency loss [S2].
  - $L_{fine}(\theta)$: High-frequency detail consistency loss [S2].

### Formulas
Two distinct formulations are cited:

1. **Multi-stage MSE and Uncertainty Loss** [S1]:
   $$ \mathcal{L} = \sum_{i=1}^D \beta_1 \mathcal{L}_{MSE}^i + \beta_2 \mathcal{L}_{unct}^i $$
   Where:
   - $D$: Current number of stages (also equal to $D-1$ times network growth iterations) [S1].
   - $\mathcal{L}_{MSE}^i$: MSE loss of the output image of layer $i$ [S1].
   - $\mathcal{L}_{unct}^i$: Uncertainty loss of the output image of layer $i$ [S1].
   - $\beta_1, \beta_2$: Weights. Typical values: $\beta_1 = 1.0, \beta_2 = 0.1$ [S1].

2. **Coarse-Fine Joint Loss** [S2]:
   $$ L(\theta) = \alpha L_{coarse}(\theta) + (1-\alpha) L_{fine}(\theta), \quad \alpha \in [0, 1] $$
   Where:
   - $L_{coarse}(\theta)$: Low-frequency consistency term comparing input and smoothed output, preserving tone and large-scale structure [S2].
   - $L_{fine}(\theta)$: Detail consistency term comprising pixel-level robust distance, structural similarity, gradient consistency, and chroma consistency [S2].

### Parameters
- $\beta_1 = 1.0, \beta_2 = 0.1$ [S1].
- $\alpha \in [0, 1]$ [S2].
- $D$: Dynamic, determined by the training stage [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_multi_scale_joint_loss(network_outputs, ground_truth, uncertainty_maps, alpha, beta1, beta2, D):
    """
    Computes the multi-scale joint training loss.
    
    Args:
        network_outputs: List of rendered images for each stage i=1..D.
        ground_truth: Reference image(s).
        uncertainty_maps: List of uncertainty maps for each stage i=1..D.
        alpha: Weight for coarse loss (used in Coarse-Fine formulation).
        beta1: Weight for MSE loss (used in Multi-stage formulation).
        beta2: Weight for uncertainty loss (used in Multi-stage formulation).
        D: Number of stages.
        
    Returns:
        Total loss scalar.
    """
    total_loss = 0.0
    
    # Formulation 1: Multi-stage MSE and Uncertainty [S1]
    # Note: Sources [S1] and [S2] describe different loss compositions.
    # Implementation depends on the specific variant used.
    
    # If using the MSE/Unct formulation from [S1]:
    mse_loss_sum = 0.0
    unct_loss_sum = 0.0
    
    for i in range(1, D + 1):
        # Compute MSE between stage i output and ground truth
        # [G] Assuming standard pixel-wise MSE
        mse_i = mean_squared_error(network_outputs[i-1], ground_truth)
        mse_loss_sum += mse_i
        
        # Compute uncertainty loss for stage i
        # [G] Assuming uncertainty loss is defined as per standard NeRF uncertainty formulations
        # e.g., based on the variance predicted by the network
        unct_i = compute_uncertainty_loss(uncertainty_maps[i-1], ground_truth)
        unct_loss_sum += unct_i
        
    loss_mse_unct = beta1 * mse_loss_sum + beta2 * unct_loss_sum
    
    # If using the Coarse-Fine formulation from [S2]:
    # L_coarse = compute_low_frequency_loss(network_outputs, ground_truth)
    # L_fine = compute_detail_consistency_loss(network_outputs, ground_truth)
    # loss_coarse_fine = alpha * L_coarse + (1 - alpha) * L_fine
    
    # [G] The sources do not specify if these are additive or mutually exclusive.
    # Typically, one variant is chosen per method. We return the MSE/Unct variant as primary example.
    return loss_mse_unct
```

### Edge Cases and Pitfalls
- **Dynamic Growth**: $D$ changes during training. The loss summation must adapt to the current number of stages [S1].
- **Uncertainty Loss**: Requires the network to output uncertainty estimates, not just color/density [S1].
- **Smoothing**: $L_{coarse}$ requires smoothing operations on input/output images [S2].

## Operating characteristics
- **Time Complexity**: Linear in the number of stages $D$ and the number of pixels, as losses are summed over all stages [S1].
- **Memory Complexity**: Requires storing outputs and uncertainties for all $D$ stages simultaneously during the backward pass [S1].
- **Strengths**: Balances global structure and local detail; robust to noise via uncertainty weighting [S1, S2].
- **Limitations**: Requires careful tuning of $\alpha, \beta_1, \beta_2$ [S1, S2].

## Background
Originated in dynamic NeRF methods where networks grow iteratively [S1] and in denoising frameworks for 3D Gaussian Splatting [S2]. It addresses the trade-off between structural fidelity and high-frequency detail in radiance field optimization.

## Sources
[S1] Recursive-NeRF: An Efficient and Dynamically Growing NeRF, https://doi.org/10.1109/tvcg.2022.3204608, fetched 2026-10-26
[S2] Denoise-GS: Self-Supervised Denoising for Sparse-View 3D Gaussian Splatting., https://doi.org/10.3390/s26020651, fetched 2026-10-26
