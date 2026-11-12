# Raw2Alpha Density to Alpha Conversion

Also called: Raw2Alpha, Raw Density to Alpha, Raw to Alpha Conversion, Density to Alpha Conversion, Raw Density to Alpha Conversion, density to alpha

## Overview
Raw2Alpha converts raw network outputs (density $\sigma$) into opacity ($\alpha$) and rendering weights for volume rendering, enabling the integration of learned density fields into standard front-to-back compositing pipelines [S1, S2, S5, S7, unofficial]. It differs from its parent [Alpha-to-Weight Computation (Forward)](2373-alpha-to-weight-computation-forward.md) by performing the initial physical conversion from density to per-sample opacity, whereas the parent handles the subsequent accumulation of transmittance and color weights [S5, S7, unofficial]. Unlike [Alpha Computation with Alpha Mask](10549-alpha-computation-with-alpha-mask.md), which corrects occlusion for planar surfaces, Raw2Alpha operates on volumetric density fields [S2, S10, unofficial]. It also differs from [Alpha-to-Extinction Conversion](13044-alpha-to-extinction-conversion.md), which bridges transfer-function alpha with physical scattering models, by directly mapping raw neural density to alpha for general volume rendering [S2, S10, unofficial].

## How it works
**Inputs:**
- `density`: Raw density values $\sigma$, shape $(N, S)$ or $(N, H, W, S)$, where $N$ is batch/ray count, $S$ is sample count [S1, S5, S7, unofficial].
- `dist`: Inter-sample distances $\delta$, shape $(N, S)$ or broadcastable [S5, S7, unofficial].
- `shift`: Optional scalar or tensor added to density before exponentiation [S2, S6, unofficial].
- `interval`: Optional scalar or tensor representing sample spacing or activation interval [S2, S6, unofficial].
- `distance_scale`: Optional scalar scaling factor for distances [S5, unofficial].

**Outputs:**
- `alpha`: Opacity values $\alpha$, shape $(N, S)$ [S5, S7, unofficial].
- `weights`: Rendering weights $w$, shape $(N, S)$ [S5, S7, unofficial].
- `T`: Final transmittance, shape $(N, 1)$ [S5, unofficial].

**Data Structures:**
- Tensors for `density`, `dist`, `alpha`, `weights`, `T`.
- No complex bit-packing; standard floating-point tensors (float32/float16) [S5, S6, unofficial].

**Steps:**
1. **Preprocessing (Optional):** If `shift` is provided, compute $e = \exp(\text{density} + \text{shift})$ [S2, S6, unofficial]. If `distance_scale` is provided, scale distances: $\delta' = \text{dist} \times \text{distance\_scale}$ [S5, unofficial].
2. **Alpha Computation:** Compute opacity $\alpha$ using one of the following formulations based on the activation mode:
   - **Standard Exponential:** $\alpha = 1 - \exp(-\sigma \cdot \delta)$ [S5, S7, unofficial][S8].
   - **Softmax-based (Per-ray):** Apply softmax to $\sigma$ along the sample dimension to get $\alpha_{\text{softmax}}$, then $\alpha = 1 - e^{-\alpha_{\text{softmax}}}$ [S1, unofficial].
   - **Interval-based (NeRF-like):** $\alpha = 1 - (1 + e)^{-\text{interval}}$, where $e = \exp(\text{density} + \text{shift})$ [S2, S6, unofficial].
   - **Linear (Legacy):** $\alpha = \text{density} \times \text{sample\_step} \times \text{val\_threshold} \times \text{brightness}$ [S3, unofficial].
3. **Transmittance Calculation:** Compute cumulative transmittance $T$. Initialize $T_0 = 1.0$. For $i > 0$, $T_i = T_{i-1} \times (1 - \alpha_{i-1} + \epsilon)$, where $\epsilon = 10^{-10}$ prevents NaNs [S5, S7, unofficial]. In code:
   $$ T = \text{cumprod}([1.0, 1 - \alpha + \epsilon], \text{dim}=-1) $$
   [S5, S7, unofficial].
4. **Weight Computation:** Compute rendering weights $w_i = \alpha_i \times T_i$ (using $T$ up to index $i-1$) [S1, S5, S7, unofficial]. Specifically, $w = \alpha \times T[:, :-1]$ [S5, S7, unofficial].
5. **Final Transmittance:** Return $T[:, -1]$ as the background transmittance [S5, unofficial].

**Formulas:**
- Standard: $\alpha = 1 - e^{-\sigma \cdot \delta}$ [S5, S7, unofficial][S8].
- Interval: $\alpha = 1 - (1 + \exp(\sigma + \text{shift}))^{-\text{interval}}$ [S2, S6, unofficial].
- Weights: $w = \alpha \cdot T$ [S1, S5, unofficial].
- Transmittance: $T_i = \prod_{j=0}^{i-1} (1 - \alpha_j + \epsilon)$ [S5, S7, unofficial].

**Parameters:**
- `shift`: Float, added to density. Typical value: Not given in the sources.
- `interval`: Float or tensor, uniform distance or activation parameter. Typical value: Not given in the sources.
- `distance_scale`: Float, scales inter-sample distances. Typical value: Not given in the sources.
- `epsilon`: Float, $10^{-10}$, added to $(1-\alpha)$ for stability [S7, unofficial].
- `brightness`: Float, scaling factor for linear alpha. Typical value: 50.0 [S3, unofficial].
- `val_threshold`: Float, scales alpha contribution. Typical value: Not given in the sources.
- `sample_step`: Float, distance per sample. Typical value: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Overflow:** $\exp(\text{density} + \text{shift})$ can overflow if density + shift is large. Handled by clamping or subsequent pow behavior [S2, S6, unofficial].
- **NaNs:** Cumulative product of $(1-\alpha)$ can produce NaNs if $\alpha > 1$ or precision issues arise. Add $\epsilon = 10^{-10}$ [S5, S7, unofficial].
- **Boundary Checks:** Assumes volume is normalized to $[0, 1]$ for boundary checks in some implementations [S3, unofficial].
- **Gradient Stability:** In backward pass, clamp exponential values to $10^{10}$ to prevent overflow [S2, S6, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def raw2alpha_forward(sigma, dist, shift=None, interval=None, distance_scale=1.0, epsilon=1e-10):
    """
    Computes alpha, weights, and final transmittance from raw density.
    
    Args:
        sigma: Tensor of shape (N, S) or (N, H, W, S). Raw density.
        dist: Tensor of shape (N, S) or broadcastable. Inter-sample distances.
        shift: Optional float or tensor. Added to density.
        interval: Optional float or tensor. Activation interval.
        distance_scale: Float. Scales distances.
        epsilon: Float. Stability constant.
        
    Returns:
        alpha: Tensor of shape (N, S). Opacity.
        weights: Tensor of shape (N, S). Rendering weights.
        T_final: Tensor of shape (N, 1). Final transmittance.
    """
    # Scale distances
    delta = dist * distance_scale
    
    # Compute alpha based on available parameters
    if shift is not None and interval is not None:
        # Interval-based activation (NeRF-like)
        e = torch.exp(sigma + shift)
        alpha = 1.0 - torch.pow(1.0 + e, -interval)
    else:
        # Standard exponential activation
        alpha = 1.0 - torch.exp(-sigma * delta)
        
    # Clip alpha to [0, 1] to prevent numerical issues
    alpha = torch.clamp(alpha, 0.0, 1.0)
    
    # Compute transmittance T
    # T[i] = product_{j=0}^{i-1} (1 - alpha[j])
    # Initialize with 1.0
    one_minus_alpha = 1.0 - alpha + epsilon
    T_cum = torch.cat([torch.ones_like(one_minus_alpha[:, :1]), one_minus_alpha[:, :-1]], dim=-1)
    T = torch.cumprod(T_cum, dim=-1)
    
    # Compute weights: w[i] = alpha[i] * T[i]
    weights = alpha * T
    
    # Final transmittance
    T_final = T[:, -1:]
    
    return alpha, weights, T_final

def raw2alpha_backward(grad_back, sigma, dist, shift=None, interval=None, distance_scale=1.0, epsilon=1e-10):
    """
    Computes gradient of loss w.r.t. density sigma.
    
    Args:
        grad_back: Gradient w.r.t. alpha, shape (N, S).
        sigma: Raw density, shape (N, S).
        dist: Inter-sample distances, shape (N, S).
        shift: Optional float or tensor.
        interval: Optional float or tensor.
        distance_scale: Float.
        epsilon: Float.
        
    Returns:
        grad_sigma: Gradient w.r.t. sigma, shape (N, S).
    """
    delta = dist * distance_scale
    
    if shift is not None and interval is not None:
        # Gradient for interval-based activation
        e = torch.exp(sigma + shift)
        # Clamp e to prevent overflow in gradient
        e_clamped = torch.clamp(e, max=1e10)
        # d(alpha)/d(sigma) = interval * (1+e)^(-interval-1) * e
        d_alpha_d_sigma = interval * torch.pow(1.0 + e_clamped, -interval - 1.0) * e_clamped
    else:
        # Gradient for standard exponential activation
        # d(alpha)/d(sigma) = delta * exp(-sigma * delta)
        d_alpha_d_sigma = delta * torch.exp(-sigma * delta)
        
    # Chain rule: grad_sigma = grad_back * d_alpha/d_sigma
    grad_sigma = grad_back * d_alpha_d_sigma
    
    return grad_sigma
```

## Subtopics

### [Alpha Computation with Alpha Mask](10549-alpha-computation-with-alpha-mask.md)
Alpha Computation with Alpha Mask corrects over-estimation of occlusion for planar surfaces with complex contours (e.g., foliage) by weighting the occlusion contribution by the average alpha value $\bar{\alpha}$ of the surface texture. It differs from standard alpha blending by applying a scalar weight to the entire surface's occlusion contribution rather than per-pixel alpha testing.

### [Alpha-to-Extinction Conversion](13044-alpha-to-extinction-conversion.md)
Alpha-to-Extinction Conversion transforms opacity values into raw extinction coefficients, bridging the gap between transfer-function-derived alpha and physical scattering models. It enables combining physically calculated cloud extinction with transfer-function alpha values.

### Post-activation Trilinear Interpolation
Post-activated Trilinear Interpolation computes activated alpha $\alpha_{\text{post}}$ by applying the exponential activation function *after* trilinear interpolation of raw values, rather than before [S9]. Formula: $\alpha_{\text{post}} = 1 - \exp(-\delta \log(1 + \exp(\text{interp}(x, V)))) = 1 - (1 + \exp(\text{interp}(x, V)))^{-\delta}$, where $\text{interp}(x, V)$ is the trilinear interpolation of raw values $V$ at position $x$, and $\delta$ is the segment length [S9].

### Raw Density to Alpha Backward Pass (Uniform)
Computes the gradient of the alpha conversion with respect to input density given the gradient of the loss w.r.t. alpha [S2, unofficial]. Formula: $\text{grad}[i_{\text{pt}}] = \min(e_{\text{d}}[i_{\text{pt}}], 10^{10}) \cdot (1 + e_{\text{d}}[i_{\text{pt}}])^{-\text{interval}-1} \cdot \text{interval} \cdot \text{grad\_back}[i_{\text{pt}}]$, where $e_{\text{d}}$ is the pre-computed exponential density [S2, unofficial]. Clamping $e_{\text{d}}$ to $10^{10}$ prevents overflow [S2, unofficial].

### Grayscale to RGBA Threshold Conversion
Converts a 1D grayscale volume buffer to a 4D RGBA buffer by replicating the scalar value to RGB channels and applying a hard alpha threshold [S10, unofficial]. Input: 1D array of `char`. Output: 1D array of `char` with stride 4 per sample, layout `[R, G, B, A]` [S10, unofficial]. Alpha is 0 if input < 30, else 255 [S10, unofficial].

### Transfer Function CDF Construction
Ensures the alpha channel of a transfer function LUT is monotonically nondecreasing for DDA ray marching optimization by computing a cumulative distribution function (CDF) of alpha values [S11, unofficial]. Steps: Copy LUT, accumulate alpha values, normalize by total integral [S11, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot S)$ for alpha and weight computation, where $N$ is batch size and $S$ is sample count [S5, S7, unofficial]. Cumulative product is linear in $S$ [S5, unofficial].
- **Memory Complexity:** $O(N \cdot S)$ for storing `alpha`, `weights`, and `T` [S5, S7, unofficial].
- **Performance:** Efficient on GPU due to vectorized tensor operations [S5, S6, unofficial].
- **Strengths:** Differentiable, enabling end-to-end training of neural radiance fields [S1, S5, S6, unofficial]. Supports various activation modes (standard, interval, softmax) [S1, S2, S6, unofficial].
- **Limitations:** Sensitive to numerical stability (requires $\epsilon$) [S5, S7, unofficial]. Standard exponential assumes uniform sampling or explicit distance input [S5, S7, unofficial]. Interval-based activation requires tuning `shift` and `interval` [S2, S6, unofficial].
- **When to Use:** When rendering neural density fields (e.g., NeRF, TensoRF) [S1, S5, S7, unofficial]. When differentiability is required [S1, S6, unofficial].
- **When Not to Use:** When using pre-computed alpha values from transfer functions without density [S2, S10, unofficial]. When high-performance non-differentiable rendering is needed and alpha is pre-computed [S3, S10, unofficial].

## Background
Raw2Alpha is a fundamental component in neural volume rendering, particularly in Neural Radiance Fields (NeRF) and its variants [S1, S2, S5, S6, S7, unofficial]. It was introduced to convert raw density outputs from neural networks into opacity values suitable for volume rendering [S1, S2, S5, unofficial]. The method has been adapted in various forms, including interval-based activation in DirectVoxGO [S2, S6, unofficial] and softmax-based weighting in PointNeRF [S1, unofficial]. It is widely used in differentiable rendering pipelines for 3D reconstruction and novel view synthesis [S1, S5, S6, S7, unofficial].

## Sources
[S1] Xharlie/pointnerf, https://github.com/Xharlie/pointnerf, fetched 2026-10-26 (unofficial)
[S2] sunset1995/DirectVoxGO, https://github.com/sunset1995/DirectVoxGO, fetched 2026-10-26 (unofficial)
[S3] GPU Raycast Volume Rendering with Transfer ... - Visualization Library, https://visualizationlibrary.org/documentation/pag_guide_raycast_volume.html, fetched 2026-10-26 (unofficial)
[S4] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
[S5] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
[S6] gkouros/ref-dvgo, https://github.com/gkouros/ref-dvgo, fetched 2026-10-28 (unofficial)
[S7] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S8] NeuManifold: Neural Watertight Manifold Reconstruction with Efficient and High-Quality Rendering Support, https://doi.org/10.1109/wacv61041.2025.00081, fetched 2026-10-29
[S9] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
[S10] abhishektrip/SliceBasedVolumeRendering, https://github.com/abhishektrip/SliceBasedVolumeRendering, fetched 2026-10-27 (unofficial)
[S11] nihofm/volren: Simple OpenGL-based renderer for ..., https://github.com/nihofm/volren, fetched 2026-10-29 (unofficial)
