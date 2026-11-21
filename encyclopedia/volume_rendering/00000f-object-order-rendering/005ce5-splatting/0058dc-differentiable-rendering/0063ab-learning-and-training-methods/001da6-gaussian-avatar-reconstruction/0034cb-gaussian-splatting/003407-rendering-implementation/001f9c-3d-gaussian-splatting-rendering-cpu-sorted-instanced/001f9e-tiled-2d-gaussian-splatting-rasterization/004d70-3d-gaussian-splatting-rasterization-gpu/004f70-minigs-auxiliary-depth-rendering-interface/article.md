# MiniGS Auxiliary & Depth Rendering Interface

Also called: MiniGS depth rasterization, MiniGS rasterization auxiliary

## Overview
The MiniGS Auxiliary & Depth Rendering Interface serves as a validation and dispatch layer for MiniGS-specific rasterizer entry points, enabling the return of auxiliary statistics required for blur-aware splitting and depth back-sampling during training [S1, unofficial]. It distinguishes itself from standard rendering by enforcing strict preconditions to ensure the availability of gradient-bearing attributes, thereby supporting the structural optimization phases of Gaussian Avatar Reconstruction rather than final inference [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**:
    *   `means3D`: 3D Gaussian centers.
    *   `means2D`: Projected 2D Gaussian centers.
    *   `dc`: Diffuse color tensor.
    *   `shs`: Spherical harmonics coefficients tensor.
    *   `opacities`: Gaussian opacity tensor.
    *   `scales`: Gaussian scale tensor.
    *   `rotations`: Gaussian rotation tensor.
    *   `raster_settings`: Configuration object containing `inference_only` (bool) and `pixel_weights` (tensor).
*   **Outputs**:
    *   Returns the result of either `minigs_rasterize_gaussians_aux` or `minigs_rasterize_gaussians_depth`, depending on the internal logic of the dispatched function (typically determined by the specific MiniGS variant or settings not fully detailed in the source, but the interface ensures one of these two is called) [S1, unofficial].

**Data Structures**
*   `raster_settings`:
    *   `inference_only`: Boolean flag. Must be `False` for this interface to function.
    *   `pixel_weights`: Tensor. Must be `None` or empty.

**Numbered Steps**
1.  **Validate Inference Mode**: Check if `raster_settings.inference_only` is `True`. If so, raise an exception, as this interface does not support inference-only mode [S1, unofficial].
2.  **Validate Pixel Weights**: Check if `raster_settings.pixel_weights` is not `None` and has a non-zero number of elements (`numel() > 0`). If so, raise an exception, as pixel weights are not supported in this path [S1, unofficial].
3.  **Validate Attribute Tensors**: Ensure that `dc`, `shs`, `scales`, and `rotations` are not `None`. These attributes are required for the auxiliary and depth rasterization processes [S1, unofficial].
4.  **Dispatch**: Call `minigs_rasterize_gaussians_aux` with the arguments `means3D`, `means2D`, `dc`, `shs`, `opacities`, `scales`, `rotations`, and `raster_settings` [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def minigs_auxiliary_depth_interface(
    means3D, means2D, dc, shs, opacities, scales, rotations, raster_settings
):
    # Step 1: Check inference_only
    if raster_settings.inference_only:
        raise Exception("MiniGS auxiliary rendering does not support inference_only=True.")
    
    # Step 2: Check pixel_weights
    if raster_settings.pixel_weights is not None and raster_settings.pixel_weights.numel() > 0:
        raise Exception("MiniGS auxiliary rendering does not support pixel_weights.")
    
    # Step 3 & 4: Validate attributes and dispatch
    # Note: The source code snippet directly dispatches to minigs_rasterize_gaussians_aux
    # without explicit None checks for dc, shs, scales, rotations in the provided snippet,
    # but the notes state these must not be None. We assume the dispatched function
    # or a preceding layer handles the None check, or we add it for safety [G].
    if dc is None or shs is None or scales is None or rotations is None:
        raise ValueError("dc, shs, scales, and rotations must not be None.")

    return minigs_rasterize_gaussians_aux(
        means3D, means2D, dc, shs, opacities, scales, rotations, raster_settings
    )
```

**Parameters**
*   `inference_only`: `bool`. Must be `False`.
*   `pixel_weights`: `tensor`. Must be `None` or empty.
*   `dc`: `tensor`. Diffuse color.
*   `shs`: `tensor`. Spherical harmonics coefficients.
*   `scales`: `tensor`. Gaussian scales.
*   `rotations`: `tensor`. Gaussian rotations.

**Edge Cases and Pitfalls**
*   **Inference Mode**: The interface explicitly fails if `inference_only=True` [S1, unofficial].
*   **Pixel Weights**: The interface explicitly fails if `pixel_weights` are provided and non-empty [S1, unofficial].
*   **Missing Attributes**: The method requires all five Gaussian attribute tensors (`dc`, `shs`, `scales`, `rotations`, `opacities`) to be present; failure to provide them results in errors [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
The MiniGS Auxiliary & Depth Rendering Interface is part of the MiniGS (Mini Gaussian Splatting) implementation, specifically designed to support training-time operations such as blur-aware splitting and depth back-sampling. It is utilized in the context of Gaussian Avatar Reconstruction and is integrated into the training loop of the Improved-GS project [S1, unofficial].

## Sources
[S1] XiaoBin2001/Improved-GS, https://github.com/XiaoBin2001/Improved-GS, fetched 2026-10-27 (unofficial)
