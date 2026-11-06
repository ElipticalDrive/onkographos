# Gradient Scaling

Also called: Sampling density based gradient scaling, Transmittance-based gradient scaling

## Overview
Gradient Scaling is a technique used in differentiable volume rendering to mitigate near-camera overfitting and the formation of "floaters" (spurious geometry artifacts) during radiance field optimization. It addresses the problem of gradient imbalance along rays, where samples far from the camera receive significantly smaller gradients than those near the camera due to the exponential decay of transmittance [S1]. Unlike [Distortion Loss with Power Transformation](4560-distortion-loss-with-power-transformation.md), which modifies the loss function to penalize metric distance, Gradient Scaling modifies the optimization process itself by rescaling gradients during backpropagation to ensure uniform gradient magnitude across all samples on a ray [S1].

## How it works
### Inputs and Outputs
*   **Inputs:**
    *   Ray $\mathbf{r}$ defined by origin and direction.
    *   Set of $N$ samples along the ray, indexed $i=1 \dots N$.
    *   For each sample $i$:
        *   Volume density $\sigma_i$.
        *   Distance to adjacent samples $\delta_i$ (typically constant step size).
        *   Spherical Harmonic (SH) coefficients $(k_\ell^m)_i$ for color representation.
        *   Transmittance $T_i$ up to sample $i$.
    *   Current rendered color $\hat{C}(\mathbf{r})$.
    *   Target color $C(\mathbf{r})$ (for photometric loss).
*   **Outputs:**
    *   Scaled gradients for SH coefficients $\nabla_{(k_\ell^m)_i} \mathcal{L}$ ready for the optimizer step.

### Data Structures
*   **Transmittance $T_i$**: A scalar value for each sample $i$, representing the probability of light passing through the volume from the camera to sample $i$.
    *   $T_1 = 1$ (assuming the first sample is at the camera origin or the start of the integration range).
    *   $T_{i+1} = T_i \cdot (1 - \alpha_i)$.
*   **Opacity $\alpha_i$**: Scalar value for each sample $i$.
    *   $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$.

### Formulas
The core mechanism relies on the observation that the gradient of the rendered color with respect to the color component $\mathbf{c}_i$ (derived from SH coefficients) is proportional to the transmittance $T_i$ [S1]:

$$
\nabla_{\mathbf{c}_i} \hat{C}(\mathbf{r}) = T_i \alpha_i
$$

Since the SH coefficients $(k_\ell^m)_i$ determine the color $\mathbf{c}_i$ via the SH basis function $Y_\ell^m$, the gradient with respect to the SH coefficients is:

$$
\nabla_{(k_\ell^m)_i} \hat{C}(\mathbf{r}) = Y_\ell^m \cdot \nabla_{\mathbf{c}_i} \hat{C}(\mathbf{r}) = Y_\ell^m \cdot T_i \alpha_i
$$

The photometric loss gradient $\nabla_{(k_\ell^m)_i} \mathcal{L}_{MSE}$ is proportional to $\nabla_{(k_\ell^m)_i} \hat{C}(\mathbf{r})$. Consequently, without scaling, the gradient magnitude decays exponentially with distance from the camera because $T_i$ decays exponentially [S1].

**Gradient Scaling Rule:**
To counteract this, the gradients for the SH coefficients are multiplied by the reciprocal of the transmittance $1/T_i$ [S1]:

$$
\nabla_{(k_\ell^m)_i}^{\text{scaled}} \mathcal{L}_{MSE} = \frac{1}{T_i} \cdot \nabla_{(k_\ell^m)_i} \mathcal{L}_{MSE}
$$

Substituting the proportionality:

$$
\nabla_{(k_\ell^m)_i}^{\text{scaled}} \mathcal{L}_{MSE} \propto \frac{1}{T_i} \cdot (Y_\ell^m \cdot T_i \alpha_i) = Y_\ell^m \cdot \alpha_i
$$

This results in gradients that are proportional only to the opacity $\alpha_i$ and the SH basis value, effectively balancing the gradient magnitude across samples regardless of their depth [S1].

**Sampling Density Based Gradient Scaling (Alternative/Complementary):**
The source also describes a related technique based on sampling density $\rho(\mathbf{p})$, which is inversely proportional to the square of the distance $\zeta_{\mathbf{p}}^i$ from the camera origin [S1]:

$$
\rho(\mathbf{p}) \approx \sum_{i=1}^n v_i(\mathbf{p}) \times \frac{1}{(\zeta_{\mathbf{p}}^i)^2}
$$

where $n$ is the number of cameras and $v_i(\mathbf{p})$ is a visibility function (1 if point $\mathbf{p}$ is in the $i$-th camera's field of view, 0 otherwise). This scaling affects both volume density $\sigma$ and SH coefficients $k$, whereas transmittance scaling primarily affects SH coefficients [S1].

### Pseudocode
The following pseudocode implements the transmittance-based gradient scaling for the SH coefficients during the optimization step. It assumes the standard volume rendering pipeline has computed $T_i$ and $\alpha_i$.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_gradient_scaling(sh_coeff_gradients, transmittances, opacities, sh_basis_values):
    """
    Applies transmittance-based gradient scaling to SH coefficient gradients.
    
    Args:
        sh_coeff_gradients: List of gradient tensors for SH coefficients for each sample i.
                            Shape: [N, num_coeffs].
        transmittances: List of transmittance values T_i for each sample i.
                        Shape: [N].
        opacities: List of opacity values alpha_i for each sample i.
                   Shape: [N].
        sh_basis_values: List of SH basis function values Y_l^m for each sample i.
                         Shape: [N, num_coeffs].
                         
    Returns:
        Scaled gradients for SH coefficients.
    """
    N = len(transmittances)
    scaled_gradients = []
    
    for i in range(N):
        T_i = transmittances[i]
        
        # Guard against division by zero if T_i is extremely close to 0
        # Although T_1=1, T_i approaches 0 for far samples.
        # In practice, T_i is rarely exactly 0 due to float precision, 
        # but a small epsilon prevents NaNs.
        if T_i < 1e-7:
            # If transmittance is negligible, the gradient contribution is negligible.
            # Scaling by 1/T_i would explode. 
            # However, the theoretical derivation suggests the gradient is proportional to alpha_i.
            # If T_i -> 0, alpha_i -> 0 (assuming sigma is finite).
            # We clamp the scaling factor to avoid numerical instability.
            scale_factor = 1.0 / max(T_i, 1e-7)
        else:
            scale_factor = 1.0 / T_i
            
        # Original gradient is proportional to Y_l^m * T_i * alpha_i
        # Scaled gradient = scale_factor * original_gradient
        #               = (1/T_i) * (Y_l^m * T_i * alpha_i)
        #               = Y_l^m * alpha_i
        
        # Note: The input sh_coeff_gradients already contains the full gradient 
        # including the T_i factor from the backpropagation of the loss.
        # We simply multiply by 1/T_i.
        
        current_grad = sh_coeff_gradients[i]
        scaled_grad = current_grad * scale_factor
        
        scaled_gradients.append(scaled_grad)
        
    return scaled_gradients

def volume_render_forward(samples_sigma, samples_sh_coeffs, ray_directions, step_size):
    """
    Standard volume rendering forward pass to compute T_i and alpha_i.
    This is a helper to provide context for the scaling application.
    """
    N = len(samples_sigma)
    alphas = []
    transmittances = []
    
    T_prev = 1.0
    transmittances.append(T_prev)
    
    for i in range(N):
        sigma = samples_sigma[i]
        delta = step_size[i] # Distance between adjacent samples
        
        # Alpha calculation
        alpha = 1.0 - exp(-sigma * delta)
        alphas.append(alpha)
        
        # Transmittance update
        T_next = T_prev * (1.0 - alpha)
        transmittances.append(T_next)
        T_prev = T_next
        
    return alphas, transmittances
```

### Parameters
*   **Scaling Factor**: $1/T_i$.
*   **Application Stage**: Early stage of training [S1].
*   **Target**: SH coefficients $(k_\ell^m)_i$ [S1].

### Edge Cases and Pitfalls
1.  **Division by Zero**: If $T_i$ becomes extremely small (near 0) for samples far from the camera, $1/T_i$ can cause numerical instability or NaNs. A small epsilon floor should be applied to $T_i$ before inversion [G].
2.  **Initial Density**: If the initial volume density $\sigma$ is very small, $T_i$ hardly attenuates (remains close to 1). In this case, the scaling factor $1/T_i \approx 1$, and the gradient imbalance is not corrected, potentially leading to slow optimization or non-convergence [S1].
3.  **Sampling Density vs. Transmittance**: Sampling density based scaling affects both $\sigma$ and $k$, while transmittance-based scaling primarily affects $k$ (SH coefficients) because transmittance attenuation primarily influences SH coefficient gradients at the beginning [S1].
4.  **Uniform Sampling**: Near-camera overfitting persists even on datasets with uniform sampling density (e.g., LLFF) due to view transformation, contradicting the hypothesis that sampling imbalance alone causes the issue. Transmittance decay is the primary culprit [S1].

## Operating characteristics
*   **Time Complexity**: Negligible extra computational cost. Transmittance $T_i$ is already computed during the forward pass of volume rendering. The scaling is a simple element-wise multiplication [S1].
*   **Memory Complexity**: No additional memory required beyond storing existing transmittance values [S1].
*   **Performance**:
    *   On the LLFF dataset (3-view), Plenoxels with Transmittance-based Gradient Scaling achieved PSNR 17.90, SSIM 0.585, LPIPS 0.393, compared to baseline Plenoxels PSNR 16.83, SSIM 0.521, LPIPS 0.438 [S1].
*   **Strengths**:
    *   Effectively reduces near-camera floaters.
    *   Generalizable to other radiance field representations [S1].
    *   Low computational overhead.
*   **Limitations**:
    *   Primarily effective in the early stages of training.
    *   Requires careful handling of numerical stability for $T_i \approx 0$.

## Background
Gradient Scaling was proposed in the context of few-shot view synthesis, specifically in the F2Plenoxels method [S1]. Previous works attributed near-camera overfitting to sampling density imbalance [S1]. However, F2Plenoxels demonstrated that this issue persists even with uniform sampling density, identifying exponential transmittance decay as the root cause. The method was validated on synthetic (Blender) and real (LLFF, Mip-NeRF 360) datasets [S1].

## Sources
[S1] F2Plenoxels: fast voxel radiance fields without neural networks for few-shot view synthesis, https://doi.org/10.1007/s44443-026-00491-2, fetched 2026-10-26
