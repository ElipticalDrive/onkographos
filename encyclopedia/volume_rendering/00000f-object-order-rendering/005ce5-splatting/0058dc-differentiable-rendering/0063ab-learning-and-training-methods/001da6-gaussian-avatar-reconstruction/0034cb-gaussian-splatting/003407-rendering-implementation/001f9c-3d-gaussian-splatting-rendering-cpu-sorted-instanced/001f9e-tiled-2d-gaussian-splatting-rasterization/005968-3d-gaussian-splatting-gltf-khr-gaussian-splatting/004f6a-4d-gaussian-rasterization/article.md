# 4D Gaussian Rasterization

Also called: 4D Gaussian Splatting Forward Pass, 4DGS Render

## Overview
4D Gaussian Rasterization renders a scene represented by 4D Gaussians (3D spatial + 1D temporal) into a 2D image for a specific viewpoint and timestamp [S1, unofficial]. It extends static 3D Gaussian Splatting by handling time-dependent opacity, position offsets, and spherical harmonics evaluation in 4D [S1, unofficial]. It differs from sibling methods like [SpacetimeGaussians](14897-4d-gaussian-splatting-dreamgaussian4d.md) by using a unified 4D Gaussian representation with explicit temporal spherical harmonics, whereas SpacetimeGaussians uses a rendering pipeline with multiple option flags for different rasterization backends [S3, unofficial].

## How it works
**Inputs:**
- `viewpoint_camera`: Object containing `FoVx`, `FoVy`, `image_height`, `image_width`, `world_view_transform` (view matrix), `full_proj_transform` (projection matrix), `camera_center`, and `timestamp`.
- `pc` (GaussianModel): Object containing attributes:
  - `get_xyz`: 3D means tensor.
  - `active_sh_degree`: Spatial SH degree.
  - `active_sh_degree_t`: Temporal SH degree.
  - `rot_4d`: Boolean flag for 4D rotation.
  - `gaussian_dim`: Integer dimension (3 or 4).
  - `force_sh_3d`: Boolean flag to force 3D SH evaluation.
  - `time_duration`: Tuple `(t_min, t_max)`.
  - `marginal_t`: Tensor of marginal time probabilities.
  - `env_map`: Environment map texture (optional).
  - `env_map_res`: Resolution of environment map (optional).
  - `scales_t`, `ts`, `rotations_r`: Time-specific attributes (if `gaussian_dim == 4`).
  - `prefilter_var`: Prefilter variance threshold (if > 0).
- `pipe`: Configuration object with flags:
  - `compute_cov3D_python`: Boolean.
  - `convert_SHs_python`: Boolean.
  - `env_map_res`: Integer or None.
- `bg_color`: Background color tensor (3 channels) on GPU.
- `scaling_modifier`: Float (default 1.0).

**Outputs:**
- Dictionary containing:
  - `rendered_image`: Tensor `(C, H, W)`.
  - `viewspace_points`: Tensor of 2D screen-space means with gradients.
  - `visibility_filter`: Boolean tensor.
  - `radii_all`: Tensor of radii.
  - `depth`: Tensor of depths.
  - `alpha`: Tensor of alpha values.
  - `flow`: Tensor of flow vectors.

**Data Structures:**
- `GaussianRasterizationSettings`: Struct holding image dimensions, FOV tangents, background color, scaling modifier, view/projection matrices, SH degrees, camera position, timestamp, time duration, 4D rotation flag, Gaussian dimension, and debug flags.
- `GaussianRasterizer`: Object initialized with settings, performing the CUDA rasterization.

**Steps:**

1.  **Initialize Gradients:**
    Create a zero tensor `screenspace_points` on CUDA with `requires_grad=True` to store gradients of 2D screen-space means [S1, unofficial].
    ```python
    screenspace_points = torch.zeros_like(pc.get_xyz, dtype=pc.get_xyz.dtype, requires_grad=True, device="cuda") + 0
    try:
        screenspace_points.retain_grad()
    except:
        pass
    ```

2.  **Compute FOV Tangents:**
    Calculate tangent of half field of view [S1, unofficial].
    ```python
    tanfovx = math.tan(viewpoint_camera.FoVx * 0.5)
    tanfovy = math.tan(viewpoint_camera.FoVy * 0.5)
    ```

3.  **Configure Rasterization Settings:**
    Construct `GaussianRasterizationSettings` with parameters [S1, unofficial]:
    - `image_height`, `image_width`: From `viewpoint_camera`.
    - `tanfovx`, `tanfovy`: From step 2.
    - `bg`: `bg_color` if `pipe.env_map_res` is not set, else `torch.zeros(3, device="cuda")`.
    - `scale_modifier`: `scaling_modifier`.
    - `viewmatrix`: `viewpoint_camera.world_view_transform`.
    - `projmatrix`: `viewpoint_camera.full_proj_transform`.
    - `sh_degree`: `pc.active_sh_degree`.
    - `sh_degree_t`: `pc.active_sh_degree_t`.
    - `campos`: `viewpoint_camera.camera_center`.
    - `timestamp`: `viewpoint_camera.timestamp`.
    - `time_duration`: `pc.time_duration[1] - pc.time_duration[0]`.
    - `rot_4d`: `pc.rot_4d`.
    - `gaussian_dim`: `pc.gaussian_dim`.
    - `force_sh_3d`: `pc.force_sh_3d`.
    - Additional flags as needed.

4.  **Initialize Rasterizer:**
    Create `GaussianRasterizer` with the configured settings [S1, unofficial].

5.  **Retrieve and Preprocess Attributes:**
    - Retrieve 3D means, opacity, and other attributes from `pc` [S1, unofficial].
    - If `pipe.compute_cov3D_python` is true:
        - Compute pre-computed 3D covariance and mean offset [S1, unofficial].
        - If `pc.rot_4d` is true, apply the mean offset to `means3D` [S1, unofficial].
        - If `pc.gaussian_dim == 4`, multiply opacity by marginal time probability `marginal_t` [S1, unofficial].
    - If `pipe.compute_cov3D_python` is false:
        - Retrieve raw scales, rotations, and time-specific scales/positions (`scales_t`, `ts`) [S1, unofficial].
        - If `pc.gaussian_dim == 4` and `pc.rot_4d` is true, retrieve `rotations_r` [S1, unofficial].
        - Retrieve `prefilter_var` if greater than 0 [S1, unofficial].

6.  **Compute Colors:**
    - If `pipe.convert_SHs_python` is true:
        - Extract SH features [S1, unofficial].
        - Compute direction vectors `dir_pp` from means (adjusted by delta_mean if cov3D is precomputed) to camera center [S1, unofficial].
        - Normalize direction [S1, unofficial].
        - If `pc.gaussian_dim == 3` or `force_sh_3d`, use `eval_sh` [S1, unofficial].
        - If `pc.gaussian_dim == 4`, compute temporal direction `dir_t` and use `eval_shfs_4d` with spatial direction, temporal direction, and time duration [S1, unofficial].
        - Clamp result `sh2rgb + 0.5` to minimum 0.0 [S1, unofficial].

7.  **Prefiltering (Optional):**
    - If `pipe.compute_cov3D_python` and `pc.gaussian_dim == 4`:
        - Create a boolean mask where `marginal_t[:,0] > 0.05` [S1, unofficial].
        - Filter all relevant tensors (`means2D`, `means3D`, `ts`, `shs`, `colors_precomp`, `opacity`, `scales`, `scales_t`, `rotations`, `rotations_r`, `cov3D_precomp`, `flow_2d`) using this mask [S1, unofficial].

8.  **Rasterize:**
    Call `rasterizer` with 3D means, 2D means, SHs, precomputed colors, flow, opacities, time scales/positions, rotations, covariance, and prefilter variance [S1, unofficial].
    Outputs: `rendered_image`, `radii`, `depth`, `alpha`, `flow`, `covs_com` [S1, unofficial].

9.  **Environment Map (Optional):**
    - If `pipe.env_map_res` is set:
        - Compute environment map background [S1, unofficial].
        - Calculate ray intersection with a sphere of radius R=60 using `rays_o` and `rays_d` [S1, unofficial].
        - Compute intersection parameter `t_inter` and intersection point `xyz_inter` [S1, unofficial].
        - Convert to spherical coordinates `tu` (theta) and `tv` (phi) normalized to [-1, 1] for texture sampling [S1, unofficial].
        - Sample `pc.env_map` using `F.grid_sample` and add to `rendered_image` weighted by `(1 - alpha)` [S1, unofficial].

10. **Construct Output Dictionary:**
    - Construct `radii_all` by zero-initializing a tensor of size `mask.shape` (if prefiltered) or using original `radii` [S1, unofficial].
    - If prefiltered, assign original `radii` to the positions indicated by `mask` [S1, unofficial].
    - Return render dictionary including `rendered_image`, `viewspace_points`, `visibility_filter` (`radii_all > 0`), `radii_all`, `depth`, `alpha`, and `flow` [S1, unofficial].

## Subtopics

### Dynamic Reconstruction with Deformation Field
An extension to dynamic MRI data modeling temporal variations of anatomical structures from time-resolved MRI sequences using a static set of 3D Gaussians and a learned deformation field network [S2, unofficial]. It differs from standard 4D Gaussian Splatting by using a canonical static set of 3D Gaussians and a spatiotemporal encoder + lightweight MLP to predict per-frame changes in position, covariance, color, and opacity [S2, unofficial]. It is chosen for medical imaging tasks where anatomical consistency is critical [S2, unofficial].

### SpacetimeGaussians Rendering Pipeline
A rendering pipeline that retrieves rasterizer settings via `getrenderpip(option)` with options like 'test_ours_full' (using `diff_gaussian_rasterization_ch9`), 'test_ours_lite' (using `forward_lite`), etc. [S3, unofficial]. It differs from the standard 4DGS rasterization by offering multiple backend configurations for testing and optimization [S3, unofficial]. It is chosen when flexible rasterization backends are needed for dynamic view synthesis [S3, unofficial].

### Efficient4D
A method that directly converts 3D Gaussians into 4D Gaussians and enhances temporal continuity by fusing spatial volumes of adjacent timestamps [S4, unofficial]. It differs from other 4D methods by focusing on synchronization to generate better cross-time multi-view images for supervision [S4, unofficial]. It is chosen when temporal continuity and cross-time supervision are prioritized [S4, unofficial].

## Operating characteristics
Not given in the sources.

## Background
4D Gaussian Rasterization is an extension of 3D Gaussian Splatting to handle dynamic scenes by adding a temporal dimension to the Gaussian representation [S1, unofficial]. It builds on [3D Gaussian Splatting Rendering (CPU-Sorted, Instanced)](52-3d-gaussian-splatting-rendering.md) by extending the static representation to 4D [S1, unofficial]. The method was introduced in the context of dynamic scene reconstruction and view synthesis, allowing for real-time rendering of time-varying scenes [S1, unofficial].

## Sources
[S1] LogosIntelligenceLab/4d-gaussian-splatting, https://github.com/LogosIntelligenceLab/4d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] InnerGS: Internal Scenes Reconstruction and Segmentation via Factorized 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2508.13287, fetched 2026-10-26 (unofficial)
[S3] oppo-us-research/SpacetimeGaussians, https://github.com/oppo-us-research/SpacetimeGaussians, fetched 2026-10-27 (unofficial)
[S4] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
