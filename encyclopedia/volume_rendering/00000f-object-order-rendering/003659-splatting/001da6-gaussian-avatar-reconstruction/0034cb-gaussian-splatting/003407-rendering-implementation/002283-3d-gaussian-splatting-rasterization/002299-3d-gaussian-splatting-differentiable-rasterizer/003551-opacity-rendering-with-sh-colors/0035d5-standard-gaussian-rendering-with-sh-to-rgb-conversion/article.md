# Standard Gaussian Rendering with SH-to-RGB Conversion

Also called: SH-to-RGB post-rasterization rendering, Spherical Harmonics Color Rendering

## Overview
This method is a rendering pass that computes RGB colors from spherical harmonics (SH) coefficients in Python before rasterization, bypassing the rasterizer's built-in SH evaluation [S1, unofficial]. It rasterizes Gaussians using precomputed geometry and opacity, then converts SH coefficients to RGB colors post-rasterization [S1, unofficial]. It differs from [Opacity Rendering with SH Colors](7480-opacity-rendering-with-sh-colors.md) by performing the SH-to-RGB conversion in the Python host code rather than precomputing colors for the rasterizer, allowing the rasterizer to use simpler color inputs while deferring the directional color evaluation to the CPU/GPU Python layer [S1, unofficial].

## How it works
**Inputs:**
- `gaussians`: A Gaussian model object containing 3D means, SH coefficients, opacities, scales, rotations, and covariance matrices.
- `viewpoint_camera`: Camera parameters including `camera_center`.
- `pipeline`: Rendering pipeline configuration.
- `background`: Background color tensor.
- `active_sh_degree`: Integer controlling the order of spherical harmonics used for color evaluation [S1, unofficial].

**Outputs:**
- `render`: Rendered RGB image.
- `depth`: Rendered depth map.
- `alpha`: Rendered alpha mask.
- `radii`: Visibility filter (radii > 0).
- `viewspace_points`, `visibility_filter`, `opacity`, `color`: Additional metadata [S1, unofficial].

**Data Structures:**
- `shs_view`: Tensor of shape `(-1, 3, (max_sh_degree+1)^2)` representing SH coefficients reshaped for evaluation [S1, unofficial].
- `dir_pp_normalized`: Normalized view direction vectors from camera center to each Gaussian [S1, unofficial].

**Steps:**
1. **Reshape SH features:** Transpose and view SH features as `(-1, 3, (max_sh_degree+1)^2)` [S1, unofficial].
   $$ \text{shs\_view} = \text{pc.get\_features}^T \text{ viewed as } (-1, 3, (\text{pc.max\_sh\_degree}+1)^2) $$
2. **Compute view directions:** Calculate normalized direction vectors from the camera center to each Gaussian center [S1, unofficial].
   $$ \text{dir\_pp} = \text{pc.get\_xyz} - \text{viewpoint\_camera.camera\_center.repeat}(\text{pc.get\_features.shape}[0], 1) $$
   $$ \text{dir\_pp\_normalized} = \frac{\text{dir\_pp}}{\|\text{dir\_pp}\|_2} $$
3. **Evaluate SH to RGB:** Use `eval_sh` with `active_sh_degree`, `shs_view`, and `dir_pp_normalized` to get RGB values [S1, unofficial].
   $$ \text{sh2rgb} = \text{eval\_sh}(\text{active\_sh\_degree}, \text{shs\_view}, \text{dir\_pp\_normalized}) $$
4. **Clamp colors:** Clamp resulting RGB values to a minimum of 0.0 after adding a 0.5 bias [S1, unofficial].
   $$ \text{color} = \max(\text{sh2rgb} + 0.5, 0.0) $$
5. **Rasterize:** Call the rasterizer with `means3D`, `means2D`, `shs`, `colors_precomp=None`, `opacities`, `scales`, `rotations`, and `cov3D_precomp` to get `rendered_image`, `radii`, `rendered_depth`, and `rendered_alpha` [S1, unofficial].
6. **Return results:** Return a dictionary containing `render`, `depth`, `alpha`, `viewspace_points`, `visibility_filter` (radii > 0), `radii`, `opacity`, and `color` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_sh_to_rgb(gaussians, viewpoint_camera, pipeline, background, active_sh_degree):
    # Step 1: Reshape SH features
    # pc.get_features shape: (N, 3, (max_sh_degree+1)^2) or similar
    # Transpose and view as (-1, 3, (max_sh_degree+1)^2)
    shs_view = gaussians.get_features.transpose(1, 2).view(-1, 3, (gaussians.max_sh_degree + 1) ** 2)
    
    # Step 2: Compute normalized view directions
    # dir_pp: (N, 3)
    dir_pp = gaussians.get_xyz - viewpoint_camera.camera_center.repeat(gaussians.get_features.shape[0], 1)
    # dir_pp_normalized: (N, 3)
    dir_pp_normalized = dir_pp / dir_pp.norm(dim=1, keepdim=True)
    
    # Step 3: Evaluate SH to RGB
    # sh2rgb: (N, 3)
    sh2rgb = eval_sh(active_sh_degree, shs_view, dir_pp_normalized)
    
    # Step 4: Clamp colors
    # color: (N, 3)
    color = torch.clamp_min(sh2rgb + 0.5, 0.0)
    
    # Step 5: Rasterize
    # Note: colors_precomp is None, SH is handled in Python
    render_pkg = rasterize_gaussians(
        means3D=gaussians.get_xyz,
        means2D=gaussians.get_means2D(), # Precomputed or computed here
        shs=gaussians.get_features,
        colors_precomp=None,
        opacities=gaussians.get_opacity,
        scales=gaussians.get_scales,
        rotations=gaussians.get_rotations,
        cov3D_precomp=gaussians.get_covariance,
        background=background,
        pipeline=pipeline
    )
    
    # Step 6: Return results
    return {
        "render": render_pkg["render"],
        "depth": render_pkg["depth"],
        "alpha": render_pkg["alpha"],
        "viewspace_points": render_pkg["viewspace_points"],
        "visibility_filter": render_pkg["radii"] > 0,
        "radii": render_pkg["radii"],
        "opacity": render_pkg["opacity"],
        "color": color
    }
```

**Parameters:**
- `active_sh_degree`: Controls the order of spherical harmonics used for color evaluation [S1, unofficial]. Typical value: 3 [S1, unofficial].

**Edge Cases and Pitfalls:**
- Precision: Clamping after adding 0.5 bias prevents negative colors [S1, unofficial].
- Boundaries: Ensure `dir_pp` is not zero vector to avoid division by zero in normalization [G].
- Degenerate input: Handle cases where `gaussians.get_features` shape does not match expected dimensions [G].

## Operating characteristics
- **Time Complexity:** O(N) for SH evaluation and clamping, where N is the number of Gaussians [S1, unofficial]. Rasterization complexity depends on the rasterizer implementation [S1, unofficial].
- **Memory Complexity:** O(N * (max_sh_degree+1)^2) for storing SH coefficients [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Linear with number of Gaussians [S1, unofficial].
- **Strengths:** Decouples SH evaluation from rasterization, allowing for flexible color updates [S1, unofficial].
- **Limitations:** Requires Python-side computation for SH evaluation, potentially slower than GPU-based SH evaluation [S1, unofficial].
- **When to use:** When SH coefficients need to be updated frequently or when using a rasterizer that does not support SH evaluation [S1, unofficial].
- **When not to use:** When performance is critical and SH evaluation can be done efficiently on the GPU [S1, unofficial].

## Background
Origin: This method is part of the Gaussian Splatting framework, specifically the rendering implementation for 3D Gaussian Splatting [S1, unofficial]. It was developed by the GRAPHDECO research group at Inria [S1, unofficial].

## Sources
[S1] Fictionarry/DNGaussian, https://github.com/Fictionarry/DNGaussian, fetched 2026-10-27 (unofficial)
