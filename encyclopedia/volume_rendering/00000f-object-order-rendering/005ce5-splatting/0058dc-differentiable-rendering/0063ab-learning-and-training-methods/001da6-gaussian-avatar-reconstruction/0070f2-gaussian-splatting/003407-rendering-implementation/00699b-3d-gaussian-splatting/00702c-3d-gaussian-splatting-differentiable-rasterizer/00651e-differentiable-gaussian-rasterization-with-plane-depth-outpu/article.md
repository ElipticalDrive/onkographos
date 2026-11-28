# Differentiable Gaussian Rasterization with Plane/Depth Output

Also called: GaussianRasterizer, diff-plane-rasterization

## Overview
This method extends the standard differentiable Gaussian rasterizer to simultaneously output color, depth, and geometric auxiliary maps per pixel [S1, unofficial]. It solves the problem of extracting dense geometric information from 3D Gaussian splats by treating depth and geometric attributes as additional channels in the alpha-compositing accumulation, allowing gradients to flow back to the 3D Gaussian parameters for geometry-aware training [S1, unofficial]. It differs from standard color-only rasterization by maintaining and backpropagating through additional output tensors (`out_all_map`, `out_plane_depth`) alongside the color buffer [S1, unofficial].

## How it works
The method operates in two phases: a forward pass that projects 3D Gaussians and accumulates color, depth, and geometric maps, and a backward pass that computes gradients for all Gaussian parameters [S1, unofficial].

### Inputs and Outputs
**Forward Inputs:**
- `raster_settings.bg`, `means3D`, `colors_precomp`, `opacities`, `scales`, `rotations`, `raster_settings.scale_modifier`, `cov3Ds_precomp`, `all_maps`, `raster_settings.viewmatrix`, `raster_settings.projmatrix`, `raster_settings.tanfovx`, `raster_settings.tanfovy`, `raster_settings.image_height`, `raster_settings.image_width`, `sh`, `raster_settings.sh_degree`, `raster_settings.campos`, `raster_settings.prefiltered`, `raster_settings.render_geo`, `raster_settings.debug` [S1, unofficial].

**Forward Outputs:**
- `color`, `radii`, `out_observe`, `out_all_map`, `out_plane_depth` [S1, unofficial].

**Backward Inputs:**
- `raster_settings.bg`, `all_map_pixels`, `means3D`, `radii`, `colors_precomp`, `all_maps`, `scales`, `rotations`, `raster_settings.scale_modifier`, `cov3Ds_precomp`, `raster_settings.viewmatrix`, `raster_settings.projmatrix`, `raster_settings.tanfovx`, `raster_settings.tanfovy`, `grad_out_color`, `grad_out_all_map`, `grad_out_plane_depth`, `sh`, `raster_settings.sh_degree`, `raster_settings.campos`, `geomBuffer`, `num_rendered`, `binningBuffer`, `imgBuffer`, `raster_settings.render_geo`, `raster_settings.debug` [S1, unofficial].

**Backward Outputs:**
- `grad_means2D`, `grad_means2D_abs`, `grad_colors_precomp`, `grad_opacities`, `grad_means3D`, `grad_cov3Ds_precomp`, `grad_sh`, `grad_scales`, `grad_rotations`, `gard_all_map` [S1, unofficial].

### Data Structures
The implementation relies on a C++/CUDA backend (`_C.rasterize_gaussians`) [S1, unofficial]. The Python interface packs arguments into ordered tuples for the backend [S1, unofficial]. Tensor layouts follow standard 3D Gaussian Splatting conventions: `means3D` $(N, 3)$, `colors_precomp` $(N, C)$, `opacities` $(N, 1)$, `scales` $(N, 3)$, `rotations` $(N, 4)$, `cov3Ds_precomp` $(N, 6)$, `all_maps` $(N, M)$, where $N$ is the number of Gaussians, $C$ is color channels, and $M$ is auxiliary channels [G].

### Algorithm Steps

#### Forward Pass
1. Restructure arguments into a single tuple in the exact order specified by the evidence [S1, unofficial].
2. Invoke the C++/CUDA rasterizer `_C.rasterize_gaussians(*args)` [S1, unofficial].
3. Save relevant tensors for backward: `out_all_map`, `colors_precomp`, `all_maps`, `means3D`, `scales`, `rotations`, `cov3Ds_precomp`, `radii`, `sh`, `geomBuffer`, `binningBuffer`, `imgBuffer` [S1, unofficial].
4. Return `color`, `radii`, `out_observe`, `out_all_map`, `out_plane_depth` [S1, unofficial].

#### Backward Pass
1. Restore saved tensors [S1, unofficial].
2. Restructure backward arguments into a tuple including gradients and intermediate buffers [S1, unofficial].
3. Invoke `_C.rasterize_gaussians_backward(*args_bw)` [S1, unofficial].
4. Return gradients for `means3D`, `means2D`, `means2D_abs`, `sh`, `colors_precomp`, `opacities`, `scales`, `rotations`, `cov3Ds_precomp`, `gard_all_map` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

# _C is the C++/CUDA extension module containing the rasterizer kernels
import _C

class GaussianRasterizerWithGeometry(torch.autograd.Function):
    @staticmethod
    def forward(ctx, bg, means3D, colors_precomp, opacities, scales, rotations,
                scale_modifier, cov3Ds_precomp, all_maps, viewmatrix, projmatrix,
                tanfovx, tanfovy, image_height, image_width, sh, sh_degree, campos,
                prefiltered, render_geo, debug):
        # 1. Restructure arguments into args tuple
        args = (bg, means3D, colors_precomp, opacities, scales, rotations,
                scale_modifier, cov3Ds_precomp, all_maps,
                viewmatrix, projmatrix, tanfovx, tanfovy,
                image_height, image_width, sh, sh_degree, campos,
                prefiltered, render_geo, debug)

        # 2. Invoke C++/CUDA rasterizer
        color, radii, out_observe, out_all_map, out_plane_depth = _C.rasterize_gaussians(*args)

        # 3. Save relevant tensors for backward
        ctx.save_for_backward(out_all_map, colors_precomp, all_maps, means3D, scales, rotations,
                              cov3Ds_precomp, radii, sh)
        # Intermediate buffers are captured from the C++ backend state
        ctx.geomBuffer = None  # Not given in the sources.
        ctx.binningBuffer = None  # Not given in the sources.
        ctx.imgBuffer = None  # Not given in the sources.
        ctx.num_rendered = 0  # Not given in the sources.

        return color, radii, out_observe, out_all_map, out_plane_depth

    @staticmethod
    def backward(ctx, grad_out_color, grad_out_all_map, grad_out_plane_depth):
        # 1. Restore saved tensors
        (out_all_map, colors_precomp, all_maps, means3D, scales, rotations,
         cov3Ds_precomp, radii, sh) = ctx.saved_tensors

        # 2. Restructure backward arguments
        args_bw = (
            ctx.bg, out_all_map, means3D, radii, colors_precomp,
            all_maps, scales, rotations, ctx.scale_modifier,
            cov3Ds_precomp, ctx.viewmatrix, ctx.projmatrix,
            ctx.tanfovx, ctx.tanfovy, grad_out_color, grad_out_all_map, grad_out_plane_depth,
            sh, ctx.sh_degree, ctx.campos,
            ctx.geomBuffer, ctx.num_rendered, ctx.binningBuffer, ctx.imgBuffer,
            ctx.render_geo, ctx.debug
        )

        # 3. Invoke backward kernel
        (grad_means2D, grad_means2D_abs, grad_colors_precomp, grad_opacities,
         grad_means3D, grad_cov3Ds_precomp, grad_sh, grad_scales, grad_rotations,
         gard_all_map) = _C.rasterize_gaussians_backward(*args_bw)

        # 4. Return gradients
        return (grad_means3D, None, grad_colors_precomp, grad_opacities, grad_scales,
                grad_rotations, None, grad_cov3Ds_precomp, None, None, None, None, None,
                None, None, None, None, None, None, None, None)
```

### Parameters and Typical Values
- `sh_degree`: Spherical harmonics degree [S1, unofficial].
- `voxel_size`: Used in downstream TSDF volume integration (e.g., 0.002) [S1, unofficial].
- `max_depth`: Culling distance for mesh extraction (e.g., 5.0) [S1, unofficial].
- `bg_color`: Background color tensor, typically `[1,1,1]` or `[0,0,0]` [S1, unofficial].

### Edge Cases and Pitfalls
- **Debug Mode**: If `debug=True`, tensors are copied to CPU before calling the C++ rasterizer to prevent corruption during exceptions. Snapshots `snapshot_fw.dump` or `snapshot_bw.dump` are saved [S1, unofficial].

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
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Performance**: Not given in the sources.
- **Strengths**: Differentiable, supports geometric outputs, efficient GPU implementation [S1, unofficial].
- **Limitations**: Requires CUDA, limited to Gaussian representations [S1, unofficial].
- **When to Use**: When geometry-aware training is needed for Gaussian splats [S1, unofficial].
- **When Not to Use**: For non-differentiable rendering or non-Gaussian representations [S1, unofficial].

## Background
The method is implemented in the zju3dv/PGSR repository [S1, unofficial]. It builds on the foundational 3D Gaussian Splatting framework [G].

## Sources
[S1] zju3dv/PGSR, https://github.com/zju3dv/PGSR, fetched 2026-10-27 (unofficial)
