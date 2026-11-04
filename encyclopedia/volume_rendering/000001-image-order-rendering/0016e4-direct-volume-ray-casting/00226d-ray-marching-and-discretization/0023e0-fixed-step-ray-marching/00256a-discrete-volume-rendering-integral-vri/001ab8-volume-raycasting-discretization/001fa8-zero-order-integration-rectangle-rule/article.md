# Zero Order Integration (Rectangle Rule)

Also called: Alpha Compositing, Discrete Volume Rendering, Piecewise-constant approximation, Quadrature Approximation, Riemann Sum Approximation, Riemann sum with rectangle rule, Uniform sampling approximation, Zero-order hold reconstruction

## Overview
Zero Order Integration approximates the continuous volume rendering integral by decomposing it into a Riemann sum over discrete segments along a ray, assuming the scalar field (density) and color are constant within each segment [S1, unofficial][S3]. It sets apart from higher-order methods like [Partial Pre-Integration](1148-partial-pre-integration.md) by requiring no pre-computed tables or linear interpolation, relying instead on a simple piecewise-constant assumption [S3]. This simplicity comes at the cost of visible discontinuities and approximation errors, particularly when sampling rates are low [S3].

## How it works
**Inputs:**
- Ray parameterization $r(t)$ entering at $t_0$ and exiting at $t_N$ [G].
- $N$ sample positions $t_i$ along the ray, typically equidistant or stratified [S1, S2, unofficial].
- Scalar field values $\sigma_i$ (density/extinction) and color values $c_i$ (emission) at each sample point $t_i$ [S1, S2, unofficial].
- Optional noise standard deviation `raw_noise_std` [S2, unofficial].

**Outputs:**
- Composite color $C(\mathbf{r})$ [S2, unofficial].
- Accumulated opacity $Acc(\mathbf{r})$ [S2, unofficial].
- Expected depth $Depth(\mathbf{r})$ [S2, unofficial].

**Data Structures:**
- Arrays of size $N$ for positions $t_i$, densities $\sigma_i$, and colors $c_i$ [S2, unofficial].
- Array of size $N$ for segment distances $\delta_i$ [S2, unofficial].
- Array of size $N$ for alphas $\alpha_i$ [S2, unofficial].
- Array of size $N$ for transmittance $T_i$ [S2, unofficial].
- Array of size $N$ for weights $w_i$ [S2, unofficial].

**Steps:**

1.  **Compute Segment Distances:**
    Calculate the distance between consecutive samples. For $i = 0$ to $N-2$, $\delta_i = t_{i+1} - t_i$. The last segment distance $\delta_{N-1}$ is set to a large value (e.g., $10^{10}$) to ensure the ray exit is fully accounted for [S2, unofficial]. If the ray direction $\mathbf{d}$ is not unit length, scale distances by its magnitude:
    $$ \delta_i = (t_{i+1} - t_i) \|\mathbf{d}\| $$
    [S2, unofficial]

2.  **Apply Noise (Optional):**
    If regularization is enabled, add Gaussian noise to the density:
    $$ \sigma_i \leftarrow \sigma_i + \text{randn}() \cdot \text{raw\_noise\_std} $$
    Typical value for `raw_noise_std` is $0.0$ [S2, unofficial].

3.  **Compute Opacity ($\alpha_i$):**
    Using the zero-order hold assumption, the density is constant $\sigma_i$ over the segment $\delta_i$ [S3]. The opacity is computed as:
    $$ \alpha_i = 1 - \exp(-\max(\sigma_i, 0) \cdot \delta_i) $$
    [S1, S2, unofficial]. The `relu` or `max(0, ...)` ensures non-negative density for the exponential [S2, unofficial].

4.  **Compute Transmittance ($T_i$):**
    Transmittance is the probability of light passing through all previous segments [G]. It is computed using an exclusive cumulative product:
    $$ T_i = \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    For $i=1$, $T_1 = 1$. To prevent numerical instability when $\alpha_j \approx 1$, add a small epsilon ($10^{-10}$) to $(1 - \alpha_j)$ before multiplication [S2, unofficial].
    $$ T_i = \prod_{j=1}^{i-1} (1 - \alpha_j + \epsilon) $$
    [S2, unofficial]

5.  **Compute Weights ($w_i$):**
    The contribution weight for each sample is the product of transmittance and opacity:
    $$ w_i = T_i \cdot \alpha_i $$
    [S2, unofficial]

6.  **Composite Color:**
    Sum the weighted color contributions:
    $$ C(\mathbf{r}) = \sum_{i=1}^{N} w_i \cdot c_i $$
    [S2, unofficial]

7.  **Compute Accumulated Opacity:**
    $$ Acc(\mathbf{r}) = \sum_{i=1}^{N} w_i $$
    [S2, unofficial]

8.  **Compute Expected Depth (Optional):**
    $$ Depth(\mathbf{r}) = \sum_{i=1}^{N} w_i \cdot t_i $$
    Clamp depth to a minimum of $10^{-10}$ to avoid division by zero in downstream disparity calculations [S2, unofficial].

9.  **Background Compositing (Optional):**
    If a white background is assumed, composite the result:
    $$ C_{final} = C(\mathbf{r}) + (1 - Acc(\mathbf{r})) \cdot \mathbf{1} $$
    [S2, unofficial]

**Pseudocode:**

```python
import numpy as np

def zero_order_integration(t_vals, sigma_vals, color_vals, ray_direction_norm=1.0, 
                           raw_noise_std=0.0, white_bkgd=False, epsilon=1e-10):
    """
    Perform zero-order integration (rectangle rule) for volume rendering.
    
    Args:
        t_vals: Array of shape (N,) containing sample positions along the ray.
        sigma_vals: Array of shape (N,) containing density/extinction at each sample.
        color_vals: Array of shape (N, 3) containing RGB color at each sample.
        ray_direction_norm: Scalar magnitude of the ray direction vector.
        raw_noise_std: Standard deviation for noise added to density.
        white_bkgd: Boolean, if True composite onto white background.
        epsilon: Small value to prevent numerical instability in transmittance.
        
    Returns:
        rgb: Composite RGB color.
        acc: Accumulated opacity.
        depth: Expected depth.
    """
    N = len(t_vals)
    
    # 1. Compute segment distances
    # Differences between consecutive samples
    dists = t_vals[1:] - t_vals[:-1]
    # Append a large value for the last segment to ensure full ray coverage
    dists = np.concatenate([dists, np.full(1, 1e10)])
    
    # Scale by ray direction magnitude if not unit length
    dists = dists * ray_direction_norm
    
    # 2. Apply optional noise to density
    if raw_noise_std > 0.0:
        sigma_vals = sigma_vals + np.random.randn(N) * raw_noise_std
        
    # 3. Compute alpha (opacity) for each segment
    # Use relu for sigma to ensure non-negative density for exp
    sigma_clamped = np.maximum(sigma_vals, 0.0)
    alpha = 1.0 - np.exp(-sigma_clamped * dists)
    
    # 4. Compute transmittance using exclusive cumulative product
    # T_i = product_{j=1}^{i-1} (1 - alpha_j)
    # Add epsilon to (1 - alpha) to prevent instability if alpha is 1.0
    one_minus_alpha = 1.0 - alpha + epsilon
    # Cumulative product of (1 - alpha)
    cum_prod = np.cumprod(one_minus_alpha)
    # Transmittance is the product of all previous segments
    # T[0] = 1.0 (no previous segments)
    # T[i] = cum_prod[i-1]
    transmittance = np.concatenate([[1.0], cum_prod[:-1]])
    
    # 5. Compute weights
    weights = transmittance * alpha
    
    # 6. Composite RGB
    # weights shape (N,), color_vals shape (N, 3)
    rgb = np.sum(weights[:, np.newaxis] * color_vals, axis=0)
    
    # 7. Compute accumulated opacity
    acc = np.sum(weights)
    
    # 8. Compute expected depth
    # Clamp depth to min 1e-10 to avoid division by zero
    depth = np.sum(weights * t_vals)
    depth = np.maximum(depth, 1e-10)
    
    # 9. Background compositing
    if white_bkgd:
        rgb = rgb + (1.0 - acc) * np.array([1.0, 1.0, 1.0])
        
    return rgb, acc, depth
```

**Parameters:**
- `raw_noise_std`: Float, standard deviation for noise added to density for regularization. Typical value: $0.0$ [S2, unofficial].
- `epsilon`: Float, small value added to $(1-\alpha)$ to prevent numerical instability. Typical value: $10^{-10}$ [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Discontinuities:** The reconstructed signal is not $C^0$ continuous, leading to visible discontinuities and significant approximation errors, especially with uniform sampling [S3].
- **Uniform Sampling Artifacts:** Assuming homogeneous material within each segment can result in artifacts if the sampling rate is insufficient to capture high-frequency details [S1, unofficial].
- **Numerical Instability:** If $\alpha_i \approx 1.0$, $(1-\alpha_i)$ becomes zero, causing transmittance to drop to zero abruptly. Adding epsilon prevents this [S2, unofficial].
- **Depth Division:** When computing disparity from depth, clamp depth to a minimum value to avoid division by zero [S2, unofficial].

## Subtopics

### [Runge-Kutta 4 (RK4) Integration](3158-runge-kutta-4-rk4-integration.md)
A higher-order technique that reduces trajectory error by querying a vector field at four positions, chosen for accurate streamline visualization where zero-order integration fails due to low accuracy.

## Operating characteristics
- **Time Complexity:** $O(N)$ per ray, where $N$ is the number of samples [G].
- **Memory Complexity:** $O(N)$ to store sample values, distances, alphas, and transmittance [G].
- **Performance:** Faster than pre-integration methods because it avoids pre-computed tables, though it still requires scalar field interpolation [S3].
- **Strengths:** Simple to implement, no pre-computation required [S3].
- **Limitations:** Low accuracy due to piecewise-constant approximation; visible artifacts (jagged edges, aliasing) if sampling rate is low [S1, unofficial][S3]. Not suitable for high-quality rendering without high sampling rates [S3].
- **When to use:** Real-time applications with limited computational resources, or as a baseline method [S1, unofficial][S3].
- **When not to use:** High-quality offline rendering, scenes with high-frequency details, or when smooth transitions are critical [S1, unofficial][S3].

## Background
Zero Order Integration, also known as the rectangle rule or Riemann sum approximation, is one of the earliest and most fundamental methods for discretizing the volume rendering integral [S1, unofficial][S3]. The method assumes that the scalar field (density) and color are constant within each ray segment, effectively performing a zero-order hold reconstruction [S3]. It remains used in neural rendering frameworks (e.g., NeRF) as a baseline quadrature method [S2, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S3] Second Order Pre-Integrated Volume Rendering, https://people.freedesktop.org/~marcheu/pacificvis08.pdf, fetched 2026-10-29
