# 3D Gaussian Splatting Differentiable Rasterizer > Backward Pass

Also called: Backward Pass of Gaussian Rasterizer, Gaussian Rasterization (Backward), ToF Gaussian Rasterization Backward Pass

## Overview
The backward pass computes the gradients of the loss function with respect to all learnable input parameters of the 3D Gaussian splatting rasterizer, enabling end-to-end differentiable training. It serves as the computational core for optimizing Gaussian attributes (means, covariances, colors, opacities, scales, rotations, and spherical harmonic coefficients) by propagating gradients from the rendered image back to the 3D scene representation. Unlike the forward pass which renders pixels, this method calculates $\frac{\partial L}{\partial \theta}$ for all $\theta \in \{\mu, \Sigma, c, \alpha, s, q, \mathbf{S}\}$, where $L$ is the reconstruction loss. It builds on [3D/2D Hybrid Gaussian Rasterization with Fallback Sigma](4812-3d-2d-hybrid-gaussian-rasterization-with-fallback-sigma.md) by providing the necessary gradient flow for the hybrid rasterization variant.

## How it works
The backward pass is implemented as a Python wrapper around a C++/CUDA kernel (`_C.rasterize_gaussians_backward`). It retrieves cached tensors from the forward pass context, restructures arguments to match the kernel's expected layout, invokes the native routine, and returns a tuple of gradients matching the forward pass input order.

### Inputs and Outputs
**Inputs:**
1.  **Gradients from downstream loss:**
    *   `grad_out_color`: Tensor of shape $(H, W, C)$ or $(H, W, 3)$, gradient of loss w.r.t. rendered color.
    *   `grad_out_phasor`: Tensor of shape $(H, W, C)$, gradient of loss w.r.t. rendered phasor (if using Time-of-Flight rendering).
    *   `grad_out_depth`: Tensor of shape $(H, W)$, gradient of loss w.r.t. rendered depth.
    *   `grad_out_normal`: Tensor of shape $(H, W, 3)$, gradient of loss w.r.t. rendered normal.
    *   `grad_out_acc`: Tensor of shape $(H, W)$, gradient of loss w.r.t. accumulated opacity.
    *   `grad_entropy`: Tensor of shape $(H, W)$, gradient of loss w.r.t. entropy regularization.
    *   `grad_depth_distortion`: Tensor of shape $(H, W)$, gradient of loss w.r.t. depth distortion.
    *   `grad_amp_distortion`: Tensor of shape $(H, W)$, gradient of loss w.r.t. amplitude distortion.
2.  **Forward Pass Context (Saved Tensors):**
    *   `means3D`: $(N, 3)$, 3D Gaussian means.
    *   `scales`: $(N, 3)$ or $(N, 1)$, Gaussian scales.
    *   `rotations`: $(N, 4)$, Gaussian quaternions.
    *   `cov3Ds_precomp`: $(N, 6)$, Precomputed 3D covariances (symmetric matrix elements).
    *   `radii`: $(N,)$, Integer radii of projected Gaussians (used for culling).
    *   `colors_precomp`: $(N, C)$ or $(N, 3)$, Precomputed colors (if SH degree is low enough to precompute, otherwise `sh` is used).
    *   `phasors_precomp`: $(N, C)$, Precomputed phasors (for ToF).
    *   `sh`: $(N, K, C)$, Spherical Harmonic coefficients.
    *   `sh_p`: $(N, K, C)$, PHASOR SH coefficients (for ToF).
    *   `geomBuffer`: Buffer containing 2D projection info (viewspace points, visibility filter).
    *   `binningBuffer`: Buffer for tile-based binning.
    *   `imgBuffer`: Buffer for image accumulation.
    *   `num_rendered`: Integer, number of Gaussians that contributed to the image.
    *   `phase_offset`: Scalar, phase offset for ToF.
    *   `dc_offset`: Scalar, DC offset for ToF.
3.  **Raster Settings:**
    *   `bg`: $(C)$, Background color.
    *   `viewmatrix`: $(4, 4)$, View transformation matrix.
    *   `projmatrix`: $(4, 4)$, Projection matrix.
    *   `tanfovx`, `tanfovy`: Floats, Tangent of horizontal/vertical field of view.
    *   `campos`: $(3)$, Camera position.
    *   `scale_modifier`: Float, Global scale modifier.
    *   `sh_degree`: Integer, Degree of SH.
    *   `use_view_dependent_phase`: Boolean, Flag for view-dependent phase.
    *   `debug`: Boolean, Debug mode flag.
    *   `near_n`, `far_n`: Floats, Near and far clipping planes.
    *   `depth_range`: Float, Depth range for normalization.

**Outputs:**
A tuple of gradients corresponding to the forward pass inputs:
1.  `grad_means3D`: $(N, 3)$
2.  `grad_means2D`: $(N, 2)$, Gradient w.r.t. 2D projected means (often unused or None if not optimized).
3.  `grad_sh`: $(N, K, C)$
4.  `grad_sh_p`: $(N, K, C)$, Gradient w.r.t. PHASOR SH coefficients.
5.  `grad_colors_precomp`: $(N, C)$
6.  `grad_phasors_precomp`: $(N, C)$
7.  `grad_opacities`: $(N,)$
8.  `grad_scales`: $(N, 3)$ or $(N, 1)$
9.  `grad_rotations`: $(N, 4)$
10. `grad_cov3Ds_precomp`: $(N, 6)$
11. `grad_phase_offset`: Scalar or None
12. `grad_dc_offset`: Scalar or None
13. `None`: For `raster_settings` (not a tensor).

### Data Structures and Layout
The C++ kernel expects arguments in a specific order. The Python wrapper restructures the inputs into a single tuple `args` before calling `_C.rasterize_gaussians_backward`.

**Argument Order for `_C.rasterize_gaussians_backward`:**
1.  `bg`
2.  `means3D`
3.  `radii`
4.  `colors_precomp`
5.  `phasors_precomp`
6.  `scales`
7.  `rotations`
8.  `scale_modifier`
9.  `cov3Ds_precomp`
10. `viewmatrix`
11. `projmatrix`
12. `tanfovx`
13. `tanfovy`
14. `grad_out_color`
15. `grad_out_phasor`
16. `grad_out_depth`
17. `grad_out_normal`
18. `grad_out_acc`
19. `grad_entropy`
20. `grad_depth_distortion`
21. `grad_amp_distortion`
22. `sh`
23. `sh_p`
24. `sh_degree`
25. `campos`
26. `geomBuffer`
27. `num_rendered`
28. `binningBuffer`
29. `imgBuffer`
30. `debug`
31. `near_n`
32. `far_n`
33. `depth_range`
34. `use_view_dependent_phase`
35. `phase_offset`
36. `dc_offset`

### Steps
1.  **Restore Context:** Retrieve all saved tensors and settings from the forward pass context. If `debug` is true, deep-copy all arguments to CPU to allow snapshotting on exception [S1, S2, unofficial].
2.  **Restructure Arguments:** Pack the inputs into the tuple order defined above. Handle optional gradients (e.g., if `grad_out_phasor` is not provided, pass a zero tensor or None depending on kernel implementation; typically, unused gradients are zeroed).
3.  **Invoke Kernel:** Call `_C.rasterize_gaussians_backward(*args)`.
4.  **Construct Return Tuple:** Create a tuple of gradients in the order expected by PyTorch's autograd engine, matching the forward pass input signature.
    *   If `optimize_phase_offset` is False, `grad_phase_offset` is `None`.
    *   If `optimize_dc_offset` is False, `grad_dc_offset` is `None`.
    *   The last element is `None` for `raster_settings`.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def backward(
    grad_out_color,
    grad_out_phasor,
    grad_out_depth,
    grad_out_normal,
    grad_out_acc,
    grad_entropy,
    grad_depth_distortion,
    grad_amp_distortion,
    bg,
    means3D,
    radii,
    colors_precomp,
    phasors_precomp,
    scales,
    rotations,
    scale_modifier,
    cov3Ds_precomp,
    viewmatrix,
    projmatrix,
    tanfovx,
    tanfovy,
    sh,
    sh_p,
    sh_degree,
    campos,
    geomBuffer,
    num_rendered,
    binningBuffer,
    imgBuffer,
    debug,
    near_n,
    far_n,
    depth_range,
    use_view_dependent_phase,
    phase_offset,
    dc_offset,
    optimize_phase_offset,
    optimize_dc_offset
):
    # Step 1: Debug handling
    if debug:
        # Deep copy to CPU for snapshotting on exception
        args_cpu = (
            bg.cpu().clone(),
            means3D.cpu().clone(),
            radii.cpu().clone(),
            colors_precomp.cpu().clone(),
            phasors_precomp.cpu().clone(),
            scales.cpu().clone(),
            rotations.cpu().clone(),
            scale_modifier,
            cov3Ds_precomp.cpu().clone(),
            viewmatrix.cpu().clone(),
            projmatrix.cpu().clone(),
            tanfovx,
            tanfovy,
            grad_out_color.cpu().clone(),
            grad_out_phasor.cpu().clone() if grad_out_phasor is not None else None,
            grad_out_depth.cpu().clone() if grad_out_depth is not None else None,
            grad_out_normal.cpu().clone() if grad_out_normal is not None else None,
            grad_out_acc.cpu().clone() if grad_out_acc is not None else None,
            grad_entropy.cpu().clone() if grad_entropy is not None else None,
            grad_depth_distortion.cpu().clone() if grad_depth_distortion is not None else None,
            grad_amp_distortion.cpu().clone() if grad_amp_distortion is not None else None,
            sh.cpu().clone(),
            sh_p.cpu().clone(),
            sh_degree,
            campos.cpu().clone(),
            geomBuffer,
            num_rendered,
            binningBuffer,
            imgBuffer,
            debug,
            near_n,
            far_n,
            depth_range,
            use_view_dependent_phase,
            phase_offset,
            dc_offset
        )
        try:
            # Step 2: Invoke Kernel
            grads = _C.rasterize_gaussians_backward(*args_cpu)
        except Exception as e:
            # Save snapshot for debugging
            import pickle
            with open('snapshot_bw.dump', 'wb') as f:
                pickle.dump(args_cpu, f)
            print(f"Exception in backward pass: {e}")
            raise e
    else:
        # Step 2: Invoke Kernel (GPU)
        args_gpu = (
            bg,
            means3D,
            radii,
            colors_precomp,
            phasors_precomp,
            scales,
            rotations,
            scale_modifier,
            cov3Ds_precomp,
            viewmatrix,
            projmatrix,
            tanfovx,
            tanfovy,
            grad_out_color,
            grad_out_phasor,
            grad_out_depth,
            grad_out_normal,
            grad_out_acc,
            grad_entropy,
            grad_depth_distortion,
            grad_amp_distortion,
            sh,
            sh_p,
            sh_degree,
            campos,
            geomBuffer,
            num_rendered,
            binningBuffer,
            imgBuffer,
            debug,
            near_n,
            far_n,
            depth_range,
            use_view_dependent_phase,
            phase_offset,
            dc_offset
        )
        grads = _C.rasterize_gaussians_backward(*args_gpu)

    # Step 3: Construct Return Tuple
    # Order must match forward pass inputs
    grad_means3D = grads[0]
    grad_means2D = grads[1]
    grad_sh = grads[2]
    grad_sh_p = grads[3]
    grad_colors_precomp = grads[4]
    grad_phasors_precomp = grads[5]
    grad_opacities = grads[6]
    grad_scales = grads[7]
    grad_rotations = grads[8]
    grad_cov3Ds_precomp = grads[9]
    
    # Handle optional phase/dc offsets
    if optimize_phase_offset:
        grad_phase_offset = grads[10]
    else:
        grad_phase_offset = None
        
    if optimize_dc_offset:
        grad_dc_offset = grads[11]
    else:
        grad_dc_offset = None
        
    # raster_settings is not a tensor
    grad_raster_settings = None

    return (
        grad_means3D,
        grad_means2D,
        grad_sh,
        grad_sh_p,
        grad_colors_precomp,
        grad_phasors_precomp,
        grad_opacities,
        grad_scales,
        grad_rotations,
        grad_cov3Ds_precomp,
        grad_phase_offset,
        grad_dc_offset,
        grad_raster_settings
    )
```

### Edge Cases and Pitfalls
*   **Debug Mode Overhead:** If `debug` is true, all tensors are copied to CPU, which is extremely slow for large scenes. This should only be used for debugging exceptions [S1, S2, unofficial].
*   **Missing Gradients:** If certain output gradients (e.g., `grad_out_phasor`) are not required by the loss function, they should be passed as `None` or zero tensors to the kernel, depending on the kernel's implementation. The kernel must handle these gracefully.
*   **Gradient Order Mismatch:** The order of gradients returned by the C++ kernel must exactly match the order of inputs in the forward pass. Any mismatch will cause incorrect gradient updates [S1, S2, unofficial].
*   **Non-Optimized Parameters:** If `phase_offset` or `dc_offset` are not optimized, their gradients are set to `None` to prevent PyTorch from trying to update non-trainable parameters [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot H \cdot W)$ in the worst case, where $N$ is the number of Gaussians and $H \cdot W$ is the image resolution. However, due to tile-based rasterization and culling, the effective complexity is closer to $O(K \cdot T)$, where $K$ is the number of Gaussians per tile and $T$ is the number of tiles.
*   **Memory Complexity:** $O(N + H \cdot W)$ for storing intermediate buffers (geomBuffer, binningBuffer, imgBuffer) and gradients.
*   **Performance:** The backward pass is typically 1.5x to 2x slower than the forward pass due to the complexity of gradient computation for splatting and blending.
*   **Strengths:** Fully differentiable, supports arbitrary loss functions, efficient tile-based implementation.
*   **Limitations:** High memory usage for large scenes, sensitive to numerical precision in gradient computation.
*   **When to Use:** When training 3D Gaussian Splatting models with differentiable rendering.
*   **When Not to Use:** For inference-only applications where gradients are not needed.

## Background
The backward pass of the 3D Gaussian Splatting rasterizer was introduced as part of the original 3D Gaussian Splatting paper [S1, unofficial] and has been extended in subsequent works such as EGGS [S1, unofficial] and gftorf [S2, unofficial] to support Time-of-Flight (ToF) rendering and other modalities. It is a critical component for optimizing the 3D scene representation from 2D images.

## Sources
[S1] Ingyu-Jeong/EGGS, https://github.com/Ingyu-Jeong/EGGS, fetched 2026-10-26 (unofficial)
[S2] brownvc/gftorf, https://github.com/brownvc/gftorf, fetched 2026-10-27 (unofficial)
