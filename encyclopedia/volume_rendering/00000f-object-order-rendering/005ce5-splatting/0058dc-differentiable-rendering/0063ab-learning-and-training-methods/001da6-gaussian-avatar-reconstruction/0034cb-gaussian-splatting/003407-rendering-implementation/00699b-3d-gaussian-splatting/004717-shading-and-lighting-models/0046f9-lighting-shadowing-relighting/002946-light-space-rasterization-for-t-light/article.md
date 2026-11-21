# Light Space Rasterization for T_light

Also called: Front Tau Rasterization, Light Space Shadow Rasterization, T_light Rasterization (Light Space)

## Overview
Light Space Rasterization computes $T_{light}$, the front-facing occlusion transmittance for each Gaussian along a directional sun direction, enabling physically based self-shadowing in 3D Gaussian Splatting. It differs from voxel-based fallbacks by providing full continuous gradients and zero EWA expansion by default, avoiding the inversion artifacts associated with early-termination in deep occlusion [S1, unofficial].

## How it works
**Inputs:**
*   3D Gaussian primitives (position, covariance, opacity, color).
*   Sun direction vector.
*   Camera parameters for the "sun camera" (see below).

**Outputs:**
*   $T_{light}$ value for each Gaussian (or pixel coverage thereof).
*   Shadowed RGB image.

**Data Structures:**
*   **Sun Camera:** A perspective camera with a narrow Field of View (FOV) placed at a large distance to simulate parallel sun rays. The parallax error is kept below 2% to avoid changing the EWA Jacobian [S1, unofficial].
*   **Splat Buffer:** Standard Gaussian splat attributes.
*   **Material Buffer:** Parallel buffer storing normal, roughness, reflectance, and albedo for relighting [4464-ref-gaussian-deferred-surfel-rendering-with-split-sum-ibl.md].
*   **T_light Buffer:** Stores the computed front tau ($\tau$) and transmittance.

**Steps:**
1.  **Camera Setup:** Construct the sun camera. The framing, sorting, coverage, and threshold branches are treated as constants during the pass to prevent incorrect gradient propagation [S1, unofficial].
2.  **Sorting:** Sort Gaussians by depth relative to the sun camera. This ensures correct front-to-back accumulation for shadowing [4487-3d-gaussian-splatting-self-shadowing-t_light.md, 4489-light-space-rasterization-for-t_light.md].
3.  **Rasterization Pass:**
    *   Project Gaussians into light space.
    *   **Crucial Difference:** Do **not** expand the projection covariance by $0.3I$ and do **not** apply amplitude compensation. This preserves the main camera pass's sampling expansion [4487-3d-gaussian-splatting-self-shadowing-t_light.md].
    *   Traverse Gaussians in depth order.
    *   For each Gaussian, accumulate front $\tau$ (optical depth) weighted by $\alpha \cdot T$. This is an energy-weighted mean over the entire sun-facing footprint, not a center-point sample [S1, unofficial].
    *   Use the `record_front_tau` channel logic.
4.  **Shadow Application:** Combine $T_{light}$ with the base color.
    *   **Pitfall Handling:** Deeply buried Gaussians (where early-termination results in $wsum=0$) are explicitly mapped to full shadow to prevent inversion artifacts [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_t_light(gaussians, sun_direction, sun_camera):
    """
    Compute T_light using light space rasterization.
    """
    # 1. Setup Sun Camera
    # sun_camera is a perspective camera with narrow FOV, large distance
    # Parallax error < 2%
    
    # 2. Sort Gaussians by depth in sun space
    sorted_gaussians = sort_by_depth(gaussians, sun_camera)
    
    # 3. Initialize T_light buffer
    t_light_buffer = zeros_like(gaussians)
    
    # 4. Rasterization
    for gaussian in sorted_gaussians:
        # Project to light space
        pos_light = sun_camera.project(gaussian.position)
        
        # Compute covariance in light space
        # NOTE: Do NOT expand covariance by 0.3*I
        # NOTE: Do NOT apply amplitude compensation
        cov_light = transform_covariance(gaussian.covariance, sun_camera.rotation)
        
        # Compute coverage and alpha
        alpha = compute_gaussian_alpha(pos_light, cov_light, gaussian.opacity)
        
        # Accumulate front tau
        # Weighted by alpha * T (transmittance from previous steps)
        # This is an energy-weighted mean over the footprint
        tau_contribution = alpha * gaussian.density * gaussian.scale_factor
        
        # Update transmittance
        # T_light = exp(-tau)
        t_light_buffer[gaussian.id] = compute_transmittance(tau_contribution)
        
    # 5. Handle Deep Occlusion
    # Map deeply buried Gaussians (wsum ~ 0) to full shadow
    for gaussian in gaussians:
        if t_light_buffer[gaussian.id] < threshold:
            t_light_buffer[gaussian.id] = 0.0
            
    return t_light_buffer
```

**Parameters:**
*   **Sun Camera FOV:** Narrow, to approximate parallel rays.
*   **Sun Camera Distance:** Large, to minimize parallax.
*   **Covariance Expansion:** None (0.0) for this pass [S1, unofficial].
*   **Amplitude Compensation:** None [S1, unofficial].

**Edge Cases:**
*   **Deep Occlusion:** If a Gaussian is deeply buried, early termination might result in $wsum=0$. Explicitly map these to full shadow ($T_{light}=0$) to avoid inversion artifacts [S1, unofficial].
*   **Gradient Propagation:** Sun camera framing, sorting, coverage, and threshold branches are treated as constants to ensure stable gradients [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \log N)$ for sorting, $O(N)$ for rasterization, where $N$ is the number of Gaussians.
*   **Memory Complexity:** $O(N)$ for storing $T_{light}$ and sorted indices.
*   **Performance:** Provides full continuous gradients and zero expansion by default, which is more accurate than voxel fallbacks [S1, unofficial].
*   **Strengths:** Physically plausible self-shadowing, differentiable, no need for specific training parameters like voxelization.
*   **Limitations:** Requires an additional rasterization pass. Sensitive to sun camera placement (parallax error).

## Background
This method is part of the 4DGS-Volume-Cloud framework for rendering volumetric clouds with 3D Gaussian Splatting. It enables relighting and self-shadowing, which are not possible with pure emission SH reconstruction (VPRF) [S1, unofficial]. The method was introduced in the context of training on datasets like `CloudDatasetUniform` with held-out sun directions for generalization testing [S1, unofficial].

## Sources
[S1] hsiang0117/4DGS-Volume-Cloud, https://github.com/hsiang0117/4DGS-Volume-Cloud, fetched 2026-10-26 (unofficial)
