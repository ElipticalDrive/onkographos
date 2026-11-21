# Vanilla 2D Gaussian Splatting Rendering

Also called: 2DGS Renderer, Vanilla2DGSRenderer

## Overview
Renders a scene represented by 2D Gaussian primitives into RGB, alpha, normal, and depth maps using a custom rasterizer [S1, unofficial]. It differs from standard 3D Gaussian Splatting by operating on surfels (2D Gaussians) projected into screen space, accumulating color and geometric attributes via front-to-back alpha compositing [Parent].

## How it works
**Inputs:**
- `means3D`: 3D mean positions of Gaussians.
- `means2D`: 2D screen-space mean positions (pre-computed or optimized).
- `shs` or `colors_precomp`: Spherical Harmonics coefficients or pre-computed colors.
- `opacities`: Per-primitive opacity values.
- `scales`: Per-primitive scaling factors.
- `rotations`: Per-primitive rotations (quaternions).
- `cov3D_precomp`: Pre-computed 3D covariance (optional; if `None`, computed from scales/rotation).
- Camera parameters: Intrinsics (FOV, image dimensions), extrinsics (view/projection matrices).
- `background_color`: Background color for alpha compositing.
- `scaling_modifier`: Scaling factor for Gaussian sizes.
- `depth_ratio`: Weighting factor for depth estimation (1 for bounded/median, 0 for unbounded/expected).

**Outputs:**
- `render_image`: RGB image.
- `render_alpha`: Alpha channel.
- `rend_normal`: View-space normals.
- `view_normal`: World-space normals.
- `rend_dist`: Expected depth.
- `surf_depth`: Surface depth (weighted combination).
- `surf_normal`: Surface normals derived from surface depth.
- `radii`: Visibility filter (pixels covered by Gaussians).
- `allmap`: Intermediate tensor containing depth, alpha, normals, and distortion.

**Data Structures:**
- `allmap`: Tensor with layout:
  - Index 0: Expected depth.
  - Index 1: Alpha.
  - Indices 2-4: View-space normals (x, y, z components).
  - Index 5: Median depth.
  - Index 6: Depth distortion.

**Steps:**
1. Initialize `means2D` with `requires_grad=True` to enable gradient capture for 2D means [S1, unofficial].
2. Compute `tan_fov_x` and `tan_fov_y` from horizontal and vertical fields of view [S1, unofficial].
3. Configure `GaussianRasterizationSettings` with image dimensions, FOV tangents, background color, scaling modifier, view/projection matrices, SH degree, camera position, and `prefiltered=False` [S1, unofficial].
4. Instantiate `GaussianRasterizer` with the settings [S1, unofficial].
5. Set `cov3D_precomp` to `None` to allow the rasterizer to compute covariance from scales and rotation [S1, unofficial].
6. Call the rasterizer with `means3D`, `means2D`, `shs`/`colors_precomp`, `opacities`, `scales`, `rotations`, and `cov3D_precomp` to obtain `render_image`, `radii`, and `allmap` [S1, unofficial].
7. Extract `render_alpha` from `allmap` index 1 [S1, unofficial].
8. Extract view-space normals from `allmap` indices 2-4 and transform to world space by multiplying by the transpose of the camera's world-to-camera rotation matrix [S1, unofficial].
9. Extract median depth from `allmap` index 5 and replace NaNs with 0 [S1, unofficial].
10. Extract expected depth from `allmap` index 0, divide by `render_alpha`, and replace NaNs with 0 [S1, unofficial].
11. Compute `surf_depth` as a weighted sum: `surf_depth = depth_ratio * median_depth + (1 - depth_ratio) * expected_depth` [S1, unofficial].
12. Compute `surf_normal` from `surf_depth` using `depth_to_normal`, then multiply by detached `render_alpha` [S1, unofficial].
13. Return dictionary containing all outputs [S1, unofficial].

**Helper Functions:**
- `depth_to_normal(depth_map)`:
  - Reconstruct 3D points from depth map using camera intrinsics and extrinsics.
  - Compute spatial derivatives (dx, dy) of the 3D points.
  - Compute normal as cross product of derivatives: `normal = normalize(cross(dx, dy))` [S1, unofficial].
- `depths_to_points(depth_map)`:
  - Compute NDC to pixel projection matrix.
  - Create meshgrid of pixel coordinates.
  - Compute ray directions.
  - Scale ray directions by depth to get 3D points [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def vanilla_2dgs_render(
    means3D, means2D, shs, opacities, scales, rotations,
    camera, background_color, scaling_modifier, depth_ratio, device
):
    # Step 1: Initialize means2D for gradients [G]
    if not means2D.requires_grad:
        means2D = means2D.clone().detach().requires_grad_(True)

    # Step 2: Compute FOV tangents [S1, unofficial]
    tan_fov_x = torch.tan(camera.fov_x / 2)
    tan_fov_y = torch.tan(camera.fov_y / 2)

    # Step 3: Configure rasterization settings [S1, unofficial]
    raster_settings = GaussianRasterizationSettings(
        image_height=camera.height,
        image_width=camera.width,
        tanfovx=tan_fov_x,
        tanfovy=tan_fov_y,
        bg=background_color,
        scale_modifier=scaling_modifier,
        viewmatrix=camera.view_matrix,
        projmatrix=camera.proj_matrix,
        sh_degree=shs.shape[-1] if shs is not None else 0,
        campos=camera.position,
        prefiltered=False
    )

    # Step 4: Instantiate rasterizer [S1, unofficial]
    rasterizer = GaussianRasterizer(raster_settings=raster_settings)

    # Step 5: Set cov3D_precomp to None [S1, unofficial]
    cov3D_precomp = None

    # Step 6: Rasterize [S1, unofficial]
    render_image, radii, allmap = rasterizer(
        means3D=means3D,
        means2D=means2D,
        shs=shs,
        colors_precomp=None,
        opacities=opacities,
        scales=scales,
        rotations=rotations,
        cov3D_precomp=cov3D_precomp
    )

    # Step 7: Extract alpha [S1, unofficial]
    render_alpha = allmap[:, 1, :, :]

    # Step 8: Extract and transform normals [S1, unofficial]
    view_normal = allmap[:, 2:5, :, :]
    # Transform to world space
    cam_rot_inv = camera.world_to_cam_rotation_matrix.t()
    view_normal = torch.matmul(cam_rot_inv, view_normal)

    # Step 9: Extract median depth [S1, unofficial]
    median_depth = allmap[:, 5, :, :]
    median_depth = torch.nan_to_num(median_depth, nan=0.0)

    # Step 10: Extract and compute expected depth [S1, unofficial]
    expected_depth = allmap[:, 0, :, :]
    expected_depth = torch.where(render_alpha > 0, expected_depth / render_alpha, 0.0)

    # Step 11: Compute surface depth [S1, unofficial]
    surf_depth = depth_ratio * median_depth + (1 - depth_ratio) * expected_depth

    # Step 12: Compute surface normal [S1, unofficial]
    surf_normal = depth_to_normal(surf_depth, camera)
    surf_normal = surf_normal * render_alpha.detach()

    # Step 13: Return results [S1, unofficial]
    return {
        "render_image": render_image,
        "viewspace_points": means2D,
        "visibility_filter": radii > 0,
        "radii": radii,
        "rend_alpha": render_alpha,
        "rend_normal": view_normal,
        "view_normal": view_normal,
        "rend_dist": expected_depth,
        "surf_depth": surf_depth,
        "surf_normal": surf_normal
    }

def depth_to_normal(depth_map, camera):
    # Reconstruct 3D points [S1, unofficial]
    points_3d = depths_to_points(depth_map, camera)

    # Compute spatial derivatives [S1, unofficial]
    # dx, dy are partial derivatives along x and y axes
    dx = torch.diff(points_3d, dim=-2, prepend=points_3d[:, :, -1:, :])
    dy = torch.diff(points_3d, dim=-1, prepend=points_3d[:, :, :, -1:])

    # Compute normal via cross product [S1, unofficial]
    normal = torch.cross(dx, dy, dim=-1)
    normal = torch.nn.functional.normalize(normal, dim=-1)
    return normal

def depths_to_points(depth_map, camera):
    # Compute NDC to pixel projection matrix [S1, unofficial]
    # This involves camera intrinsics and extrinsics
    # Create meshgrid of pixel coordinates
    h, w = depth_map.shape[-2:]
    y, x = torch.meshgrid(torch.arange(h), torch.arange(w), indexing='ij')
    pixel_coords = torch.stack([x, y], dim=-1).float().to(depth_map.device)

    # Compute ray directions [S1, unofficial]
    # Using camera intrinsics to convert pixel coords to ray directions
    ray_dirs = pixel_to_ray_directions(pixel_coords, camera.intrinsics)

    # Scale by depth [S1, unofficial]
    points_3d = ray_dirs * depth_map.unsqueeze(-1)

    # Transform to world space using camera extrinsics [G]
    # Assuming points_3d are in camera space
    points_3d_world = transform_to_world(points_3d, camera.extrinsics)
    return points_3d_world
```

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of Gaussians and pixels covered. Dominated by rasterization and compositing steps.
- **Memory Complexity:** Linear with respect to the number of Gaussians and image resolution. Stores intermediate tensors like `allmap`.
- **Performance:** Fast rendering due to efficient rasterization. Performance scales with image resolution and number of Gaussians.
- **Strengths:** Efficient rendering of 2D Gaussian primitives. Provides rich geometric information (normals, depth).
- **Limitations:** Requires pre-computed or optimized 2D means. Depth estimation depends on the `depth_ratio` parameter.
- **When to use:** When rendering 2D Gaussian splats is required, and geometric outputs (normals, depth) are needed.
- **When not to use:** When 3D Gaussian primitives are used, or when real-time performance is critical and 2D means are not pre-computed.

## Background
Originated from the implementation of 2D Gaussian Splatting in the `gaussian-splatting-lightning` repository [S1, unofficial]. It extends the 3D Gaussian Splatting framework to handle 2D primitives, providing a way to render surfels with geometric attributes.

## Sources
[S1] yzslab/gaussian-splatting-lightning, https://github.com/yzslab/gaussian-splatting-lightning, fetched 2026-10-27 (unofficial)
