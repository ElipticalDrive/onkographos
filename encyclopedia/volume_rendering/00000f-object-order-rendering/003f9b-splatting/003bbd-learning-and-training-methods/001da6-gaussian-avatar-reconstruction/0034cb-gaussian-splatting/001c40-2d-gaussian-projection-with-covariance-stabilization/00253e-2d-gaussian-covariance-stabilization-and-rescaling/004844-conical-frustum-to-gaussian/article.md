# Conical Frustum to Gaussian

Also called: Conical Frustum to Gaussian Approximation, cone ray featurization

## Overview
This method approximates the density of a conical frustum (or cylinder) segment along a ray as a 3D Gaussian distribution, defined by a mean position and a covariance matrix. It solves the problem of integrating positional encodings and density contributions over a volume that varies in width along the ray, enabling anti-aliased neural rendering. It differs from its parent, [2D Gaussian Covariance Stabilization and Rescaling](3467-2d-gaussian-covariance-stabilization-and-rescaling.md), which operates on the projected 2D covariance; this method operates in 3D space to lift the 1D ray-space statistics into a full 3D Gaussian before projection.

## How it works
The method computes the mean and covariance of a Gaussian that approximates the integral of a conical frustum's density. It consists of two main phases: calculating the statistical moments of the frustum in ray-space (time and radius), and lifting these moments into 3D space using the ray direction.

### Inputs and Outputs
**Inputs:**
- `origins`: Tensor of shape $[B, 3]$, ray origin coordinates.
- `directions`: Tensor of shape $[B, 3]$, normalized ray direction vectors.
- `t_samples`: Tensor of shape $[B, n_{sample}+1]$, fencepost distances along the ray defining the frustum segments.
- `radii`: Tensor of shape $[B, 1]$, base radii of the rays (for cones).
- `ray_shape`: String, either `'cone'` or `'cylinder'`.
- `diagonal`: Boolean, if `True`, returns diagonal covariance matrices; otherwise, full $3 \times 3$ matrices.

**Outputs:**
- `means`: Tensor of shape $[B, n_{sample}, 3]$, 3D mean positions.
- `covs`: Tensor of shape $[B, n_{sample}, 3, 3]$ (or $[B, n_{sample}, 3]$ if diagonal), 3D covariance matrices.

### Data Structures and Layout
The input `t_samples` defines $n_{sample}$ segments. For each segment $i$ ($0 \le i < n_{sample}$), the frustum is bounded by $t_0 = t\_samples[:, i]$ and $t_1 = t\_samples[:, i+1]$.

### Parameters
- `eps`: Machine epsilon for `float16` (or the precision of `t_samples`), used for numerical stability.
- `ray_shape`: Determines the geometric model.
- `diagonal`: Determines the covariance representation.

### Steps

1. **Compute Ray-Space Moments:**
   For each segment defined by $t_0$ and $t_1$:
   - Compute intermediate parameters:
     $$ \mu = \frac{t_0 + t_1}{2} $$
     $$ hw = \frac{t_1 - t_0}{2} $$
   - Compute the stability denominator:
     $$ \text{denom} = \max(\epsilon, 3\mu^2 + hw^2) $$
     where $\epsilon$ is the machine epsilon of the input precision.

   - **If `ray_shape` is 'cone':**
     Calculate the mean and variance in the time (depth) dimension and the radial variance:
     $$ t_{\text{mean}} = \frac{3(t_1^4 - t_0^4)}{4(t_1^3 - t_0^3)} $$
     $$ t_{\text{mosq}} = \frac{3}{5} \frac{t_1^5 - t_0^5}{t_1^3 - t_0^3} $$
     $$ t_{\text{var}} = t_{\text{mosq}} - t_{\text{mean}}^2 $$
     $$ r_{\text{var}} = \text{base\_radius}^2 \left( \frac{3}{20} \frac{t_1^5 - t_0^5}{t_1^3 - t_0^3} \right) $$
     *Note: The source [S1, unofficial] provides an alternative formula for $t_{\text{mean}}$ and $t_{\text{var}}$ using $\mu, hw, \text{denom}$ which is numerically more stable. Implementers should use the stable formulas below for consistency with the "stable" flag in the source code.*
     
     Stable formulas for cone:
     $$ t_{\text{mean}} = \mu + \frac{2\mu hw^2}{\text{denom}} $$
     $$ t_{\text{var}} = \frac{hw^2}{3} - \frac{4}{15} \frac{hw^4 (12\mu^2 - hw^2)}{\text{denom}^2} $$
     $$ r_{\text{var}} = \text{base\_radius}^2 \left( \frac{\mu^2}{4} + \frac{5}{12}hw^2 - \frac{4}{15} \frac{hw^4}{\text{denom}} \right) $$

   - **If `ray_shape` is 'cylinder':**
     $$ r_{\text{var}} = \text{base\_radius}^2 \left( \frac{\mu^2}{4} + \frac{5}{12}hw^2 - \frac{4}{15} \frac{hw^4}{\text{denom}} \right) $$
     $$ t_{\text{mean}} = \mu $$
     $$ t_{\text{var}} = \frac{hw^2}{3} $$
     *(Note: The source [S1, unofficial] implies cylinder handling but raises `NotImplementedError` in the reference code. The formulas above are derived from standard cylinder moments.)*

2. **Lift to 3D:**
   Using the computed $t_{\text{mean}}$, $t_{\text{var}}$, and $r_{\text{var}}$, and the ray directions $d$:

   - **If `diagonal` is True:**
     $$ d_{\text{outer\_diag}} = d^2 $$
     $$ n_{\text{outer\_diag}} = 1 - \frac{d^2}{\sum d^2 + 10^{-10}} $$
     $$ t_{\text{cov\_diag}} = t_{\text{var}} \cdot d_{\text{outer\_diag}} $$
     $$ xy_{\text{cov\_diag}} = r_{\text{var}} \cdot n_{\text{outer\_diag}} $$
     $$ \text{cov\_diag} = t_{\text{cov\_diag}} + xy_{\text{cov\_diag}} $$
     $$ \text{mean} = d \cdot t_{\text{mean}} $$

   - **If `diagonal` is False:**
     Compute the full covariance matrix $\Sigma$. The source code implies a decomposition into axial and radial components.
     $$ \Sigma = t_{\text{var}} (d \otimes d) + r_{\text{var}} (I - \frac{d \otimes d}{\|d\|^2}) $$
     *(Note: The source [S1, unofficial] code snippet for diagonal case suggests the radial component is orthogonal to the direction. For full covariance, this projection is applied to the identity matrix.)*

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def conical_frustum_to_gaussian(origins, directions, t_samples, radii, ray_shape='cone', diagonal=True):
    """
    Approximates conical frustums as Gaussians.
    
    Args:
        origins: [B, 3]
        directions: [B, 3] (normalized)
        t_samples: [B, n_samples + 1]
        radii: [B, 1]
        ray_shape: 'cone' or 'cylinder'
        diagonal: bool
        
    Returns:
        means: [B, n_samples, 3]
        covs: [B, n_samples, 3, 3] or [B, n_samples, 3]
    """
    # t_samples shape: [B, n_samples + 1]
    t0 = t_samples[:, :-1]  # [B, n_samples]
    t1 = t_samples[:, 1:]  # [B, n_samples]
    
    # Intermediate parameters
    mu = (t0 + t1) / 2.0
    hw = (t1 - t0) / 2.0
    
    # Stability denominator
    eps = torch.finfo(t0.dtype).eps
    denom = torch.maximum(eps, 3.0 * mu**2 + hw**2)
    
    # Compute variances and means based on shape
    if ray_shape == 'cone':
        # Stable formulas from [S1, unofficial]
        t_mean = mu + (2.0 * mu * hw**2) / denom
        t_var = (hw**2) / 3.0 - (4.0 / 15.0) * (hw**4 * (12.0 * mu**2 - hw**2)) / (denom**2)
        r_var = (radii**2) * ((mu**2) / 4.0 + (5.0 / 12.0) * hw**2 - (4.0 / 15.0) * (hw**4) / denom)
    elif ray_shape == 'cylinder':
        # Derived from standard cylinder moments, using stable denom for consistency
        t_mean = mu
        t_var = (hw**2) / 3.0
        r_var = (radii**2) * ((mu**2) / 4.0 + (5.0 / 12.0) * hw**2 - (4.0 / 15.0) * (hw**4) / denom)
    else:
        raise ValueError("ray_shape must be 'cone' or 'cylinder'")
        
    # Lift to 3D
    # directions: [B, 3]
    # t_mean: [B, n_samples]
    # t_var: [B, n_samples]
    # r_var: [B, n_samples]
    
    # Expand dimensions for broadcasting
    # directions: [B, 1, 3]
    # t_mean, t_var, r_var: [B, n_samples, 1]
    
    d = directions.unsqueeze(1)  # [B, 1, 3]
    t_m = t_mean.unsqueeze(2)    # [B, n_samples, 1]
    t_v = t_var.unsqueeze(2)     # [B, n_samples, 1]
    r_v = r_var.unsqueeze(2)     # [B, n_samples, 1]
    
    # Mean
    means = d * t_m  # [B, n_samples, 3]
    
    if diagonal:
        # d_outer_diag: [B, 1, 3]
        d_outer_diag = d**2
        # sum(d^2) is 1.0 if normalized, but we add epsilon for safety
        # norm_sq = torch.sum(d**2, dim=2, keepdim=True) # [B, 1, 1]
        # However, directions are assumed normalized. 
        # Source uses: 1 - d**2 / (sum(d**2) + 1e-10)
        # If normalized, sum(d**2) is 1.
        
        # To be safe and match source logic exactly:
        norm_sq = torch.sum(d**2, dim=2, keepdim=True) + 1e-10
        null_outer_diag = 1.0 - d**2 / norm_sq
        
        t_cov_diag = t_v * d_outer_diag
        xy_cov_diag = r_v * null_outer_diag
        cov_diag = t_cov_diag + xy_cov_diag
        
        # cov_diag shape: [B, n_samples, 3]
        return means, cov_diag.squeeze(2) # [B, n_samples, 3]
    else:
        # Full covariance
        # Sigma = t_var * (d @ d.T) + r_var * (I - (d @ d.T) / ||d||^2)
        
        # d @ d.T: [B, n_samples, 3, 3]
        outer_dd = torch.bmm(d.transpose(1, 2), d) # [B, 1, 3, 3] -> [B, n_samples, 3, 3]
        # Actually d is [B, 1, 3]. d.transpose(1,2) is [B, 3, 1].
        # bmm([B, 3, 1], [B, 1, 3]) -> [B, 3, 3]
        # We need to broadcast t_v and r_v
        
        # Let's do it element-wise for clarity
        # d: [B, 1, 3]
        # outer_dd[i,j,k,l] = d[i,j,k] * d[i,j,l]
        outer_dd = d.unsqueeze(3) * d.unsqueeze(2) # [B, n_samples, 3, 3]
        
        # Identity matrix
        I = torch.eye(3, device=d.device, dtype=d.dtype).unsqueeze(0).unsqueeze(0) # [1, 1, 3, 3]
        
        # Projection matrix P = I - (d @ d.T) / ||d||^2
        # ||d||^2 is sum(d^2) over dim 2
        norm_sq = torch.sum(d**2, dim=2, keepdim=True) + 1e-10 # [B, 1, 1]
        P = I - outer_dd / norm_sq
        
        # Sigma = t_var * outer_dd + r_var * P
        # t_v: [B, n_samples, 1, 1]
        # r_v: [B, n_samples, 1, 1]
        
        covs = t_v * outer_dd + r_v * P
        
        return means, covs
```

### Edge Cases and Pitfalls
- **Numerical Stability:** Using non-stable formulas (direct division by $t_1^3 - t_0^3$) causes catastrophic failure when $t_0 \approx t_1$ or near the origin. The stable denominator `denom = max(eps, 3*mu^2 + hw^2)` must be used [S1, unofficial].
- **Normalized Directions:** The lifting step assumes `directions` are normalized. If not, `norm_sq` must be computed and used in the projection matrix $P$ [S1, unofficial].
- **Cylinder Support:** The reference code raises `NotImplementedError` for cylinders. Implementers must provide the cylinder moment formulas derived from standard geometry [S1, unofficial].

## Subtopics

### 2D Gaussian Covariance Stabilization and Rescaling
Computes the conic matrix and rescale factor for the 2D projection. Chosen when anti-aliasing requires minimum pixel coverage enforcement after 3D-to-2D projection.

### Mip-NeRF
The parent framework that introduced conical frustums for anti-aliased volume rendering. Chosen for general neural radiance field training with multi-scale representations.

## Operating characteristics
- **Time Complexity:** $O(N \cdot B)$, where $N$ is the number of samples per ray and $B$ is the batch size. The operations are element-wise tensor operations.
- **Memory Complexity:** $O(N \cdot B \cdot D)$, where $D$ is the dimensionality (3 for mean, 9 or 3 for covariance).
- **Strengths:** Enables differentiable integration of positional encodings and density over varying-width volumes. Provides anti-aliasing by accounting for the frustum's width.
- **Limitations:** Approximation error exists between the true frustum integral and the Gaussian moment. The diagonal covariance assumption reduces accuracy for off-axis viewing angles but is faster.
- **When to use:** When training neural radiance fields that require anti-aliasing (e.g., Mip-NeRF, 3D Gaussian Splatting variants).
- **When not to use:** For simple ray marching without positional encoding integration, or when exact volume integration is required without approximation.

## Background
The method originates from Mip-NeRF (Barron et al., 2021), which proposed representing rays as conical frustums to integrate neural network inputs over the volume of the ray, effectively anti-aliasing the rendering [S1, unofficial]. The specific "conical frustum to Gaussian" approximation allows these frustums to be represented as Gaussians, facilitating efficient computation of integrated positional encodings. This technique is a core component of 3D Gaussian Splatting and its variants, where 3D Gaussians are projected to 2D for rendering.

## Sources
[S1] AlexSheldrick/density_bounded_ray_weights, https://github.com/AlexSheldrick/density_bounded_ray_weights, fetched 2026-10-27 (unofficial)
