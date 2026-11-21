# 3D Gaussian Splatting to Point Cloud

Also called: 3DGS to Point Cloud Conversion, 3DGS-to-PC, Gaussian Splatting Point Cloud Generation, Gaussian Splatting to Dense Point Clouds

## Overview
This method converts a trained 3D Gaussian Splatting (3DGS) scene representation into a dense, standard point cloud format (e.g., PLY) that approximates the original geometry and appearance, enabling compatibility with general 3D software that does not support Gaussian splats [S1, unofficial]. It distinguishes itself from sibling methods like [SuGaR Mesh Extraction](12580-sugar-mesh-extraction.md) by focusing on generating a high-fidelity point cloud (and optionally a mesh via Poisson reconstruction) directly from the Gaussian distributions, rather than extracting a watertight mesh through explicit surface optimization [S1, unofficial]. The process involves filtering Gaussians by visibility and opacity, sampling points from the Gaussian probability distributions, and optionally re-rendering colors from camera views to ensure photorealistic appearance in the final point cloud [S1, unofficial].

## How it works
**Inputs:**
- A 3DGS scene file (PLY or splat) containing $N$ Gaussians. Each Gaussian $i$ has mean $\boldsymbol{\mu}_i$, covariance $\boldsymbol{\Sigma}_i$ (derived from scale $\mathbf{S}_i$ and rotation $\mathbf{R}_i$), opacity $\alpha_i$, and spherical harmonic (SH) coefficients for color.
- Camera parameters (transforms, intrinsics) if color rendering is enabled.
- Configuration settings (see Parameters).

**Outputs:**
- A point cloud file (PLY) containing points $\mathbf{p}_j$, colors $\mathbf{c}_j$, and optionally normals $\mathbf{n}_j$.
- Optionally, a mesh file (PLY) generated from the point cloud.

**Data Structures:**
- `GaussPointCloudSettings`: NamedTuple containing:
  - `renderer_type`: str ('cuda' or 'python')
  - `num_points`: int (default 10,000,000)
  - `prioritise_visible_gaussians`: bool (default True)
  - `mahalanobis_distance_std`: float (default 2.0)
  - `camera_skip_rate`: int (default 0)
  - `render_colours`: bool (default True)
  - `min_opacity`: float (default 0.0)
  - `bounding_box_min`: list of 3 floats (default None)
  - `bounding_box_max`: list of 3 floats (default None)
  - `calculate_normals`: bool (default True)
  - `cull_large_percentage`: float (default 0.0)
  - `remove_unrendered_gaussians`: bool (default True)
  - `colour_resolution`: int (mapped from `colour_quality`)
  - `max_sh_degree`: int (default 3)
  - `exact_num_points`: bool (default False)
  - `visibility_threshold`: float (default 0.05)
  - `surface_distance_std`: float (default None)
  - `generate_mesh`: bool (default False)
  - `quiet`: bool (default False)
  - `device`: str (default 'cuda')
- `PointCloudData`: NamedTuple containing:
  - `points`: torch.Tensor of shape $(M, 3)$
  - `colours`: torch.Tensor of shape $(M, 3)$
  - `normals`: torch.Tensor of shape $(M, 3)$

**Parameters:**
- `num_points`: Total target points. Default: 10,000,000 [S1, unofficial].
- `exact_num_points`: If True, forces the output to match `num_points` closely (slower). Default: False [S1, unofficial].
- `no_prioritise_visible_gaussians`: If True, ignores visibility contributions when distributing points. Default: False [S1, unofficial].
- `visibility_threshold`: Minimum Gaussian contribution to be included. Default: 0.05 [S1, unofficial].
- `surface_distance_std`: Cull Gaussians further than X standard deviations from predicted surfaces. Default: Disabled [S1, unofficial].
- `clean_pointcloud`: Remove outliers using Open3D. Default: False [S1, unofficial].
- `generate_mesh`: Generate mesh via Poisson reconstruction. Default: False [S1, unofficial].
- `poisson_depth`: Depth for Poisson reconstruction. Default: 10 [S1, unofficial].
- `laplacian_iterations`: Laplacian smoothing iterations for mesh. Default: 10 [S1, unofficial].
- `camera_skip_rate`: Cameras to skip during rendering. Default: 0 [S1, unofficial].
- `no_render_colours`: Skip color rendering. Default: False [S1, unofficial].
- `colour_quality`: Quality level ('tiny', 'low', 'medium', 'high', 'ultra', 'original'). Default: 'high' [S1, unofficial].
- `bounding_box_min/max`: Spatial bounds for inclusion. Default: Disabled [S1, unofficial].
- `no_calculate_normals`: Skip normal calculation. Default: False [S1, unofficial].
- `mahalanobis_distance_std`: Max Mahalanobis distance for sampling. Default: 2.0 [S1, unofficial].
- `min_opacity`: Minimum opacity threshold. Default: 0.0 [S1, unofficial].
- `cull_gaussian_sizes`: Percentage of largest Gaussians to remove. Default: 0.0 [S1, unofficial].
- `max_sh_degrees`: SH degree of input. Default: 3 [S1, unofficial].
- `num_sample_attempts`: Attempts per Gaussian (5 if `exact_num_points` is False, else 100) [S1, unofficial].
- `renderer_type`: 'cuda' or 'python'. Default: 'cuda' [S1, unofficial].

**Steps:**

1. **Load and Validate Gaussians:**
   Load Gaussian parameters ($\boldsymbol{\mu}_i, \boldsymbol{\Sigma}_i, \alpha_i, \mathbf{c}_i$) from the input file. Validate covariance matrices $\boldsymbol{\Sigma}_i$ to ensure they are positive semidefinite. If any are not, clamp them or remove the corresponding Gaussians [S1, unofficial].

2. **Filter Gaussians:**
   Apply culling criteria to the set of $N$ Gaussians:
   - **Opacity:** Remove Gaussians where $\alpha_i < \text{min\_opacity}$ [S1, unofficial].
   - **Bounding Box:** Remove Gaussians where $\boldsymbol{\mu}_i$ is outside $[\text{bounding\_box\_min}, \text{bounding\_box\_max}]$ [S1, unofficial].
   - **Size:** Remove the top $\text{cull\_gaussian\_sizes} \times 100\%$ of Gaussians ranked by size (approximated by surface area) [S1, unofficial].
   - **Visibility:** If `prioritise_visible_gaussians` is True, render the scene from all camera views (or a subset based on `camera_skip_rate`) to compute the total contribution of each Gaussian. Remove Gaussians with total contribution $< \text{visibility\_threshold}$ [S1, unofficial].
   - **Surface Distance:** If `surface_distance_std` is set, cull Gaussians that are further than this threshold from predicted surfaces [S1, unofficial].

3. **Calculate Point Allocation:**
   Determine the number of points to sample from each remaining Gaussian.
   - Compute the surface area $A_i$ of each Gaussian ellipsoid using its eigenvalues $\lambda_1, \lambda_2, \lambda_3$ from $\boldsymbol{\Sigma}_i$:
     $$ a = \sqrt{\lambda_1}, \quad b = \sqrt{\lambda_2}, \quad c = \sqrt{\lambda_3} $$
     $$ A_i \approx 4\pi \left( \frac{a^p b^p + a^p c^p + b^p c^p}{3} \right)^{1/p}, \quad p = 1.6075 $$
   - If `prioritise_visible_gaussians` is True, use the visibility contribution $v_i$ as the weight; otherwise, use opacity $\alpha_i$.
   - Calculate magnitude $m_i$:
     $$ m_i = \sqrt{A_i} \times v_i $$
   - Distribute the total `num_points` among Gaussians proportional to $m_i$. If `exact_num_points` is False, use a fixed number of sample attempts per Gaussian (5); otherwise, use 100 attempts [S1, unofficial].

4. **Sample Points:**
   For each Gaussian $i$:
   - Sample points $\mathbf{p}$ from the multivariate normal distribution $\mathcal{N}(\boldsymbol{\mu}_i, \boldsymbol{\Sigma}_i)$.
   - Reject samples where the Mahalanobis distance $d_M$ exceeds $\text{mahalanobis\_distance\_std}$:
     $$ d_M = \sqrt{(\mathbf{p} - \boldsymbol{\mu}_i)^\top \boldsymbol{\Sigma}_i^{-1} (\mathbf{p} - \boldsymbol{\mu}_i)} $$
   - Add valid points to the point cloud. If `exact_num_points` is False, stop sampling for this Gaussian once the allocated count is reached or attempts are exhausted [S1, unofficial].

5. **Render Colors (Optional):**
   If `render_colours` is True:
   - Initialize a Gaussian renderer with the loaded Gaussians and camera parameters.
   - For each camera (skipping based on `camera_skip_rate`), render the scene to obtain per-pixel colors and depths.
   - Map the generated points to the rendered images. Update the point colors with the rendered values, weighted by the Gaussian contributions [S1, unofficial].
   - Color resolution is determined by `colour_quality`: tiny=180, low=360, medium=720, high=1280, ultra=1920, original=None [S1, unofficial].

6. **Calculate Normals (Optional):**
   If `calculate_normals` is True:
   - Compute normals for each point. For points sampled directly from the Gaussian mean, use the normal derived from the Gaussian's covariance matrix (principal axis). For other points, estimate normals from local neighborhood or use the Gaussian's orientation [S1, unofficial].

7. **Post-Processing:**
   - If `clean_pointcloud` is True, remove outliers using Open3D's statistical outlier removal [S1, unofficial].
   - If `generate_mesh` is True:
     - Identify surface Gaussians using `surface_distance_std` or other heuristics.
     - Cull non-surface Gaussians.
     - Generate a separate point cloud with a fixed average of 25 points per surface Gaussian, capped at half the total requested points.
     - Perform Poisson surface reconstruction with `poisson_depth` and apply `laplacian_iterations` of smoothing [S1, unofficial].

8. **Save Output:**
   Save the final points, colors, and normals to a PLY file [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np
from scipy.spatial import transform

def load_gaussians(file_path):
    """Load 3DGS PLY file. Returns xyz, scales, rots, opacities, sh_coeffs."""
    # Implementation details omitted; assume standard PLY loader for 3DGS
    pass

def validate_covariances(xyz, scales, rots):
    """Ensure covariances are positive semidefinite."""
    # Construct covariance from scale and rotation
    # R = quat_to_rot_matrix(rots)
    # S = diag(scales)
    # Sigma = R @ S @ S.T @ R.T
    # Check eigenvalues. Clamp or remove invalid Gaussians.
    pass

def get_gaussian_magnitudes(gaussians, contributions=None):
    """Calculate magnitude for point allocation based on surface area and contribution."""
    # Compute eigenvalues of covariance
    eigvals = torch.linalg.eigvals(gaussians.covariances).real
    a = torch.sqrt(eigvals[:, 0])
    b = torch.sqrt(eigvals[:, 1])
    c = torch.sqrt(eigvals[:, 2])
    p = 1.6075
    radicand = (torch.pow(a*b, p) + torch.pow(a*c, p) + torch.pow(b*c, p)) / 3.0
    surface_area = 4.0 * torch.pi * torch.pow(radicand, 1.0/p)
    surface_area = torch.sqrt(surface_area)
    
    if contributions is None:
        contributions = gaussians.opacities
    
    magnitudes = surface_area * contributions
    return magnitudes

def sample_points_from_gaussians(gaussians, num_points_per_gaussian, mahalanobis_std):
    """Sample points from Gaussian distributions, rejecting those outside Mahalanobis distance."""
    points = []
    for i, g in enumerate(gaussians):
        mean = g.xyz
        cov = g.covariance
        num_samples = num_points_per_gaussian[i]
        
        # Sample from MultivariateNormal
        samples = torch.distributions.multivariate_normal.MultivariateNormal(mean, cov).sample((num_samples,))
        
        # Calculate Mahalanobis distance
        diff = samples - mean
        # Solve Sigma @ x = diff for x, then x.T @ diff
        # Or use cholesky decomposition
        L = torch.linalg.cholesky(cov)
        y = torch.linalg.solve(L, diff.T)
        mahalanobis_dist = torch.norm(y, dim=0)
        
        valid_mask = mahalanobis_dist <= mahalanobis_std
        valid_samples = samples[valid_mask]
        points.append(valid_samples)
        
        if len(points[-1]) == 0:
            # Fallback: add mean if no samples valid
            points[-1] = mean.unsqueeze(0)
            
    return torch.cat(points, dim=0)

def render_colors(points, gaussians, cameras, renderer_type, colour_resolution):
    """Render colors for points using Gaussian splatting."""
    # Initialize renderer
    # For each camera:
    #   Render image
    #   Project points to image plane
    #   Assign color from rendered image
    pass

def convert_3dgs_to_pointcloud(input_path, output_path, settings):
    # 1. Load
    gaussians = load_gaussians(input_path)
    
    # 2. Validate
    gaussians.validate_covariances()
    
    # 3. Filter
    if settings.min_opacity > 0:
        gaussians.apply_min_opacity(settings.min_opacity)
    if settings.bounding_box_min is not None:
        gaussians.apply_bounding_box(settings.bounding_box_min, settings.bounding_box_max)
    if settings.cull_gaussian_sizes > 0:
        gaussians.cull_large_gaussians(settings.cull_gaussian_sizes)
        
    contributions = None
    if settings.prioritise_visible_gaussians:
        # Render to get contributions
        contributions = get_visibility_contributions(gaussians, settings)
        gaussians.apply_visibility_threshold(settings.visibility_threshold, contributions)
        
    # 4. Allocate Points
    magnitudes = get_gaussian_magnitudes(gaussians, contributions)
    total_mag = magnitudes.sum()
    if total_mag == 0:
        raise Exception("No Gaussians remain after culling.")
        
    if settings.exact_num_points:
        num_attempts = 100
    else:
        num_attempts = 5
        
    # Distribute points proportionally to magnitude
    points_per_gaussian = (magnitudes / total_mag * settings.num_points).long()
    
    # 5. Sample
    points = sample_points_from_gaussians(gaussians, points_per_gaussian, settings.mahalanobis_distance_std)
    
    # 6. Colors
    if settings.render_colours:
        colors = render_colors(points, gaussians, settings.cameras, settings.renderer_type, settings.colour_resolution)
    else:
        colors = gaussians.get_initial_colors(points) # Fallback color
        
    # 7. Normals
    if settings.calculate_normals:
        normals = calculate_normals(points, gaussians)
    else:
        normals = None
        
    # 8. Save
    save_ply(output_path, points, colors, normals)
```

## Operating characteristics
- **Time Complexity:** Dominated by the visibility rendering step if `prioritise_visible_gaussians` is True, which is $O(N \cdot C \cdot R)$ where $N$ is Gaussians, $C$ is cameras, and $R$ is resolution. Sampling is $O(M)$ where $M$ is the number of generated points.
- **Memory Complexity:** $O(N + M)$ to store Gaussians and the resulting point cloud.
- **Performance:** Generating 10 million points typically takes several minutes on a modern GPU, depending on the number of cameras and resolution. Using `cuda` renderer is significantly faster than `python`.
- **Strengths:** Produces high-quality, dense point clouds that closely mimic the original 3DGS appearance. Flexible filtering options allow for noise reduction and scene simplification.
- **Limitations:** The point cloud is not watertight. Color accuracy depends on the number of cameras used for rendering; sparse views may result in inconsistent colors. `exact_num_points` is computationally expensive.
- **When to use:** When a standard point cloud format is required for downstream applications (e.g., meshing, visualization in non-Gaussian software).
- **When not to use:** When a watertight mesh is needed directly (use [SuGaR Mesh Extraction](12580-sugar-mesh-extraction.md) or other meshing methods) or when real-time rendering is required (use the original 3DGS representation).

## Background
3D Gaussian Splatting (3DGS) was introduced by Kerbl et al. in 2023 as a real-time radiance field representation [S1, unofficial]. While 3DGS offers high-quality rendering, it is not compatible with many standard 3D processing tools. The "3DGS to Point Cloud" conversion method, as implemented in the repository by Lewis-Stuart, addresses this interoperability gap by sampling points from the Gaussian distributions [S1, unofficial]. This approach leverages the fact that a Gaussian distribution can be approximated by a set of points, allowing the scene to be represented as a traditional point cloud.

## Sources
[S1] Lewis-Stuart-11/3DGS-to-PC, https://github.com/Lewis-Stuart-11/3DGS-to-PC, fetched 2026-10-27 (unofficial)
