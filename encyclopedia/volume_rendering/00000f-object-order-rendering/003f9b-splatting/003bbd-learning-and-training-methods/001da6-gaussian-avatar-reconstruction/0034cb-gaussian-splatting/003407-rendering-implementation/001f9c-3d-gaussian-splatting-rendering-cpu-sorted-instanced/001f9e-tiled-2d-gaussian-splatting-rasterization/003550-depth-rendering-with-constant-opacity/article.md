# Depth Rendering with Constant Opacity

Also called: Depth-Only Rendering Mode

## Overview
A specialized rendering pass for 3D Gaussian Splatting that extracts clean depth maps by forcing uniform opacity and color across all Gaussians. It solves the problem of depth ambiguity caused by varying Gaussian opacities and shading, ensuring that the rasterizer's depth accumulation is driven strictly by geometric distance rather than opacity weighting. It differs from standard rendering by disabling gradient computation for geometric parameters and setting all Gaussians to a constant opacity value (default 0.95) and white color.

## How it works
**Inputs:**
- `pc`: Gaussian model object containing `get_xyz`, `get_scaling`, `get_rotation`, `get_covariance`, `get_sh`, `get_opacity`.
- `viewpoint_camera`: Camera object with `FoVx`, `FoVy`, `world_view_transform`, `full_proj_transform`, `image_width`, `image_height`.
- `value`: Constant opacity multiplier (default 0.95).
- `pipe`: Pipeline parameters (e.g., `compute_cov3D_python`).

**Outputs:**
- Dictionary containing:
  - `render`: Rendered image (all white).
  - `depth`: Rendered depth map.
  - `alpha`: Rendered alpha map (uniform).
  - `viewspace_points`: Tensor of 2D screen-space points with gradients retained.
  - `visibility_filter`: Boolean tensor indicating visible Gaussians (`radii > 0`).
  - `radii`: Integer tensor of visible Gaussian radii.

**Data Structures:**
- `screenspace_points`: Tensor of shape `(N, 2)` or `(N, 3)` depending on implementation, initialized to zeros, dtype matches `pc.get_xyz`, device "cuda", `requires_grad=True`.
- `opacity`: Tensor of shape `(N, 1)`, dtype float32, device "cuda", values equal to `value`.
- `colors_precomp`: Tensor of shape `(N, 3)`, dtype float32, device "cuda", values equal to 1.0.
- `scales`, `rotations`, `cov3D_precomp`: Set to `None` or detached tensors as per `pipe.compute_cov3D_python`.
- `shs`: Set to `None`.

**Steps:**
1. **Initialize Screen-Space Points:** Create `screenspace_points` tensor of zeros with same shape and dtype as `pc.get_xyz`, on CUDA device, with `requires_grad=True`. Attempt to retain gradients: `try: screenspace_points.retain_grad() except: pass`. [S1, unofficial]
2. **Compute FOV Tangents:** Calculate `tanfovx = math.tan(viewpoint_camera.FoVx * 0.5)` and `tanfovy = math.tan(viewpoint_camera.FoVy * 0.5)`. [S1, unofficial]
3. **Configure Rasterization Settings:** Create `GaussianRasterizationSettings` with:
   - `image_width`, `image_height` from `viewpoint_camera`.
   - `tanfovx`, `tanfovy`.
   - `background`: White `[1,1,1]` or black `[0,0,0]` based on dataset setting.
   - `scale_modifier`: Default or from pipeline.
   - `viewmatrix`, `projmatrix`: From `viewpoint_camera`.
   - `sh_degree`: From `pc`.
   - `camera_position`: From `viewpoint_camera`.
   - `prefiltered`: `False`.
   - `debug`: `False`. [S1, unofficial]
4. **Prepare Gaussian Attributes:**
   - Set `means3D = pc.get_xyz`.
   - Set `means2D = screenspace_points`.
   - Set `opacity = torch.ones(pc.get_xyz.shape[0], 1, device=pc.get_xyz.device) * value`.
   - Set `colors_precomp = torch.ones_like(pc.get_xyz)`.
   - Set `shs = None`.
   - If `pipe.compute_cov3D_python` is true:
     - Compute `cov3D_precomp = pc.get_covariance(scaling_modifier)`.
     - Set `scales = None`, `rotations = None`.
   - Else:
     - Set `scales = pc.get_scaling.detach()`.
     - Set `rotations = pc.get_rotation.detach()`.
     - Set `cov3D_precomp = None`. [S1, unofficial]
5. **Render:** Call the rasterizer with `means3D`, `means2D`, `shs=None`, `colors_precomp`, `opacity`, `scales`, `rotations`, `cov3D_precomp`, and `rasterization_settings`. Obtain `rendered_image`, `radii`, `rendered_depth`, `rendered_alpha`. [S1, unofficial]
6. **Return Results:** Return dictionary with `render=rendered_image`, `depth=rendered_depth`, `alpha=rendered_alpha`, `viewspace_points=screenspace_points`, `visibility_filter=(radii > 0)`, `radii=radii`. [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def render_depth_constant_opacity(pc, viewpoint_camera, value=0.95, pipe=None):
    """
    Render depth map with constant opacity for all Gaussians.
    
    Args:
        pc: GaussianModel instance with methods get_xyz, get_scaling, get_rotation, get_covariance, get_sh, get_opacity.
        viewpoint_camera: Camera object with FoVx, FoVy, world_view_transform, full_proj_transform, image_width, image_height.
        value: Constant opacity multiplier (default 0.95).
        pipe: PipelineParams with compute_cov3D_python attribute.
        
    Returns:
        dict: Dictionary with render, depth, alpha, viewspace_points, visibility_filter, radii.
    """
    # Step 1: Initialize screen-space points
    screenspace_points = torch.zeros_like(pc.get_xyz, dtype=pc.get_xyz.dtype, requires_grad=True, device="cuda") + 0
    try:
        screenspace_points.retain_grad()
    except:
        pass

    # Step 2: Compute FOV tangents
    tanfovx = math.tan(viewpoint_camera.FoVx * 0.5)
    tanfovy = math.tan(viewpoint_camera.FoVy * 0.5)

    # Step 3: Configure rasterization settings
    # Assuming GaussianRasterizationSettings is available from gaussian_renderer
    from gaussian_renderer import GaussianRasterizationSettings
    
    bg_color = [1, 1, 1] if hasattr(viewpoint_camera, 'white_background') and viewpoint_camera.white_background else [0, 0, 0]
    background = torch.tensor(bg_color, dtype=torch.float32, device="cuda")
    
    rasterization_settings = GaussianRasterizationSettings(
        image_height=int(viewpoint_camera.image_height),
        image_width=int(viewpoint_camera.image_width),
        tanfovx=tanfovx,
        tanfovy=tanfovy,
        bg=background,
        scale_modifier=1.0, # Default scale modifier, adjust if pipe provides it
        viewmatrix=viewpoint_camera.world_view_transform,
        projmatrix=viewpoint_camera.full_proj_transform,
        sh_degree=pc.active_sh_degree,
        campos=viewpoint_camera.camera_center,
        prefiltered=False,
        debug=False
    )

    # Step 4: Prepare Gaussian attributes
    means3D = pc.get_xyz
    means2D = screenspace_points
    
    # Constant opacity
    opacity = torch.ones(pc.get_xyz.shape[0], 1, device=pc.get_xyz.device) * value
    
    # Constant color (white)
    colors_precomp = torch.ones_like(pc.get_xyz)
    
    # Shaders disabled
    shs = None
    
    # Covariance handling
    if pipe.compute_cov3D_python:
        cov3D_precomp = pc.get_covariance(1.0) # scaling_modifier=1.0
        scales = None
        rotations = None
    else:
        cov3D_precomp = None
        scales = pc.get_scaling.detach()
        rotations = pc.get_rotation.detach()

    # Step 5: Render
    # Assuming GaussianRasterizer is available
    from gaussian_renderer import GaussianRasterizer
    
    rasterizer = GaussianRasterizer(rasterization_settings=rasterization_settings)
    
    rendered_image, radii, rendered_depth, rendered_alpha = rasterizer(
        means3D=means3D,
        means2D=means2D,
        shs=shs,
        colors_precomp=colors_precomp,
        opacities=opacity,
        scales=scales,
        rotations=rotations,
        cov3D_precomp=cov3D_precomp
    )

    # Step 6: Return results
    visibility_filter = radii > 0
    
    return {
        "render": rendered_image,
        "depth": rendered_depth,
        "alpha": rendered_alpha,
        "viewspace_points": screenspace_points,
        "visibility_filter": visibility_filter,
        "radii": radii
    }
```

**Parameters:**
- `value`: Constant opacity multiplier for all Gaussians. Default is 0.95. [S1, unofficial]

**Edge Cases and Pitfalls:**
- Gradient leakage: Scales and rotations must be detached or set to `None` to prevent gradients from flowing into depth estimation during training. [S1, unofficial]
- Covariance computation: If `pipe.compute_cov3D_python` is true, covariance is computed explicitly; otherwise, precomputed or detached scales/rotations are used. Failure to handle this correctly may cause errors or incorrect depth. [S1, unofficial]
- Screen-space points: Must be initialized with `requires_grad=True` and gradients retained to allow backpropagation to 3D positions if needed for depth-based loss. [S1, unofficial]

## Subtopics

### [Normalized Depth Rendering](8298-normalized-depth-rendering.md)
Computes depth normalized by accumulated opacity to correct for transparency artifacts in Gaussian splatting renders.

## Operating characteristics
- **Time Complexity:** O(N) for Gaussian setup, O(T) for rasterization where T is number of tiles/pixels. Similar to standard Gaussian Splatting rasterization.
- **Memory Complexity:** O(N) for Gaussian attributes, O(W*H) for output buffers.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with number of Gaussians and image resolution, similar to base rasterization.
- **Strengths:** Produces clean, unambiguous depth maps by removing opacity and color variations. Simple to implement as a specialized pass.
- **Limitations:** Does not account for transparency effects in depth; depth is purely geometric. Requires detachment of geometric parameters to avoid gradient issues.
- **When to use:** When clean depth maps are needed for downstream tasks (e.g., depth supervision, normal estimation) and transparency artifacts are undesirable.
- **When not to use:** When depth should reflect opacity-weighted contributions or when transparency is important for the application.

## Background
Originated in the DNGaussian repository as an unofficial implementation detail for depth extraction. [S1, unofficial]

## Sources
[S1] Fictionarry/DNGaussian, https://github.com/Fictionarry/DNGaussian, fetched 2026-10-27 (unofficial)
