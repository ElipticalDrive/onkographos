# Depth Rendering with Constant Opacity

Also called: Depth-Only Rendering Mode

## Overview
Depth Rendering with Constant Opacity is a specialized rendering pass for 3D Gaussian Splatting designed to extract clean, geometrically accurate depth maps by decoupling depth accumulation from opacity weighting [S1, unofficial]. Unlike standard rendering where depth is weighted by the alpha compositing process, this method forces all Gaussians to have uniform opacity and color, ensuring that the rasterizer's depth accumulation is driven purely by geometric distance [S1, unofficial]. This approach corrects for transparency artifacts inherent in standard Gaussian splatting compositing, making it suitable for applications requiring precise depth estimation without shading variations [S1, unofficial].

## How it works
**Inputs:**
- `pc`: The Gaussian model object containing 3D parameters (xyz, scaling, rotation, covariance, SH coefficients).
- `viewpoint_camera`: The camera object containing field of view (FoV) and projection matrices.
- `image_size`: Tuple of (width, height) for the output depth map.
- `value`: The constant opacity multiplier (default 0.95) [S1, unofficial].

**Outputs:**
- `render`: The rendered image (all ones, effectively unused for depth).
- `depth`: The accumulated depth buffer.
- `alpha`: The accumulated opacity buffer (constant).
- `viewspace_points`: The transformed 3D points in view space.
- `visibility_filter`: Boolean mask of visible Gaussians (radii > 0).
- `radii`: The radius of each Gaussian's bounding box in screen space.

**Data Structures:**
- `screenspace_points`: A tensor of shape `(num_gaussians, 2)` initialized to zeros, with gradient retention enabled to allow backpropagation if needed, though typically detached during inference [S1, unofficial].
- `opacity`: A tensor of shape `(num_gaussians, 1)` filled with the constant `value` [S1, unofficial].
- `colors_precomp`: A tensor of shape `(num_gaussians, 3)` filled with ones, removing color information [S1, unofficial].
- `scales`, `rotations`, `cov3D_precomp`: Set to `None` or detached values to prevent gradient leakage and simplify computation [S1, unofficial].

**Steps:**

1. **Initialization**:
   - Create a tensor `screenspace_points` of shape `(num_gaussians, 2)` initialized to zeros on the CUDA device.
   - Enable gradient retention: `screenspace_points.retain_grad()` [S1, unofficial].

2. **Camera Setup**:
   - Compute tangent field of view values:
     $$ \text{tanfovx} = \tan(\frac{\text{FoV}_x}{2}) $$
     $$ \text{tanfovy} = \tan(\frac{\text{FoV}_y}{2}) $$
   - Configure `GaussianRasterizationSettings` with:
     - Image dimensions (width, height).
     - Tangent FOV values (`tanfovx`, `tanfovy`).
     - Background color (typically white `[1,1,1]` or black `[0,0,0]`).
     - Scale modifier (usually 1.0).
     - View and projection matrices from `viewpoint_camera`.
     - SH degree (usually 0 for constant color).
     - Camera position.
     - `prefiltered=False`.
     - `debug=False` [S1, unofficial].

3. **Gaussian Parameter Preparation**:
   - Set `means3D` to `pc.get_xyz()` [S1, unofficial].
   - Set `means2D` to `screenspace_points` [S1, unofficial].
   - Set `opacity` to `torch.ones(num_gaussians, 1, device=device) * value` [S1, unofficial].
   - Set `colors_precomp` to `torch.ones_like(pc.get_xyz)` [S1, unofficial].
   - Set `shs` to `None` [S1, unofficial].

4. **Scale and Rotation Handling**:
   - Enter a `torch.no_grad()` context to prevent gradient leakage [S1, unofficial].
   - If `pipe.compute_cov3D_python` is true:
     - Compute `cov3D_precomp` using `pc.get_covariance(scaling_modifier)` [S1, unofficial].
     - Set `scales` and `rotations` to `None`.
   - Else:
     - Set `scales` to `pc.get_scaling().detach()` [S1, unofficial].
     - Set `rotations` to `pc.get_rotation().detach()` [S1, unofficial].
     - Set `cov3D_precomp` to `None` [S1, unofficial].

5. **Rasterization**:
   - Call the rasterizer with:
     - `means3D`
     - `means2D`
     - `shs=None`
     - `colors_precomp`
     - `opacity`
     - `scales`
     - `rotations`
     - `cov3D_precomp`
   - The rasterizer returns `rendered_image`, `radii`, `rendered_depth`, and `rendered_alpha` [S1, unofficial].

6. **Post-Processing**:
   - Return a dictionary containing:
     - `render`: `rendered_image`
     - `depth`: `rendered_depth`
     - `alpha`: `rendered_alpha`
     - `viewspace_points`: `screenspace_points`
     - `visibility_filter`: `radii > 0`
     - `radii`: `radii` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def depth_rendering_with_constant_opacity(pc, viewpoint_camera, image_size, value=0.95):
    """
    Render depth map with constant opacity.
    
    Args:
        pc: Gaussian model object with get_xyz, get_scaling, get_rotation, get_covariance methods.
        viewpoint_camera: Camera object with FoVx, FoVy, world_view_transform, full_proj_transform, camera_center.
        image_size: Tuple (width, height).
        value: Constant opacity multiplier (default 0.95).
        
    Returns:
        Dictionary with render, depth, alpha, viewspace_points, visibility_filter, radii.
    """
    num_gaussians = pc.get_xyz.shape[0]
    device = pc.get_xyz.device
    
    # Step 1: Initialize screenspace_points with gradient retention
    screenspace_points = torch.zeros(num_gaussians, 2, dtype=pc.get_xyz.dtype, device=device, requires_grad=True) + 0
    try:
        screenspace_points.retain_grad()
    except:
        pass
    
    # Step 2: Camera Setup
    tanfovx = math.tan(viewpoint_camera.FoVx * 0.5)
    tanfovy = math.tan(viewpoint_camera.FoVy * 0.5)
    
    # Assuming GaussianRasterizationSettings is available from the rasterizer library
    raster_settings = GaussianRasterizationSettings(
        image_height=image_size[1],
        image_width=image_size[0],
        tanfovx=tanfovx,
        tanfovy=tanfovy,
        bg=torch.tensor([1.0, 1.0, 1.0], dtype=torch.float32, device=device), # Default white background
        scale_modifier=1.0,
        viewmatrix=viewpoint_camera.world_view_transform,
        projmatrix=viewpoint_camera.full_proj_transform,
        sh_degree=0,
        campos=viewpoint_camera.camera_center,
        prefiltered=False,
        debug=False
    )
    
    # Step 3: Gaussian Parameter Preparation
    means3D = pc.get_xyz()
    means2D = screenspace_points
    opacity = torch.ones(num_gaussians, 1, device=device) * value
    colors_precomp = torch.ones_like(pc.get_xyz)
    shs = None
    
    # Step 4: Scale and Rotation Handling
    with torch.no_grad():
        if hasattr(pc, 'compute_cov3D_python') and pc.compute_cov3D_python:
            cov3D_precomp = pc.get_covariance(1.0)
            scales = None
            rotations = None
        else:
            cov3D_precomp = None
            scales = pc.get_scaling().detach()
            rotations = pc.get_rotation().detach()
            
    # Step 5: Rasterization
    # Assuming rasterizer is the compiled CUDA kernel or wrapper
    rasterizer = GaussianRasterizer(raster_settings)
    
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
    
    # Step 6: Post-Processing
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
- `value`: Constant opacity multiplier for depth rendering. Default is 0.95 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Gradient Leakage**: Scales and rotations must be detached or set to `None` to prevent gradients from flowing into depth estimation during training [S1, unofficial].
- **Covariance Computation**: The method for computing `cov3D_precomp` depends on the pipeline setting `compute_cov3D_python`. Failure to handle this branch correctly will cause errors [S1, unofficial].
- **Background Color**: The background color affects the depth accumulation at pixels with no Gaussians. It should be set appropriately (e.g., far plane distance or a specific value) depending on the downstream use of the depth map [S1, unofficial].

## Subtopics

### [Normalized Depth Rendering](8298-normalized-depth-rendering.md)
Normalized Depth Rendering computes a depth map by dividing the accumulated depth buffer by the accumulated opacity buffer, correcting for transparency artifacts inherent in standard Gaussian splatting compositing. Unlike its parent method, which forces constant opacity to decouple depth from alpha weighting, this method preserves the original scene's opacity distribution while producing a geometrically accurate depth estimate.

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of Gaussians and pixels, similar to standard Gaussian Splatting rasterization, but with reduced computational overhead due to constant opacity and color [S1, unofficial].
- **Memory Complexity**: Linear with respect to the number of Gaussians for storing parameters, plus O(W*H) for the depth and alpha buffers [S1, unofficial].
- **Performance**: Faster than standard rendering due to simplified color and opacity handling, but still bound by the rasterization kernel's efficiency [S1, unofficial].
- **Strengths**: Produces clean, artifact-free depth maps; decouples depth from opacity; simple to implement as a specialized pass [S1, unofficial].
- **Limitations**: Does not provide color or shading information; requires a separate pass for standard rendering; depth values may be biased if background color is not handled correctly [S1, unofficial].
- **When to Use**: When geometric depth estimation is the primary goal, such as for depth completion, stereo matching, or 3D reconstruction [S1, unofficial].
- **When Not to Use**: When color, shading, or opacity information is required [S1, unofficial].

## Background
Depth Rendering with Constant Opacity is a technique derived from the 3D Gaussian Splatting framework, specifically adapted for depth extraction tasks. It was popularized by implementations such as DNGaussian, which introduced this specialized rendering pass to address the limitations of standard Gaussian Splatting in producing accurate depth maps [S1, unofficial]. The method leverages the rasterization pipeline of Gaussian Splatting but modifies the input parameters to enforce uniformity, thereby simplifying the depth accumulation process [S1, unofficial].

## Sources
[S1] Fictionarry/DNGaussian, https://github.com/Fictionarry/DNGaussian, fetched 2026-10-27 (unofficial)
