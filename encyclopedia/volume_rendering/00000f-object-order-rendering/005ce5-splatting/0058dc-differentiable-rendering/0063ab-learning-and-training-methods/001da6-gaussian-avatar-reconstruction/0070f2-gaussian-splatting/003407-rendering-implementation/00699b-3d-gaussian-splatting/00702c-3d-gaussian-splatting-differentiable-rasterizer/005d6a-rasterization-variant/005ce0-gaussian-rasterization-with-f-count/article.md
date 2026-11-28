# Gaussian Rasterization with F-Count

Also called: LightGaussian rasterization

## Overview
Gaussian Rasterization with F-Count is a variant of the 3D Gaussian Splatting differentiable rasterizer that computes an additional output: the number of times each Gaussian is activated (rendered) during the forward pass, alongside the standard color and radii outputs [S1, unofficial]. It sets itself apart from the standard rasterizer by exposing per-Gaussian rendering statistics, which are used in downstream optimization pipelines (such as LightGaussian) for importance scoring and pruning, whereas the standard variant only returns pixel colors and hit radii [S1, unofficial].

## How it works
The method modifies the standard 3D Gaussian rasterization pipeline by introducing a boolean flag `f_count` in the rasterization settings. When enabled, the Python dispatcher invokes a specialized C++/CUDA backend function (`_C.count_gaussians`) instead of the standard `_C.rasterize_gaussians`. This backend function performs the standard projection, sorting, and alpha compositing steps defined in the parent [Rasterization Variant](17388-rasterization-variant.md) but additionally accumulates a count of active Gaussians per pixel or per Gaussian instance, depending on the internal implementation details of the specific fork [S1, unofficial].

### Inputs and Outputs
**Inputs:**
The forward pass accepts a tuple of arguments constructed in a specific order. The inputs are:
1.  `raster_settings.bg`: Background color tensor.
2.  `means3D`: 3D Gaussian means tensor.
3.  `colors_precomp`: Precomputed colors tensor (or `None` if SHs are used).
4.  `opacities`: Gaussian opacities tensor.
5.  `scales`: Gaussian scales tensor (or `None` if cov3Ds are used).
6.  `rotations`: Gaussian rotations tensor (or `None` if cov3Ds are used).
7.  `raster_settings.scale_modifier`: Scale modifier float.
8.  `cov3Ds_precomp`: Precomputed 3D covariances tensor (or `None` if scales/rotations are used).
9.  `raster_settings.viewmatrix`: View matrix tensor.
10. `raster_settings.projmatrix`: Projection matrix tensor.
11. `raster_settings.tanfovx`: Tangent of horizontal field of view.
12. `raster_settings.tanfovy`: Tangent of vertical field of view.
13. `raster_settings.image_height`: Image height integer.
14. `raster_settings.image_width`: Image width integer.
15. `sh`: Spherical Harmonics coefficients tensor (or `None` if colors_precomp are used).
16. `raster_settings.sh_degree`: SH degree integer.
17. `raster_settings.campos`: Camera position tensor.
18. `raster_settings.prefiltered`: Prefiltered boolean.
19. `raster_settings.debug`: Debug boolean.
20. `raster_settings.f_count`: Boolean flag enabling F-Count mode.

**Outputs:**
If `f_count` is true, the forward pass returns a tuple:
1.  `gaussians_count`: Tensor containing the count of activated Gaussians.
2.  `important_score`: Tensor containing importance scores derived from the counts.
3.  `color`: Rendered color image tensor.
4.  `radii`: Hit radii tensor.

If `f_count` is false, it returns the standard tuple: `(color, radii)`.

### Data Structures
**GaussianRasterizationSettings:**
A `NamedTuple` containing the following fields:
*   `image_height`: `int`
*   `image_width`: `int`
*   `tanfovx`: `float`
*   `tanfovy`: `float`
*   `bg`: `torch.Tensor`
*   `scale_modifier`: `float`
*   `viewmatrix`: `torch.Tensor`
*   `projmatrix`: `torch.Tensor`
*   `sh_degree`: `int`
*   `campos`: `torch.Tensor`
*   `prefiltered`: `bool`
*   `debug`: `bool`
*   `f_count`: `bool`

### Numbered Steps

1.  **Input Validation:**
    *   Verify that exactly one of `shs` or `colors_precomp` is provided. If both are `None` or both are not `None`, raise an exception [S1, unofficial].
    *   Verify that exactly one of the following pairs is provided:
        *   `scales` and `rotations` (both not `None`).
        *   `cov3D_precomp` (not `None`).
        *   If `(scales is None or rotations is None)` AND `cov3D_precomp is None`, raise an exception.
        *   If `(scales is not None or rotations is not None)` AND `cov3D_precomp is not None`, raise an exception [S1, unofficial].

2.  **Argument Construction:**
    *   Construct the `args` tuple for the C++ backend in the specific order listed in "Inputs and Outputs" above.
    *   Ensure `raster_settings.f_count` is the 20th element of this tuple [S1, unofficial].

3.  **Dispatch:**
    *   Check the value of `raster_settings.f_count`.
    *   If `True`, call `_C.count_gaussians(*args)`.
    *   If `False`, call `_C.rasterize_gaussians(*args)` [S1, unofficial].

4.  **Execution and Return:**
    *   The C++ backend performs the standard rasterization steps (projection, sorting, compositing) as defined in the parent article [17388-rasterization-variant.md].
    *   If `f_count` is true, the backend additionally computes `gaussians_count` and `important_score`.
    *   Return the appropriate tuple based on the mode [S1, unofficial].

5.  **Backward Pass:**
    *   Construct the backward `args` tuple in this specific order:
        1.  `raster_settings.bg`
        2.  `means3D`
        3.  `radii`
        4.  `colors_precomp`
        5.  `scales`
        6.  `rotations`
        7.  `raster_settings.scale_modifier`
        8.  `cov3Ds_precomp`
        9.  `raster_settings.viewmatrix`
        10. `raster_settings.projmatrix`
        11. `raster_settings.tanfovx`
        12. `raster_settings.tanfovy`
        13. `grad_out_color`
        14. `sh`
        15. `raster_settings.sh_degree`
        16. `raster_settings.campos`
        17. `geomBuffer`
        18. `num_rendered`
        19. `binningBuffer`
        20. `imgBuffer`
        21. `raster_settings.debug`
    *   Call `_C.rasterize_gaussians_backward(*args)` [S1, unofficial].
    *   The backward function returns: `(grad_means2D, grad_colors_precomp, grad_opacities, grad_means3D, grad_cov3Ds_precomp, grad_sh, grad_scales, grad_rotations)`.
    *   Return the gradients corresponding to the forward inputs: `(grad_means3D, grad_means2D, grad_sh, grad_colors_precomp, grad_opacities, grad_scales, grad_rotations, grad_cov3Ds_precomp, None)`. The last `None` corresponds to the non-differentiable `raster_settings` argument [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import copy

# Assuming _C is the imported C++/CUDA extension module
# Assuming GaussianRasterizationSettings is the NamedTuple defined in the source

def gaussian_rasterization_with_f_count(
    means3D: torch.Tensor,
    colors_precomp: torch.Tensor,
    opacities: torch.Tensor,
    scales: torch.Tensor,
    rotations: torch.Tensor,
    cov3Ds_precomp: torch.Tensor,
    sh: torch.Tensor,
    raster_settings: GaussianRasterizationSettings
) -> tuple:
    """
    Differentiable rasterization of 3D Gaussians with optional F-Count output.
    
    Args:
        means3D: Tensor of 3D Gaussian means.
        colors_precomp: Tensor of precomputed colors (or None).
        opacities: Tensor of Gaussian opacities.
        scales: Tensor of Gaussian scales (or None).
        rotations: Tensor of Gaussian rotations (or None).
        cov3Ds_precomp: Tensor of precomputed 3D covariances (or None).
        sh: Tensor of Spherical Harmonics coefficients (or None).
        raster_settings: GaussianRasterizationSettings NamedTuple.
        
    Returns:
        Tuple of tensors: (gaussians_count, important_score, color, radii) if f_count is True.
        Tuple of tensors: (color, radii) if f_count is False.
    """
    
    # Input Validation
    if colors_precomp is None and sh is None:
        raise ValueError("Either colors_precomp or sh must be provided.")
    if colors_precomp is not None and sh is not None:
        raise ValueError("Only one of colors_precomp or sh can be provided.")
        
    if (scales is None or rotations is None) and cov3Ds_precomp is None:
        raise ValueError("Either (scales and rotations) or cov3Ds_precomp must be provided.")
    if (scales is not None or rotations is not None) and cov3Ds_precomp is not None:
        raise ValueError("Only one of (scales and rotations) or cov3Ds_precomp can be provided.")

    # Construct Forward Arguments
    # Order: bg, means3D, colors_precomp, opacities, scales, rotations, scale_modifier, 
    # cov3Ds_precomp, viewmatrix, projmatrix, tanfovx, tanfovy, image_height, image_width, 
    # sh, sh_degree, campos, prefiltered, debug, f_count
    args = (
        raster_settings.bg,
        means3D,
        colors_precomp,
        opacities,
        scales,
        rotations,
        raster_settings.scale_modifier,
        cov3Ds_precomp,
        raster_settings.viewmatrix,
        raster_settings.projmatrix,
        raster_settings.tanfovx,
        raster_settings.tanfovy,
        raster_settings.image_height,
        raster_settings.image_width,
        sh,
        raster_settings.sh_degree,
        raster_settings.campos,
        raster_settings.prefiltered,
        raster_settings.debug,
        raster_settings.f_count
    )

    try:
        if raster_settings.f_count:
            # Call specialized count function
            # Returns: (gaussians_count, important_score, num_rendered, color, radii, geomBuffer, binningBuffer, imgBuffer)
            forward_output = _C.count_gaussians(*args)
            
            # Extract relevant outputs
            gaussians_count = forward_output[0]
            important_score = forward_output[1]
            color = forward_output[3]
            radii = forward_output[4]
            
            return (gaussians_count, important_score, color, radii)
        else:
            # Call standard rasterization function
            # Returns: (num_rendered, important_score/None, num_rendered, color, radii, geomBuffer, binningBuffer, imgBuffer)
            forward_output = _C.rasterize_gaussians(*args)
            
            color = forward_output[3]
            radii = forward_output[4]
            
            return (color, radii)
            
    except Exception as e:
        if raster_settings.debug:
            # Deep copy args to CPU and save for debugging
            args_cpu = copy.deepcopy(args)
            torch.save(args_cpu, 'snapshot_fw.dump')
        raise e

def gaussian_rasterization_with_f_count_backward(
    grad_out_color: torch.Tensor,
    means3D: torch.Tensor,
    radii: torch.Tensor,
    colors_precomp: torch.Tensor,
    scales: torch.Tensor,
    rotations: torch.Tensor,
    cov3Ds_precomp: torch.Tensor,
    sh: torch.Tensor,
    raster_settings: GaussianRasterizationSettings,
    geomBuffer,
    num_rendered,
    binningBuffer,
    imgBuffer
) -> tuple:
    """
    Backward pass for Gaussian rasterization with F-Count.
    
    Args:
        grad_out_color: Gradient of the loss with respect to the output color.
        means3D: 3D Gaussian means.
        radii: Hit radii from forward pass.
        colors_precomp: Precomputed colors.
        scales: Gaussian scales.
        rotations: Gaussian rotations.
        cov3Ds_precomp: Precomputed 3D covariances.
        sh: Spherical Harmonics coefficients.
        raster_settings: GaussianRasterizationSettings NamedTuple.
        geomBuffer: Geometry buffer from forward pass.
        num_rendered: Number of rendered Gaussians from forward pass.
        binningBuffer: Binning buffer from forward pass.
        imgBuffer: Image buffer from forward pass.
        
    Returns:
        Tuple of gradients: (grad_means3D, grad_means2D, grad_sh, grad_colors_precomp, 
        grad_opacities, grad_scales, grad_rotations, grad_cov3Ds_precomp, None)
    """
    
    # Construct Backward Arguments
    # Order: bg, means3D, radii, colors_precomp, scales, rotations, scale_modifier, 
    # cov3Ds_precomp, viewmatrix, projmatrix, tanfovx, tanfovy, grad_out_color, 
    # sh, sh_degree, campos, geomBuffer, num_rendered, binningBuffer, imgBuffer, debug
    args_bw = (
        raster_settings.bg,
        means3D,
        radii,
        colors_precomp,
        scales,
        rotations,
        raster_settings.scale_modifier,
        cov3Ds_precomp,
        raster_settings.viewmatrix,
        raster_settings.projmatrix,
        raster_settings.tanfovx,
        raster_settings.tanfovy,
        grad_out_color,
        sh,
        raster_settings.sh_degree,
        raster_settings.campos,
        geomBuffer,
        num_rendered,
        binningBuffer,
        imgBuffer,
        raster_settings.debug
    )

    try:
        # Call backward function
        # Returns: (grad_means2D, grad_colors_precomp, grad_opacities, grad_means3D, 
        # grad_cov3Ds_precomp, grad_sh, grad_scales, grad_rotations)
        grad_output = _C.rasterize_gaussians_backward(*args_bw)
        
        grad_means2D = grad_output[0]
        grad_colors_precomp = grad_output[1]
        grad_opacities = grad_output[2]
        grad_means3D = grad_output[3]
        grad_cov3Ds_precomp = grad_output[4]
        grad_sh = grad_output[5]
        grad_scales = grad_output[6]
        grad_rotations = grad_output[7]
        
        # Return gradients in the order expected by the autograd engine for the forward inputs
        # Forward inputs: means3D, colors_precomp, opacities, scales, rotations, cov3Ds_precomp, sh, raster_settings
        return (
            grad_means3D,
            grad_means2D,
            grad_sh,
            grad_colors_precomp,
            grad_opacities,
            grad_scales,
            grad_rotations,
            grad_cov3Ds_precomp,
            None
        )
        
    except Exception as e:
        if raster_settings.debug:
            args_bw_cpu = copy.deepcopy(args_bw)
            torch.save(args_bw_cpu, 'snapshot_bw.dump')
        raise e
```

### Parameters
*   `f_count`: A boolean flag in `GaussianRasterizationSettings`. When `True`, the rasterizer invokes `_C.count_gaussians` and returns `gaussians_count` and `important_score`. When `False`, it invokes `_C.rasterize_gaussians` and returns only `color` and `radii` [S1, unofficial].

### Edge Cases and Pitfalls
*   **Input Validation:** The forward pass raises an exception if both `shs` and `colors_precomp` are `None`, or if both are not `None`. It must receive exactly one of either [S1, unofficial].
*   **Covariance Input:** The forward pass raises an exception if `(scales is None or rotations is None)` AND `cov3D_precomp is None`, OR if `(scales is not None or rotations is not None)` AND `cov3D_precomp is not None`. It must receive exactly one of either scale/rotation pair or precomputed 3D covariance [S1, unofficial].
*   **Debugging Snapshots:** If an exception occurs during the forward pass and `raster_settings.debug` is `True`, the arguments are deep-copied to CPU and saved to `snapshot_fw.dump` before re-raising the exception. Similarly, backward pass exceptions save to `snapshot_bw.dump` [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The time complexity is dominated by the standard 3D Gaussian rasterization steps (projection, sorting, compositing) as defined in the parent article [17388-rasterization-variant.md]. The additional computation for `f_count` is minimal, involving simple increment operations during the compositing loop.
*   **Memory Complexity:** The memory complexity is similar to the standard rasterizer, with the addition of storing the `gaussians_count` and `important_score` tensors. The buffers (`geomBuffer`, `binningBuffer`, `imgBuffer`) are reused from the forward pass [S1, unofficial].
*   **Performance:** The specific performance impact of `f_count` is not quantified in the sources beyond the fact that it is a lightweight addition to the forward pass [S1, unofficial].
*   **Strengths:** Enables importance scoring and pruning strategies (e.g., in LightGaussian) by providing per-Gaussian rendering statistics [S1, unofficial].
*   **Limitations:** Only available in specific forks/implementations (e.g., Fov-3DGS) and not part of the core 3DGS standard [S1, unofficial].

## Background
This variant is associated with the "LightGaussian" approach and is implemented in the `Fov-3DGS` repository by Horizon Research [S1, unofficial]. It is used in training pipelines where Gaussian pruning or importance-based optimization is required, allowing the system to track how often each Gaussian contributes to the final image [S1, unofficial].

## Sources
[S1] horizon-research/Fov-3DGS, https://github.com/horizon-research/Fov-3DGS, fetched 2026-10-27 (unofficial)
