# Normal Field Supervision via Rendering and Depth Derivatives

Also called: Lemma 2, Supervision of Normal Field

## Overview
This method supervises the learnable surface normal field of a 3D Gaussian Splatting representation by enforcing geometric consistency between the rendered normal maps and the spatial derivatives of the rendered depth map. While [Surface Normal Rendering via External Channel](2099-surface-normal-rendering-via-external-channel.md) provides the mechanism to render normals as an external attribute, this method provides the loss signal. It differs from sibling methods like [Learnable Normals Parameterization](9615-learnable-normals-parameterization.md) by relying on self-supervision from the scene's own depth geometry rather than external pseudo-normal targets or lighting priors.

## How it works
The method establishes a supervisory loss by comparing two estimates of the surface normal at each pixel: one derived directly from the Gaussian attributes and one derived from the rendered depth field.

### Inputs and Outputs
*   **Inputs:**
    *   3D Gaussian Splatting representation $\mathcal{G}$, consisting of Gaussians $G_i$ with position $\mu_i$, covariance $\Sigma_i$, opacity $\alpha_i$, and learnable normal $n_i$.
    *   Camera parameters (origin $\mathbf{o}$, direction $\mathbf{w}$).
    *   Pixel coordinates $(u, v)$.
*   **Outputs:**
    *   Supervision loss $\mathcal{L}_{normal}$ minimizing the angular difference between the two normal estimates.

### Data Structures
*   **Rendered Normal Map:** A 2D array of 3D vectors $\mathbf{N}_{rendered}(u,v)$, computed via alpha-blending of Gaussian normals.
*   **Rendered Depth Map:** A 2D array of scalar depths $D(u,v)$, specifically the median depth as defined below.
*   **Depth Derivative:** A 2D array of 3D vectors $\mathbf{N}_{depth}$, computed via finite differences of $D(u,v)$.

### Steps
1.  **Render Normal Map:** Rasterize the Gaussian splats using standard alpha-blending. For a ray $\mathbf{r}_{o,w}$, the rendered normal $\mathbf{N}_{rendered}$ is the weighted average of the Gaussian normals $n_i$, weighted by their contribution to the color/opacity.
2.  **Render Median Depth:** Compute the depth $t^G_{o,w}$ for each pixel. This is defined as the depth of the first Gaussian along the ray where the accumulated transmittance $T_{o,w}(t)$ drops to or below 0.5.
3.  **Compute Depth Derivative:** Estimate the surface normal $\mathbf{N}_{depth}$ by taking the spatial gradient of the rendered median depth map $D(u,v)$.
4.  **Compute Loss:** Calculate the angular loss between $\mathbf{N}_{rendered}$ and $\mathbf{N}_{depth}$.

### Formulas
The core insight is that the derivative of the depth map corresponds to the normal of the expected visible surface. The method uses the **median** free-flight distance $t^*_{o,w}$ for robustness against outliers, rather than the expectancy.

The median depth $t$ is defined as the infimum of $t$ such that the transmittance $T_{o,w}(t) \leq 0.5$ [S1]:

$$
\text{Median}_M(t_{\mathbf{o},\mathbf{w}}^*) = \inf \{ t \in [0, +\infty) : T_{\mathbf{o},\mathbf{w}}(t) \leq 0.5 \}
$$

In the context of point-based rendering, this is approximated as the depth $t^G_{o,w}$ of the first Gaussian $i$ along the ray such that:

$$
T_{o,w}(t^G_{o,w}) \leq 0.5
$$

where $T_{o,w}(t)$ is the transmittance up to distance $t$.

The rendered normal $\mathbf{N}_{rendered}$ is obtained by alpha-compositing the learnable normals $n_i$ of the splats. The depth-derived normal $\mathbf{N}_{depth}$ is computed by taking the gradient of the depth map $D(u,v)$ in the image plane and converting it to world space.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_normal_supervision_loss(gaussians, camera, image_coords):
    """
    Computes the supervision loss for normal fields using depth derivatives.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
                   - position: (3,)
                   - opacity: float
                   - normal: (3,) unit vector n_i
                   - covariance: (3,3) or scale/rotation
        camera: Camera object with origin o and view/projection matrices.
        image_coords: List of (u, v) pixel coordinates to render.
        
    Returns:
        loss: Scalar tensor representing the angular difference.
    """
    
    # Step 1: Render Normal Map
    # Rasterize Gaussians. For each pixel, accumulate normals weighted by alpha.
    # N_rendered = sum(alpha_i * n_i) / sum(alpha_i) for visible splats
    N_rendered = render_normal_map(gaussians, camera, image_coords)
    
    # Step 2: Render Median Depth Map
    # For each ray, find the first Gaussian where transmittance <= 0.5
    # T(t) = product(1 - alpha_j) for all j before t
    # We sort splats by depth along the ray.
    D_median = render_median_depth(gaussians, camera, image_coords)
    
    # Step 3: Compute Depth Derivative (Normal from Depth)
    # Compute spatial gradients of the depth map
    # dD_dx, dD_dy are gradients in image space
    dD_dx, dD_dy = compute_depth_gradients(D_median)
    
    # Convert depth gradients to surface normal in camera space
    # N_depth_cam = (dD_dx, dD_dy, -1) normalized, adjusted for focal length
    N_depth_cam = depth_gradient_to_normal(dD_dx, dD_dy, camera.focal_length)
    
    # Transform to world space if necessary to match N_rendered
    N_depth_world = transform_to_world(N_depth_cam, camera.extrinsics)
    
    # Step 4: Compute Loss
    # Angular loss between rendered normal and depth-derived normal
    # loss = 1 - dot(N_rendered, N_depth_world)
    dot_product = dot(N_rendered, N_depth_world)
    # Clamp to avoid numerical issues with acos
    dot_product = clamp(dot_product, -1.0, 1.0)
    loss = 1.0 - dot_product
    
    return loss

def render_median_depth(gaussians, camera, image_coords):
    """
    Renders the median depth map.
    Median depth is the depth of the first Gaussian where transmittance <= 0.5.
    """
    depth_map = zeros_like(image_coords)
    
    for each pixel (u, v) in image_coords:
        ray = camera.get_ray(u, v)
        # Get all Gaussians intersecting this ray, sorted by distance
        splats = get_sorted_splats_along_ray(gaussians, ray)
        
        transmittance = 1.0
        median_depth = None
        
        for splat in splats:
            # Calculate transmittance contribution of this splat
            # In standard Gaussian Splatting, alpha is pre-computed or derived from covariance
            alpha = splat.alpha
            
            # If adding this splat drops transmittance below 0.5, this is the median depth
            if transmittance * (1 - alpha) <= 0.5:
                median_depth = splat.depth_along_ray
                break
            
            transmittance *= (1 - alpha)
            
        if median_depth is None:
            # Fallback if transmittance never drops below 0.5 (e.g., transparent background)
            # Not given in sources. Typically set to max depth or infinity.
            median_depth = MAX_DEPTH 
            
        depth_map[u, v] = median_depth
        
    return depth_map
```

### Parameters
*   **Transmittance Threshold:** 0.5. This defines the median. [S1]
*   **Gradient Window:** Not given in the sources. Standard finite difference (e.g., Sobel or central difference) is implied by "derivative of the rendered depth" [S1].

### Edge Cases and Pitfalls
*   **Specular/Transparent Regions:** Artifacts in the normal field correspond to under-reconstructed parts of the scene (e.g., specular spots, sky, background) [S1]. The depth derivative may be noisy or undefined in these regions.
*   **Outliers:** The method explicitly chooses median depth over expectancy to be robust to statistical outliers [S1]. Using expectancy would make the supervision sensitive to deep, transparent Gaussians.
*   **Orientation Ambiguity:** The normal derived from depth derivatives has an inherent sign ambiguity (inward vs outward). The learnable normals $n_i$ must be oriented consistently with the depth gradient direction. [G] Typically, the dot product between the rendered normal and the depth-derived normal is maximized, or the depth-derived normal is flipped if the dot product is negative.

## Subtopics

### [Learnable Normal Estimation with Pseudo-Normal Supervision](10917-learnable-normal-estimation-with-pseudo-normal-supervision.md)
This method introduces a learnable unit normal per 3D Gaussian to replace noisy, gradient-based normal estimates, preventing the optimization from overfitting to training lighting conditions by aligning rendered normals with geometry priors derived from depth gradients. It differs from pure gradient-based approaches by assigning explicit normal attributes to Gaussians and using expected depth to generate stable pseudo-normal targets for supervision.

## Operating characteristics
*   **Time Complexity:** Dominated by the rasterization step for normal and depth maps, which is $O(N)$ per pixel where $N$ is the number of splats. The depth derivative computation is $O(1)$ per pixel.
*   **Memory Complexity:** Requires storing the rendered normal map and depth map. $O(W \times H)$ per frame.
*   **Strengths:** Self-supervised; does not require ground-truth normals or external geometry. Robust to outliers via median depth.
*   **Limitations:** Relies on the accuracy of the depth map. Fails in textureless or specular regions where depth estimation is ambiguous. The "first Gaussian" approximation for median depth may be inaccurate if Gaussians are large or overlapping significantly.

## Background
Originating from the paper "From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians" [S1], this method (Lemma 2) leverages the closed-form equation for vacancy and the ability to compute transmittance to define a robust median depth. It builds on the observation that the derivative of the depth map estimates the normal of the expected visible surface [S1].

## Sources
[S1] From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians, https://doi.org/10.1007/978-3-032-37095-2_4, fetched 2026-10-29
