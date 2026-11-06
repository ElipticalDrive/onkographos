# NeRF Rendering Formula

Also called: Discrete Volumetric Rendering Equation, Volume Rendering (Raw to Outputs)

## Overview
The NeRF Rendering Formula transforms raw neural network predictions (density and color features) into final RGB, depth, and accumulation maps by discretizing the volume rendering integral along camera rays [S4, unofficial]. It differs from generic volume rendering by explicitly handling sigmoid color activation, density noise injection for regularization, and the specific discrete weighting scheme required for differentiable training of implicit representations [S1][S4, unofficial].

## How it works
**Inputs:**
- `z_vals`: Array of shape $(..., N)$ containing sample depths along rays [S4, unofficial].
- `raw`: Array of shape $(..., N, 4)$ containing raw network outputs. The first 3 channels are RGB features, the 4th is density $\sigma$ [S4, unofficial].
- `rays_d`: Ray direction vectors used for real-world distance scaling [S4, unofficial].
- `raw_noise_std`: Scalar standard deviation for density noise [S4, unofficial].
- `white_bkgd`: Boolean flag for background compositing [S4, unofficial].

**Outputs:**
- `rgb_map`: Composite RGB color, shape $(..., 3)$ [S4, unofficial].
- `depth_map`: Composite depth, shape $(...)$ [S4, unofficial].
- `acc_map`: Accumulated opacity, shape $(...)$ [S4, unofficial].

**Data Structures:**
- `dists`: Segment lengths, shape $(..., N)$ [S4, unofficial].
- `alpha`: Opacity per segment, shape $(..., N)$ [S4, unofficial].
- `transmittance`: Cumulative transmittance, shape $(..., N)$ [S4, unofficial].
- `weights`: Contribution weights, shape $(..., N)$ [S4, unofficial].

**Steps:**

1.  **Compute Segment Distances:**
    Calculate distances between adjacent samples. Append a large value ($10^{10}$) for the last segment to handle boundary conditions [S4, unofficial]. Convert to real-world distance by multiplying by the ray direction norm [S4, unofficial]:
    $$ \text{dists} = \text{concat}(z_{i+1} - z_i, 10^{10}) $$
    $$ \text{dists} = \text{dists} \times \| \mathbf{d} \| $$

2.  **Process Raw Predictions:**
    Extract density and color. Apply sigmoid to color features to bound them to $[0, 1]$ [S4, unofficial]. Add Gaussian noise to density if `raw_noise_std > 0` for regularization [S4, unofficial]:
    $$ \text{rgb} = \sigma_{\text{sigmoid}}(\text{raw}[:, :3]) $$
    $$ \sigma_{\text{noisy}} = \text{relu}(\text{raw}[:, 3] + \mathcal{N}(0, \text{raw\_noise\_std})) $$

3.  **Compute Opacity (Alpha):**
    Calculate segment opacity using the exponential attenuation model [S4, unofficial]:
    $$ \alpha = 1 - \exp(-\sigma_{\text{noisy}} \cdot \text{dists}) $$

4.  **Compute Transmittance:**
    Calculate cumulative transmittance $T$ using exclusive cumulative product. The first segment has $T=1$. A small epsilon ($10^{-10}$) is added to prevent numerical instability in the product [S4, unofficial]:
    $$ T_i = \prod_{j=0}^{i-1} (1 - \alpha_j + \epsilon) $$
    Specifically:
    $$ \text{transmittance} = \text{cumprod}(\text{concat}(1, 1 - \alpha + 10^{-10}), \text{axis}=-1)[\dots, :-1] $$

5.  **Compute Weights:**
    Calculate the weight $w_i$ for each sample, representing the probability that the ray terminates at sample $i$ [S1][S4, unofficial]:
    $$ w_i = T_i \cdot (1 - \exp(-\sigma_i \cdot \Delta t_i)) = T_i \cdot \alpha_i $$
    Note: Since $\alpha_i = 1 - \exp(-\sigma_i \cdot \Delta t_i)$, this simplifies to $w_i = T_i \cdot \alpha_i$.

6.  **Aggregate Results:**
    Compute the final RGB, depth, and accumulation maps by summing weighted contributions [S4, unofficial]:
    $$ \text{rgb\_map} = \sum_i w_i \cdot \text{rgb}_i $$
    $$ \text{depth\_map} = \sum_i w_i \cdot z_i $$
    $$ \text{acc\_map} = \sum_i w_i $$

7.  **Background Compositing (Optional):**
    If `white_bkgd` is true, composite the result onto a white background using the accumulated opacity [S4, unofficial]:
    $$ \text{rgb\_map} = \text{rgb\_map} + (1 - \text{acc\_map}) $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def nerf_rendering_formula(z_vals, raw, rays_d, raw_noise_std, white_bkgd):
    """
    z_vals: (..., N) sample depths
    raw: (..., N, 4) raw network output [RGB, density]
    rays_d: (..., 3) ray directions
    raw_noise_std: float, noise std for density
    white_bkgd: bool, composite on white background
    """
    # 1. Compute distances
    # dists shape: (..., N)
    dists = z_vals[..., 1:] - z_vals[..., :-1]
    # Append infinity for the last segment
    dists = np.concatenate([dists, np.full(dists.shape[:-1] + (1,), 1e10)], axis=-1)
    
    # Convert to real-world distance
    # rays_d_norm shape: (..., 1)
    rays_d_norm = np.linalg.norm(rays_d, axis=-1, keepdims=True)
    # Guard against zero-direction rays to prevent division by zero downstream
    rays_d_norm = np.maximum(rays_d_norm, 1e-10)
    dists = dists * rays_d_norm
    
    # 2. Process raw predictions
    # rgb shape: (..., N, 3)
    rgb = 1.0 / (1.0 + np.exp(-raw[..., :3])) # Sigmoid
    
    # Density with noise
    # raw[..., 3] shape: (..., N)
    if raw_noise_std > 0:
        noise = np.random.randn(*raw[..., 3].shape) * raw_noise_std
        density = np.maximum(raw[..., 3] + noise, 0) # ReLU
    else:
        density = np.maximum(raw[..., 3], 0) # ReLU
        
    # 3. Compute alpha
    # alpha shape: (..., N)
    alpha = 1.0 - np.exp(-density * dists)
    
    # 4. Compute transmittance (exclusive cumprod)
    # ones shape: (..., 1)
    ones = np.ones_like(alpha[..., :1])
    # 1 - alpha + epsilon
    term = 1.0 - alpha + 1e-10
    # Concatenate ones at the start
    terms = np.concatenate([ones, term], axis=-1)
    # Cumprod
    cumprod = np.cumprod(terms, axis=-1)
    # Transmittance is cumprod[..., :-1]
    transmittance = cumprod[..., :-1]
    
    # 5. Compute weights
    # weights shape: (..., N)
    weights = alpha * transmittance
    
    # 6. Aggregate
    # rgb_map shape: (..., 3)
    rgb_map = np.sum(weights[..., np.newaxis] * rgb, axis=-2)
    
    # depth_map shape: (...,)
    depth_map = np.sum(weights * z_vals, axis=-1)
    
    # acc_map shape: (...,)
    acc_map = np.sum(weights, axis=-1)
    # Guard against division by zero in downstream depth normalization
    acc_map = np.maximum(acc_map, 1e-10)
    
    # 7. Background compositing
    if white_bkgd:
        rgb_map = rgb_map + (1.0 - acc_map[..., np.newaxis])
        
    return rgb_map, depth_map, acc_map
```

**Parameters:**
- `raw_noise_std`: Typically $0.0$ during training if noise is disabled, or a small value like $0.01$–$0.1$ for regularization [S4, unofficial].
- `epsilon`: $10^{-10}$ for numerical stability in transmittance calculation [S4, unofficial].
- `infinity`: $10^{10}$ for the last segment distance [S4, unofficial].

**Edge Cases:**
- **Zero Density:** If density is zero, alpha is zero, and the sample contributes nothing to color but passes transmittance through [G].
- **Full Opacity:** If alpha is 1, transmittance for subsequent samples becomes 0 [G].
- **Numerical Stability:** The addition of $10^{-10}$ in the transmittance product prevents `cumprod` from becoming exactly zero due to floating-point underflow when alpha is very small but non-zero, or exactly 1 when alpha is 0 [S4, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ per ray, where $N$ is the number of samples [G]. The dominant operations are element-wise multiplications and a cumulative product [G].
- **Memory Complexity:** $O(N)$ per ray for storing intermediate arrays (`dists`, `alpha`, `transmittance`, `weights`) [G].
- **Performance:** Efficient on GPUs due to vectorizable operations. Memory usage scales with batch size and number of samples [G]. Large `netchunk` values (e.g., $1024 \times 64$) are used to process many points in parallel to maximize GPU utilization [S4, unofficial].
- **Strengths:** Differentiable, enabling end-to-end training. Handles transparency and depth estimation naturally [S3].
- **Limitations:** Sensitive to sampling strategy (uniform vs. importance sampling). Requires careful tuning of `raw_noise_std` to avoid overfitting or blurry results [S4, unofficial].

## Background
The NeRF Rendering Formula implements the discretized volume rendering equation specifically tailored for implicit neural representations [S3]. It assumes that radiance is calculated by a single-path summation along the ray trajectory, avoiding path tracing and the actual RTE/VRE [S3]. The learned parameters (color and opacity of each point) are not interpreted literally in the physical sense but serve as an approximation learnable from offline samples [S3]. The formula is widely used in subsequent 3D reconstruction and novel view synthesis methods based on neural implicit fields [G].

## Sources
[S1] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S2] sxyu/svox2, https://github.com/sxyu/svox2, fetched 2026-10-27 (unofficial)
[S3] 14 Rendering Volume and Subsurface Scattering - Foundations of Visual ..., https://learnvisualcomputing.github.io/rendering-rte.html, fetched 2026-10-27
[S4] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
