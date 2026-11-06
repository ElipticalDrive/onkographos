# Gaussian Kernel Volumetric Primitive

Also called: 3D Gaussian PDF for Volume Rendering, 3D Gaussian Splatting (Method), Normalized Gaussian Kernel

## Overview
A method for volumetric rendering that represents a scene as a set of 3D Gaussian distributions, parameterized by mean (position), covariance (shape/direction), and color, allowing for efficient simulation of 3D structure details without meshing [S2, unofficial]. It differs from traditional mesh-based reconstruction by avoiding meshing processes and from NeRF by offering faster training and rendering speeds while maintaining comparable quality [S2, unofficial]. Unlike other kernel methods like Epanechnikov, Gaussian kernels have infinite support, which requires clipping to maintain efficiency [S1, unofficial].

## How it works
**Inputs:** A set of 3D points (point cloud) or primitives, each defined by a mean position $\mathbf{c}_i$, a covariance matrix $\Sigma_i$, and color attributes.
**Outputs:** A 2D image with pixel colors and alpha values.

**Data Structures:**
Each primitive $i$ is stored with:
- $\mathbf{c}_i \in \mathbb{R}^3$: Mean position (XYZ).
- $\Sigma_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix defining shape and orientation.
- Color attributes (e.g., RGB or spherical harmonics coefficients).

**Steps:**
1.  **Parameterization:** Each point in the input point cloud is parameterized as a Gaussian kernel defined by its mean, covariance, and color [S2, unofficial]. The normalized Gaussian kernel function is:
    $$
    g_i(\mathbf{x}) = \frac{1}{(2\pi)^{3/2} |\Sigma_i|^{1/2}} e^{-\frac{1}{2} (\mathbf{x}-\mathbf{c}_i)^T \Sigma_i^{-1} (\mathbf{x}-\mathbf{c}_i)}
    $$
    where $\mathbf{c}_i$ is the center and $\Sigma_i$ is the covariance matrix [S1, unofficial].

2.  **Projection:** Each 3D Gaussian is projected from 3D space onto the 2D screen space to determine its influence range and size [S2, unofficial].

3.  **Transmittance and Sampling:** For ray tracing, transmittance is computed using the error function ($\text{erf}$) for a closed-form solution [S1, unofficial]. Free-flight distance for a single Gaussian primitive is sampled using the inverse error function ($\text{erf}^{-1}$) for a closed-form solution [S1, unofficial].

4.  **Blending:** The contributions of all Gaussians are combined using alpha blending to generate the final pixel color [S2, unofficial]. Alpha blending allows multiple Gaussian points to combine smoothly, handling complex lighting and reflections [S2, unofficial].

**Pitfalls:**
- Gaussian kernels have infinite support, requiring clipping to avoid evaluating all primitives, which reduces efficiency [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_gaussian_volumetric(scene_gaussians, camera, image_width, image_height):
    """
    Renders a scene represented by 3D Gaussian primitives.
    
    Args:
        scene_gaussians: List of Gaussian primitives. Each primitive is a dict/object with:
            - mean: (3,) array, position c_i
            - cov: (3,3) array, covariance matrix Sigma_i
            - color: (3,) array or SH coefficients
        camera: Camera object with projection matrix and view matrix.
        image_width: int
        image_height: int
        
    Returns:
        color_image: (H, W, 3) array
        alpha_image: (H, W, 1) array
    """
    color_image = np.zeros((image_height, image_width, 3))
    alpha_image = np.zeros((image_height, image_width, 1))
    
    # Initialize transmittance for each pixel
    transmittance = np.ones((image_height, image_width, 1))
    
    # Sort Gaussians by depth or process in screen-space order
    # Note: Exact sorting strategy depends on implementation details not fully specified in sources [G]
    
    for gaussian in scene_gaussians:
        c_i = gaussian['mean']
        Sigma_i = gaussian['cov']
        color = gaussian['color']
        
        # Project 3D Gaussian to 2D screen space
        # Determine influence range and size [S2, unofficial]
        # This involves projecting the covariance matrix and finding the bounding box
        # 2D Covariance Sigma_2D is derived from 3D Sigma_i and camera projection [G]
        
        # Get bounding box of the projected Gaussian
        bbox = get_bounding_box_2d(Sigma_i, c_i, camera)
        
        # Clip to image bounds
        bbox = clip_to_image(bbox, image_width, image_height)
        
        if not bbox:
            continue
            
        y_min, y_max, x_min, x_max = bbox
        
        # Iterate over pixels in the bounding box
        for y in range(y_min, y_max):
            for x in range(x_min, x_max):
                # Get pixel center in screen space
                pixel_center = get_pixel_center(x, y, camera)
                
                # Evaluate Gaussian contribution at this pixel
                # g_i(x) is the density/value of the Gaussian at the ray intersection or pixel center
                # For volume rendering, we often use the error function for closed-form transmittance [S1, unofficial]
                
                # Calculate density/value
                # Using the normalized Gaussian kernel formula [S1, unofficial]
                # dist_sq = (pixel_center - c_i)^T * Sigma_i^-1 * (pixel_center - c_i)
                # value = exp(-0.5 * dist_sq) / sqrt(det(Sigma_i))
                
                # For volume rendering with alpha blending:
                # Alpha is derived from the integral of the density along the ray
                # Closed-form solution uses erf [S1, unofficial]
                
                alpha = compute_alpha_gaussian(gaussian, pixel_center, camera)
                
                # Alpha blending
                # C_out = C_in * (1 - alpha) + C_gauss * alpha
                # A_out = A_in * (1 - alpha) + alpha
                
                current_alpha = alpha_image[y, x, 0]
                current_color = color_image[y, x, :]
                
                # Blend
                blend_factor = alpha * transmittance[y, x, 0]
                color_image[y, x, :] += color * blend_factor
                alpha_image[y, x, 0] += alpha * transmittance[y, x, 0]
                transmittance[y, x, 0] *= (1 - alpha)
                
    return color_image, alpha_image

def compute_alpha_gaussian(gaussian, pixel_center, camera):
    """
    Computes the alpha value for a Gaussian primitive at a specific pixel.
    Uses closed-form solution involving error function [S1, unofficial].
    """
    # Implementation details for erf-based alpha calculation are not fully specified in sources [G]
    # Typically involves projecting the 3D Gaussian to 2D and integrating along the ray
    # alpha = 1 - exp(-integral_of_density)
    # For Gaussian, integral relates to erf
    pass
```

## Subtopics

### [Gaussian Interpolation](2347-gaussian-interpolation.md)
Uses a Gaussian function as an interpolation kernel. It is the only filter that is both separable and rotationally symmetric.

### [Normalized 3D Gaussian Function](2348-normalized-3d-gaussian-function.md)
Defines the mathematical form of the normalized 3D Gaussian used as the primitive.

### [Gaussian Splatting](2346-gaussian-splatting.md)
The broader rendering technique that uses these primitives for view synthesis.

## Operating characteristics
- **Time Complexity:** Rendering is efficient, achieving real-time performance.
- **Memory Complexity:** Depends on the number of Gaussians $N$.
- **Performance:** Achieves high-quality real-time view synthesis at 1080p resolution with a frame rate of ≥ 100 fps (or ≥ 30 fps in other cited contexts) for unbounded, complete scenes [S2, unofficial].
- **Strengths:** Faster training and rendering speeds compared to NeRF [S2, unofficial]. Does not require meshing [S2, unofficial]. Integrates with physical-based rendering engines (scattering, emissive media) [S1, unofficial].
- **Limitations:** Infinite support of Gaussian kernels requires clipping, reducing efficiency compared to finite-support kernels like Epanechnikov [S1, unofficial]. Epanechnikov kernels may be faster and better for sharp edges [S1, unofficial].

## Background
3D Gaussian Splatting is a method for 3D reconstruction and rendering that represents a point cloud as a set of 3D Gaussian distributions [S2, unofficial]. It was introduced as a way to efficiently simulate 3D structure details without meshing [S2, unofficial]. The method allows for efficient simulation of 3D structure details without meshing [S2, unofficial]. It is used in 3D reconstruction and rendering [S2, unofficial]. The approach is part of the Gaussian Splatting family, which aims to provide faster alternatives to Neural Radiance Fields (NeRF) [S2, unofficial].

## Sources
[S1] Don’t Splat your Gaussians: Volumetric Ray-Traced Primitives for Modeling and Rendering Scattering and Emissive Media (2025), https://qiita.com/matumu20/items/67b33681dc97ccbcc849, fetched 2026-10-27 (unofficial)
[S2] 3D Gaussian Splatting 入门原创, https://blog.csdn.net/qq_35831906/article/details/143322205, fetched 2026-10-30 (unofficial)
