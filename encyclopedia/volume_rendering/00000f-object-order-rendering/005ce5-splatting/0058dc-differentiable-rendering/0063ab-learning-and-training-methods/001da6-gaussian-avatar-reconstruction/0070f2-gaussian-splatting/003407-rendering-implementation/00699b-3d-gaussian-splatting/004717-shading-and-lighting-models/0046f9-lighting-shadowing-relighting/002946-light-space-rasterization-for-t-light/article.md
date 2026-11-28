# Light Space Rasterization for T_light

Also called: Front Tau Rasterization, Light Space Shadow Rasterization, T_light Rasterization (Light Space)

## Overview
Light Space Rasterization for $T_{light}$ computes the front-facing occlusion transmittance for each 3D Gaussian primitive along a directional sun direction [S1, unofficial]. It solves the self-shadowing problem in differentiable Gaussian splatting by rasterizing Gaussians in a virtual light-space camera to determine which primitives are occluded by those in front [S1, unofficial]. Unlike voxel-based fallback methods, this approach provides full continuous gradients and zero covariance expansion by default, enabling accurate relighting of volumetric scenes like clouds [S1, unofficial]. It builds on [Lighting, Shadowing & Relighting](11894-lighting-shadowing-relighting.md) by providing the specific shading term $T_{light}$ required for physically based illumination.

## How it works
### Inputs and Outputs
**Inputs:**
*   3D Gaussian primitives: mean $\boldsymbol{\mu}$, covariance $\boldsymbol{\Sigma}$ (or scales and orientation), opacity $o$, and color $\mathbf{c}$ [G].
*   Sun direction: a unit vector $\mathbf{d}_{sun}$ defining the light source [S1, unofficial].
*   Light-space camera parameters: a distant narrow FOV perspective camera setup to approximate parallel rays [S1, unofficial].

**Outputs:**
*   $T_{light}$: The front-facing occlusion transmittance for each Gaussian [S1, unofficial].
*   Shadow mask: Binary or continuous shadow information per pixel/primitive [S1, unofficial].

### Data Structures
*   **Gaussian Buffer:** Contains $\boldsymbol{\mu}, \boldsymbol{\Sigma}, o, \mathbf{c}$ for all active Gaussians [G].
*   **Light-Space Projection Buffer:** Stores projected 2D positions, covariances, and depths for Gaussians sorted by depth in light space [S1, unofficial].
*   **Tau Accumulation Buffer:** Stores the accumulated front $\tau$ (optical depth) weighted by $\alpha \cdot T$ for each pixel during rasterization [S1, unofficial].

### Steps
1.  **Light-Space Camera Setup:** Construct a virtual perspective camera with a narrow field of view (FOV) placed at a large distance along the sun direction $\mathbf{d}_{sun}$ [S1, unofficial]. This setup ensures parallax error is less than 2%, preserving the validity of the EWA (Elliptical Weighted Average) Jacobian used in standard splatting [S1, unofficial].
2.  **Projection:** Project all 3D Gaussians into the light-space camera. Compute the 2D mean and covariance in screen space. **Crucially**, do not expand the projection covariance by $0.3I$ and do not apply amplitude compensation, as this pass is for shadowing, not final shading [S1, unofficial].
3.  **Sorting:** Sort the projected Gaussians by their depth (z-coordinate) in light space, from front to back [S1, unofficial]. This sorting is essential for correct front-to-back accumulation [Parent].
4.  **Rasterization (CUDA Kernel):** Iterate through the sorted Gaussians. For each Gaussian, rasterize its 2D ellipse onto the light-space image buffer [S1, unofficial].
    *   Use the `record_front_tau` channel to accumulate values [S1, unofficial].
    *   For each pixel covered by the Gaussian, update the accumulated front $\tau$ weighted by $\alpha \cdot T$ [S1, unofficial]. This represents an energy-weighted mean over the entire sun-facing footprint, not just a center-point sample [S1, unofficial].
5.  **Transmittance Calculation:** Compute $T_{light}$ for each Gaussian based on the accumulated $\tau$ from the rasterization pass. $T_{light}$ represents the fraction of light reaching the Gaussian from the sun direction [S1, unofficial].
6.  **Gradient Handling:** Enable full continuous gradients for $\tau$, weights, normalization, weak density backup transmittance, and projected position/scale/rotation [S1, unofficial]. Treat sun camera framing, sorting, coverage, and threshold branches as constants to prevent incorrect gradient propagation [S1, unofficial].
7.  **Fallback Handling:** Explicitly map deeply buried Gaussians (those resulting in early termination with $wsum=0$) to full shadow ($T_{light}=0$) to prevent inversion artifacts [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def light_space_rasterization(gaussians, sun_direction):
    # 1. Setup Light-Space Camera
    # Create a perspective camera at a large distance along sun_direction
    # with narrow FOV to approximate parallel rays.
    light_cam = PerspectiveCamera(
        position=sun_direction * LARGE_DISTANCE,
        target=Vector3(0, 0, 0),
        fov=NEAR_FOV,
        resolution=IMAGE_RESOLUTION
    )

    # 2. Project Gaussians
    projected_gaussians = []
    for g in gaussians:
        # Project mean and covariance to light space
        mu_2d, sigma_2d = project_gaussian(g, light_cam)
        
        # CRITICAL: Do NOT expand covariance by 0.3*I
        # CRITICAL: Do NOT apply amplitude compensation
        projected_gaussians.append({
            'mu': mu_2d,
            'sigma': sigma_2d,
            'opacity': g.opacity,
            'depth': g.depth_in_light_space,
            'original_index': g.index
        })

    # 3. Sort by depth (front-to-back)
    projected_gaussians.sort(key=lambda x: x['depth'])

    # 4. Rasterize and Accumulate Tau
    # Initialize tau buffer to 0
    tau_buffer = ZeroBuffer(IMAGE_WIDTH, IMAGE_HEIGHT)
    
    # Initialize T_light for each Gaussian
    t_light_values = [1.0] * len(gaussians)

    for pg in projected_gaussians:
        # Rasterize the 2D ellipse
        covered_pixels = get_covered_pixels(pg['mu'], pg['sigma'])
        
        for pixel in covered_pixels:
            # Calculate alpha for this pixel
            alpha = compute_gaussian_alpha(pg['mu'], pg['sigma'], pixel)
            
            # Update accumulated front tau weighted by alpha * T
            # Note: T is initially 1.0 for the first occluder
            current_tau = tau_buffer[pixel]
            tau_buffer[pixel] += alpha * current_tau
            
            # Store the tau value before this gaussian for later T_light calculation
            # This is an approximation; precise implementation requires 
            # storing the tau value *before* the current gaussian's contribution
            # for each gaussian covering the pixel.
            # For simplicity, we assume the buffer tracks the cumulative tau
            # and we can derive T_light from the final buffer state relative to depth.
            pass

    # 5. Compute T_light for each Gaussian
    # T_light is the transmittance *reaching* the Gaussian.
    # This is exp(-accumulated_tau_from_front_to_this_gaussian)
    for pg in projected_gaussians:
        # Retrieve the accumulated tau from pixels covered by this gaussian
        # Average or sum appropriately to get a representative tau for the gaussian
        avg_tau = compute_average_tau_for_gaussian(pg, tau_buffer)
        
        # Handle deeply buried Gaussians (wsum=0 case)
        if avg_tau is None or avg_tau == 0:
            t_light_values[pg['original_index']] = 0.0 # Full shadow
        else:
            t_light_values[pg['original_index']] = exp(-avg_tau)

    return t_light_values
```

### Parameters
*   **Parallax Error Threshold:** < 2% [S1, unofficial].
*   **Covariance Expansion:** 0 (disabled) [S1, unofficial].
*   **Amplitude Compensation:** Disabled [S1, unofficial].
*   **Gradient Mode:** Full continuous gradients enabled by default [S1, unofficial].

### Edge Cases and Pitfalls
*   **Deeply Buried Gaussians:** Gaussians that are fully occluded (early termination, $wsum=0$) must be explicitly mapped to full shadow ($T_{light}=0$) to avoid inversion artifacts [S1, unofficial].
*   **Gradient Propagation:** Sun camera framing, sorting, coverage, and threshold branches must be treated as constants during backpropagation to prevent incorrect gradients [S1, unofficial].
*   **Covariance Expansion:** Applying the standard $0.3I$ expansion or amplitude compensation in this pass will corrupt the shadow calculation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot A)$, where $N$ is the number of Gaussians and $A$ is the average number of pixels covered by each Gaussian's projection. Sorting is $O(N \log N)$.
*   **Memory Complexity:** $O(N + W \cdot H)$, where $W \cdot H$ is the light-space image resolution. Requires buffers for projected Gaussians and the tau accumulation buffer.
*   **Performance:** Provides full continuous gradients and zero expansion by default, offering higher accuracy than voxel-based fallbacks without requiring specific training parameters [S1, unofficial].
*   **Strengths:** Accurate self-shadowing for volumetric scenes; differentiable; compatible with standard EWA splatting.
*   **Limitations:** Requires an additional rasterization pass; sensitive to sun direction distribution in training data [S1, unofficial].
*   **When to Use:** When physically based relighting and self-shadowing are required for Gaussian splatting models, especially for volumetric data like clouds [S1, unofficial].
*   **When Not to Use:** For simple opaque scenes where self-shadowing is negligible; when computational resources are extremely limited and voxel fallback is acceptable [S1, unofficial].

## Background
This method was introduced in the context of 4D Gaussian Splatting for Volume Clouds, specifically for handling dynamic lighting and self-shadowing [S1, unofficial]. It addresses the limitation of pure emission-based SH (Spherical Harmonics) reconstruction, which cannot support relighting [S1, unofficial]. The method is unique to held-out sun relighting scenarios and relies on uniform directional coverage in the training dataset to ensure healthy geometric shadow gradients [S1, unofficial].

## Sources
[S1] hsiang0117/4DGS-Volume-Cloud, https://github.com/hsiang0117/4DGS-Volume-Cloud, fetched 2026-10-26 (unofficial)
