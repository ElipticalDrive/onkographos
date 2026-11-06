# Regularised Gaussian Fitting Loss

Also called: L1+Regularization Loss for Gaussians, Regularised Loss Function for Gaussian Fitting

## Overview
The Regularised Gaussian Fitting Loss is a composite objective function used to optimize Gaussian splat representations of volumetric data by combining a data-fidelity term with penalties on splat amplitude and extent. It addresses the problem of overfitting and instability in differentiable volume rendering by constraining the learned Gaussian parameters to physically plausible ranges. It distinguishes itself from sibling regularization methods like Total Variation by focusing specifically on the statistical moments (amplitude and covariance diagonal) of the Gaussian primitives rather than spatial smoothness of the field.

## How it works
The method minimizes a total loss $\mathcal{L}$ composed of a reconstruction loss and two regularization terms. The optimization updates the Gaussian splat parameters (amplitude, diagonal covariance, position) to match a target volume.

**Inputs:**
- Target volume data.
- Current Gaussian splat parameters: amplitude $a_k$, diagonal covariance matrix entries $L_{k,ii}$ for $K$ splats in $D$ dimensions.
- Hyperparameters: learning rate $lr$, regularization weights $\lambda_a, \lambda_d$, minimum covariance floor $\sigma_{min}$.

**Outputs:**
- Gradients for the splat parameters to be passed to the optimizer (e.g., Adam).

**Data Structures:**
- Splat parameters are stored as tensors. The diagonal covariance entries $L_{k,ii}$ are typically softplus-activated parameters to ensure positivity, with a floor $\sigma_{min}$.
- Amplitudes $a_k$ may be subject to a clamp $a_{max}$ during evaluation but the regularization is applied to the raw parameters before clamping.

**Formulas:**
The total loss is defined as [S1, unofficial]:
$$
\mathcal{L} = \mathcal{L}_{recon} + \lambda_a \sum_{k=1}^{K} |a_k| + \lambda_d \sum_{k=1}^{K} \sum_{i=1}^{D} |L_{k,ii} - \sigma_{min}|
$$
Where:
- $\mathcal{L}_{recon}$ is the L1 reconstruction loss between the predicted volume (rendered from the Gaussians) and the target volume.
- $K$ is the number of splats.
- $D$ is the dimensionality of the volume (typically 3).
- $a_k$ is the amplitude of splat $k$.
- $L_{k,ii}$ is the $i$-th diagonal entry of the covariance matrix for splat $k$.
- $\sigma_{min}$ is the minimum floor for the covariance diagonal entries.
- $\lambda_a$ and $\lambda_d$ are the regularization weights.

**Parameters:**
- $\lambda_a = 0.1 \times lr$. With a standard learning rate $lr = 0.01$, $\lambda_a = 10^{-3}$ [S1, unofficial].
- $\lambda_d = 0.01 \times lr$. With $lr = 0.01$, $\lambda_d = 10^{-4}$ [S1, unofficial].
- $\sigma_{min}$: Not given in the sources.
- $\tau$ (truncation threshold): Used in the optional boundary term, but not in the primary loss. Not given in the sources for the primary loss.

**Optional Boundary Term:**
An optional boundary-containment term $\mathcal{L}_{boundary}$ can be added with weight $\lambda_b$:
$$
\mathcal{L}_{boundary} = \sum_{k=1}^{K} \sum_{i=1}^{D} \left[ \max(0, r_{k,i} - \mu_{k,i})^2 + \max(0, r_{k,i} - (S_i - 1 - \mu_{k,i}))^2 \right]
$$
Where:
- $r_{k,i} = \tau \sqrt{\sum_j L_{k,ij}}$ is the effective radius of splat $k$ along dimension $i$ at truncation threshold $\tau$.
- $\mu_{k,i}$ is the center of splat $k$ along dimension $i$.
- $S_i$ is the volume extent along dimension $i$.
- This term is disabled ($\lambda_b = 0$) in reported benchmarks [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def regularised_gaussian_fitting_loss(target_volume, predicted_volume, amplitudes, cov_diagonals, lr, sigma_min):
    """
    Computes the Regularised Gaussian Fitting Loss.
    
    Args:
        target_volume: Tensor of target voxel values.
        predicted_volume: Tensor of rendered voxel values from Gaussians.
        amplitudes: Tensor of shape (K,) containing raw amplitude parameters a_k.
        cov_diagonals: Tensor of shape (K, D) containing raw diagonal covariance parameters L_{k,ii}.
        lr: Learning rate scalar.
        sigma_min: Minimum floor for covariance diagonal.
        
    Returns:
        Total loss scalar.
    """
    # 1. L1 Reconstruction Loss
    # L_recon is the mean L1 error between predicted and target volumes
    l_recon = torch.mean(torch.abs(target_volume - predicted_volume))
    
    # 2. Regularization Weights
    # lambda_a = 0.1 * lr
    # lambda_d = 0.01 * lr
    lambda_a = 0.1 * lr
    lambda_d = 0.01 * lr
    
    # 3. Amplitude Regularization
    # Sum of absolute amplitudes over K splats
    # Note: Applied to raw parameters before amplitude clamp a_max
    l_amp = torch.sum(torch.abs(amplitudes))
    
    # 4. Extent (Covariance Diagonal) Regularization
    # Sum of absolute differences between diagonal entries and sigma_min
    # Applied to softplus-activated parameters above the floor sigma_min
    # cov_diagonals are assumed to be the raw parameters that are softplus-activated
    # The term is |L_{k,ii} - sigma_min|
    l_ext = torch.sum(torch.abs(cov_diagonals - sigma_min))
    
    # 5. Total Loss
    # L = L_recon + lambda_a * sum(|a_k|) + lambda_d * sum(|L_{k,ii} - sigma_min|)
    # The regularization terms are means over K splats and D entries in the source formula context,
    # but the summation notation implies summing. The text says "Both penalties are means over the K splats...".
    # However, the formula shows summation. We follow the formula structure but normalize if "mean" is strictly required.
    # The source says: "lambda_a sum |a_k| + lambda_d sum sum |L_{k,ii} - sigma_min|"
    # And "Both penalties are means over the K splats (and D diagonal entries)".
    # This implies the loss terms should be divided by K and D respectively, or the lambda absorbs it.
    # Given lambda is small, we assume the formula sums, but the text says "means".
    # To be safe and consistent with "means over K splats", we divide by K.
    # For D, it says "means over ... D diagonal entries".
    
    # Re-interpreting "means over K splats (and D diagonal entries)":
    # L_reg_a = (1/K) * sum(|a_k|)
    # L_reg_d = (1/(K*D)) * sum(sum(|L_{k,ii} - sigma_min|))
    
    K = amplitudes.shape[0]
    D = cov_diagonals.shape[1]
    
    l_amp_mean = torch.sum(torch.abs(amplitudes)) / K
    l_ext_mean = torch.sum(torch.abs(cov_diagonals - sigma_min)) / (K * D)
    
    total_loss = l_recon + lambda_a * l_amp_mean + lambda_d * l_ext_mean
    
    return total_loss
```

**Edge Cases and Pitfalls:**
- The regularization is applied to the raw parameters before any amplitude clamping ($a_{max}$), ensuring the weight does not grow with splat count $K$ as the pool is swept [S1, unofficial].
- The covariance diagonal entries are softplus-activated parameters; the regularization term $|L_{k,ii} - \sigma_{min}|$ operates on these parameters above the floor $\sigma_{min}$ [S1, unofficial].
- L1 loss is preferred over MSE or Poisson deviance for reconstruction, as MSE performs worse on held-out PSNR for these regularised, early-stopped fits [S1, unofficial].

## Subtopics

### [Progressive Multi-Pass Fitting](3185-progressive-multi-pass-fitting.md)
A strategy where a nominal splat budget is distributed across sequential passes, fitting residuals left by frozen earlier passes. It is evaluated as an alternative to single-pass fitting within this framework, trading off some PSNR for potential SSIM gains.

## Operating characteristics
- **Time Complexity:** The loss computation is $O(K \cdot D)$ for the regularization terms and $O(N)$ for the reconstruction loss (where $N$ is the number of voxels). The optimization uses standard Adam, which has constant tensor shapes due to fixed-pool relocation, preserving vectorized efficiency [S1, unofficial].
- **Memory Complexity:** Requires storage for $K$ splat parameters (amplitude, covariance diagonal, position) and the target/predicted volumes.
- **Performance:** L1 reconstruction loss beats MSE on 11 of 17 volumes by +0.20 to +7.78 dB in held-out PSNR and never trails by more than 0.28 dB [S1, unofficial].
- **Strengths:** Prevents overfitting by constraining amplitude and extent; L1 loss provides robust reconstruction.
- **Limitations:** Single-pass fitting generally achieves higher training PSNR than progressive variants; the boundary term is disabled in benchmarks, so boundary artifacts may occur if not handled elsewhere.
- **When to use:** When optimizing Gaussian splats for volumetric data where stability and physical plausibility of splat sizes are important.
- **When not to use:** If maximum PSNR is the sole metric and progressive fitting is not an option, single-pass with L1 is preferred.

## Background
The method originates from the Luxar project, which applies Gaussian splatting to microscopy and scientific data visualization. The specific loss formulation was detailed in an unofficial preprint/report on Luxar [S1, unofficial]. It was designed to handle the specific challenges of volumetric data, where resolution varies relative to splat size (resolution over splat FWHM 1.15–8.7, median 2.4) [S1, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
