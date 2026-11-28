# Inner 3D Gaussian Splatting

Also called: Inner Gaussian Splatting, InnerGS

## Overview
Inner 3D Gaussian Splatting (InnerGS) is a framework for direct volumetric inference that models continuous volumetric density using 3D Gaussians to reconstruct internal structures from sparse, pose-free sliced data [S1, unofficial]. It solves the problem that standard 3D Gaussian Splatting (3DGS) relies on camera parameters (view/projection matrices) which are absent in sliced data, and that learned Gaussians concentrate on the object's surface, leaving the interior volume empty [S1, unofficial]. It sets itself apart from siblings like [GS-IR](14852-gs-ir.md) by eliminating the need for camera poses and using a slice-based rendering pipeline centered on Conditional Splatting, which dynamically adapts the sampling of Gaussians for each 2D slice [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Sparse, pose-free sliced data (e.g., medical CT/MRI slices or cross-sections). Each slice provides 2D image data at a specific depth $t$.
*   **Output:** A 3D Gaussian representation (means $\mu$, covariances $\Sigma$, opacities $\alpha$, colors $c$) and the rendered volumetric density/color for any depth slice $t$.

**Data Structures:**
The method uses the standard 3D Gaussian representation defined in [3D Gaussian Splatting Implementation Details](9128-3d-gaussian-splatting-implementation-details.md). Each primitive $i$ is defined by:
*   Mean position $\mu_i \in \mathbb{R}^3$
*   Covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$
*   Opacity $\alpha_i$
*   Color $c_i$ (modeled via Spherical Harmonics coefficients) [S1, unofficial]

**Formulas:**
The screen-space contribution of the $i$-th splat at pixel $(u, v)$ on a slice at depth $t$ is computed as density $p_i(u, v, t)$ multiplied by opacity $\alpha_i$ and color $c_i$. The final color is accumulated front-to-back [S1, unofficial]:

$$ I(u, v, t) = \sum_{i=1}^{N} p_i(u, v, t)\alpha_i c_i \prod_{j=1}^{i-1} (1 - p_j(u, v, t)\alpha_j) $$

where $N$ is the total number of Gaussians, and $(u, v)$ are pixel coordinates with $u \in [0, W - 1]$ and $v \in [0, H - 1]$ for an image of width $W$ and height $H$ [S1, unofficial].

The 3D Gaussian density is defined by an unnormalized Gaussian function [S1, unofficial]:

$$ p(x) = e^{-\frac{1}{2} (x-\mu)^\top \Sigma^{-1} (x-\mu)} $$

The covariance matrix $\Sigma$ is decomposed into a scaling component $S$ (diagonal matrix with scale factors $s_x, s_y, s_z$) and a rotation component $R$ (derived from unit quaternion $q$):

$$ \Sigma = R S S^\top R^\top $$

**Steps:**
1.  **Conditional 2D Gaussian Splatting:** For each depth slice $t$, compute conditional 2D Gaussian splats. This determines the 2D Gaussian center and influence radii in the image plane for that specific slice [S1, unofficial]. This step adapts the bounding box per slice based on conditional Gaussians, unlike 3D Ellipsoid Projection which computes a cube and projects identical bounding boxes across slices [S1, unofficial].
2.  **Marginal 1D Gaussian Combination:** Combine the conditional 2D Gaussians with a marginal 1D Gaussian along the depth axis to formulate the complete 3D Gaussian Splatting (3DGS) density [S1, unofficial]. This allows the representation to capture continuous volumetric information between sparse slices.
3.  **Tile-based Rasterization:** Perform tile-based rasterization using the formulated 3DGS density. The Gaussians are sorted and rendered in front-to-back order to reconstruct smooth and detailed internal structures [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_inner_gs(slices, gaussians, width, height):
    """
    Render internal structures from sparse slices using InnerGS.
    
    Args:
        slices: List of 2D image arrays, one per depth slice.
        gaussians: List of 3D Gaussian primitives. Each primitive has:
                   - mu: (3,) mean position
                   - S: (3,) scale factors
                   - q: (4,) unit quaternion for rotation
                   - alpha: float opacity
                   - sh_coeffs: array of spherical harmonics coefficients
        width: int image width W
        height: int image height H
        
    Returns:
        rendered_volumes: List of 2D image arrays, one per depth slice.
    """
    rendered_volumes = []
    
    # Iterate over each depth slice t
    for t, slice_data in enumerate(slices):
        # Initialize output buffer for this slice
        color_buffer = np.zeros((height, width, 3))
        alpha_buffer = np.zeros((height, width))
        
        # Step 1 & 2: Formulate 3DGS density for this slice
        # This involves computing conditional 2D Gaussians and combining 
        # with marginal 1D Gaussian along depth.
        # The result is a set of 2D splats valid for slice t.
        splats_for_slice = compute_conditional_splats(gaussians, t, width, height)
        
        # Sort splats by depth (front-to-back) for this slice
        # Note: In InnerGS, depth is the slice index t, but splats have 
        # 3D extent. Sorting is typically done by the z-coordinate of 
        # the Gaussian center projected or evaluated at slice t.
        splats_for_slice.sort(key=lambda s: s.z_center)
        
        # Step 3: Tile-based rasterization
        # For each pixel (u, v) in the slice
        for v in range(height):
            for u in range(width):
                accumulated_color = np.array([0.0, 0.0, 0.0])
                accumulated_alpha = 0.0
                
                for splat in splats_for_slice:
                    # Compute density p_i(u, v, t) for this splat at pixel (u, v)
                    # p_i is derived from the 3D Gaussian density evaluated 
                    # at the slice t and projected to 2D.
                    density = compute_gaussian_density(splat, u, v, t)
                    
                    # Get color c_i from SH coefficients and view direction (if applicable)
                    # For internal scenes, view direction might be fixed or averaged.
                    color = compute_color(splat.sh_coeffs)
                    
                    alpha = splat.alpha
                    
                    # Alpha compositing
                    transmittance = 1.0 - accumulated_alpha
                    contribution = density * alpha * color * transmittance
                    
                    accumulated_color += contribution
                    accumulated_alpha += density * alpha * transmittance
                    
                    # Early exit if fully opaque
                    if accumulated_alpha >= 1.0:
                        break
                
                color_buffer[v, u] = accumulated_color
                alpha_buffer[v, u] = accumulated_alpha
        
        rendered_volumes.append(color_buffer)
        
    return rendered_volumes

def compute_conditional_splats(gaussians, t, width, height):
    """
    Compute conditional 2D Gaussian splats for a specific depth slice t.
    This adapts the bounding box per slice based on conditional Gaussians.
    """
    splats = []
    for g in gaussians:
        # Compute 2D Gaussian parameters conditioned on slice t
        # This involves combining the 3D Gaussian with a marginal 1D Gaussian 
        # along the depth axis.
        # The center and covariance in the image plane are adjusted for slice t.
        center_2d, cov_2d = project_gaussian_to_slice(g, t)
        
        # Compute influence radius based on cov_2d
        radius = compute_radius(cov_2d)
        
        # Only include splats that overlap with the image bounds
        if overlaps_image(center_2d, radius, width, height):
            splats.append({
                'mu_2d': center_2d,
                'cov_2d': cov_2d,
                'alpha': g.alpha,
                'sh_coeffs': g.sh_coeffs,
                'z_center': g.mu[2], # For sorting
                'radius': radius
            })
    return splats

def compute_gaussian_density(splat, u, v, t):
    """
    Compute the density p_i(u, v, t) for a splat at pixel (u, v) on slice t.
    """
    # Extract 2D center and covariance
    mu_2d = splat['mu_2d']
    cov_2d = splat['cov_2d']
    
    # Vector from center to pixel
    delta = np.array([u - mu_2d[0], v - mu_2d[1]])
    
    # Compute exponent: -0.5 * delta^T * cov_2d^-1 * delta
    inv_cov = np.linalg.inv(cov_2d)
    exponent = -0.5 * delta @ inv_cov @ delta
    
    # Density
    density = np.exp(exponent)
    return density

def compute_color(sh_coeffs):
    """
    Compute color from SH coefficients.
    For internal scenes, view direction might be simplified.
    """
    # Standard SH evaluation, assuming a default view direction or 
    # averaged over views if not specified.
    # This is a placeholder for the specific SH evaluation used in InnerGS.
    # Typically: c = SH_eval(sh_coeffs, view_direction)
    # Since InnerGS is pose-free, the view direction might be implicit 
    # or the SH coefficients are optimized for a canonical view.
    # [G] Assuming a default view direction (0, 0, 1) for simplicity.
    view_direction = np.array([0.0, 0.0, 1.0])
    return evaluate_sh(sh_coeffs, view_direction)

def evaluate_sh(sh_coeffs, view_direction):
    """
    Evaluate spherical harmonics for a given view direction.
    """
    # Standard SH basis evaluation
    # This function depends on the degree of SH used (e.g., degree 3 for 45 coeffs)
    # [G] Placeholder for actual SH evaluation logic.
    # In practice, this involves projecting the view direction onto SH basis functions.
    pass

def project_gaussian_to_slice(gaussian, t):
    """
    Project a 3D Gaussian to a 2D slice at depth t.
    Combines conditional 2D Gaussian with marginal 1D Gaussian.
    """
    mu = gaussian.mu
    cov = gaussian.cov # Sigma = RSS^T R^T
    
    # The marginal 1D Gaussian along depth is used to condition the 2D Gaussian.
    # The 2D center and covariance are adjusted based on the slice t.
    # [G] Specific formula for this conditioning is not detailed in the source,
    # but it implies a factorization of the 3D Gaussian into 2D (slice) and 1D (depth) components.
    # The center in the image plane is likely the projection of mu, adjusted for t.
    # The covariance is the projection of Sigma, adjusted for t.
    
    # Simplified projection:
    # 2D center is the x,y component of mu.
    # 2D covariance is the upper-left 2x2 block of Sigma, potentially scaled by 
    # the 1D Gaussian density at t.
    
    mu_2d = mu[:2]
    cov_2d = cov[:2, :2]
    
    return mu_2d, cov_2d

def compute_radius(cov_2d):
    """
    Compute influence radius from 2D covariance.
    """
    # Radius is typically proportional to the square root of the largest eigenvalue
    # or a multiple of the standard deviation.
    eigenvalues = np.linalg.eigvals(cov_2d)
    max_eigenvalue = np.max(eigenvalues)
    radius = 3.0 * np.sqrt(max_eigenvalue) # 3 sigma rule
    return radius

def overlaps_image(center, radius, width, height):
    """
    Check if a splat overlaps with the image bounds.
    """
    x, y = center
    if x - radius < 0 or x + radius >= width or y - radius < 0 or y + radius >= height:
        return False
    return True
```

**Parameters:**
*   Pixel coordinates $(u, v)$ where $u \in [0, W - 1]$ and $v \in [0, H - 1]$ [S1, unofficial].
*   Position vector $\mu = (\mu_x, \mu_y, \mu_z)$ [S1, unofficial].
*   Covariance matrix $\Sigma$ decomposed into scaling $S$ and rotation $R$ [S1, unofficial].
*   Spherical harmonics (SH) coefficients for view-dependent color [S1, unofficial].
*   Opacity parameter $\alpha$ [S1, unofficial].

**Pitfalls:**
*   Standard 3DGS projection-based mechanism fails for internal volumetric structures because learned Gaussians concentrate on the object's surface, leaving the interior volume empty [S1, unofficial].
*   Standard 3DGS relies on camera parameters (view/projection matrices) which are absent in sliced data [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. However, it uses tile-based rasterization similar to 3DGS, which is generally efficient. The conditional splatting adds overhead per slice.
*   **Memory Complexity:** Not given in the sources. Depends on the number of Gaussians $N$ and the number of slices.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Eliminates the need for camera poses; plug-and-play; inherently compatible with any data modalities; reconstructs smooth and detailed internal structures from sparse sliced data [S1, unofficial].
*   **Limitations:** Relies on sparse sliced data; may not capture complex view-dependent effects as well as full 3DGS if SH degree is low [G].
*   **When to use:** When reconstructing internal structures from sparse, pose-free sliced data (e.g., medical imaging).
*   **When not to use:** When camera poses are available and the goal is surface reconstruction or novel-view synthesis of external scenes (use standard 3DGS).

## Background
Inner 3D Gaussian Splatting was proposed by Shuxin Liang et al. in 2025 [S1, unofficial]. It addresses the limitations of NeRF and standard 3DGS in modeling internal scenes. NeRF requires heavy MLPs and is slow to render, while standard 3DGS fails for internal structures due to its projection-based nature and reliance on camera poses [S1, unofficial]. InnerGS provides a direct volumetric inference framework using 3D Gaussian density [S1, unofficial].

## Sources
[S1] InnerGS: Internal Scenes Reconstruction and Segmentation via Factorized 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2508.13287, fetched 2026-10-26 (unofficial)
