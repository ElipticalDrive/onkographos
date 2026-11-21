# Depth Rendering with Constant Opacity

Also called: Depth-Only Rendering Mode

## Overview
Depth Rendering with Constant Opacity is a specialized rendering pass for 3D Gaussian Splatting that produces clean depth maps by forcing all Gaussians to have uniform opacity and color [S2, unofficial]. Unlike standard alpha-blending which weights depth by Gaussian opacity and transmittance, this method ensures that depth accumulation is driven purely by geometric distance and rasterization order, eliminating shading artifacts and opacity-induced depth bias [S1][S2, unofficial]. It is distinct from [Depth Rendering with Variable Opacity](12622-depth-rendering-with-variable-opacity.md), which respects the learned alpha values of the Gaussians.

## How it works
The method modifies the standard Gaussian rasterization pipeline to strip away appearance information (color and variable opacity) and gradient information related to scale and rotation, leaving only the geometric position and a constant alpha value to drive the depth accumulation.

### Inputs and Outputs
**Inputs:**
*   `pc`: The Gaussian model object containing 3D parameters (`get_xyz`, `get_scaling`, `get_rotation`, `get_covariance`).
*   `viewpoint_camera`: The camera object providing intrinsic parameters (`FoVx`, `FoVy`, projection matrices).
*   `value`: The constant opacity multiplier applied to all Gaussians (default 0.95) [S2, unofficial].
*   `pipe`: Pipeline configuration object (e.g., `compute_cov3D_python`, `debug`) [S2, unofficial].

**Outputs:**
*   `render`: The rendered image (all ones, effectively a mask).
*   `depth`: The accumulated depth buffer (NDC depth).
*   `alpha`: The accumulated opacity buffer.
*   `viewspace_points`: The 3D points in view space.
*   `visibility_filter`: Boolean mask of visible Gaussians (radii > 0).
*   `radii`: The radius of each Gaussian's bounding box in screen space.

### Data Structures
*   `screenspace_points`: A tensor of shape `(num_gaussians, 2)` initialized to zeros. It is allocated with `requires_grad=True` and `retain_grad()` enabled to allow gradients to flow back to the 2D mean positions during optimization, even though they are not used in the forward pass of this specific mode [S2, unofficial].
*   `opacity`: A tensor of shape `(num_gaussians, 1)` filled with the constant `value`.
*   `colors_precomp`: A tensor of shape `(num_gaussians, 3)` filled with ones (white).
*   `scales`, `rotations`, `cov3D_precomp`: Set to `None` or detached tensors to prevent gradient leakage into depth estimation [S2, unofficial].

### Parameters
*   `value`: Constant opacity multiplier. Default is 0.95 [S2, unofficial].
*   `background`: Background color tensor (typically `[1,1,1]` or `[0,0,0]`) [S2, unofficial].

### Steps
1.  **Initialize Screenspace Points:** Create a tensor `screenspace_points` with the same shape and device as `pc.get_xyz`[:, :2], initialized to zero. Enable gradient retention:
    ```python
    screenspace_points = torch.zeros_like(pc.get_xyz[:, :2], dtype=pc.get_xyz.dtype, requires_grad=True, device="cuda") + 0
    try:
        screenspace_points.retain_grad()
    except:
        pass
    ```
    [S2, unofficial]

2.  **Compute FOV Tangents:** Calculate the tangent of half the field of view from the camera:
    $$ \text{tanfovx} = \tan(\frac{\text{FoVx}}{2}) $$
    $$ \text{tanfovy} = \tan(\frac{\text{FoVy}}{2}) $$
    [S2, unofficial]

3.  **Configure Rasterization Settings:** Create `GaussianRasterizationSettings` with:
    *   Image dimensions (width, height).
    *   `tanfovx`, `tanfovy`.
    *   `background` color.
    *   `scale_modifier`: 1.0 (or as configured).
    *   `viewmatrix` and `projmatrix`: From the camera.
    *   `sh_degree`: From the pipeline.
    *   `camera_position`: From the camera.
    *   `prefiltered`: False.
    *   `debug`: From the pipeline.
    [S2, unofficial]

4.  **Prepare Gaussian Attributes:**
    *   Set `means3D` to `pc.get_xyz`.
    *   Set `means2D` to `screenspace_points`.
    *   Set `opacity` to `torch.ones(num_gaussians, 1, device=device) * value`.
    *   Set `colors_precomp` to `torch.ones_like(pc.get_xyz)` (shape `(num_gaussians, 3)`).
    *   Set `shs` to `None`.
    [S2, unofficial]

5.  **Handle Scales and Rotations:** Inside a `torch.no_grad()` block:
    *   If `pipe.compute_cov3D_python` is true:
        *   Compute `cov3D_precomp` using `pc.get_covariance(scaling_modifier)`.
        *   Set `scales` and `rotations` to `None`.
    *   Else:
        *   Set `scales` to `pc.get_scaling.detach()`.
        *   Set `rotations` to `pc.get_rotation.detach()`.
        *   Set `cov3D_precomp` to `None`.
    [S2, unofficial]

6.  **Rasterize:** Call the Gaussian rasterizer with:
    *   `means3D`
    *   `means2D`
    *   `shs=None`
    *   `colors_precomp`
    *   `opacity`
    *   `scales`
    *   `rotations`
    *   `cov3D_precomp`
    *   `rasterization_settings`
    
    The rasterizer returns `rendered_image`, `radii`, `rendered_depth`, and `rendered_alpha`. Note that `rendered_image` will be uniform due to `colors_precomp` being ones.
    [S2, unofficial]

7.  **Return Results:** Return a dictionary containing:
    *   `render`: `rendered_image`
    *   `depth`: `rendered_depth`
    *   `alpha`: `rendered_alpha`
    *   `viewspace_points`: The viewspace points used.
    *   `visibility_filter`: `radii > 0`
    *   `radii`: The radii tensor.
    [S2, unofficial]

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def render_depth_constant_opacity(pc, viewpoint_camera, value=0.95, pipe=None, background=None):
    """
    Render depth map with constant opacity.
    
    Args:
        pc: GaussianModel object with methods get_xyz, get_scaling, get_rotation, get_covariance.
        viewpoint_camera: Camera object with FoVx, FoVy, projection matrices.
        value: Constant opacity multiplier (default 0.95).
        pipe: PipelineParams object.
        background: Tensor of background color.
        
    Returns:
        dict: Render results including depth, alpha, etc.
    """
    if background is None:
        # Default to white if not specified, assuming standard 3DGS background handling
        background = torch.tensor([1.0, 1.0, 1.0], dtype=torch.float32, device="cuda")
    
    if pipe is None:
        pipe = type('obj', (object,), {'compute_cov3D_python': False, 'debug': False})()

    num_gaussians = pc.get_xyz.shape[0]
    device = pc.get_xyz.device

    # Step 1: Initialize screenspace_points with gradient retention
    screenspace_points = torch.zeros_like(pc.get_xyz[:, :2], dtype=pc.get_xyz.dtype, requires_grad=True, device=device) + 0
    try:
        screenspace_points.retain_grad()
    except:
        pass

    # Step 2: Compute FOV tangents
    tanfovx = math.tan(viewpoint_camera.FoVx * 0.5)
    tanfovy = math.tan(viewpoint_camera.FoVy * 0.5)

    # Step 3: Configure Rasterization Settings
    # Note: In actual implementation, this creates a specific settings object for the C++/CUDA rasterizer
    rasterization_settings = {
        "image_width": viewpoint_camera.image_width,
        "image_height": viewpoint_camera.image_height,
        "tanfovx": tanfovx,
        "tanfovy": tanfovy,
        "bg": background,
        "scale_modifier": 1.0, # Assuming default scale modifier
        "viewmatrix": viewpoint_camera.world_view_transform,
        "projmatrix": viewpoint_camera.full_proj_transform,
        "sh_degree": pc.active_sh_degree,
        "camera_position": viewpoint_camera.camera_center,
        "prefiltered": False,
        "debug": pipe.debug
    }

    # Step 4: Prepare Attributes
    means3D = pc.get_xyz
    means2D = screenspace_points
    opacity = torch.ones((num_gaussians, 1), dtype=torch.float32, device=device) * value
    colors_precomp = torch.ones_like(pc.get_xyz) # Shape (N, 3)
    shs = None

    # Step 5: Handle Scales and Rotations
    with torch.no_grad():
        if pipe.compute_cov3D_python:
            cov3D_precomp = pc.get_covariance(1.0) # Assuming scaling_modifier 1.0
            scales = None
            rotations = None
        else:
            cov3D_precomp = None
            scales = pc.get_scaling.detach()
            rotations = pc.get_rotation.detach()

    # Step 6: Rasterize
    # The rasterizer function signature depends on the specific implementation (e.g., DiffGaussian, DNGaussian)
    # Assuming a standard call structure found in the sources
    rendered_image, radii, rendered_depth, rendered_alpha = gaussian_rasterizer(
        means3D=means3D,
        means2D=means2D,
        shs=shs,
        colors_precomp=colors_precomp,
        opacities=opacity,
        scales=scales,
        rotations=rotations,
        cov3D_precomp=cov3D_precomp,
        **rasterization_settings
    )

    # Step 7: Return Results
    visibility_filter = radii > 0
    
    return {
        "render": rendered_image,
        "depth": rendered_depth,
        "alpha": rendered_alpha,
        "viewspace_points": screenspace_points, # Or full viewspace points if available
        "visibility_filter": visibility_filter,
        "radii": radii
    }
```
[G] The `gaussian_rasterizer` call is a placeholder for the specific C++/CUDA kernel invocation used in the 3DGS implementation (e.g., `GaussianRasterization`). The exact argument names and order may vary slightly between forks (e.g., `DiffGaussian` vs `DNGaussian`), but the logical grouping of parameters remains consistent. The `viewspace_points` returned in the dictionary in the source notes refers to `screenspace_points` initialized in Step 1, which is a common pattern in these implementations to store the 2D means for gradient tracking.

### Edge Cases and Pitfalls
*   **Gradient Leakage:** Failing to detach `scales` and `rotations` or set them to `None` can cause gradients to flow into the depth estimation process, corrupting the optimization of the Gaussian positions [S2, unofficial].
*   **Precision:** The depth values are in NDC (Normalized Device Coordinates). Ensure the downstream consumer understands the coordinate system (e.g., converting to metric depth if required).
*   **Uniform Opacity:** Setting opacity to a constant value (e.g., 0.95) means that the final accumulated opacity at a pixel will depend on the number of overlapping Gaussians and their order. This can lead to "over-saturated" alpha values if many Gaussians overlap, but the depth is still correctly weighted by the transmittance $T_i$ [S1].
*   **No Color Information:** Since `colors_precomp` is set to ones, the `render` output is not a valid color image. It is only useful for extracting the `depth` and `alpha` buffers.

## Subtopics

### [Normalized Depth Rendering](8298-normalized-depth-rendering.md)
Computes a depth map by dividing the accumulated depth buffer by the accumulated opacity buffer to correct for transparency artifacts.

### [Depth Rendering with Variable Opacity](12622-depth-rendering-with-variable-opacity.md)
Produces a per-pixel depth buffer while respecting the variable opacity of the Gaussians, unlike this method which forces uniform opacity.

## Operating characteristics
*   **Time Complexity:** Same as standard Gaussian Splatting rasterization, $O(N)$ where $N$ is the number of Gaussians, as the computational cost is dominated by the rasterization and alpha-blending steps, which are unchanged in structure (only the input values differ) [S2, unofficial].
*   **Memory Complexity:** $O(N)$ for storing Gaussian attributes. The `screenspace_points` tensor adds $O(N)$ memory for gradient storage [S2, unofficial].
*   **Strengths:** Produces clean, unbiased depth maps that reflect the geometric distribution of Gaussians rather than their learned opacity. Useful for tasks like depth supervision, pruning, and normal estimation where consistent depth weighting is required [S1][S2, unofficial].
*   **Limitations:** Does not account for the actual transparency of the scene. If Gaussians have varying densities, the depth map may still exhibit artifacts from "floaters" or misplaced Gaussians, as the depth is influenced by all Gaussians along the ray [S1].
*   **When to Use:** When training a Gaussian Splatting model with depth supervision, or when extracting depth maps for downstream geometric tasks where opacity bias is undesirable.
*   **When Not to Use:** When visualizing the scene appearance or when the actual transparency of the object is important for the depth interpretation.

## Background
The method originates from the need to extract reliable depth information from 3D Gaussian Splatting models for training and optimization purposes. Standard alpha-blending depth rendering can be biased by the learned opacity values, which may not accurately reflect the geometric surface. By forcing constant opacity, the depth accumulation becomes a function of geometric distance and rasterization order, providing a more stable signal for gradient-based optimization [S1][S2, unofficial]. The specific implementation details, such as gradient retention for `screenspace_points` and detaching scales/rotations, are documented in the DNGaussian repository [S2, unofficial].

## Sources
[S1] Sparsegs: Sparse View Synthesis Using 3D Gaussian Splatting, https://doi.org/10.1109/3dv66043.2025.00100, fetched 2026-10-26
[S2] Fictionarry/DNGaussian, https://github.com/Fictionarry/DNGaussian, fetched 2026-10-27 (unofficial)
