# Raw2Alpha Density to Alpha Conversion

Also called: Raw2Alpha, Raw Density to Alpha, Raw to Alpha Conversion, Density to Alpha Conversion, Raw Density to Alpha Conversion, density to alpha

## Overview
Raw2Alpha converts raw neural network outputs (density $\sigma$) into per-sample opacity ($\alpha$) and ray-space weights ($w$) for volume rendering [S1, S5, S7, unofficial]. It solves the problem of mapping unbounded or arbitrary density values to the bounded $[0, 1]$ opacity range required for front-to-back compositing [S8]. It is distinct from its parent [Alpha-to-Weight Computation (Forward)](2373-alpha-to-weight-computation-forward.md) because it performs the initial density-to-opacity transformation, whereas the parent handles the subsequent accumulation of transmittance and weights from already-computed alpha values.

## How it works
**Inputs:**
*   `sigma`: Raw density values, shape $(N, S)$ or $(N,)$, where $N$ is the number of rays and $S$ is the number of samples per ray [S1, S5, unofficial].
*   `dist`: Inter-sample distances (step sizes $\delta$), shape $(N, S)$ or $(N,)$ [S5, S7, unofficial].
*   `shift`: Optional scalar or tensor added to density before exponentiation [S2, S6, unofficial].
*   `interval`: Optional scalar or tensor representing sample spacing or activation interval [S2, S6, unofficial].

**Outputs:**
*   `alpha`: Opacity values, shape $(N, S)$, values in $[0, 1)$ [S5, S7, unofficial].
*   `weights`: Ray-space weights for color blending, shape $(N, S)$ [S5, S7, unofficial].
*   `T`: Transmittance values, shape $(N, S+1)$ or $(N, 1)$ for final transmittance [S5, unofficial].

**Data Structures:**
*   Tensors for `sigma`, `dist`, `alpha`, `weights`, `T`.
*   If using the uniform interval variant with shift, `exp_d` (pre-computed exponential density) is stored for the backward pass [S2, S6, unofficial].

**Steps:**

1.  **Compute Exponential Density (Optional Variant):** If using the shifted/interval variant, compute $e = \exp(\sigma + \text{shift})$ [S2, S6, unofficial].
2.  **Compute Opacity ($\alpha$):**
    *   *Standard Variant:* Compute $\alpha = 1 - \exp(-\sigma \cdot \text{dist})$ [S5, S7, unofficial][S8].
    *   *Shifted/Interval Variant:* Compute $\alpha = 1 - (1 + e)^{-\text{interval}}$ [S2, S6, unofficial].
    *   *Linear Variant (Legacy):* Compute $\alpha = \sigma \cdot \text{sample\_step} \cdot \text{val\_threshold} \cdot \text{brightness}$ [S3, unofficial].
3.  **Compute Transmittance ($T$):**
    *   Initialize $T$ with a leading 1.0: $T_{init} = [1.0, 1-\alpha_0, 1-\alpha_1, \dots, 1-\alpha_{S-1}]$ [S1, S5, unofficial].
    *   Add epsilon ($\epsilon = 10^{-10}$) to $(1-\alpha)$ terms to prevent NaNs in cumulative product [S5, S7, unofficial].
    *   Compute cumulative product along the sample axis: $T = \text{cumprod}(T_{init})$ [S1, S5, unofficial].
4.  **Compute Weights ($w$):**
    *   $w_i = \alpha_i \cdot T_{i}$, where $T_i$ is the transmittance *before* sample $i$ (i.e., $T[:, :-1]$) [S1, S5, S7, unofficial].
5.  **Return:** $\alpha$, $w$, and final transmittance $T_{final} = T[:, -1:]$ [S5, unofficial].

**Formulas:**

Standard exponential mapping [S5, S7, unofficial][S8]:
$$ \alpha_i = 1 - \exp(-\sigma_i \cdot \delta_i) $$
$$ T_i = \prod_{j=0}^{i-1} (1 - \alpha_j + \epsilon) $$
$$ w_i = \alpha_i \cdot T_i $$

Shifted/Interval mapping [S2, S6, unofficial]:
$$ e_i = \exp(\sigma_i + \text{shift}) $$
$$ \alpha_i = 1 - (1 + e_i)^{-\text{interval}} $$
$$ \frac{\partial \alpha_i}{\partial \sigma_i} = \text{interval} \cdot (1 + e_i)^{-\text{interval}-1} \cdot e_i $$

Linear mapping [S3, unofficial]:
$$ \alpha = \sigma \cdot \text{sample\_step} \cdot \text{val\_threshold} \cdot \text{brightness} $$

**Parameters:**
*   `epsilon`: $10^{-10}$, added to $(1-\alpha)$ to ensure numerical stability in `cumprod` [S5, S7, unofficial].
*   `shift`: Float added to density before exponentiation [S2, unofficial].
*   `interval`: Float or tensor representing uniform distance or activation power [S2, S6, unofficial].
*   `brightness`: Constant scaling factor for alpha, typically 50.0 [S3, unofficial].
*   `val_threshold`: Uniform float parameter to scale alpha contribution [S3, unofficial].
*   `distance_scale`: Float scaling inter-sample distances [S5, unofficial].

**Pitfalls:**
*   **Overflow:** $\exp(\sigma + \text{shift})$ can overflow to infinity if density is large [S2, unofficial]. Clamping `exp_d` to $10^{10}$ in the backward pass prevents gradient instability [S2, unofficial].
*   **NaNs:** Cumulative product of $(1-\alpha)$ can become NaN if $\alpha > 1$ due to precision errors or linear mapping without clamping. Adding epsilon and clamping $\alpha$ to $[0, 1)$ is recommended [S5, S7, unofficial].
*   **Boundary Checks:** In shader implementations, assume volume bounds are $[0, 1]$; break ray marching if position exceeds these bounds [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def raw2alpha(sigma: torch.Tensor, dist: torch.Tensor, shift: float = 0.0, interval: float = 1.0, epsilon: float = 1e-10) -> tuple:
    """
    Converts raw density and distances to alpha, weights, and transmittance.
    
    Args:
        sigma: Raw density values, shape (N, S) or (N,)
        dist: Inter-sample distances, shape (N, S) or (N,)
        shift: Scalar added to density before exponentiation (for shifted variant)
        interval: Scalar representing interval/power (for shifted variant)
        epsilon: Small constant to prevent NaNs in cumprod
        
    Returns:
        alpha: Opacity values, shape (N, S)
        weights: Ray-space weights, shape (N, S)
        T_final: Final transmittance, shape (N, 1)
    """
    # Ensure consistent shapes
    if sigma.dim() == 1:
        sigma = sigma.unsqueeze(1)
    if dist.dim() == 1:
        dist = dist.unsqueeze(1)
        
    # Standard exponential mapping
    # alpha = 1 - exp(-sigma * dist)
    # Using torch.exp for numerical stability
    exponent = -sigma * dist
    # Clamp exponent to prevent underflow/overflow issues in exp
    exponent = torch.clamp(exponent, min=-100, max=0)
    alpha = 1.0 - torch.exp(exponent)
    
    # Clip alpha to [0, 1) to prevent NaNs in cumprod
    alpha = torch.clamp(alpha, min=0.0, max=1.0 - epsilon)
    
    # Compute transmittance T
    # T = cumprod([1, 1-alpha_0, 1-alpha_1, ...])
    # Shape of (1-alpha) is (N, S)
    one_minus_alpha = 1.0 - alpha + epsilon
    
    # Prepend 1.0 to the start of the sequence for each ray
    # ones shape: (N, 1)
    ones = torch.ones_like(one_minus_alpha[:, :1])
    T_input = torch.cat([ones, one_minus_alpha], dim=-1)
    
    # Cumulative product along the sample axis (dim=-1)
    T = torch.cumprod(T_input, dim=-1)
    
    # Compute weights: w_i = alpha_i * T_i (where T_i is transmittance BEFORE sample i)
    # T[:, :-1] gives transmittance at the start of each sample
    weights = alpha * T[:, :-1]
    
    # Final transmittance is the last element of T
    T_final = T[:, -1:]
    
    return alpha, weights, T_final

def raw2alpha_backward(alpha: torch.Tensor, grad_back: torch.Tensor, sigma: torch.Tensor, dist: torch.Tensor, shift: float = 0.0, interval: float = 1.0) -> torch.Tensor:
    """
    Computes gradient of loss w.r.t. sigma.
    """
    # Gradient of alpha w.r.t sigma for standard mapping:
    # d(alpha)/d(sigma) = -dist * exp(-sigma * dist) = dist * (1 - alpha)
    
    # However, if using shifted mapping, gradient is different.
    # Assuming standard mapping for simplicity unless shift is used.
    # If shift is used, we need exp_d.
    
    # For standard mapping:
    # grad_sigma = grad_back * dist * (1 - alpha)
    
    # Note: This is a simplified backward pass. For the shifted variant, 
    # see [S2, unofficial] for the specific gradient formula involving exp_d.
    
    grad_sigma = grad_back * dist * (1.0 - alpha)
    
    return grad_sigma
```

## Subtopics

### [Alpha Computation with Alpha Mask](10549-alpha-computation-with-alpha-mask.md)
Corrects over-estimation of occlusion for planar surfaces with complex contours (e.g., foliage) by weighting the occlusion contribution by the average alpha ($\bar{\alpha}$) of the surface texture. Distinct from standard alpha blending as it applies a scalar weight to the entire surface's occlusion contribution rather than per-pixel alpha testing.

### Post-activation Trilinear Interpolation
Computes activated alpha by applying the exponential activation function *after* trilinear interpolation of raw values, rather than before [S9]. Formula: $\alpha_{\text{post}} = 1 - (1 + \exp(\text{interp}(x, V)))^{-\delta}$, where $\text{interp}(x, V)$ is the trilinear interpolation of raw values $V$ at position $x$, and $\delta$ is the segment length [S9].

### Raw Density to Alpha Backward Pass (Uniform)
Computes the gradient of alpha conversion w.r.t. input density [S2, unofficial]. Formula: $\text{grad}[i] = \min(e_d[i], 10^{10}) \cdot (1 + e_d[i])^{-\text{interval}-1} \cdot \text{interval} \cdot \text{grad\_back}[i]$, where $e_d$ is the pre-computed exponential density [S2, unofficial]. Clamping $e_d$ to $10^{10}$ prevents overflow [S2, unofficial].

### Grayscale to RGBA Threshold Conversion
Converts a 1D grayscale volume buffer to a 4D RGBA buffer by replicating the scalar value to RGB channels and applying a hard alpha threshold (e.g., 30) [S10, unofficial]. Input: 1D `char` array; Output: 1D `char` array with stride 4 per sample [S10, unofficial].

### Transfer Function CDF Construction
Ensures the alpha channel of a transfer function LUT is monotonically nondecreasing for DDA ray marching optimization by computing a cumulative distribution function (CDF) of alpha values [S11, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$ for forward pass, where $N$ is rays and $S$ is samples. Dominated by `exp` and `cumprod` operations [S5, S7, unofficial].
*   **Memory Complexity:** $O(N \cdot S)$ to store `alpha`, `weights`, and `T` [S5, unofficial].
*   **Performance:** Highly parallelizable on GPUs. The `cumprod` operation can be a bottleneck if not optimized, but modern libraries handle it efficiently [S5, S7, unofficial].
*   **Strengths:** Differentiable, supports variable step sizes, numerically stable with epsilon [S5, S7, unofficial].
*   **Limitations:** Standard exponential mapping assumes constant density over the step; large steps may introduce artifacts [S8]. Shifted variant requires careful tuning of `shift` and `interval` [S2, unofficial].

## Background
Originated in neural radiance fields (NeRF) and related volume rendering techniques to map neural network density outputs to opacity [S1, S5, S7, unofficial]. The standard exponential mapping $\alpha = 1 - e^{-\sigma \delta}$ is derived from the Beer-Lambert law [S8]. Variants like the shifted/interval mapping were introduced to improve convergence and flexibility in optimization [S2, S6, unofficial].

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
