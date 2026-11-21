# Differentiable Gaussian Rasterization with Plane/Depth Output

Also called: GaussianRasterizer, diff-plane-rasterization

## Overview
This method extends the standard differentiable Gaussian rasterizer to simultaneously output color, depth, and geometric auxiliary maps (such as normals or albedo) per pixel. It solves the problem of extracting dense geometric information from 3D Gaussian splats by treating depth and geometric attributes as additional channels in the alpha-compositing accumulation, allowing gradients to flow back to the 3D Gaussian parameters for geometry-aware training. It differs from standard color-only rasterization by maintaining and backpropagating through additional output tensors (`out_all_map`, `out_plane_depth`) alongside the color buffer.

## How it works
The method operates in two phases: a forward pass that projects 3D Gaussians and accumulates color, depth, and geometric maps, and a backward pass that computes gradients for all Gaussian parameters.

### Inputs and Outputs
**Forward Inputs:**
- `raster_settings.bg`: Background color tensor.
- `means3D`: 3D mean positions of Gaussians.
- `colors_precomp`: Pre-computed color attributes.
- `opacities`: Opacity of Gaussians.
- `scales`: Scale factors.
- `rotations`: Quaternion rotations.
- `raster_settings.scale_modifier`: Scale adjustment factor.
- `cov3Ds_precomp`: Pre-computed 3D covariance matrices.
- `all_maps`: Auxiliary geometric maps (e.g., normals, depth) associated with Gaussians.
- `raster_settings.viewmatrix`: View transformation matrix.
- `raster_settings.projmatrix`: Projection matrix.
- `raster_settings.tanfovx`, `raster_settings.tanfovy`: Tangent of horizontal/vertical field of view.
- `raster_settings.image_height`, `raster_settings.image_width`: Output image dimensions.
- `sh`: Spherical Harmonics coefficients (if used).
- `raster_settings.sh_degree`: SH degree.
- `raster_settings.campos`: Camera position.
- `raster_settings.prefiltered`: Boolean for pre-filtering.
- `raster_settings.render_geo`: Boolean flag to enable geometric output.
- `raster_settings.debug`: Boolean for debug mode.

**Forward Outputs:**
- `color`: Rendered color image.
- `radii`: Radius of each Gaussian in screen space.
- `out_observe`: Visibility mask.
- `out_all_map`: Rendered auxiliary geometric maps.
- `out_plane_depth`: Rendered depth map.

**Backward Inputs:**
- All forward inputs plus:
- `all_map_pixels`: Pixel-wise auxiliary map data.
- `radii`: From forward pass.
- `grad_out_color`: Gradient of loss w.r.t. color.
- `grad_out_all_map`: Gradient of loss w.r.t. auxiliary maps.
- `grad_out_plane_depth`: Gradient of loss w.r.t. depth.
- `geomBuffer`, `num_rendered`, `binningBuffer`, `imgBuffer`: Intermediate buffers from forward pass.

**Backward Outputs:**
- Gradients for: `means3D`, `means2D`, `means2D_abs`, `sh`, `colors_precomp`, `opacities`, `scales`, `rotations`, `cov3Ds_precomp`, `gard_all_map`.

### Data Structures
The implementation relies on a C++/CUDA backend (`_C.rasterize_gaussians`). The Python interface manages tensor layouts:
- `means3D`: Shape $(N, 3)$, float32.
- `colors_precomp`: Shape $(N, C)$, float32, where $C$ is color channels.
- `opacities`: Shape $(N, 1)$, float32.
- `scales`: Shape $(N, 3)$, float32.
- `rotations`: Shape $(N, 4)$, float32 (quaternions).
- `cov3Ds_precomp`: Shape $(N, 6)$, float32 (upper triangular covariance).
- `all_maps`: Shape $(N, M)$, float32, where $M$ is the number of auxiliary channels.
- `raster_settings`: A namespace or dictionary containing camera matrices and image dimensions.

### Algorithm Steps

#### Forward Pass
1.  **Restructure Arguments**: Pack all inputs into a single tuple `args` in the specific order required by the C++ backend:
    ```python
    args = (
        raster_settings.bg, means3D, colors_precomp, opacities, scales, rotations,
        raster_settings.scale_modifier, cov3Ds_precomp, all_maps,
        raster_settings.viewmatrix, raster_settings.projmatrix,
        raster_settings.tanfovx, raster_settings.tanfovy,
        raster_settings.image_height, raster_settings.image_width,
        sh, raster_settings.sh_degree, raster_settings.campos,
        raster_settings.prefiltered, raster_settings.render_geo,
        raster_settings.debug
    )
    ```
2.  **Invoke Rasterizer**: Call the CUDA kernel `_C.rasterize_gaussians(*args)`. This performs:
    -   3D to 2D projection of Gaussians.
    -   Sorting of Gaussians by depth.
    -   Alpha compositing for color, depth, and auxiliary maps.
3.  **Save for Backward**: Store all tensors required for gradient computation in the context:
    ```python
    ctx.save_for_backward(
        out_all_map, colors_precomp, all_maps, means3D, scales, rotations,
        cov3Ds_precomp, radii, sh, geomBuffer, binningBuffer, imgBuffer
    )
    ```
4.  **Return Results**: Return `color`, `radii`, `out_observe`, `out_all_map`, and `out_plane_depth`.

#### Backward Pass
1.  **Restore Tensors**: Retrieve saved tensors from the context.
2.  **Restructure Backward Arguments**: Pack inputs including gradients and intermediate buffers:
    ```python
    args_bw = (
        raster_settings.bg, all_map_pixels, means3D, radii, colors_precomp,
        all_maps, scales, rotations, raster_settings.scale_modifier,
        cov3Ds_precomp, raster_settings.viewmatrix, raster_settings.projmatrix,
        tanfovx, tanfovy, grad_out_color, grad_out_all_map, grad_out_plane_depth,
        sh, raster_settings.sh_degree, raster_settings.campos,
        geomBuffer, num_rendered, binningBuffer, imgBuffer,
        raster_settings.render_geo, raster_settings.debug
    )
    ```
3.  **Invoke Backward Kernel**: Call `_C.rasterize_gaussians_backward(*args_bw)`.
4.  **Return Gradients**: Return the computed gradients for all Gaussian parameters.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
from diff_gaussian_rasterization import GaussianRasterizationSettings, GaussianRasterizer

def forward_gaussian_rasterization_with_geometry(
    means3D, colors_precomp, opacities, scales, rotations,
    cov3Ds_precomp, all_maps, viewmatrix, projmatrix,
    tanfovx, tanfovy, image_height, image_width, sh=None,
    sh_degree=0, campos=None, bg_color=torch.tensor([0,0,0]),
    scale_modifier=1.0, prefiltered=False, render_geo=True, debug=False
):
    """
    Forward pass for differentiable Gaussian rasterization with plane/depth output.
    """
    # 1. Setup rasterization settings
    raster_settings = GaussianRasterizationSettings(
        image_height=int(image_height),
        image_width=int(image_width),
        tanfovx=tanfovx,
        tanfovy=tanfovy,
        bg=bg_color,
        scale_modifier=scale_modifier,
        viewmatrix=viewmatrix,
        projmatrix=projmatrix,
        sh_degree=sh_degree,
        campos=campos,
        prefiltered=prefiltered,
        debug=debug
    )

    # 2. Initialize rasterizer
    rasterizer = GaussianRasterizer(raster_settings=raster_settings)

    # 3. Perform rasterization
    # The C++ backend handles the accumulation of color, depth, and all_maps
    # Returns: colors, radii, viewspace_points, visibility_filter, radii_all_map
    colors, radii, viewspace_points, visibility_filter, radii_all_map = rasterizer(
        means3D=means3D,
        means2D=torch.zeros_like(means3D), # Placeholder if not provided
        colors_precomp=colors_precomp,
        opacities=opacities,
        scales=scales,
        rotations=rotations,
        cov3D_precomp=cov3Ds_precomp,
        all_maps=all_maps # Passed to backend for geometric accumulation
    )

    # 4. Extract depth and geometric maps if enabled
    # Note: The exact extraction of 'out_plane_depth' and 'out_all_map' depends on the specific
    # C++ implementation's return structure. In PGSR, these are often part of the output tensors
    # or derived from the accumulated buffers.
    out_plane_depth = colors[:, 2:3, :, :] # Example: assuming depth is stored in a specific channel or separate output
    out_all_map = radii_all_map # Example: assuming auxiliary maps are returned in radii_all_map or similar

    return colors, radii, out_all_map, out_plane_depth

def backward_gaussian_rasterization_with_geometry(
    grad_out_color, grad_out_all_map, grad_out_plane_depth,
    means3D, colors_precomp, opacities, scales, rotations,
    cov3Ds_precomp, all_maps, sh, radii, geomBuffer, binningBuffer, imgBuffer,
    viewmatrix, projmatrix, tanfovx, tanfovy, campos, bg_color,
    scale_modifier, render_geo, debug
):
    """
    Backward pass for differentiable Gaussian rasterization with plane/depth output.
    """
    # 1. Restructure arguments for C++ backward kernel
    args_bw = (
        bg_color, all_maps, means3D, radii, colors_precomp,
        all_maps, scales, rotations, scale_modifier,
        cov3Ds_precomp, viewmatrix, projmatrix,
        tanfovx, tanfovy, grad_out_color, grad_out_all_map, grad_out_plane_depth,
        sh, 0, campos, # sh_degree assumed 0 for simplicity, adjust as needed
        geomBuffer, len(radii), binningBuffer, imgBuffer,
        render_geo, debug
    )

    # 2. Invoke backward kernel
    # Returns gradients for means2D, means2D_abs, colors_precomp, opacities, means3D,
    # cov3Ds_precomp, sh, scales, rotations, gard_all_map
    grad_means2D, grad_means2D_abs, grad_colors_precomp, grad_opacities, grad_means3D, grad_cov3Ds_precomp, grad_sh, grad_scales, grad_rotations, gard_all_map = _C.rasterize_gaussians_backward(*args_bw)

    return grad_means3D, grad_cov3Ds_precomp, grad_scales, grad_rotations, grad_opacities, grad_colors_precomp, grad_sh, gard_all_map
```

### Parameters and Typical Values
-   `sh_degree`: Typically 0 for color-only, higher for SH-based color.
-   `voxel_size`: Used in post-processing (e.g., 0.002 in PGSR).
-   `max_depth`: Culling distance (e.g., 5.0 in PGSR).
-   `bg_color`: White `[1,1,1]` or black `[0,0,0]`.

### Edge Cases and Pitfalls
-   **Debug Mode**: If `debug=True`, tensors are copied to CPU before calling the C++ rasterizer to prevent corruption during exceptions. Snapshots `snapshot_fw.dump` or `snapshot_bw.dump` are saved [S1, S1, unofficial].
-   **Precision**: Floating-point precision in depth accumulation can affect gradient stability.
-   **Degenerate Input**: Gaussians with zero scale or opacity may cause division by zero or invalid gradients.

## Subtopics

### [Gaussian Splat Rasterization (Vertex Shader)](3253-gaussian-splat-rasterization-vertex-shader.md)
Uses vertex shaders for rasterization, differing in hardware acceleration and pipeline integration.

### [Opacity Rendering with SH Colors](7480-opacity-rendering-with-sh-colors.md)
Focuses on SH color representation, differing in color encoding.

### [3D Gaussian Unscented Transform (3DGUT)](9154-3d-gaussian-unscented-transform-3dgut.md)
Uses unscented transform for uncertainty, differing in probabilistic modeling.

### [RoofGS Quadratic Form Expansion and Coefficient Folding](10258-roofgs-quadratic-form-expansion-and-coefficient-folding.md)
Optimizes quadratic forms, differing in computational efficiency.

### [Transmittance of Gaussian Splatting](10263-transmittance-of-gaussian-splatting.md)
Computes transmittance, differing in physical accuracy.

### [Rendering Pipeline and Configuration](11406-rendering-pipeline-and-configuration.md)
General pipeline setup, differing in configuration scope.

### [Differentiable Backpropagation Mechanisms](11407-differentiable-backpropagation-mechanisms.md)
General backpropagation, differing in gradient flow details.

### [Multi-view Mini-batch Rasterization](13386-multi-view-mini-batch-rasterization.md)
Handles multiple views, differing in batch processing.

### [Gaussian Normal Estimation](13447-gaussian-normal-estimation.md)
Estimates normals, differing in geometric output.

### [Rasterization Variant](17388-rasterization-variant.md)
Alternative rasterization methods, differing in implementation.

### [Camera Pose Gradient Computation](18044-camera-pose-gradient-computation.md)
Computes camera pose gradients, differing in optimization targets.

### [Continuous Stochastic Solids Volume Rendering](18428-continuous-stochastic-solids-volume-rendering.md)
Volume rendering approach, differing in rendering paradigm.

### [Single-Pass Rasterization with Alpha-Blending](19202-single-pass-rasterization-with-alpha-blending.md)
Single-pass alpha blending, differing in pass structure.

## Operating characteristics
-   **Time Complexity**: Dominated by the CUDA kernel execution, scaling with the number of Gaussians and image resolution.
-   **Memory Complexity**: Linear in the number of Gaussians for storing parameters and intermediate buffers.
-   **Performance**: High performance due to GPU acceleration. Measured performance depends on hardware and scene complexity.
-   **Strengths**: Differentiable, supports geometric outputs, efficient GPU implementation.
-   **Limitations**: Requires CUDA, limited to Gaussian representations.
-   **When to Use**: When geometry-aware training is needed for Gaussian splats.
-   **When Not to Use**: For non-differentiable rendering or non-Gaussian representations.

## Background
The method originates from the PGSR (Point-based Gaussian Splatting for Real-time Dynamic View Synthesis) project, which extends 3D Gaussian Splatting to support geometric outputs [S1, S1, unofficial]. It builds on the foundational 3D Gaussian Splatting work by Kerbl et al. [S1, S1, unofficial].

## Sources
[S1] zju3dv/PGSR, https://github.com/zju3dv/PGSR, fetched 2026-10-27 (unofficial)
