# Ray Compositing for Training (Uncertainty)


## Overview
This method composites rays for training neural radiance fields by estimating and backpropagating uncertainty alongside standard volume rendering outputs (RGB, sigma, ambient light) [S1, unofficial]. It differs from standard training compositing by explicitly computing gradients for an uncertainty term, allowing the model to learn confidence metrics per pixel [S1, unofficial]. It builds on [Training Ray Compositing (Sigma)](3868-training-ray-compositing-sigma.md) by extending the output space to include uncertainty [S1, unofficial].

## How it works
**Inputs:**
- `sigma`: Tensor of volume densities, shape `[N, num_samples]`.
- `rgb`: Tensor of colors, shape `[N, num_samples, 3]`.
- `ambient`: Tensor of ambient light, shape `[N, num_samples, amb_dim]`.
- `uncertainty`: Tensor of per-sample uncertainty values ($\beta$), shape `[N, num_samples]`.
- `dist`: Tensor of sample distances along each ray, shape `[N, num_samples]`.
- `T_thresh`: Float threshold for transmittance culling, default `1e-4` [S1, unofficial].

**Outputs:**
- `rgb_out`: Composite RGB, shape `[N, 3]`.
- `sigma_out`: Composite sigma, shape `[N]`.
- `ambient_out`: Composite ambient, shape `[N, amb_dim]`.
- `uncertainty_out`: Composite uncertainty, shape `[N]`.
- `weights`: Transmittance weights, shape `[N, num_samples]` (saved for backward).
- `uncertainty_sum`: Intermediate sum for backward pass, shape `[N]` (saved for backward).

**Data Structures:**
- Tensors are assumed to be contiguous in memory. If not, they are made contiguous before kernel invocation [S1, unofficial].

**Algorithm:**

1.  **Preprocessing:** Ensure `uncertainty` tensor is contiguous [S1, unofficial]. Allocate `uncertainty_sum` tensor of size `[N]` [S1, unofficial].
2.  **Forward Pass:** Call the forward kernel to perform standard volume rendering integration for RGB, sigma, and ambient, while simultaneously accumulating the weighted uncertainty [S1, unofficial].
    - The composite uncertainty is calculated as:
      $$ \text{uncertainty\_out}_i = \sum_{j=1}^{M} w_{i,j} \cdot \beta_{i,j} $$
      where $w_{i,j}$ is the transmittance weight for ray $i$ at sample $j$, and $\beta_{i,j}$ is the per-sample uncertainty [G].
    - The kernel uses `T_thresh` to cull samples where transmittance drops below the threshold [S1, unofficial].
3.  **Save State:** Save `uncertainty`, `uncertainty_sum`, `weights`, and other necessary tensors for the backward pass [S1, unofficial].
4.  **Backward Pass:**
    - Ensure `grad_uncertainty_sum` (gradient of the loss w.r.t `uncertainty_out`) is contiguous [S1, unofficial].
    - Initialize `grad_uncertainty` tensor with zeros, shape `[N, num_samples]` [S1, unofficial].
    - Call the backward kernel to compute gradients for `sigma`, `rgb`, `ambient`, and `uncertainty` using the saved forward state [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def composite_rays_train_uncertainty_forward(sigma, rgb, ambient, uncertainty, dist, T_thresh=1e-4):
    """
    Forward pass for training ray compositing with uncertainty.
    Implements standard volume rendering integration with uncertainty accumulation.
    """
    N, M = len(sigma), len(sigma[0])
    D = len(ambient[0][0])
    
    # Allocate output tensors
    rgb_out = [[0.0, 0.0, 0.0] for _ in range(N)]
    sigma_out = [0.0] * N
    ambient_out = [[0.0] * D for _ in range(N)]
    uncertainty_out = [0.0] * N
    weights = [[0.0] * M for _ in range(N)]
    uncertainty_sum = [0.0] * N

    for i in range(N):
        T = 1.0  # Initialize transmittance
        for j in range(M):
            alpha = 1.0 - math.exp(-sigma[i][j] * dist[i][j])
            w = alpha * T
            weights[i][j] = w
            
            rgb_out[i][0] += w * rgb[i][j][0]
            rgb_out[i][1] += w * rgb[i][j][1]
            rgb_out[i][2] += w * rgb[i][j][2]
            
            sigma_out[i] += w * sigma[i][j]
            
            for k in range(D):
                ambient_out[i][k] += w * ambient[i][j][k]
                
            uncertainty_sum[i] += w * uncertainty[i][j]
            
            T *= (1.0 - alpha)
            if T < T_thresh:
                break
        uncertainty_out[i] = uncertainty_sum[i]
        
    return rgb_out, sigma_out, ambient_out, uncertainty_out, weights, uncertainty_sum

def composite_rays_train_uncertainty_backward(
    grad_rgb_out, grad_sigma_out, grad_ambient_out, grad_uncertainty_out,
    sigma, rgb, ambient, uncertainty, weights, uncertainty_sum, dist, T_thresh=1e-4
):
    """
    Backward pass for training ray compositing with uncertainty.
    Computes gradients for sigma, rgb, ambient, and uncertainty.
    """
    N, M = len(sigma), len(sigma[0])
    D = len(ambient[0][0])
    
    # Initialize gradients
    grad_sigma = [[0.0] * M for _ in range(N)]
    grad_rgb = [[[0.0, 0.0, 0.0] for _ in range(M)] for _ in range(N)]
    grad_ambient = [[[0.0] * D for _ in range(M)] for _ in range(N)]
    grad_uncertainty = [[0.0] * M for _ in range(N)]

    for i in range(N):
        T = 1.0
        for j in range(M):
            alpha = 1.0 - math.exp(-sigma[i][j] * dist[i][j])
            w = alpha * T
            
            # Gradient for uncertainty: dU/d_beta_j = w_j
            grad_uncertainty[i][j] = grad_uncertainty_out[i] * w
            
            # Gradient for rgb and ambient: dC/d_c_j = w_j * g_c_out
            grad_rgb[i][j][0] = w * grad_rgb_out[i][0]
            grad_rgb[i][j][1] = w * grad_rgb_out[i][1]
            grad_rgb[i][j][2] = w * grad_rgb_out[i][2]
            
            for k in range(D):
                grad_ambient[i][j][k] = w * grad_ambient_out[i][k]
                
            # Gradient for sigma: dC/d_sigma_j = (g_rgb * rgb_j + g_amb * amb_j) * T * dist_j + g_sigma_out * w_j
            rgb_dot = (grad_rgb_out[i][0] * rgb[i][j][0] +
                       grad_rgb_out[i][1] * rgb[i][j][1] +
                       grad_rgb_out[i][2] * rgb[i][j][2])
            amb_dot = sum(grad_ambient_out[i][k] * ambient[i][j][k] for k in range(D))
            grad_sigma[i][j] = (rgb_dot + amb_dot) * T * dist[i][j] + grad_sigma_out[i] * w
            
            T *= (1.0 - alpha)
            if T < T_thresh:
                break
                
    return grad_sigma, grad_rgb, grad_ambient, grad_uncertainty
```

**Edge Cases and Pitfalls:**
- Non-contiguous tensors cause kernel launch failures or incorrect results; ensure contiguity before invocation [S1, unofficial].
- `T_thresh` defaults to `1e-4` [S1, unofficial].
- Degenerate input (e.g., zero-length rays) should be handled by the backend kernel, typically resulting in zero outputs [G].

## Subtopics

### [Uncertainty Rendering](14231-uncertainty-rendering.md)
Estimates the epistemic uncertainty of a volume rendering result by aggregating per-sample uncertainty metrics along a ray. It solves the problem of quantifying confidence in the final pixel color by weighting local uncertainty estimates by the ray compositing weights. It differs from standard direct volume rendering by outputting an additional uncertainty channel alongside RGB, derived from the variance of the neural field's predictions rather than just the mean.

## Operating characteristics
Time complexity is linear in the number of rays and samples due to per-sample integration [G]. Memory complexity is linear in rays and samples for storing weights and intermediate tensors [G]. Performance depends on the efficiency of the forward and backward kernels [S1, unofficial]. Strengths include enabling uncertainty-aware training, potentially improving robustness in occluded or sparse regions [S1, unofficial]. Limitations include increased memory usage due to storing uncertainty gradients and intermediate sums [S1, unofficial].

## Background
Used in neural radiance field training frameworks that support uncertainty estimation, specifically referenced in the ER-NeRF project [S1, unofficial]. It extends standard volume rendering compositing to include uncertainty gradients, allowing the network to learn confidence metrics [S1, unofficial].

## Sources
[S1] Fictionarry/ER-NeRF, https://github.com/Fictionarry/ER-NeRF, fetched 2026-10-26 (unofficial)
