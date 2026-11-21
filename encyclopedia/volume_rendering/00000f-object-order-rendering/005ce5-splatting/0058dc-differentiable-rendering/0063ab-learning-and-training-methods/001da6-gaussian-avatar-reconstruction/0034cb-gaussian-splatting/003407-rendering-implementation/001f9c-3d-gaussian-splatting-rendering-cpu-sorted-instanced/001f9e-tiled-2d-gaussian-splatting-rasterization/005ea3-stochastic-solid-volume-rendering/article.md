# Stochastic Solid Volume Rendering

Also called: SSVR

## Overview
Stochastic Solid Volume Rendering (SSVR) is a volume rendering technique used as an alternative to vanilla 3D Gaussian Splatting (3DGS) rasterization to improve geometric fidelity, particularly for depth and surface reconstruction [S1, unofficial]. It produces RGB images nearly identical to standard 3DGS rasterization under suitable conditions but offers geometric advantages by reducing holes and blurred boundaries in depth maps [S1, unofficial]. It is equivalent to 3DGS rasterization when an appropriate vacancy function is applied [S1, unofficial].

## How it works
**Inputs:**
- 3D Gaussian primitives defined by parameters $\theta$ (position, covariance, opacity, color).
- Camera parameters (view matrix, projection matrix, intrinsics).
- Image resolution and tile configuration (if using tiled rasterization).

**Outputs:**
- RGB image.
- Depth map (median depth).

**Data Structures:**
- Gaussian primitives: Standard 3DGS representation (mean $\mu$, covariance $\Sigma$, opacity $\alpha$, spherical harmonics coefficients for color).
- Ray parameters: Origin $o$, direction $d$.

**Core Concept:**
SSVR reformulates the rendering integral using a "vacancy function" $v(x)$ [S1, unofficial]. The density $\sigma(x, \omega)$ is defined as:
$$ \sigma(x, \omega) = \omega^\top \nabla \log v(x) $$
where $\omega$ represents the view-dependent color coefficients [S1, unofficial].

**Equivalence Condition:**
To obtain RGB images equivalent to 3DGS rasterization, the vacancy function must be:
$$ v(x) = \sqrt{1 - G(x)} $$
where $G(x)$ is the Gaussian density function [S1, unofficial].

**Rendering Steps:**

1.  **Ray-Gaussian Intersection:** For each pixel, cast a ray. For each Gaussian $i$ intersecting the ray, compute the position $t$ along the ray relative to the Gaussian's maximum $t^*_i$ along the ray direction. $t^*_i$ is the sample point where the Gaussian density is maximized along the ray [S1, unofficial].

2.  **Transmittance Calculation:** Compute the accumulated transmittance $T_i(t)$ for each Gaussian $i$. The transmittance is defined piecewise based on the position $t$ relative to $t^*_i$ [S1, unofficial]:
    $$
    T_i(t) =
    \begin{cases}
    \sqrt{1 - G_i(t)}, & t \le t^*_i \\
    \sqrt{1 - G_i(t^*_i)}, & t > t^*_i
    \end{cases}
    $$
    where $G_i(t)$ is the cumulative opacity (or density integral) of Gaussian $i$ at position $t$ [S1, unofficial]. Note: The source text shows $T_i(t) = 1 - G_i(t)$ in the formula block but explicitly states $v(x) = \sqrt{1-G(x)}$ and $T(t) = \exp(-\int \sigma ds) = v(x)/v(t_n)$. The piecewise formula in the source explicitly uses the square root form $\sqrt{1-G_i(t)}$ [S1, unofficial]. We follow the explicit piecewise formula provided in Equation 3 of the source [S1, unofficial].

3.  **Color Computation:** The final color $C$ is computed by integrating the radiance along the ray:
    $$ C = \int_{t_n}^{t_f} T(t) \sigma(x(t), \omega) c(x(t), \omega) dt $$
    where $T(t)$ is the accumulated transmittance from all Gaussians, $\sigma$ is the density, and $c$ is the color [S1, unofficial]. In practice, this is discretized over the sorted Gaussians.

4.  **Depth Rendering (Median Depth):** For depth rendering, compute the median depth $t_{med}$ defined by the point where the accumulated transmittance $T(t_{med}) = 0.5$ [S1, unofficial].
    -   Find $t_{med}$ using a fixed number of bisection iterations [S1, unofficial].

5.  **Backpropagation:** During training, differentiate the fixed condition $T(t_{med}) = 0.5$ with respect to Gaussian parameters $\theta$ [S1, unofficial]. Using the chain rule, the gradient is computed as [S1, unofficial]:
    $$ \frac{\partial T}{\partial t} \frac{\partial t_{med}}{\partial \theta} + \frac{\partial T}{\partial \theta} = 0 $$
    This allows gradients to flow through the depth definition to optimize the Gaussian parameters [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def stochastic_solid_volume_rendering(gaussians, camera, image_shape):
    """
    Renders RGB and Depth using Stochastic Solid Volume Rendering.
    
    Args:
        gaussians: List of Gaussian primitives. Each has:
            - mean: (3,)
            - covariance: (3,3)
            - opacity: scalar
            - color: (3,) or SH coefficients
        camera: Camera object with project() and unproject() methods.
        image_shape: (H, W)
        
    Returns:
        rgb: (H, W, 3)
        depth: (H, W)
    """
    H, W = image_shape
    rgb = np.zeros((H, W, 3))
    depth = np.zeros((H, W))
    
    # Pre-project Gaussians to 2D to get bounding boxes and screen-space covariance
    # This step is assumed to be handled by the parent Tiled 2D Gaussian Splatting Rasterization
    # which assigns Gaussians to tiles. Here we assume we iterate over visible Gaussians per tile/pixel.
    
    for y in range(H):
        for x in range(W):
            # Cast ray for pixel (x, y)
            ray_o, ray_d = camera.get_ray(x, y)
            
            # Collect intersecting Gaussians for this ray
            # In a tiled implementation, this would be done per tile
            visible_gaussians = []
            for g in gaussians:
                # Check if Gaussian intersects ray and is within tile bounds
                if gaussian_intersects_ray(g, ray_o, ray_d):
                    # Compute t* (position of max density along ray)
                    t_star = compute_gaussian_max_position(g, ray_o, ray_d)
                    visible_gaussians.append((g, t_star))
            
            # Sort Gaussians by t_star (front to back)
            visible_gaussians.sort(key=lambda item: item[1])
            
            # Initialize accumulated transmittance
            T_acc = 1.0
            color_acc = np.zeros(3)
            depth_samples = []
            
            # Render loop
            for g, t_star in visible_gaussians:
                # Sample points along the ray segment covered by this Gaussian
                # For simplicity, assume we evaluate at the center or integrate
                # The source defines T_i(t) piecewise around t_star
                
                # Compute G_i(t) at current ray position t
                # Note: In discrete rasterization, we often evaluate at the center of the splat
                # or use the precomputed 2D Gaussian.
                # Let's assume we have a function to get G(t) for a Gaussian at ray position t
                t_current = compute_ray_position(g, ray_o, ray_d) # Placeholder for actual intersection
                
                G_i_t = compute_gaussian_density(g, t_current)
                
                # Compute T_i(t) according to Eq 3
                if t_current <= t_star:
                    T_i = np.sqrt(1.0 - G_i_t)
                else:
                    G_i_t_star = compute_gaussian_density(g, t_star)
                    T_i = np.sqrt(1.0 - G_i_t_star)
                
                # Update accumulated transmittance
                # T_acc_new = T_acc_old * T_i
                # However, the source defines T(t) as the accumulated transmittance.
                # In standard volume rendering, T(t) = exp(-int sigma ds).
                # Here, the vacancy function approach implies a specific integration.
                # We follow the source's definition: T_i(t) is the transmittance contribution.
                
                # Standard alpha blending with vacancy function modification
                # The source implies T(t) is the product of individual T_i(t) or similar.
                # Let's assume standard accumulation: T_acc *= T_i
                
                # Actually, the source says "accumulated transmittance is then computed as shown in Equation 3"
                # Equation 3 defines T_i(t) for a single Gaussian.
                # The total T(t) is likely the product of all T_i(t) for Gaussians behind the current point.
                
                # For color accumulation:
                # C = int T(t) sigma c dt
                # With vacancy function, sigma = omega^T grad log v
                # This simplifies to color blending with modified weights.
                
                # Simplified implementation for 3DGS-like rasterization:
                # Weight w_i = T_acc * (1 - T_i) / (1 - T_i) ? No.
                # Standard 3DGS: w_i = alpha_i * prod(T_j)
                # Here, we use the vacancy function.
                
                # Let's use the explicit formula for depth and color from the source context.
                # The source emphasizes the equivalence to 3DGS when v(x) = sqrt(1-G).
                # So we can use standard 3DGS rasterization logic but with the vacancy function for depth.
                
                # Compute opacity contribution
                # In 3DGS, alpha = 1 - exp(-sigma * delta)
                # Here, we use the vacancy function directly.
                
                # For RGB:
                # Color contribution from Gaussian i
                # C_i = T_acc * (1 - T_i) * c_i ? 
                # The source doesn't give the exact color blending formula, only the transmittance.
                # However, it states equivalence to 3DGS.
                # So we use standard 3DGS color blending:
                # alpha_i = 1 - T_i (if T_i is the transmittance *after* the Gaussian)
                # But T_i(t) is defined as the transmittance *of* the Gaussian.
                
                # Let's assume standard alpha blending with the computed T_i
                # T_acc is the transmittance *before* this Gaussian.
                # The transmittance *after* this Gaussian is T_acc * T_i.
                # The absorbed color is T_acc * (1 - T_i) * c_i.
                
                # Wait, T_i(t) in Eq 3 is defined as sqrt(1-G).
                # If G is the cumulative opacity, then 1-G is the remaining transmittance?
                # No, G is density.
                
                # Let's stick to the source's explicit depth formula and general equivalence.
                
                # Update T_acc
                T_acc *= T_i
                
                # Accumulate color
                # Assuming standard 3DGS color blending logic applies due to equivalence
                # alpha_contribution = 1 - T_i (if T_i is the transmittance through the Gaussian)
                # But T_i is defined as sqrt(1-G).
                # If G is small, T_i ~ 1.
                
                # Let's use the standard 3DGS rasterization for color, as they are equivalent.
                # The difference is in depth.
                
                # For depth, we need to track the transmittance curve.
                depth_samples.append((t_current, T_acc))
            
            # Compute RGB using standard 3DGS blending (due to equivalence)
            # This is a placeholder for the actual color integration
            rgb[y, x] = compute_color_from_gaussians(visible_gaussians, ray_o, ray_d)
            
            # Compute Depth using Median Transmittance
            # Find t_med such that T(t_med) = 0.5
            # T(t) is the accumulated transmittance.
            # We have samples (t, T_acc).
            # Interpolate to find t where T_acc = 0.5
            
            t_med = find_median_depth(depth_samples, target_T=0.5)
            depth[y, x] = t_med
            
    return rgb, depth

def compute_gaussian_density(g, t):
    """Compute G(t) for a Gaussian at position t along the ray."""
    # Standard 3DGS density computation
    # G(t) = alpha * exp(-0.5 * (t - mu)^T Sigma^-1 (t - mu))
    # This is a placeholder. The exact implementation depends on the 3DGS density function.
    pass

def compute_gaussian_max_position(g, ray_o, ray_d):
    """Compute t* where Gaussian density is maximized along the ray."""
    # This is the projection of the Gaussian mean onto the ray, adjusted for covariance.
    # For a 3D Gaussian, the maximum along a ray is at the intersection of the ray with the Gaussian's principal axis?
    # Or simply the point on the ray closest to the mean?
    # In 3DGS, the 2D projection is used.
    # For 3D volume rendering, we need the 3D density.
    # The source mentions t* is the location of the Gaussian maximum along the ray.
    pass

def find_median_depth(samples, target_T):
    """Find t such that T(t) = target_T using bisection."""
    # Sort samples by t
    samples.sort(key=lambda x: x[0])
    
    # Bisection
    # We need to interpolate between samples to find t where T(t) = target_T
    # T(t) is piecewise constant or linear?
    # In volume rendering, T(t) is continuous.
    # We can interpolate T(t) between samples.
    
    # Simple bisection on the ray segment
    t_min = samples[0][0]
    t_max = samples[-1][0]
    
    for _ in range(10): # Fixed number of iterations
        t_mid = (t_min + t_max) / 2.0
        T_mid = interpolate_T(t_mid, samples)
        if T_mid > target_T:
            t_min = t_mid
        else:
            t_max = t_mid
            
    return (t_min + t_max) / 2.0

def interpolate_T(t, samples):
    """Interpolate accumulated transmittance T(t) at position t."""
    # Find the segment containing t
    # T(t) is the product of T_i for all Gaussians with t_i < t
    # This requires re-evaluating the product, which is expensive.
    # In practice, we precompute the T values at sample points.
    # Here, we assume samples are dense enough or we use linear interpolation.
    pass
```

**Edge Cases and Pitfalls:**
-   **Precision:** The square root in the vacancy function $v(x) = \sqrt{1 - G(x)}$ requires $G(x) \le 1$. If $G(x) > 1$ due to numerical errors, the result is undefined. Guard against this by clamping $G(x)$ to $[0, 1]$ [S1, unofficial].
-   **Boundaries:** The piecewise definition of $T_i(t)$ depends on $t^*_i$. Incorrect computation of $t^*_i$ leads to incorrect transmittance. $t^*_i$ must be computed accurately as the position of maximum density along the ray [S1, unofficial].
-   **Degenerate Input:** If a Gaussian is degenerate (e.g., zero covariance), the density is infinite. Handle by capping density or skipping such Gaussians [G].

## Operating characteristics
-   **Time Complexity:** Similar to 3DGS rasterization, $O(N \cdot P)$ where $N$ is the number of Gaussians and $P$ is the number of pixels, assuming efficient sorting and rasterization. The depth computation adds a small constant factor due to bisection [S1, unofficial].
-   **Memory Complexity:** Same as 3DGS, $O(N)$ for Gaussian storage. Additional memory for depth samples during bisection is negligible [S1, unofficial].
-   **Performance:** Real-time performance is achievable on modern GPUs, similar to 3DGS [S1, unofficial].
-   **Strengths:** Improved geometric fidelity for depth and surface reconstruction; reduced holes and blurred boundaries in depth maps [S1, unofficial].
-   **Limitations:** Slightly more complex implementation due to the vacancy function and median depth computation [S1, unofficial].
-   **When to use:** When geometric accuracy (depth, surface) is more important than raw RGB fidelity, or when 3DGS rasterization produces poor depth maps [S1, unofficial].
-   **When not to use:** When RGB fidelity is the only concern and 3DGS rasterization is sufficient, as SSVR offers no RGB advantage [S1, unofficial].

## Background
Stochastic Solid Volume Rendering was introduced as an optimization-based method for 3D Gaussian Splatting to improve geometric representation [S1, unofficial]. It builds on the work of Miller et al. (2023), Zhang et al. (2026), and Gomez et al. (2026) who proved the equivalence between SSVR and 3DGS rasterization under suitable conditions [S1, unofficial]. The method is used in contexts requiring accurate camera parameters, often estimated with SfM pipelines, and can be combined with test-time optimization for feed-forward models [S1, unofficial].

## Sources
[S1] VoxelTTO: Voxel-Aligned Feed-Forward 3D Gaussian Splatting with Test-Time Optimization, https://doi.org/10.48550/arxiv.2609.21498, fetched 2026-10-26 (unofficial)
