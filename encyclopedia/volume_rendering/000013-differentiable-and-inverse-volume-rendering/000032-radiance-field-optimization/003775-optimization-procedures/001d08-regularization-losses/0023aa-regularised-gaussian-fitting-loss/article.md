# Regularised Gaussian Fitting Loss

Also called: L1+Regularization Loss for Gaussians, Regularised Loss Function for Gaussian Fitting

## Overview
This method defines the objective function for optimizing a set of $K$ Gaussian splats to reconstruct a target volume, combining data fidelity with structural regularization [S1, unofficial]. It differs from standard reconstruction losses by explicitly penalizing extreme splat amplitudes and extents to prevent overfitting and ensure stable optimization [S1, unofficial]. It is typically used in single-pass benchmarks for differentiable volume rendering [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: Target volume data, current set of $K$ Gaussian splats. Each splat $k$ has parameters including diagonal covariance entries $L_{k,ii}$ (related to extent) and amplitude parameter $a_k$ [S1, unofficial].
- **Output**: A scalar loss value $L$ used for gradient-based optimization [S1, unofficial].

**Data Structures**
- **Splat Parameters**:
  - $L_{k,ii}$: Diagonal covariance entries for splat $k$ along dimension $i$ [S1, unofficial].
  - $a_k$: Amplitude parameter for splat $k$ [S1, unofficial].
  - $\mu_{k,i}$: Center coordinate of splat $k$ along dimension $i$ [S1, unofficial].
  - $S_i$: Volume extent (size) along dimension $i$ [S1, unofficial].
- **Constants**:
  - $\tau$: Truncation threshold for effective radius calculation [S1, unofficial].
  - $\sigma_{min}$: Minimum floor for covariance softplus [S1, unofficial].
  - $\lambda_a, \lambda_d, \lambda_b$: Regularization weights [S1, unofficial].

**Formulas**
The total loss is defined as [S1, unofficial]:

$$
L = L_{recon} + \lambda_a \sum_{k=1}^K |a_k| + \lambda_d \sum_{k=1}^K \sum_{i=1}^D |L_{k,ii} - \sigma_{min}|
$$

Where:
- $L_{recon}$ is the L1 reconstruction loss between the predicted volume (rendered from the Gaussians) and the target volume [S1, unofficial].
- $\lambda_a = 0.1 \times lr$ regularizes the splat amplitude [S1, unofficial].
- $\lambda_d = 0.01 \times lr$ regularizes the splat extent (diagonal covariance) [S1, unofficial].
- $lr$ is the learning rate (typically $0.01$) [S1, unofficial].
- $D$ is the number of dimensions (typically $3$) [S1, unofficial].
- The regularization terms are computed as means over the $K$ splats and $D$ diagonal entries [S1, unofficial].
- The terms $|a_k|$ and $|L_{k,ii} - \sigma_{min}|$ are evaluated on the softplus-activated parameters before any amplitude clamp $a_{max}$ is applied [S1, unofficial].

**Optional Boundary Term**
An optional boundary-containment term can be added:
$$
L_{boundary} = \sum_{k=1}^K \sum_{i=1}^D \left[ \max(0, r_{k,i} - \mu_{k,i})^2 + \max(0, r_{k,i} - (S_i - 1 - \mu_{k,i}))^2 \right]
$$
Where $r_{k,i} = \tau \sqrt{\sum_j L_{k,ij}}$ is the effective radius of splat $k$ along dimension $i$ at truncation threshold $\tau$ [S1, unofficial]. This term is disabled ($\lambda_b = 0$) in standard benchmarks [S1, unofficial].

**Parameters**
- $\lambda_a$: Regularization weight for amplitude. Set to $10^{-3}$ (assuming $lr=0.01$) [S1, unofficial].
- $\lambda_d$: Regularization weight for extent. Set to $10^{-4}$ (assuming $lr=0.01$) [S1, unofficial].
- $\lambda_b$: Weight for boundary term. Set to $0$ (disabled) [S1, unofficial].
- $lr$: Learning rate. Typically $0.01$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def softplus(x):
    return math.log(math.exp(x) + 1.0)

def regularised_gaussian_fitting_loss(target_volume, predicted_volume, splats, lr, sigma_min, tau, volume_size):
    """
    Computes the Regularised Gaussian Fitting Loss.
    
    Args:
        target_volume: List/Array of target voxel values.
        predicted_volume: List/Array of rendered voxel values from splats.
        splats: List of dictionaries containing:
            - 'a': amplitude parameter (raw)
            - 'L_diag': diagonal covariance entries [D]
            - 'mu': center coordinates [D]
            - 'L_full': full covariance matrix [D, D] (for radius calc)
        lr: Learning rate.
        sigma_min: Minimum floor for covariance softplus.
        tau: Truncation threshold.
        volume_size: Size of volume along each dimension [D].
        
    Returns:
        loss: Scalar loss value.
    """
    K = len(splats)
    D = len(splats[0]['mu'])
    
    # 1. L1 Reconstruction Loss
    l_recon = sum(abs(t - p) for t, p in zip(target_volume, predicted_volume)) / len(target_volume)
    
    # 2. Amplitude Regularization (Mean over K splats)
    sum_amplitude = sum(abs(s['a']) for s in splats)
    l_amp = sum_amplitude / K
    
    # 3. Extent Regularization (Mean over K splats and D entries)
    sum_extent = 0.0
    for s in splats:
        for i in range(D):
            val = softplus(s['L_diag'][i])
            sum_extent += abs(val - sigma_min)
    l_extent = sum_extent / (K * D)
    
    # 4. Combine Losses
    lambda_a = 0.1 * lr
    lambda_d = 0.01 * lr
    
    loss = l_recon + lambda_a * l_amp + lambda_d * l_extent
    
    # Optional Boundary Term (Disabled by default in benchmarks)
    # lambda_b = 0
    # if lambda_b > 0:
    #     l_boundary = 0.0
    #     for s in splats:
    #         for i in range(D):
    #             r_ki = tau * math.sqrt(sum(s['L_full'][i][j] for j in range(D)))
    #             term1 = max(0.0, r_ki - s['mu'][i]) ** 2
    #             term2 = max(0.0, r_ki - (volume_size[i] - 1 - s['mu'][i])) ** 2
    #             l_boundary += term1 + term2
    #     loss += lambda_b * (l_boundary / (K * D))
        
    return loss
```

**Edge Cases and Pitfalls**
- **Amplitude Clamp**: The regularization terms are evaluated *before* any amplitude clamp $a_{max}$ is applied, ensuring the weight does not grow with $K$ as splat count increases [S1, unofficial].
- **Softplus Floor**: The extent regularization uses the softplus-activated covariance values, with the penalty measuring deviation from the floor $\sigma_{min}$ [S1, unofficial].
- **L1 vs MSE**: L1 is preferred over MSE for training PSNR in this context, as MSE performs worse on held-out PSNR for 11 of 17 volumes and is not the best training loss for early-stopped fits [S1, unofficial].

## Subtopics

### [Progressive Multi-Pass Fitting](3185-progressive-multi-pass-fitting.md)
Distributes splat budget across sequential passes, optimizing residuals left by frozen earlier passes. Trades marginal PSNR gains for improved SSIM on complex structures.

## Operating characteristics
- **Time Complexity**: Depends on the number of splats $K$, volume resolution, and rendering samples. The loss computation itself is $O(K \cdot D)$ for regularization terms plus the cost of rendering $L_{recon}$ [S1, unofficial].
- **Memory Complexity**: Constant with respect to splat count for optimizer states due to a fixed-pool relocation mechanism [S1, unofficial].
- **Performance**: L1 reconstruction loss yields better held-out PSNR than MSE on most volumes (11 of 17), with improvements ranging from $+0.20$ to $+7.78$ dB [S1, unofficial].
- **Strengths**: Prevents overfitting via amplitude and extent regularization; stable optimization with standard Adam; L1 loss provides robust reconstruction [S1, unofficial].
- **Limitations**: Requires tuning of $\lambda_a$ and $\lambda_d$ relative to learning rate; boundary term is optional and not used in standard benchmarks [S1, unofficial].

## Background
The loss function is used in differentiable volume rendering optimization pipelines to fit Gaussian splat representations to volumetric data [S1, unofficial]. It balances reconstruction fidelity with structural constraints to enable efficient rendering and visualization [S1, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
