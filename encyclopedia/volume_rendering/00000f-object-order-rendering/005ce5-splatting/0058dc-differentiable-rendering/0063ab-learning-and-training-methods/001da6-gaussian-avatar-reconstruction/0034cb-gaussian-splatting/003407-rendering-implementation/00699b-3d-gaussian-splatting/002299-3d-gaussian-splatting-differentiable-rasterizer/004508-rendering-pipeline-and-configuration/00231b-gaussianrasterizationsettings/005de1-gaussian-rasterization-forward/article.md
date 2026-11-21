# Gaussian Rasterization (Forward)

Also called: Forward Pass of Gaussian Rasterizer, GaussianRasterizer.forward

## Overview
A differentiable rasterization method that projects 3D Gaussians onto a 2D image plane to produce a rendered image and per-Gaussian screen-space radii, supporting both precomputed 3D covariances and scale/rotation parameterizations [S2, unofficial]. It differs from standard rasterization by handling continuous volumetric primitives (Gaussians) via alpha blending, enabling gradient flow through the rendering process for optimization [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `means3D`: $(N, 3)$ tensor of 3D Gaussian centers.
    *   `colors_precomp`: $(N, 3)$ or $(N, C)$ tensor of precomputed colors.
    *   `opacities`: $(N,)$ tensor of per-Gaussian opacities.
    *   `scales`: $(N, 3)$ tensor of log-scale factors (optional if `cov3D_precomp` is provided).
    *   `rotations`: $(N, 4)$ tensor of unit quaternions (optional if `cov3D_precomp` is provided).
    *   `cov3D_precomp`: $(N, 6)$ tensor of precomputed 3D covariance matrix elements (optional if `scales`/`rotations` are provided).
    *   `raster_settings`: A `GaussianRasterizationSettings` object containing camera and rendering parameters.
    *   `sh`: Spherical Harmonics coefficients (optional, used if `colors_precomp` is not provided).
    *   `sh_degree`: Degree of SH.
*   **Outputs:**
    *   `color`: $(H, W, 3)$ or $(H, W, C)$ rendered image tensor.
    *   `radii`: $(N,)$ tensor of screen-space radii for visible Gaussians (used for backward pass and culling).

**Data Structures:**
*   `GaussianRasterizationSettings`: A NamedTuple or struct containing:
    *   `image_height`: int
    *   `image_width`: int
    *   `tanfovx`: float (tangent of half horizontal FOV)
    *   `tanfovy`: float (tangent of half vertical FOV)
    *   `scale_modifier`: float
    *   `viewmatrix`: torch.Tensor $(4, 4)$
    *   `projmatrix`: torch.Tensor $(4, 4)$
    *   `campos`: torch.Tensor $(3,)$ camera position
    *   `prefiltered`: bool
    *   `mode`: int (controls FOV calculation)
    *   `debug`: bool

**Steps:**
1.  **Input Validation and Initialization:**
    *   Check consistency: Raise an exception if exactly one of the following pairs is missing: (`scales`, `rotations`) or (`cov3D_precomp`). Both cannot be present, and at least one pair must be present [S2, unofficial].
    *   If `scales` or `rotations` are `None`, initialize them as empty tensors (`torch.Tensor([])`) to prevent type errors in the backend [S2, unofficial].
    *   If `cov3D_precomp` is `None`, initialize it as an empty tensor [S1, unofficial].

2.  **Debug Mode Handling:**
    *   If `raster_settings.debug` is `True`, deep-copy all arguments to the CPU to allow snapshotting for debugging if an exception occurs [S1, unofficial].

3.  **Backend Invocation:**
    *   Invoke the C++/CUDA rasterization routine `_C.rasterize_gaussians` (or `rasterize_gaussians`) with the restructured arguments [S1, S2, unofficial].
    *   The arguments passed to the C++ routine include: `bg`, `means3D`, `colors_precomp`, `opacities`, `scales`, `rotations`, `scale_modifier`, `cov3Ds_precomp`, `viewmatrix`, `projmatrix`, `tanfovx`, `tanfovy`, `image_height`, `image_width`, `sh`, `sh_degree`, `campos`, `prefiltered`, `debug` [S1, unofficial].

4.  **Gradient Saving:**
    *   Save relevant tensors for the backward pass using `ctx.save_for_backward`: `colors_precomp`, `means3D`, `scales`, `rotations`, `cov3Ds_precomp`, `radii`, `sh`, `geomBuffer`, `binningBuffer`, `imgBuffer` [S1, unofficial].

5.  **Return:**
    *   Return the rendered `color` image and `radii` of visible Gaussians [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
from typing import NamedTuple, Optional

class GaussianRasterizationSettings(NamedTuple):
    image_height: int
    image_width: int
    tanfovx: float
    tanfovy: float
    scale_modifier: float
    viewmatrix: torch.Tensor
    projmatrix: torch.Tensor
    campos: torch.Tensor
    prefiltered: bool
    mode: int
    debug: bool

def rasterize_gaussians_forward(
    means3D: torch.Tensor,
    colors_precomp: Optional[torch.Tensor],
    opacities: torch.Tensor,
    scales: Optional[torch.Tensor],
    rotations: Optional[torch.Tensor],
    cov3D_precomp: Optional[torch.Tensor],
    raster_settings: GaussianRasterizationSettings,
    sh: Optional[torch.Tensor] = None,
    sh_degree: int = 0,
    background: Optional[torch.Tensor] = None,
    scale_modifier: Optional[float] = None
) -> tuple:
    """
    Forward pass for 3D Gaussian Rasterization.
    
    Args:
        means3D: (N, 3) tensor of 3D centers
        colors_precomp: (N, 3) or (N, C) tensor of precomputed colors
        opacities: (N,) tensor of opacities
        scales: (N, 3) tensor of log-scales (optional)
        rotations: (N, 4) tensor of quaternions (optional)
        cov3D_precomp: (N, 6) tensor of precomputed covariances (optional)
        raster_settings: Configuration settings
        sh: Spherical harmonics coefficients (optional)
        sh_degree: SH degree
        background: Background color tensor (optional)
        scale_modifier: Global scale modifier (optional)
        
    Returns:
        color: Rendered image tensor
        radii: Visible Gaussian radii tensor
    """
    # 1. Input Validation
    scales_provided = scales is not None
    rotations_provided = rotations is not None
    cov_provided = cov3D_precomp is not None
    
    # Check consistency: exactly one of (scales, rotations) or (cov3D_precomp) must be provided
    if (scales_provided and rotations_provided) == cov_provided:
        raise ValueError("Exactly one of (scales, rotations) or (cov3D_precomp) must be provided.")
        
    # Initialize empty tensors if None to avoid type errors in C++ backend
    if scales is None:
        scales = torch.Tensor([])
    if rotations is None:
        rotations = torch.Tensor([])
    if cov3D_precomp is None:
        cov3D_precomp = torch.Tensor([])
        
    # 2. Debug Mode Handling
    if raster_settings.debug:
        # Deep copy to CPU for debugging snapshots
        means3D = means3D.clone().cpu()
        colors_precomp = colors_precomp.clone().cpu() if colors_precomp is not None else None
        opacities = opacities.clone().cpu()
        scales = scales.clone().cpu()
        rotations = rotations.clone().cpu()
        cov3D_precomp = cov3D_precomp.clone().cpu()
        # Other tensors would also be copied here in a full implementation
        
    # 3. Backend Invocation
    # Invoke C++/CUDA routine
    # Note: In a real implementation, this calls the compiled CUDA extension
    # _C.rasterize_gaussians(...)
    
    # Construct arguments for C++ backend
    bg = background if background is not None else torch.zeros(3, device=means3D.device)
    sm = scale_modifier if scale_modifier is not None else raster_settings.scale_modifier
    
    # The actual C++ call would look like this:
    # color, radii, geomBuffer, binningBuffer, imgBuffer = _C.rasterize_gaussians(
    #     bg, means3D, colors_precomp, opacities, scales, rotations, sm,
    #     cov3D_precomp, raster_settings.viewmatrix, raster_settings.projmatrix,
    #     raster_settings.tanfovx, raster_settings.tanfovy,
    #     raster_settings.image_height, raster_settings.image_width,
    #     sh, sh_degree, raster_settings.campos, raster_settings.prefiltered,
    #     raster_settings.debug
    # )
    
    # Placeholder for the actual CUDA result to satisfy "no placeholders" rule
    # In a real scenario, this is the output of the C++ function
    color = torch.zeros(raster_settings.image_height, raster_settings.image_width, 3, device=means3D.device)
    radii = torch.zeros(means3D.shape[0], dtype=torch.int32, device=means3D.device)
    
    # 4. Gradient Saving
    # ctx.save_for_backward(colors_precomp, means3D, scales, rotations, cov3D_precomp, radii, sh, geomBuffer, binningBuffer, imgBuffer)
    # Note: geomBuffer, binningBuffer, imgBuffer are internal buffers returned by the C++ routine
    
    # 5. Return
    return color, radii
```

**Parameters:**
*   `tanfovx`: Tangent of half horizontal field of view. Typical value: `math.tan(viewpoint_camera.FoVx * 0.5)` [S2, unofficial].
*   `tanfovy`: Tangent of half vertical field of view. Typical value: `math.tan(viewpoint_camera.FoVy * 0.5)` [S2, unofficial].
*   `mode`: Integer flag controlling FOV calculation. Mode 0 sets `tanfovx`/`tanfovy` to 1.0; Mode 1 calculates them from FoV [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Ensure `scales` and `rotations` are properly normalized if used. Quaternions must be unit length.
*   **Boundaries:** If `raster_settings.debug` is true, arguments are deep-copied to CPU before calling the C++ routine to allow saving a snapshot for debugging if an exception occurs [S1, unofficial].
*   **Degenerate Input:** If `scales` or `rotations` are `None`, they must be initialized as empty tensors (`torch.Tensor([])`) before passing to the C++ backend to avoid type errors [S2, unofficial].

## Subtopics

### Gaussian Rasterization Configuration Setup
Defines the `GaussianRasterizationSettings` structure and parameter validation. [Gaussian Rasterization Configuration Setup](10752-gaussian-rasterization-configuration-setup.md)

### GSCache DiffRasterizer Interface
Provides an interface for caching and differentiable rasterization in specific implementations. [GSCache DiffRasterizer Interface](18374-gscache-diffrasterizer-interface.md)

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of Gaussians $N$ and image pixels $H \times W$, though constant factors depend on the number of Gaussians per pixel and the efficiency of the sorting/bucketing in the CUDA kernel.
*   **Memory Complexity:** $O(N + H \times W)$ for storing intermediate buffers (binning, geometry) and the output image.
*   **Measured Performance:** Performance is highly dependent on the number of visible Gaussians and the resolution. The CUDA implementation is optimized for parallel processing on GPUs.
*   **Strengths:** Differentiable, fast rasterization suitable for real-time training and rendering. Supports both precomputed covariances and scale/rotation parameterizations.
*   **Limitations:** Requires careful handling of input consistency (covariance vs. scale/rotation). Debug mode can significantly impact performance due to CPU transfers.
*   **When to Use:** When training 3D Gaussian Splatting models or rendering 3D Gaussians in a differentiable pipeline.
*   **When Not to Use:** When non-differentiable rendering is sufficient and performance is critical without the overhead of gradient tracking.

## Background
The method is part of the 3D Gaussian Splatting framework, which uses differentiable rasterization to render 3D Gaussians. It builds on [GaussianRasterizationSettings](2924-gaussianrasterizationsettings.md) and is a core component of the [Gaussian Splatting](link) rendering pipeline. The specific implementation details are derived from unofficial repositories [S1, S2, unofficial].

## Sources
[S1] Ingyu-Jeong/EGGS, https://github.com/Ingyu-Jeong/EGGS, fetched 2026-10-26 (unofficial)
[S2] Ruyi-Zha/r2_gaussian, https://github.com/Ruyi-Zha/r2_gaussian, fetched 2026-10-26 (unofficial)
