# Gaussian Rasterization (Forward)

Also called: CudaRasterizer::Rasterizer::forward, Differentiable Gaussian Rasterization Forward Pass, Forward Pass of Gaussian Rasterizer, GaussianRasterizer forward pass, GaussianRasterizer.forward

## Overview
A differentiable rasterization method that projects 3D Gaussians onto a 2D image plane to produce a rendered image and per-Gaussian screen-space radii, supporting both precomputed 3D covariances and scale/rotation parameterizations [S2, unofficial]. It differs from standard rasterization by handling continuous volumetric primitives via alpha blending, enabling gradient flow through the rendering process for optimization [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `means3D`: $(N, 3)$ tensor of 3D Gaussian centers.
    *   `colors_precomp`: $(N, 3)$ or $(N, C)$ tensor of precomputed colors.
    *   `opacities`: $(N,)$ tensor of per-Gaussian opacities.
    *   `scales`: $(N, 3)$ tensor of log-scale factors (optional if `cov3D_precomp` is provided).
    *   `rotations`: $(N, 4)$ tensor of unit quaternions (optional if `cov3D_precomp` is provided).
    *   `cov3D_precomp`: $(N, 6)$ tensor of precomputed 3D covariance matrix elements (optional if `scales`/`rotations` are provided) [S2, unofficial].
    *   `raster_settings`: A `GaussianRasterizationSettings` object containing camera and rendering parameters.
    *   `sh`: Spherical Harmonics coefficients (optional, used if `colors_precomp` is not provided).
    *   `sh_degree`: Degree of SH.
*   **Outputs:**
    *   `color`: $(H, W, 3)$ or $(H, W, C)$ rendered image tensor.
    *   `radii`: $(N,)$ tensor of screen-space radii for visible Gaussians (used for backward pass and culling) [S1, unofficial].

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
    *   `debug`: bool [S2, unofficial]
*   `GeometryState`: Contains `depths` (float, size $P$), `clamped` (float, size $P \times 3$), `internal_radii` (float, size $P$), `means2D` (float2, size $P$), `cov3D` (float, size $P \times 6$), `conic_opacity` (float4, size $P$), `rgb` (float, size $P \times 3$), `semantic_feature` (float, size $P \times \text{NUM\_SEMANTIC\_CHANNELS}$), `tiles_touched` (uint32_t, size $P$), `scanning_space` (temporary for scan), `point_offsets` (uint32_t, size $P$) [S3, unofficial].
*   `ImageState`: Contains `accum_alpha` (float, size $N=H \times W$), `n_contrib` (uint32_t, size $N$), `ranges` (uint2, size $N$) [S3, unofficial].
*   `BinningState`: Contains `point_list` (uint32_t, size $P$), `point_list_unsorted` (uint32_t, size $P$), `point_list_keys` (uint32_t, size $P$), `point_list_keys_unsorted` (uint32_t, size $P$), `list_sorting_space` (temporary for sort) [S3, unofficial].

**Steps:**
1.  **Input Validation and Initialization:** Check consistency and raise an exception if exactly one of the following pairs is missing: (`scales`, `rotations`) or (`cov3D_precomp`) [S2, unofficial]. Initialize missing optional inputs (`shs`, `colors_precomp`, `scales`, `rotations`, `cov3D_precomp`) to empty PyTorch tensors if they are `None` [S3, unofficial].
2.  **Debug Mode Handling:** If `raster_settings.debug` is `True`, deep-copy all arguments to the CPU before calling the C++ routine to allow saving a snapshot for debugging if an exception occurs [S1, unofficial].
3.  **Backend Invocation:** Invoke the C++/CUDA rasterization routine `_C.rasterize_gaussians` with the restructured arguments [S1, unofficial].
4.  **Gradient Saving:** Save relevant tensors for the backward pass using `ctx.save_for_backward`: `colors_precomp`, `means3D`, `scales`, `rotations`, `cov3Ds_precomp`, `radii`, `sh`, `geomBuffer`, `binningBuffer`, `imgBuffer` [S1, unofficial].
5.  **Return:** Return the rendered `color` image and `radii` of visible Gaussians [S1, unofficial].
6.  **Internal CUDA Pipeline (executed inside `_C.rasterize_gaussians`):**
    *   Allocate `GeometryState`, `ImageState`, and `BinningState` buffers via callbacks [S3, unofficial].
    *   Compute tile grid dimensions: `tile_grid.x = (width + BLOCK_X - 1) / BLOCK_X`, `tile_grid.y = (height + BLOCK_Y - 1) / BLOCK_Y` [S3, unofficial].
    *   Call `FORWARD::preprocess` to transform Gaussians, compute 2D means, depths, covariances, colors, conic matrices, and radii [S3, unofficial].
    *   Compute inclusive prefix sum over `tiles_touched` to get `point_offsets` using `geomState.scanning_space` [S3, unofficial].
    *   Retrieve `num_rendered` from the last element of `point_offsets` (`point_offsets[P-1]`) [S3, unofficial].
    *   Allocate `BinningState` buffers with size `num_rendered` [S3, unofficial].
    *   Launch `duplicateWithKeys` kernel to create sorting keys and duplicated Gaussian indices from `means2D`, `depths`, `point_offsets`, `radii`, and `tile_grid` [S3, unofficial].
    *   Calculate bit depth for radix sort: `bit = getHigherMsb(tile_grid.x * tile_grid.y)` [S3, unofficial].
    *   Sort pairs using `cub::DeviceRadixSort::SortPairs` over range `0` to `32+bit` [S3, unofficial].
    *   Zero out `imgState.ranges` array [S3, unofficial].
    *   If `num_rendered > 0`, launch `identifyTileRanges` kernel to compute start/end indices in the sorted `point_list` for each tile [S3, unofficial].
    *   Determine feature pointer: use `colors_precomp` if not null, otherwise use `geomState.rgb` [S3, unofficial].
    *   Launch `FORWARD::render` kernel to compute `out_color` and `out_feature_map` [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
from typing import NamedTuple, Optional, Tuple

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
) -> Tuple[torch.Tensor, torch.Tensor]:
    # 1. Input Validation & Initialization
    scales_provided = scales is not None and scales.numel() > 0
    rotations_provided = rotations is not None and rotations.numel() > 0
    cov_provided = cov3D_precomp is not None and cov3D_precomp.numel() > 0

    if (scales_provided and rotations_provided) == cov_provided:
        raise ValueError("Exactly one of (scales, rotations) or (cov3D_precomp) must be provided.")

    if scales is None or scales.numel() == 0:
        scales = torch.Tensor([])
    if rotations is None or rotations.numel() == 0:
        rotations = torch.Tensor([])
    if cov3D_precomp is None or cov3D_precomp.numel() == 0:
        cov3D_precomp = torch.Tensor([])

    # 2. Debug Mode Handling
    if raster_settings.debug:
        means3D = means3D.clone().cpu()
        colors_precomp = colors_precomp.clone().cpu() if colors_precomp is not None else None
        opacities = opacities.clone().cpu()
        scales = scales.clone().cpu()
        rotations = rotations.clone().cpu()
        cov3D_precomp = cov3D_precomp.clone().cpu()
        sh = sh.clone().cpu() if sh is not None else None

    # 3. Gradient Saving
    # ctx.save_for_backward(colors_precomp, means3D, scales, rotations, cov3D_precomp, radii, sh, geomBuffer, binningBuffer, imgBuffer)

    # 4. Backend Invocation
    bg = background if background is not None else torch.zeros(3, device=means3D.device)
    sm = scale_modifier if scale_modifier is not None else raster_settings.scale_modifier

    # The C++/CUDA routine handles the full pipeline:
    color, radii, geomBuffer, binningBuffer, imgBuffer = _C.rasterize_gaussians(
        bg, means3D, colors_precomp, opacities, scales, rotations, sm,
        cov3D_precomp, raster_settings.viewmatrix, raster_settings.projmatrix,
        raster_settings.tanfovx, raster_settings.tanfovy,
        raster_settings.image_height, raster_settings.image_width,
        sh, sh_degree, raster_settings.campos, raster_settings.prefiltered,
        raster_settings.debug
    )

    return color, radii
```

**Parameters:**
*   `tanfovx`: Tangent of half horizontal field of view. Typical value: `math.tan(viewpoint_camera.FoVx * 0.5)` [S2, unofficial].
*   `tanfovy`: Tangent of half vertical field of view. Typical value: `math.tan(viewpoint_camera.FoVy * 0.5)` [S2, unofficial].
*   `mode`: Integer flag controlling FOV calculation. Mode 0 sets `tanfovx`/`tanfovy` to 1.0; Mode 1 calculates them from FoV [S2, unofficial].
*   `width`, `height`: Image dimensions in pixels [S3, unofficial].
*   `tan_fovx`, `tan_fovy`: Tangent of horizontal and vertical field of view angles [S3, unofficial].
*   Focal lengths are derived as `focal_y = height / (2.0f * tan_fovy)` and `focal_x = width / (2.0f * tan_fovx)` [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Ensure `scales` and `rotations` are properly normalized if used. Quaternions must be unit length [G].
*   **Boundaries:** If `raster_settings.debug` is true, arguments are deep-copied to CPU before calling the C++ routine to allow saving a snapshot for debugging if an exception occurs [S1, unofficial].
*   **Degenerate Input:** If `scales` or `rotations` are `None`, they must be initialized as empty tensors (`torch.Tensor([])`) before passing to the C++ backend to avoid type errors [S2, unofficial].
*   **Channel Mismatch:** If `NUM_CHANNELS != 3` and `colors_precomp` is null, throw a runtime error requiring precomputed colors [S3, unofficial].

## Subtopics

### Gaussian Rasterization Configuration Setup
Defines the `GaussianRasterizationSettings` structure and parameter validation. [Gaussian Rasterization Configuration Setup](10752-gaussian-rasterization-configuration-setup.md)

### GSCache DiffRasterizer Interface
Provides an interface for caching and differentiable rasterization in specific implementations. [GSCache DiffRasterizer Interface](18374-gscache-diffrasterizer-interface.md)

### 3D Gaussian Splatting with Feature Splatting
Extends the forward pass to render additional feature channels alongside color. [3D Gaussian Splatting with Feature Splatting](19779-3d-gaussian-splatting-with-feature-splatting.md)

### DOFGS Rendering Pipeline
Adapts the rasterization steps for depth-of-field Gaussian splatting. [DOFGS Rendering Pipeline](21470-dofgs-rendering-pipeline.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Differentiable rasterization suitable for training and rendering pipelines. Supports both precomputed covariances and scale/rotation parameterizations [S2, unofficial].
*   **Limitations:** Requires careful handling of input consistency (covariance vs. scale/rotation). Debug mode impacts performance due to CPU transfers [S1, unofficial].
*   **When to Use:** When training 3D Gaussian Splatting models or rendering 3D Gaussians in a differentiable pipeline [G].
*   **When Not to Use:** When non-differentiable rendering is sufficient and performance is critical without the overhead of gradient tracking [G].

## Background
The method is part of the 3D Gaussian Splatting framework, which uses differentiable rasterization to render 3D Gaussians [S2, unofficial]. It builds on [GaussianRasterizationSettings](2924-gaussianrasterizationsettings.md) and is a core component of the [Gaussian Splatting](link) rendering pipeline [S1, unofficial].

## Sources
[S1] Ingyu-Jeong/EGGS, https://github.com/Ingyu-Jeong/EGGS, fetched 2026-10-26 (unofficial)
[S2] Ruyi-Zha/r2_gaussian, https://github.com/Ruyi-Zha/r2_gaussian, fetched 2026-10-26 (unofficial)
[S3] Mercerai/PFGS, https://github.com/Mercerai/PFGS, fetched 2026-10-28 (unofficial)
