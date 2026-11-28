# Trilinear Interpolation for X-ray Attenuation Approximation

Also called: Fast approximation of X-ray attenuation, Interpolatory quadrature via the rectangular rule

## Overview
This method approximates the line integral of the Linear Attenuation Coefficient (LAC) along an X-ray beam using trilinear interpolation and the rectangular rule, serving as a differentiable forward model for sparse-view Cone-Beam CT (CBCT) reconstruction [S1, unofficial]. It sets itself apart from exact geometric methods like Siddon's method by trading precise voxel intersection calculations for speed and a regularization effect where rendered pixels influence non-intersected voxels, thereby preventing overfitting in sparse-view scenarios [S1, unofficial].

## How it works
**Inputs:**
- Ray defined by source position $\mathbf{s} \in \mathbb{R}^3$ and detector pixel position $\mathbf{p} \in \mathbb{R}^3$ [S1, unofficial].
- Voxel grid of LAC values $\hat{\mu}$, sampled via trilinear interpolation [S1, unofficial].
- Number of sample points $M$ along the ray [S1, unofficial].

**Outputs:**
- Approximated X-ray intensity $I_{\hat{\mu}}(\vec{r})$ [S1, unofficial].

**Data Structures:**
- $\hat{\mu}$: Scalar field defined on a 3D rectilinear grid. Trilinear interpolation $\hat{\mu}[\mathbf{x}]$ returns the weighted average of the 8 corner voxels surrounding coordinate $\mathbf{x}$ [S1, unofficial].
- $\alpha_m$: Scalar parameters for evenly spaced points along the ray segment from source to pixel [S1, unofficial].

**Steps:**
1. Define the ray parametric equation $\vec{r}(\alpha) = \mathbf{s} + \alpha(\mathbf{p} - \mathbf{s})$ for $\alpha \in [0, 1]$ [S1, unofficial].
2. Sample $M$ evenly spaced points $\alpha_1, \dots, \alpha_M$ along the ray such that $\alpha_1 = 0$ and $\alpha_M = 1$ [S1, unofficial].
3. Initialize the sum $S = 0$ [S1, unofficial].
4. Iterate $m$ from $1$ to $M-1$:
   a. Evaluate the LAC at the start point of the segment using trilinear interpolation: $\hat{\mu}_m = \hat{\mu}[\mathbf{s} + \alpha_m(\mathbf{p} - \mathbf{s})]$ [S1, unofficial].
   b. Calculate the weighted segment length factor: $w_m = \frac{\alpha_{m+1} - \alpha_{m-1}}{2}$ [S1, unofficial]. For the first segment ($m=1$), $\alpha_0$ is undefined in the source; a standard boundary convention for this fast approximation is to clamp $\alpha_0 = \alpha_1$ [G].
   c. Accumulate the product: $S \leftarrow S + \hat{\mu}_m \cdot w_m$ [S1, unofficial].
5. Multiply the accumulated sum by the total ray length: $I_{\hat{\mu}}(\vec{r}) = \|\mathbf{p} - \mathbf{s}\| \cdot S$ [S1, unofficial].

**Formula:**
$$I_{\hat{\mu}}(\vec{r}) \approx \|\mathbf{p} - \mathbf{s}\| \sum_{m=1}^{M-1} \hat{\mu} [\mathbf{s} + \alpha_m(\mathbf{p} - \mathbf{s})] \frac{(\alpha_{m+1} - \alpha_{m-1})}{2}$$
where:
- $\hat{\mu}[\cdot]$ is the trilinear interpolation of the voxel grid [S1, unofficial].
- $\alpha_m$ are evenly spaced parameters in $[0, 1]$ [S1, unofficial].
- $M$ is the number of sample points [S1, unofficial].

**Parameters:**
- $M = 500$ sample points per ray [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trilinear_xray_attenuation(s, p, mu_grid, M=500):
    """
    Approximates X-ray attenuation using trilinear interpolation and rectangular rule.
    
    Args:
        s: Source position (3D vector)
        p: Detector pixel position (3D vector)
        mu_grid: Trilinear interpolator for the voxel grid LAC
        M: Number of sample points along the ray
        
    Returns:
        Approximated intensity I_mu(r)
    """
    # 1. Define ray direction and length
    ray_dir = (p[0] - s[0], p[1] - s[1], p[2] - s[2])
    ray_length = (ray_dir[0]**2 + ray_dir[1]**2 + ray_dir[2]**2)**0.5
    
    # 2. Sample M evenly spaced points alpha_1 ... alpha_M
    # alpha_1 = 0, alpha_M = 1
    alphas = [i / (M - 1) for i in range(M)]
    
    # 3. Initialize sum
    total_sum = 0.0
    
    # 4. Iterate m from 1 to M-1
    for m in range(1, M):
        # Map formula indices (1-based) to list indices (0-based)
        alpha_m = alphas[m - 1]
        alpha_m_plus_1 = alphas[m]
        alpha_m_minus_1 = alphas[0] if m == 1 else alphas[m - 2]
        
        # Calculate weight
        weight = (alpha_m_plus_1 - alpha_m_minus_1) / 2.0
        
        # Evaluate trilinear interpolation at s + alpha_m * (p - s)
        point = (s[0] + alpha_m * ray_dir[0], 
                 s[1] + alpha_m * ray_dir[1], 
                 s[2] + alpha_m * ray_dir[2])
        mu_val = mu_grid.interpolate(point)  # [G] Calls standard trilinear interpolation from parent article
        
        # Add to sum
        total_sum += mu_val * weight
        
    # 5. Multiply by ray length
    intensity = ray_length * total_sum
    
    return intensity
```

## Operating characteristics
- **Time Complexity:** Linear in the number of sample points $M$ ($O(M)$) [S1, unofficial]. It is faster than Siddon's method, which scales cubically with the resolution of the voxel grid $\hat{\mu}$ due to plane intersection calculations [S1, unofficial].
- **Memory Complexity:** Depends on the storage of the voxel grid $\hat{\mu}$ and the ray batch size [S1, unofficial]. The batch size for trilinear interpolation was set to $1.8 \times 10^6$ rays, larger than Siddon's $5.5 \times 10^5$ due to lower memory overhead per ray [S1, unofficial].
- **Measured Performance:** On a dataset of 40 walnuts with 15 input views, using a single NVIDIA RTX A6000 GPU, the method achieved SSIM 0.912, PSNR 37.23, MSE $1.43 \times 10^{-5}$, and PCC 0.96 [S1, unofficial]. The reconstruction runtime was 10.0 minutes for this specific configuration [S1, unofficial]. For 60-view reconstruction, the runtime was 39.1 minutes [S1, unofficial].
- **Strengths:** Faster than Siddon's method; prevents overfitting in sparse-view settings by allowing ground truth pixels to supervise more voxels (including those not directly intersected); approximates second-order effects like X-ray scatter [S1, unofficial].
- **Limitations:** Provides modestly lower quality than Siddon's method in novel view synthesis, particularly in very sparse-view settings (5-20 views) where Siddon's exact geometry yields higher fidelity [S1, unofficial].

## Background
This method was introduced in the DiffVox framework for differentiable CBCT reconstruction [S1, unofficial]. It addresses the computational bottleneck of exact ray-voxel intersection (Siddon's method) in differentiable rendering pipelines [S1, unofficial]. By using interpolatory quadrature, it enables a fully differentiable forward model that is linear in sample count, facilitating faster optimization of the voxel grid $\hat{\mu}$ [S1, unofficial].

## Sources
[S1] Differentiable Voxel-based X-ray Rendering Improves Sparse-View 3D CBCT Reconstruction, https://doi.org/10.48550/arxiv.2411.19224, fetched 2026-10-26 (unofficial)
