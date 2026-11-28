# PyTorch Rasterization Wrapper

Also called: GaussianRasterizer Wrapper, PyTorch Autograd Rasterizer Wrapper

## Overview
The PyTorch Rasterization Wrapper is a `torch.autograd.Function` that bridges Python and a C++ rasterization kernel for 3D Gaussian Splatting [S1, unofficial]. It handles the construction of argument tuples for the forward and backward passes, manages buffer persistence, and optionally supports debug mode by copying arguments to the CPU [S1, unofficial]. It serves as the Python-facing interface for the underlying GPU-accelerated rasterizer [S1, unofficial].

## How it works
**Inputs:**
- `means3D`: Tensor of shape $(N, 3)$, 3D Gaussian means.
- `opacities`: Tensor of shape $(N, 1)$, Gaussian opacities.
- `scales`: Tensor of shape $(N, 3)$, Gaussian scales.
- `rotations`: Tensor of shape $(N, 4)$, Gaussian rotations (quaternions).
- `raster_settings`: Object containing `scale_modifier`, `cov3Ds_precomp`, `viewmatrix`, `projmatrix`, `tanfovx`, `tanfovy`, `image_height`, `image_width`, `campos`, `prefiltered`, `mode`, and `debug` flags.
- `grad_out_color`: Tensor of shape $(H, W, 3)$, gradient of the output color image (used in backward pass).

**Outputs:**
- `color`: Tensor of shape $(H, W, 3)$, rendered color image.
- `radii`: Tensor of shape $(N)$, radii of projected Gaussians (used to identify visible Gaussians).

**Data Structures:**
- `geomBuffer`, `binningBuffer`, `imgBuffer`: Internal buffers managed by the C++ kernel. These are saved in `ctx` for the backward pass.

**Steps:**

1.  **Forward Pass Argument Construction:**
    Construct the `args` tuple for the C++ forward rasterizer:
    ```python
    args = (
        means3D, opacities, scales, rotations,
        raster_settings.scale_modifier, cov3Ds_precomp,
        raster_settings.viewmatrix, raster_settings.projmatrix,
        raster_settings.tanfovx, raster_settings.tanfovy,
        raster_settings.image_height, raster_settings.image_width,
        raster_settings.campos, raster_settings.prefiltered,
        raster_settings.mode, raster_settings.debug
    )
    ```

2.  **Debug Mode Handling:**
    If `raster_settings.debug` is true, perform a deep copy of `args` to CPU tensors for inspection [S1, unofficial].

3.  **C++ Forward Rasterization:**
    Call the C++ rasterizer function `_C.rasterize_gaussians(*args)` [S1, unofficial].
    ```python
    num_rendered, color, radii, geomBuffer, binningBuffer, imgBuffer = _C.rasterize_gaussians(*args)
    ```

4.  **Save for Backward:**
    Save the necessary tensors and buffers in `ctx` for the backward pass [S1, unofficial]:
    ```python
    ctx.save_for_backward(means3D, scales, rotations, cov3Ds_precomp, radii, geomBuffer, binningBuffer, imgBuffer)
    ```
    Store `raster_settings` and `num_rendered` in `ctx` as well (implied by backward usage).

5.  **Backward Pass Argument Construction:**
    Restore saved tensors from `ctx`. Construct the `args` tuple for the C++ backward rasterizer:
    ```python
    args_backward = (
        means3D, radii, scales, rotations,
        raster_settings.scale_modifier, cov3Ds_precomp,
        raster_settings.viewmatrix, raster_settings.projmatrix,
        raster_settings.tanfovx, raster_settings.tanfovy,
        grad_out_color, raster_settings.campos,
        geomBuffer, num_rendered, binningBuffer, imgBuffer,
        raster_settings.mode, raster_settings.debug
    )
    ```

6.  **C++ Backward Rasterization:**
    Call the C++ backward rasterizer function `_C.rasterize_gaussians_backward(*args_backward)` [S1, unofficial]:
    ```python
    (grad_means2D, grad_opacities, _, grad_means3D, grad_cov3Ds_precomp, grad_scales, grad_rotations) = _C.rasterize_gaussians_backward(*args_backward)
    ```

7.  **Return Gradients:**
    Return the gradients corresponding to the inputs. Note that `grad_opacities` and `grad_means2D` are returned, but `grad_opacities` is often not used in standard training pipelines if opacities are not being optimized or if a simplified gradient flow is assumed. The `_` corresponds to an unused gradient (likely related to `cov3Ds_precomp` or an intermediate buffer depending on the C++ signature).

**Edge Cases and Pitfalls:**
- **Debug Mode:** Copying arguments to CPU can be slow for large $N$. Only enable for debugging [S1, unofficial].
- **Buffer Management:** The buffers (`geomBuffer`, etc.) must be preserved exactly as returned by the forward pass for the backward pass to work correctly. Modifying them or reusing them across different forward calls without proper handling will cause errors.
- **Tensor Types:** Ensure all input tensors are on the same device (typically CUDA) as expected by the C++ extension.

## Operating characteristics
- **Time Complexity:** Dominated by the C++ rasterization kernel. The wrapper itself adds minimal overhead (tuple construction, function calls).
- **Memory Complexity:** Requires storing intermediate buffers (`geomBuffer`, `binningBuffer`, `imgBuffer`) and saved tensors for the backward pass.
- **Strengths:** Provides automatic differentiation for the rasterization process, enabling end-to-end training of 3D Gaussian parameters.
- **Limitations:** Relies on a compiled C++ extension. Debug mode can significantly slow down execution.

## Background
The PyTorch Rasterization Wrapper is part of the `r2_gaussian` project, which implements 3D Gaussian Splatting for medical imaging (CT reconstruction) [S1, unofficial]. It adapts the original 3D Gaussian Splatting rendering pipeline to work within a PyTorch training loop for gradient-based optimization.

## Sources
[S1] Ruyi-Zha/r2_gaussian, https://github.com/Ruyi-Zha/r2_gaussian, fetched 2026-10-26 (unofficial)
