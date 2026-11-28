# 3D Gaussian Splatting to Point Cloud

Also called: 3DGS to Point Cloud Conversion, 3DGS-to-PC, Gaussian Splatting Point Cloud Generation, Gaussian Splatting to Dense Point Clouds

## Overview
This method converts a trained 3D Gaussian Splatting scene into a dense point cloud that imitates the original scene, with optional mesh generation and color rendering [S1, unofficial]. It distinguishes itself from sibling methods like [SuGaR Mesh Extraction](12580-sugar-mesh-extraction.md) by generating points directly from the Gaussian probability distributions and optionally using rendered camera views to weight contributions and filter surface proximity, rather than optimizing an explicit surface representation [S1, unofficial].

## How it works
**Inputs:**
- A 3DGS scene file (PLY or splat) containing $N$ Gaussians with parameters (xyz, scales, rots, colours, opacities, SH coefficients) [S1, unofficial].
- Camera parameters (transforms, intrinsics) if color rendering is enabled [S1, unofficial].
- Configuration settings (see Parameters) [S1, unofficial].

**Outputs:**
- A point cloud file (PLY) containing points, colours, and optionally normals [S1, unofficial].
- Optionally, a mesh file (PLY) generated from the point cloud [S1, unofficial].

**Data Structures:**
- `GaussPointCloudSettings`: NamedTuple containing `renderer_type` (str), `num_points` (int), `prioritise_visible_gaussians` (bool), `mahalanobis_distance_std` (float), `camera_skip_rate` (int), `render_colours` (bool), `min_opacity` (float), `bounding_box_min` (list), `bounding_box_max` (list), `calculate_normals` (bool), `cull_large_percentage` (float), `remove_unrendered_gaussians` (bool), `colour_resolution` (int), `max_sh_degree` (int), `exact_num_points` (bool), `visibility_threshold` (float), `surface_distance_std` (float), `generate_mesh` (bool), `quiet` (bool), `device` (str) [S1, unofficial].
- `PointCloudData`: NamedTuple containing `points` (torch.Tensor), `colours` (torch.Tensor), `normals` (torch.Tensor) [S1, unofficial].

**Parameters:**
- `num_points`: Total number of points to generate for the point cloud. Default: 10,000,000 [S1, unofficial].
- `exact_num_points`: Boolean flag to force the number of generated points to more closely match `num_points` (slower). Default: False [S1, unofficial].
- `no_prioritise_visible_gaussians`: Boolean flag to disable giving more points to Gaussians that contribute most to the scene. Default: False [S1, unofficial].
- `visibility_threshold`: Minimum contribution each Gaussian must have to be included in the final point cloud generation. Default: 0.05 [S1, unofficial].
- `surface_distance_std`: Cull Gaussians that are a minimum of X standard deviations away from the predicted scene surfaces. Default: Disabled [S1, unofficial].
- `clean_pointcloud`: Boolean flag to remove outliers on the point cloud after generation using Open3D. Default: False [S1, unofficial].
- `generate_mesh`: Boolean flag to also generate a mesh based on the created point cloud. Default: False [S1, unofficial].
- `poisson_depth`: Depth used in the Poisson surface reconstruction algorithm for meshing. Default: 10 [S1, unofficial].
- `laplacian_iterations`: Number of iterations to perform Laplacian mesh smoothing. Default: 10 [S1, unofficial].
- `camera_skip_rate`: Number of cameras to skip for each rendered image to reduce compute time. Default: 0 [S1, unofficial].
- `no_render_colours`: Boolean flag to skip rendering colours for faster processing. Default: False [S1, unofficial].
- `colour_quality`: Quality of colours when generating the point cloud. Options: tiny, low, medium, high, ultra, original. Default: high [S1, unofficial].
- `bounding_box_min`: Minimum position values of Gaussians to include in generating the new point cloud. Default: Disabled [S1, unofficial].
- `bounding_box_max`: Maximum position values of Gaussians to include in generating the new point cloud. Default: Disabled [S1, unofficial].
- `no_calculate_normals`: Boolean flag to set to not calculate and save normals for the points. Default: False [S1, unofficial].
- `mahalanobis_distance_std`: Maximum Mahalanobis distance each point can be from the centre of their Gaussian. Default: 2.0 [S1, unofficial].
- `min_opacity`: Minimum opacity for Gaussians that will be included. Must be between 0-1. Default: 0.0 [S1, unofficial].
- `cull_gaussian_sizes`: Percentage of Gaussians to remove from largest to smallest. Must be between 0-1. Default: 0.0 [S1, unofficial].
- `max_sh_degrees`: Number of spherical harmonics of the loaded point cloud. Default: 3 [S1, unofficial].
- `num_sample_attempts`: Number of attempts to sample points per Gaussian. Set to 5 if `exact_num_points` is false, else 100 [S1, unofficial].
- `renderer_type`: Type of renderer to use ('cuda' or 'python'). Default: 'cuda' [S1, unofficial].

**Steps:**
1. Load Gaussian parameters (xyz, scales, rots, colours, opacities, SH coefficients) from the input file (PLY or splat) [S1, unofficial].
2. Validate covariance matrices to ensure they are positive semidefinite, removing invalid Gaussians [S1, unofficial].
3. Filter Gaussians by `min_opacity`, `visibility_threshold`, bounding box constraints, and `cull_gaussian_sizes` percentage before sampling [S1, unofficial].
4. If rendering colours is enabled, initialize a Gaussian renderer with the loaded Gaussians and camera parameters [S1, unofficial]. Iterate through all provided camera transforms (skipping based on `camera_skip_rate`). For each camera, render the scene to obtain per-pixel contributions and depth maps [S1, unofficial]. Retrieve the total contribution of each Gaussian across all rendered images to prioritize visible Gaussians [S1, unofficial]. Update Gaussian colours with the rendered colours if rendering was performed [S1, unofficial].
5. Calculate the number of points to sample from each remaining Gaussian. Compute the surface area of each Gaussian ellipsoid from the eigenvalues of its covariance matrix. Calculate magnitude $m_i = \sqrt{4\pi \left(\frac{a^p b^p + a^p c^p + b^p c^p}{3}\right)^{1/p}} \times v_i$, where $a,b,c$ are the square roots of the eigenvalues, $p=1.6075$, and $v_i$ is the visibility contribution if `prioritise_visible_gaussians` is True, otherwise opacity $\alpha_i$ [S1, unofficial]. Distribute the total `num_points` among Gaussians proportional to $m_i$ [S1, unofficial]. Set `num_sample_attempts` to 5 if `exact_num_points` is False, else 100 [S1, unofficial].
6. Generate points for the entire scene by iterating over remaining Gaussians. For each Gaussian, add its mean as a point, then sample additional points from the Gaussian distribution $\mathcal{N}(\boldsymbol{\mu}_i, \boldsymbol{\Sigma}_i)$, rejecting samples where the Mahalanobis distance from the Gaussian center exceeds `mahalanobis_distance_std` [S1, unofficial].
7. If generating a mesh, identify surface Gaussians using a predicted surface standard deviation threshold (`surface_distance_std`) [S1, unofficial]. Cull non-surface Gaussians and generate a separate point cloud with a fixed average of 25 points per surface Gaussian, capped at half the total requested points [S1, unofficial].
8. If `clean_pointcloud` is enabled, remove outliers on the point cloud after generation using Open3D [S1, unofficial].
9. Save the final point cloud (and mesh if requested) to a PLY file [S1, unofficial].

**Pitfalls:**
- If no Gaussians remain after culling, the process raises an exception because a point cloud cannot be generated [S1, unofficial].
- If `transform_path` is not provided but rendering is enabled, an exception is raised because camera poses are required [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def load_gaussians(file_path):
    # Standard 3DGS PLY loader returning tensors: xyz, scales, rots, opacities, sh_coeffs
    pass # Implementation depends on file format; assume standard loader [G]

def validate_covariances(xyz, scales, rots, epsilon=1e-6):
    # Construct covariance matrices from scale and rotation
    # R = quat_to_rot_matrix(rots)
    # S = diag(scales)
    # Sigma = R @ S @ S.T @ R.T
    # Check eigenvalues. Clamp or remove invalid Gaussians.
    pass # Standard covariance validation [G]

def get_gaussian_magnitudes(gaussians, contributions=None):
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
    magnitudes = (surface_area * contributions).to(torch.float64)
    return magnitudes

def sample_points_from_gaussians(gaussians, num_points_per_gaussian, mahalanobis_std):
    all_points = []
    for i, g in enumerate(gaussians):
        mean = g.xyz
        cov = g.covariance
        num_samples = num_points_per_gaussian[i]
        samples = torch.distributions.multivariate_normal.MultivariateNormal(mean, cov).sample((num_samples,))
        diff = samples - mean
        L = torch.linalg.cholesky(cov)
        y = torch.linalg.solve(L, diff.T)
        mahalanobis_dist = torch.norm(y, dim=0)
        valid_mask = mahalanobis_dist <= mahalanobis_std
        valid_samples = samples[valid_mask]
        if len(valid_samples) == 0:
            valid_samples = mean.unsqueeze(0)
        all_points.append(valid_samples)
    return torch.cat(all_points, dim=0)

def render_colors(points, gaussians, cameras, renderer_type, colour_resolution):
    # Initialize renderer based on renderer_type
    # For each camera: render scene, project points, assign colors
    pass # Standard Gaussian rasterization [G]

def calculate_normals(points, gaussians):
    # Compute normals from Gaussian covariance principal axes or local neighborhood
    pass # Standard normal estimation [G]

def save_ply(output_path, points, colours, normals):
    # Write points, colours, normals to PLY file
    pass # Standard PLY writer [G]

def convert_3dgs_to_pointcloud(input_path, output_path, settings):
    gaussians = load_gaussians(input_path)
    gaussians.validate_covariances()
    if settings.min_opacity > 0:
        gaussians.apply_min_opacity(settings.min_opacity)
    if settings.bounding_box_min is not None:
        gaussians.apply_bounding_box(settings.bounding_box_min, settings.bounding_box_max)
    if settings.cull_gaussian_sizes > 0:
        gaussians.cull_large_gaussians(settings.cull_gaussian_sizes)
    contributions = None
    if settings.prioritise_visible_gaussians:
        contributions = get_visibility_contributions(gaussians, settings)
        gaussians.apply_visibility_threshold(settings.visibility_threshold, contributions)
    magnitudes = get_gaussian_magnitudes(gaussians, contributions)
    total_mag = magnitudes.sum()
    if total_mag == 0:
        raise Exception("No Gaussians remain after culling.")
    num_attempts = 100 if settings.exact_num_points else 5
    points_per_gaussian = (magnitudes / total_mag * settings.num_points).long()
    points = sample_points_from_gaussians(gaussians, points_per_gaussian, settings.mahalanobis_distance_std)
    if settings.render_colours:
        colors = render_colors(points, gaussians, settings.cameras, settings.renderer_type, settings.colour_resolution)
    else:
        colors = gaussians.get_initial_colors(points)
    if settings.calculate_normals:
        normals = calculate_normals(points, gaussians)
    else:
        normals = None
    save_ply(output_path, points, colors, normals)
```

## Operating characteristics
- **Time Complexity:** Dominated by the visibility rendering step if `prioritise_visible_gaussians` is True, which is $O(N \cdot C \cdot R)$ where $N$ is Gaussians, $C$ is cameras, and $R$ is resolution [S1, unofficial]. Sampling is $O(M)$ where $M$ is the number of generated points [G].
- **Memory Complexity:** $O(N + M)$ to store Gaussians and the resulting point cloud [G].
- **Performance:** Generating 10 million points typically takes several minutes on a modern GPU, depending on the number of cameras and resolution [S1, unofficial]. Using the `cuda` renderer is significantly faster than `python` [S1, unofficial].
- **Strengths:** Produces high-quality, dense point clouds that closely mimic the original 3DGS appearance [S1, unofficial]. Flexible filtering options allow for noise reduction and scene simplification [S1, unofficial].
- **Limitations:** The point cloud is not watertight [G]. Color accuracy depends on the number of cameras used for rendering; sparse views may result in inconsistent colors [S1, unofficial]. `exact_num_points` is computationally expensive [S1, unofficial].
- **When to use:** When a standard point cloud format is required for downstream applications (e.g., meshing, visualization in non-Gaussian software) [S1, unofficial].
- **When not to use:** When a watertight mesh is needed directly (use [SuGaR Mesh Extraction](12580-sugar-mesh-extraction.md) or other meshing methods) or when real-time rendering is required (use the original 3DGS representation) [G].

## Background
The method originates from the repository by Lewis-Stuart, which provides scripts for converting 3D Gaussian Splatting scenes into dense point clouds [S1, unofficial]. It was presented at the ICCV 2025 Workshop on 3D Vision and Spatial AI (3D-VAST) [S1, unofficial]. The approach addresses the interoperability gap between Gaussian splatting and standard 3D processing tools by sampling points from the Gaussian distributions, allowing the scene to be represented as a traditional point cloud [S1, unofficial].

## Sources
[S1] Lewis-Stuart-11/3DGS-to-PC, https://github.com/Lewis-Stuart-11/3DGS-to-PC, fetched 2026-10-27 (unofficial)
