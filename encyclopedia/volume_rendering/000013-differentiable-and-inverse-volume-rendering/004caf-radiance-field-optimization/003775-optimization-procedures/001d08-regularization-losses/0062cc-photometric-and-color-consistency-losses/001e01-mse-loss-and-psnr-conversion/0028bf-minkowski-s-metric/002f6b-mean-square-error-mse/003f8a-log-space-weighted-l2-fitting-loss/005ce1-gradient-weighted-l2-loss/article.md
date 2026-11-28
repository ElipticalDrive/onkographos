# Gradient-weighted L2 Loss

Also called: Gradient-weighted loss (reweighted L2), RawNeRF loss, Weighted L2 Loss for HDR, reweighted L2 loss

## Overview
Gradient-weighted L2 Loss is a regularization technique for differentiable volume rendering that stabilizes training on high dynamic range (HDR) and noisy raw images. It approximates the gradient of a tonemapped loss (such as L2 on log-transformed values) by weighting the standard L2 residual by the derivative of the tone curve, allowing stable convergence where standard L2 or L1 losses diverge [S2]. It sets itself apart from standard MSE by dynamically suppressing gradients in bright regions and enhancing supervision in dark regions, addressing the inadequacy of uniform weighting in scenes with large intensity variations [S1].

## How it works
**Inputs and Outputs**
- **Inputs**: Rendered color $\hat{y}_i$ (scalar or vector per pixel/channel), reference noisy color $y_i$, small constant $\epsilon$.
- **Output**: Scalar loss value $L_\psi$.

**Data Structures**
- $\hat{y}_i$: Rendered color value at pixel $i$.
- $y_i$: Ground truth noisy color value at pixel $i$.
- $\epsilon$: Numerical stability constant.

**Formula**
The loss is defined as:
$$ L_\psi(\hat{y}, y) = \sum_i \left( \frac{\hat{y}_i - y_i}{\text{sg}(\hat{y}_i) + \epsilon} \right)^2 $$
where $\text{sg}(\cdot)$ is the stop-gradient operator [S1, S2].

**Derivation**
The method approximates the loss $L_\psi(\hat{y}, y) = \sum_i (\psi(\hat{y}_i) - \psi(y_i))^2$ using a first-order Taylor expansion around $\hat{y}_i$ [S2]:
$$ \psi(\hat{y}_i) - \psi(y_i) \approx \psi'(\hat{y}_i)(\hat{y}_i - y_i) $$
Substituting this into the squared error yields the weighted L2 form with weight $w_i = \psi'(\hat{y}_i)$ [S2]. The stop-gradient is applied to $\hat{y}_i$ in the denominator to prevent the gradient of the weight from flowing back through the network, which ensures the loss remains a stable regularizer rather than a divergent primary loss [S2].

**Parameters**
- $\epsilon$: $1 \times 10^{-3}$ [S1, S2].
- Loss Weight: The gradient-weighted loss is typically added as a regularizer with a weight between $1 \times 10^{-2}$ and $1 \times 10^{-1}$ relative to the primary rendering loss [S2]. Higher weights are used for noisier or darker scenes to suppress "floater" artifacts [S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gradient_weighted_l2_loss(rendered_color, target_color, epsilon=1e-3):
    """
    Computes the gradient-weighted L2 loss.
    
    Args:
        rendered_color (Tensor): Rendered color values \hat{y}.
        target_color (Tensor): Noisy reference color values y.
        epsilon (float): Numerical stability constant.
        
    Returns:
        Tensor: Scalar loss value.
    """
    # Compute residual
    residual = rendered_color - target_color
    
    # Apply stop-gradient to rendered color for the denominator
    # This prevents the gradient of the weighting term from affecting the network weights
    stable_denominator = rendered_color.detach() + epsilon
    
    # Compute weighted squared residual
    # Note: In practice, this is often summed over pixels/channels
    loss = (residual / stable_denominator) ** 2
    
    return loss.sum()
```

**Edge Cases and Pitfalls**
- **Divergence**: Standard unweighted L2 or L1 losses cause the model to diverge early in training, particularly in very noisy scenes [S2].
- **Log Tone Curve**: Directly applying a log tone curve before the L2 loss (i.e., computing L2 on $\log(\hat{y})$ and $\log(y)$) causes training to diverge [S2].
- **sRGB Gamma**: Using an unclipped sRGB gamma curve (extended linearly below zero and exponentially above 1) causes many color artifacts in dark regions [S2].
- **NaN Gradients**: Standard L2 losses are prone to producing NaN gradient values in high-noise data, requiring careful gradient clipping [S2]. The reweighted loss is more stable [S2].

## Operating characteristics
- **Stability**: Significantly more stable than standard L2 or L1 losses in noisy data, avoiding NaN gradients and divergence [S2].
- **Sharpness**: Applying the regularizer with a high weight can result in a minor loss of sharpness, which can be ameliorated by annealing its weight from 0 to 1 over the course of training [S2].
- **Noise Handling**: Specifically designed to handle high levels of noise in raw images, making it suitable for HDR view synthesis from noisy RAW images [S1, S2].
- **Limitations**: The method cannot scale to arbitrary amounts of noise; extremely noisy scenes may still require multiple COLMAP runs or parameter tuning for camera poses [S2].

## Background
The method was introduced in the context of "NeRF in the Dark" (RawNeRF) by Mildenhall et al. (2022) [S2]. It was developed to address the challenge of training neural radiance fields on high dynamic range, noisy raw images where standard photometric losses fail. The approach leverages the derivative of a tone curve to reweight the L2 loss, providing a stable approximation to tonemapped losses without the instability of direct tonemapping in the loss function [S2]. It is also referenced in subsequent works such as "Lighting Every Darkness with 3DGS" [S1].

## Sources
[S1] Lighting Every Darkness with 3DGS: Fast Training and Real-Time Rendering for HDR View Synthesis, https://doi.org/10.52202/079017-2549, fetched 2026-10-26
[S2] NeRF in the Dark: High Dynamic Range View Synthesis from Noisy Raw Images, https://doi.org/10.1109/cvpr52688.2022.01571, fetched 2026-10-26
