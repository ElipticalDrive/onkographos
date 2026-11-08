# ToF Scene Rendering Setup

Also called: ToF Gaussian Splatting Setup, ToF render

## Overview
This method configures the rendering pipeline for Time-of-Flight (ToF) cameras within a 3D Gaussian Splatting framework, enabling the simultaneous optimization of color and ToF data (phasors, depth, amplitude). It differs from standard RGB-only splatting by maintaining separate rasterization settings for color and ToF views, handling multi-modal Gaussian attributes (such as phasor features), and supporting dynamic scene decomposition via motion masks.

## How it works
The setup initializes CUDA tensors for gradient tracking, computes camera projection parameters for both RGB and ToF sensors, and prepares Gaussian attributes for static and dynamic regions before invoking the respective rasterizers.

### Inputs and Outputs
**Inputs:**
- `pc`: GaussianModel instance containing attributes `xyz`, `opacity`, `scaling`, `rotation`, `features_color`, `features_phasor`, `use_view_dependent_phase`.
- `viewpoint_camera`: Object containing `FoVx`, `FoVy`, `FoVx_tof`, `FoVy_tof`, `world_view_transform`, `full_proj_transform`, `camera_center`, `tof_image_height`, `tof_image_width`, `depth_range`.
- `opt`: Optimization configuration containing `optimize_phase_offset`, `optimize_dc_offset`.
- `render_regions`: List of strings indicating which parts of the scene to render (e.g., `['static', 'dynamic']`).
- `motion_mask`: Boolean tensor identifying dynamic Gaussians.

**Outputs:**
- `color_image`: Rendered RGB image.
- `tof_data`: Rendered ToF data including phasor, depth, pixel indices, and distribution.
- `screenspace_points`: Tensor storing gradients of 2D means.

### Data Structures
1.  **GaussianRasterizationSettings**:
    -   Fields: `image_height`, `image_width`, `tanfovx`, `tanfovy`, `bg`, `scale_modifier`, `viewmatrix`, `projmatrix`, `sh_degree`, `campos`, `prefiltered`, `debug`, `near_n`, `far_n`.
    -   ToF-specific fields: `depth_range`, `use_view_dependent_phase`, `optimize_phase_offset`, `optimize_dc_offset`.
2.  **Gaussian Attributes Tensors** (CUDA, `requires_grad=True`):
    -   `means3D`: Shape $(N, 3)$.
    -   `means2D`: Shape $(N, 2)$.
    -   `opacity`: Shape $(N, 1)$.
    -   `scales`: Shape $(N, 3)$.
    -   `rotations`: Shape $(N, 4)$ (quaternions).
    -   `shs`: Shape $(N, K, 3)$ (Spherical Harmonics coefficients for color).
    -   `shs_p`: Shape $(N, K, P)$ (Spherical Harmonics coefficients for phasor/ToF).
    -   `screenspace_points`: Shape $(N, 2)$, initialized to zero, with `retain_grad()` called.

### Steps
1.  **Initialize Gradient Tensors**:
    Create `screenspace_points` on CUDA with `requires_grad=True` and call `retain_grad()` to preserve gradients for 2D mean updates [S1, unofficial].

2.  **Compute Field of View Tangents**:
    Calculate $\tan(\text{FoV}_x)$ and $\tan(\text{FoV}_y)$ for the RGB camera:
    $$
    \text{tanfovx} = \tan(\text{viewpoint\_camera.FoVx})
    $$
    $$
    \text{tanfovy} = \tan(\text{viewpoint\_camera.FoVy})
    $$
    [S1, unofficial]

3.  **Configure RGB Rasterization Settings**:
    Create `GaussianRasterizationSettings` for color:
    -   `image_height`, `image_width`: From `viewpoint_camera`.
    -   `tanfovx`, `tanfovy`: From Step 2.
    -   `bg`: Background color tensor.
    -   `scale_modifier`: Scaling factor.
    -   `viewmatrix`: `viewpoint_camera.world_view_transform`.
    -   `projmatrix`: `viewpoint_camera.full_proj_transform`.
    -   `sh_degree`: Spherical Harmonics degree.
    -   `campos`: `viewpoint_camera.camera_center`.
    -   `prefiltered`: `False`.
    -   `debug`: `False`.
    -   `near_n`, `far_n`: `znear`, `zfar` from `viewpoint_camera`.
    [S1, unofficial]

4.  **Configure ToF Rasterization Settings** (if `viewpoint_camera.FoVx_tof` is not None):
    Create `GaussianRasterizationSettings` for ToF:
    -   `image_height`: `viewpoint_camera.tof_image_height`.
    -   `image_width`: `viewpoint_camera.tof_image_width`.
    -   `tanfovx`: $\tan(\text{viewpoint\_camera.FoVx\_tof})$.
    -   `tanfovy`: $\tan(\text{viewpoint\_camera.FoVy\_tof})$.
    -   `bg`, `scale_modifier`, `prefiltered`, `debug`, `near_n`, `far_n`: Same as RGB settings.
    -   `viewmatrix`: `viewpoint_camera.world_view_transform_tof`.
    -   `projmatrix`: `viewpoint_camera.full_proj_transform_tof`.
    -   `campos`: `viewpoint_camera.camera_center_tof`.
    -   `depth_range`: `viewpoint_camera.depth_range.item()`.
    -   `use_view_dependent_phase`: `pc.use_view_dependent_phase`.
    -   `optimize_phase_offset`: `opt.optimize_phase_offset`.
    -   `optimize_dc_offset`: `opt.optimize_dc_offset`.
    [S1, unofficial]

5.  **Initialize Gaussian Attribute Tensors**:
    Create zero tensors on CUDA for `means3D`, `means2D`, `opacity`, `scales`, `rotations`, `shs`, `shs_p` with shapes matching `pc`'s attributes [S1, unofficial].

6.  **Populate Static Attributes**:
    If `'static'` is in `render_regions`:
    -   Copy `pc.xyz`, `pc.opacity`, `pc.scaling`, `pc.rotation`, `pc.features_color`, `pc.features_phasor` to the zero tensors.
    -   Exclude Gaussians marked by `motion_mask` [S1, unofficial].

7.  **Populate Dynamic Attributes**:
    If `'dynamic'` is in `render_regions`:
    -   Copy base attributes from `pc`.
    -   Apply differentials:
        -   `means3D` += `d_xyz`.
        -   `rotations`: Apply `rotation_activation` to `(_rotation + d_rot)`.
        -   `shs` += `d_sh`.
        -   `shs_p` += `d_sh_p`.
    [S1, unofficial]

8.  **Render Color Image**:
    Call `rasterizer_color` with:
    -   `means3D`, `means2D`, `shs`, `shs_p`, `opacity`, `scales`, `rotations`.
    -   `cov3D_precomp`: `None`.
    [S1, unofficial]

9.  **Render ToF Data**:
    Call `rasterizer_tof` with the same Gaussian attributes (`means3D`, `means2D`, `shs`, `shs_p`, `opacity`, `scales`, `rotations`) to produce phasor, depth, pixel, and distribution outputs [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def setup_tof_rendering(pc, viewpoint_camera, opt, render_regions, motion_mask):
    # Step 1: Initialize gradient tensors
    screenspace_points = torch.zeros(pc.get_xyz.shape[0], 2, device='cuda', requires_grad=True)
    screenspace_points.retain_grad()

    # Step 2: Compute FoV tangents for RGB
    tanfovx = math.tan(viewpoint_camera.FoVx)
    tanfovy = math.tan(viewpoint_camera.FoVy)

    # Step 3: Configure RGB Rasterization Settings
    rgb_settings = GaussianRasterizationSettings(
        image_height=viewpoint_camera.image_height,
        image_width=viewpoint_camera.image_width,
        tanfovx=tanfovx,
        tanfovy=tanfovy,
        bg=viewpoint_camera.bg,
        scale_modifier=viewpoint_camera.scale_modifier,
        viewmatrix=viewpoint_camera.world_view_transform,
        projmatrix=viewpoint_camera.full_proj_transform,
        sh_degree=pc.active_sh_degree,
        campos=viewpoint_camera.camera_center,
        prefiltered=False,
        debug=False,
        near_n=viewpoint_camera.znear,
        far_n=viewpoint_camera.zfar
    )

    # Step 4: Configure ToF Rasterization Settings
    tof_settings = None
    if viewpoint_camera.FoVx_tof is not None:
        tof_settings = GaussianRasterizationSettings(
            image_height=viewpoint_camera.tof_image_height,
            image_width=viewpoint_camera.tof_image_width,
            tanfovx=math.tan(viewpoint_camera.FoVx_tof),
            tanfovy=math.tan(viewpoint_camera.FoVy_tof),
            bg=viewpoint_camera.bg,
            scale_modifier=viewpoint_camera.scale_modifier,
            viewmatrix=viewpoint_camera.world_view_transform_tof,
            projmatrix=viewpoint_camera.full_proj_transform_tof,
            sh_degree=pc.active_sh_degree,
            campos=viewpoint_camera.camera_center_tof,
            prefiltered=False,
            debug=False,
            near_n=viewpoint_camera.znear,
            far_n=viewpoint_camera.zfar,
            depth_range=viewpoint_camera.depth_range.item(),
            use_view_dependent_phase=pc.use_view_dependent_phase,
            optimize_phase_offset=opt.optimize_phase_offset,
            optimize_dc_offset=opt.optimize_dc_offset
        )

    # Step 5: Initialize Gaussian Attribute Tensors
    N = pc.get_xyz.shape[0]
    means3D = torch.zeros(N, 3, device='cuda', requires_grad=True)
    means2D = torch.zeros(N, 2, device='cuda', requires_grad=True)
    opacity = torch.zeros(N, 1, device='cuda', requires_grad=True)
    scales = torch.zeros(N, 3, device='cuda', requires_grad=True)
    rotations = torch.zeros(N, 4, device='cuda', requires_grad=True)
    shs = torch.zeros(N, pc.active_sh_degree + 1, 3, device='cuda', requires_grad=True)
    shs_p = torch.zeros(N, pc.active_sh_degree + 1, pc.phasor_dim, device='cuda', requires_grad=True)

    # Step 6: Populate Static Attributes
    if 'static' in render_regions:
        static_mask = ~motion_mask
        means3D[static_mask] = pc.get_xyz[static_mask]
        opacity[static_mask] = pc.get_opacity[static_mask]
        scales[static_mask] = pc.get_scaling[static_mask]
        rotations[static_mask] = pc.get_rotation[static_mask]
        shs[static_mask] = pc.get_features_color[static_mask]
        shs_p[static_mask] = pc.get_features_phasor[static_mask]

    # Step 7: Populate Dynamic Attributes
    if 'dynamic' in render_regions:
        dynamic_mask = motion_mask
        # Base copy
        means3D[dynamic_mask] = pc.get_xyz[dynamic_mask]
        opacity[dynamic_mask] = pc.get_opacity[dynamic_mask]
        scales[dynamic_mask] = pc.get_scaling[dynamic_mask]
        rotations[dynamic_mask] = pc.get_rotation[dynamic_mask]
        shs[dynamic_mask] = pc.get_features_color[dynamic_mask]
        shs_p[dynamic_mask] = pc.get_features_phasor[dynamic_mask]
        
        # Apply differentials
        means3D[dynamic_mask] += pc.d_xyz[dynamic_mask]
        # Assuming rotation_activation normalizes quaternions
        rotations[dynamic_mask] = rotation_activation(pc.get_rotation[dynamic_mask] + pc.d_rot[dynamic_mask])
        shs[dynamic_mask] += pc.d_sh[dynamic_mask]
        shs_p[dynamic_mask] += pc.d_sh_p[dynamic_mask]

    # Step 8: Render Color Image
    rasterizer_color = GaussianRasterizer(rgb_settings)
    color_image = rasterizer_color(
        means3D=means3D,
        means2D=means2D,
        shs=shs,
        shs_p=shs_p,
        opacity=opacity,
        scales=scales,
        rotations=rotations,
        cov3D_precomp=None
    )

    # Step 9: Render ToF Data
    rasterizer_tof = GaussianRasterizer(tof_settings)
    tof_data = rasterizer_tof(
        means3D=means3D,
        means2D=means2D,
        shs=shs,
        shs_p=shs_p,
        opacity=opacity,
        scales=scales,
        rotations=rotations,
        cov3D_precomp=None
    )

    return color_image, tof_data, screenspace_points
```

### Parameters and Edge Cases
-   **Parameters**: `render_regions` controls which Gaussians are active. `motion_mask` distinguishes static from dynamic Gaussians. `depth_range` limits ToF raycasting.
-   **Edge Cases**: If `viewpoint_camera.FoVx_tof` is `None`, ToF settings are skipped. If `render_regions` is empty, no attributes are copied, resulting in empty renders. Precision issues may arise in `rotation_activation` if quaternions are not normalized after adding differentials [G].

## Operating characteristics
-   **Time Complexity**: Dominated by rasterization steps. ToF rendering adds overhead due to additional phasor calculations and potentially larger attribute tensors (`shs_p`).
-   **Memory Complexity**: Increased by storing `shs_p` (phasor SH coefficients) and separate gradient tensors for ToF-specific parameters.
-   **Strengths**: Enables joint optimization of color and depth/phase data, improving scene reconstruction accuracy in reflective or transparent surfaces where RGB alone fails.
-   **Limitations**: Requires careful handling of ToF camera calibration and noise models. Dynamic scene support adds complexity to attribute management.

## Background
This method is part of the Gaussian Splatting framework extended for Time-of-Flight cameras, as implemented in the `gftorf` project. It builds on [Non-Pinhole Camera Training](6088-non-pinhole-camera-training.md) by adding specific support for ToF sensor characteristics.

## Sources
[S1] brownvc/gftorf, https://github.com/brownvc/gftorf, fetched 2026-10-27 (unofficial)
