# FaCT-GS

## Overview
FaCT-GS is a framework for fast and flexible Computed Tomography (CT) reconstruction that optimizes the voxelization and rasterization pipelines of 3D Gaussian Splatting [S1]. It solves the problem of slow convergence and high computational cost in Gaussian-based tomographic reconstruction by removing view-dependent occlusion logic, coalescing data structures, and enabling rapid fitting to pre-existing volumes for warm-starting [S1]. It is distinguished from standard 3D Gaussian Splatting by its removal of depth-sorting overhead and from its sibling [GPS-Gaussian](9197-gps-gaussian.md) by its specific focus on CT geometry and voxelization-based initialization [S1].
## How it works
### Inputs and Outputs
**Inputs:**
*   A set of $N$ X-ray projection images (views).
*   Camera parameters for each view (pinhole or cone-beam geometry) [S1].
*   Optional: A pre-existing volumetric prior (e.g., from FDK reconstruction) for initialization [S1].

**Outputs:**
*   A set of $M$ 3D Gaussians $\{G_i\}_{i=1}^M$ representing the 3D volume.
*   Each Gaussian $i$ is defined by a center $\mathbf{p}_i$, a covariance matrix $\boldsymbol{\Sigma}_i$, an opacity/weight $G_i$, and other attributes [S1].

### Data Structures
To reduce memory traffic, parameters are coalesced into compact `float4` structs [S1]:
1.  **Voxelizer Struct:** Combines position and radius.
2.  **Rasterizer Struct:** Combines the projected conic (inverse covariance) $\hat{\boldsymbol{\Sigma}}_i^{-1}$ and the integration factor $\mu_i$ [S1].

### Rendering Formula
The projection of a 3D Gaussian onto the 2D detector plane yields a 2D Gaussian. The intensity $I(\hat{\mathbf{x}})$ at a pixel coordinate $\hat{\mathbf{x}}$ on the detector plane is computed as [S1]:

$$
I(\hat{\mathbf{x}}) = \sum_{i=1}^M G_i^2 \left( \hat{\mathbf{x}} \mid \mu_i \rho_i, \hat{\mathbf{p}}_i, \hat{\boldsymbol{\Sigma}}_i \right)
$$

Where:
*   $M$ is the number of Gaussians.
*   $G_i$ is the Gaussian weight.
*   $\mu_i$ is an integration factor [S1].
*   $\rho_i$ is a scaling factor [S1].
*   $\hat{\mathbf{p}}_i$ is the projected center of the $i$-th Gaussian.
*   $\hat{\boldsymbol{\Sigma}}_i$ is the projected covariance matrix of the $i$-th Gaussian.
*   The term $G_i^2 (\dots)$ represents the 2D Gaussian function evaluated at $\hat{\mathbf{x}}$ with the specified parameters [S1].

### Algorithm Steps
1.  **Initialization:** Initialize the set of 3D Gaussians. This can be done via random initialization, gradient-based initialization from an FDK reconstruction, or by fitting Gaussians to a pre-existing volumetric prior [S1].
2.  **Voxelization:** Discretize the 3D space into a regular grid. Accumulate Gaussian contributions within each voxel to produce a dense volumetric representation. This step avoids mapping Gaussians to a 2D plane, simplifying the logic but incurring cubic growth in grid elements [S1].
3.  **Projection:** For each view, project the 3D Gaussians onto the 2D detector plane. This transforms the 3D covariance $\boldsymbol{\Sigma}_i$ into the 2D projected covariance $\hat{\boldsymbol{\Sigma}}_i$ and the center $\mathbf{p}_i$ into $\hat{\mathbf{p}}_i$ [S1].
4.  **Culling:** For each pixel, identify contributing Gaussians. Accumulate contributions only for Gaussians satisfying the bounding box condition [S1]:
    $$
    |\hat{\mathbf{x}} - \hat{\mathbf{p}}_i| < 3\lambda_{\text{MAX}}(\hat{\boldsymbol{\Sigma}}_i)
    $$
    where $\lambda_{\text{MAX}}$ is the largest eigenvalue of the projected covariance $\hat{\boldsymbol{\Sigma}}_i$ [S1].
5.  **Rasterization:** Compute the intensity contribution of each culling-passed Gaussian using the rendering formula.
    *   **Optimization:** Unlike standard 3D Gaussian Splatting, do **not** sort Gaussians by depth or accumulate in a specific order. In CT, the integration order along the ray does not affect the measured signal. Removing this ordering logic reduces overhead [S1].
    *   Use the coalesced `float4` structs for memory-efficient access to conics and $\mu$ values [S1].
6.  **Optimization Loop:** Compare the rendered projections with the acquired X-ray images. Compute the loss and update the Gaussian parameters (centers, covariances, weights, etc.) via gradient descent [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_fact_gs(projections, gaussians, camera_params):
    """
    Renders 3D Gaussians to 2D projections for CT reconstruction.
    
    Args:
        projections: List of target projection images.
        gaussians: List of 3D Gaussian objects with attributes:
                   - center (3D vector)
                   - covariance (3x3 matrix)
                   - weight (scalar)
                   - mu (scalar, integration factor)
                   - rho (scalar, scaling factor)
        camera_params: Camera intrinsics and extrinsics for each view.
        
    Returns:
        rendered_images: List of rendered projection images.
    """
    rendered_images = []
    
    for view_idx, target_projection in enumerate(projections):
        camera = camera_params[view_idx]
        height, width = target_projection.shape
        
        # Initialize output image
        image = zeros((height, width))
        
        # Project all Gaussians to 2D for this view
        projected_gaussians = []
        for g in gaussians:
            # Project 3D center and covariance to 2D detector plane
            p_2d, sigma_2d = project_3d_gaussian_to_2d(g.center, g.covariance, camera)
            
            # Calculate bounding box for culling
            # lambda_max is the largest eigenvalue of sigma_2d
            lambda_max = max(eigenvalues(sigma_2d))
            radius = 3.0 * sqrt(lambda_max)
            
            projected_gaussians.append({
                'center': p_2d,
                'covariance': sigma_2d,
                'weight': g.weight,
                'mu': g.mu,
                'rho': g.rho,
                'radius': radius
            })
        
        # Rasterize
        # Note: No depth sorting is performed, as integration order is irrelevant in CT
        for g_proj in projected_gaussians:
            # Determine pixel range covered by the Gaussian's bounding box
            min_x, max_x, min_y, max_y = get_bounding_box(g_proj['center'], g_proj['radius'])
            
            # Clip to image bounds
            min_x = max(0, min_x)
            max_x = min(width, max_x)
            min_y = max(0, min_y)
            max_y = min(height, max_y)
            
            for y in range(min_y, max_y):
                for x in range(min_x, max_x):
                    pixel_coord = (x, y)
                    
                    # Culling check: distance from center
                    dist = norm(pixel_coord - g_proj['center'])
                    if dist < g_proj['radius']:
                        # Evaluate 2D Gaussian
                        # I(x) = G^2 * N(x | mu*rho, p, Sigma)
                        # N is the 2D Gaussian PDF
                        pdf = evaluate_2d_gaussian_pdf(
                            pixel_coord, 
                            g_proj['mu'] * g_proj['rho'], 
                            g_proj['center'], 
                            g_proj['covariance']
                        )
                        contribution = (g_proj['weight'] ** 2) * pdf
                        image[y, x] += contribution
        
        rendered_images.append(image)
        
    return rendered_images

def project_3d_gaussian_to_2d(center_3d, cov_3d, camera):
    """
    Projects a 3D Gaussian onto the 2D detector plane.
    Returns 2D center and 2D covariance.
    """
    # Apply camera transformation (rotation and translation)
    # This is a simplified representation; actual implementation depends on camera model
    # (pinhole or cone-beam)
    center_2d = project_point(center_3d, camera)
    
    # Project covariance matrix
    # This involves Jacobian of the projection function
    cov_2d = project_covariance(cov_3d, camera)
    
    return center_2d, cov_2d

def evaluate_2d_gaussian_pdf(x, scale_factor, mean, covariance):
    """
    Evaluates the 2D Gaussian PDF.
    """
    diff = x - mean
    # Inverse covariance
    inv_cov = inverse(covariance)
    # Mahalanobis distance squared
    exponent = -0.5 * dot(diff, dot(inv_cov, diff))
    # Normalization constant
    norm_const = 1.0 / (2.0 * pi * sqrt(determinant(covariance)))
    return norm_const * exp(exponent) * scale_factor
```

### Parameters
*   **Culling Threshold:** $3\lambda_{\text{MAX}}$ [S1].
*   **Execution Time:** Approximately 2 minutes per scan on 75 projections of $512^2$ size; competitive results after 45 seconds [S1].
*   **Scalability:** Generates high-quality reconstructions in about 1 minute for 1k projections and 4 minutes for 2k projections [S1].

### Edge Cases and Pitfalls
*   **Occlusion Logic:** Standard 3D Gaussian Splatting accumulates splat contributions in a specific order to model view-dependent occlusion. In CT, integration order along the ray has no effect on the measured signal. Removing this ordering logic is critical to avoid unnecessary overhead [S1].
*   **Memory Traffic:** Failure to coalesce function parameters into compact `float4` structs (combining position/radius in voxelizer, conics/$\mu$ in rasterizer) leads to higher memory traffic and reduced performance [S1].
*   **Voxelization Cost:** Voxelization is computationally expensive due to the cubic growth in the number of grid elements. Care must be taken with grid resolution [S1].

## Operating characteristics
*   **Time Complexity:** Significantly faster than State of the Art GS CT reconstruction. Over 4x faster on standard $512^2$ projections and over 13x faster on 2k projections [S1].
*   **Memory Complexity:** Reduced by coalescing parameters into `float4` structs [S1]. Voxelization incurs cubic growth in grid elements [S1].
*   **Scalability:** Scales well with projection and output volume size [S1].
*   **Strengths:** Fast execution, flexible initialization (including warm-starting from pre-existing volumes), and compression properties comparable to 2D JPEG stacks [S1].
*   **Limitations:** Voxelization can be computationally expensive for high-resolution grids [S1].
*   **When to use:** Sparse-view tomography, fast CT reconstruction, scenarios requiring warm-starting from prior volumes [S1].
*   **When not to use:** Standard 3D rendering with view-dependent occlusion is required [S1].

## Background
FaCT-GS was introduced to address the limitations of Gaussian Splatting-based CT reconstruction, specifically the slow convergence and high computational cost [S1]. It builds on the observation that projecting a 3D Gaussian yields a 2D Gaussian, allowing for simplified rendering equations [S1]. The method was developed to enable rapid fitting of Gaussians to pre-existing volumes, serving as a prior for warm-starting or as a compressed representation [S1]. It is related to R2-Gaussian, which also highlighted the irrelevance of integration order in CT, but FaCT-GS fully removes the ordering logic from the pipeline [S1].

## Sources
[S1] FaCT-GS: Fast and Scalable CT Reconstruction with Gaussian Splatting, https://doi.org/10.1007/978-3-032-37464-6_13, fetched 2026-10-26
