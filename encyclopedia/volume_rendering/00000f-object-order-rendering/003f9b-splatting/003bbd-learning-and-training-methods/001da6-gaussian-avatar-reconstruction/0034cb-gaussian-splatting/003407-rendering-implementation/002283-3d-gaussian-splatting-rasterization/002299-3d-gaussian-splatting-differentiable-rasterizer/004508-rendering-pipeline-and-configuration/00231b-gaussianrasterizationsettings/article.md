# GaussianRasterizationSettings

Also called: 4D Gaussian Splatting Rasterization, GaussianRasterizationSettings configuration, GaussianRasterizer

## Overview
`GaussianRasterizationSettings` is a configuration structure (typically a `NamedTuple` or struct) that defines the camera parameters, background, and rendering flags required by the differentiable 3D Gaussian splatting rasterizer. It serves as the primary input container for the rasterization kernel, bridging the gap between the scene representation (Gaussians) and the camera projection. It is distinct from the rasterization kernel itself; it sets up the state that the kernel consumes [S2, S3, unofficial]. It enables the rasterizer to handle both static 3D Gaussians and dynamic 4D Gaussians (time-varying position, scale, rotation, and opacity) by including temporal parameters [S2, unofficial].

## How it works
The settings object is instantiated with specific camera and rendering parameters. Before being passed to the CUDA rasterization kernel (`_C.rasterize_gaussians`), the settings and associated Gaussian tensors undergo strict validation and normalization to ensure device consistency and mutual exclusivity of representation formats [Parent, S2].

### Inputs and Outputs
*   **Inputs**: Camera matrices (`viewmatrix`, `projmatrix`), field of view (`tanfovx`, `tanfovy`), image dimensions (`image_height`, `image_width`), background color tensor (`bg`), and optional flags for debugging, SH degree, and 4D temporal support.
*   **Outputs**: The settings object itself is not an output; it is consumed by the rasterizer to produce rendered images, gradients, and auxiliary buffers (radii, depth, flow).

### Data Structures and Layout
The structure varies slightly between implementations (static 3D-GS vs. dynamic 4D-GS), but shares core fields.

**Standard 3D-GS Layout [S1, S3, unofficial]:**
*   `image_height`: `int`
*   `image_width`: `int`
*   `tanfovx`: `float` (Tangent of half horizontal FOV)
*   `tanfovy`: `float` (Tangent of half vertical FOV)
*   `cx`: `float` (Principal point x, optional in some variants)
*   `cy`: `float` (Principal point y, optional in some variants)
*   `bg`: `torch.Tensor` (Background color, shape `(3,)` or `(4,)`)
*   `scale_modifier`: `float` (Multiplicative scalar for Gaussian scales)
*   `viewmatrix`: `torch.Tensor` (4x4 view matrix)
*   `projmatrix`: `torch.Tensor` (4x4 projection matrix)
*   `sh_degree`: `int` (Spherical Harmonics degree)
*   `campos`: `torch.Tensor` (Camera position)
*   `prefiltered`: `bool` (Whether SH coefficients are prefiltered)
*   `backward_geometry`: `bool` (Enable geometry gradients)
*   `computer_pseudo_normal`: `bool` (Enable pseudo-normal computation)
*   `debug`: `bool` (Enable debug mode)

**4D-GS Layout [S2, unofficial]:**
Includes all 3D fields plus:
*   `sh_degree_t`: `int` (Temporal SH degree)
*   `timestamp`: `float` (Current time step)
*   `time_duration`: `float` (Total time duration)
*   `rot_4d`: `bool` (Enable 4D rotation)
*   `gaussian_dim`: `int` (Dimensionality of Gaussian parameters)
*   `force_sh_3d`: `bool` (Force 3D SH even in 4D mode)

### Numbered Steps for Configuration and Validation

1.  **Initialize Settings**: Create the `GaussianRasterizationSettings` object with the required camera and rendering parameters.
2.  **Normalize None Inputs**: Convert any input tensor that is `None` to an empty `torch.Tensor([])` or a tensor of shape $(N, 0)$ to prevent `None` propagation into the CUDA kernel [Parent, S2].
    *   If `shs` is `None`, set `shs = torch.Tensor([])`.
    *   If `colors_precomp` is `None`, set `colors_precomp = torch.Tensor([])`.
    *   If `flow_2d` is `None`, set `flow_2d = torch.Tensor([])`.
    *   If `ts` is `None`, set `ts = torch.Tensor([])`.
    *   If `scales` is `None`, set `scales = torch.Tensor([])`.
    *   If `scales_t` is `None`, set `scales_t = torch.Tensor([])`.
    *   If `rotations` is `None`, set `rotations = torch.Tensor([])`.
    *   If `rotations_r` is `None`, set `rotations_r = torch.Tensor([])`.
    *   If `cov3D_precomp` is `None`, set `cov3D_precomp = torch.Tensor([])`.
3.  **Validate Color Representation**:
    *   If `shs` is `None` AND `colors_precomp` is `None`, raise Exception: "Please provide exactly one of either SHs or precomputed colors!".
    *   If `shs` is not `None` AND `colors_precomp` is not `None`, raise Exception: "Please provide exactly one of either SHs or precomputed colors!".
4.  **Validate Geometry Representation**:
    *   If `(scales is None OR rotations is None)` AND `cov3D_precomp` is `None`, raise Exception: "Please provide exactly one of either scale/rotation pair or precomputed 3D covariance!".
    *   If `(scales is not None OR rotations is not None)` AND `cov3D_precomp` is not `None`, raise Exception: "Please provide exactly one of either scale/rotation pair or precomputed 3D covariance!".
5.  **Validate 4D Geometry (if applicable)**:
    *   If `rot_4d` is `True` AND `cov3D_precomp` is `None`, then `rotations_r`, `scales_t`, and `ts` must all be non-`None`. Otherwise, raise Exception: "Please provide exactly rotations_r and scales_t and ts if rot_4d and cov3D_precomp is None!".
6.  **Ensure Device Consistency**: Ensure all input tensors (`means3D`, `colors_precomp`, `opacities`, `scales`, `rotations`, `shs`, `bg`, `viewmatrix`, `projmatrix`, etc.) reside on the same device (typically CUDA) [Parent].

### Pseudocode for Settings Construction and Validation

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
from typing import NamedTuple, Optional

class GaussianRasterizationSettings(NamedTuple):
    image_height: int
    image_width: int
    tanfovx: float
    tanfovy: float
    bg: torch.Tensor
    scale_modifier: float
    viewmatrix: torch.Tensor
    projmatrix: torch.Tensor
    sh_degree: int
    campos: torch.Tensor
    prefiltered: bool
    debug: bool = False
    # 4D specific fields (optional, default to None/0/False)
    sh_degree_t: int = 0
    timestamp: float = 0.0
    time_duration: float = 1.0
    rot_4d: bool = False
    gaussian_dim: int = 3
    force_sh_3d: bool = False
    backward_geometry: bool = False
    computer_pseudo_normal: bool = False

def validate_and_construct_settings(
    raster_settings: GaussianRasterizationSettings,
    means3D: torch.Tensor,
    colors_precomp: Optional[torch.Tensor],
    shs: Optional[torch.Tensor],
    opacities: torch.Tensor,
    scales: Optional[torch.Tensor],
    rotations: Optional[torch.Tensor],
    cov3D_precomp: Optional[torch.Tensor],
    flow_2d: Optional[torch.Tensor] = None,
    ts: Optional[torch.Tensor] = None,
    scales_t: Optional[torch.Tensor] = None,
    rotations_r: Optional[torch.Tensor] = None
) -> tuple:
    """
    Validates inputs and normalizes None tensors for the Gaussian rasterizer.
    Returns the validated settings and normalized tensors.
    """
    # Step 2: Normalize None Inputs
    if colors_precomp is None:
        colors_precomp = torch.Tensor([])
    if shs is None:
        shs = torch.Tensor([])
    if flow_2d is None:
        flow_2d = torch.Tensor([])
    if ts is None:
        ts = torch.Tensor([])
    if scales is None:
        scales = torch.Tensor([])
    if scales_t is None:
        scales_t = torch.Tensor([])
    if rotations is None:
        rotations = torch.Tensor([])
    if rotations_r is None:
        rotations_r = torch.Tensor([])
    if cov3D_precomp is None:
        cov3D_precomp = torch.Tensor([])

    # Step 3: Validate Color Representation
    has_sh = shs.numel() > 0
    has_colors = colors_precomp.numel() > 0
    if not has_sh and not has_colors:
        raise Exception("Please provide exactly one of either SHs or precomputed colors!")
    if has_sh and has_colors:
        raise Exception("Please provide exactly one of either SHs or precomputed colors!")

    # Step 4: Validate Geometry Representation
    has_scale_rot = scales.numel() > 0 and rotations.numel() > 0
    has_cov = cov3D_precomp.numel() > 0
    
    if (not has_scale_rot) and not has_cov:
        raise Exception("Please provide exactly one of either scale/rotation pair or precomputed 3D covariance!")
    if has_scale_rot and has_cov:
        raise Exception("Please provide exactly one of either scale/rotation pair or precomputed 3D covariance!")

    # Step 5: Validate 4D Geometry
    if raster_settings.rot_4d and not has_cov:
        if ts.numel() == 0 or scales_t.numel() == 0 or rotations_r.numel() == 0:
            raise Exception("Please provide exactly rotations_r and scales_t and ts if rot_4d and cov3D_precomp is None!")

    # Step 6: Device Consistency (Implicitly handled by caller, but checked here for safety)
    # Ensure all tensors are on the same device as raster_settings.bg
    device = raster_settings.bg.device
    tensors_to_check = [means3D, opacities, scales, rotations, cov3D_precomp, shs, colors_precomp, flow_2d, ts, scales_t, rotations_r, raster_settings.viewmatrix, raster_settings.projmatrix, raster_settings.campos]
    for t in tensors_to_check:
        if t.numel() > 0 and t.device != device:
            raise Exception(f"Tensor {t} is not on device {device}")

    return raster_settings, means3D, colors_precomp, shs, opacities, scales, rotations, cov3D_precomp, flow_2d, ts, scales_t, rotations_r
```

### Parameters with Typical Values
*   `tanfovx`/`tanfovy`: Computed as `math.tan(camera.FoVx * 0.5)` and `math.tan(camera.FoVy * 0.5)` [S1, unofficial].
*   `scale_modifier`: Default value is `1.0` [S1, unofficial].
*   `cx`/`cy`: Principal point coordinates in pixels, extracted from `intrinsic[0, 2]` and `intrinsic[1, 2]` [S1, unofficial].

### Edge Cases and Pitfalls
*   **Precision**: Floating-point precision issues may arise in the CUDA kernel if inputs are not normalized. Ensure `bg` tensor is `float32` [S1, unofficial].
*   **Boundaries**: The rasterizer assumes valid camera matrices. Invalid `viewmatrix` or `projmatrix` will cause undefined behavior or crashes [Parent].
*   **Degenerate Input**: Providing both `shs` and `colors_precomp` or neither will raise an exception [S2, unofficial]. Similarly, providing both `scales`/`rotations` and `cov3D_precomp` will raise an exception [S2, unofficial].

## Subtopics

### 3D Half-Gaussian Rasterization
A rasterization technique that represents a 3D Gaussian kernel as two halves split by a plane through its center, allowing distinct opacity values for each half to better represent sharp edges and planar surfaces [S4]. It uses the three unused normal parameters in the original 3D-GS kernel to store the splitting plane normal $n = [n_1, n_2, n_3]^T$ [S4]. Opacity $\alpha_1$ is for the half where $n^T(x-\mu) \ge 0$, and $\alpha_2$ is for the half where $n^T(x-\mu) < 0$ [S4]. This increases memory by only one additional parameter per Gaussian compared to 3D-GS [S4].

### Restricting Gaussian Kernel Support
A technique to reduce Gaussian footprints and superfluous ray-Gaussian intersections using a generalized kernel exponent and adaptive alpha clamping [S5]. The generalized Gaussian kernel is defined as:
$$ \exp \left[ -\frac{1}{2n_{\text{expo}}} ((x - \mu)^T \Sigma^{-1} (x - \mu))^{n_{\text{expo}}} \right] $$
where $n_{\text{expo}}$ is the exponent controlling bulkiness [S5]. 3DGRT uses $4.5 n_{\text{expo}}$ in the denominator, while GRay uses $1 2^{n_{\text{expo}}}$ to make initialization less sensitive and result in smaller initial Gaussians [S5]. Dynamic clamping can cause visible artifacts for large Gaussians, but these are less apparent when using DI since most Gaussians stay small [S5].

## Operating characteristics
*   **Time Complexity**: The rasterization kernel is $O(N \cdot M)$ where $N$ is the number of Gaussians and $M$ is the number of pixels, but optimized with visibility culling and binning to be much faster in practice [S2, unofficial].
*   **Memory Complexity**: $O(N)$ for storing Gaussian parameters and $O(M)$ for the output image and auxiliary buffers [S2, unofficial].
*   **Performance**: Real-time performance is achievable on modern GPUs (e.g., NVIDIA RTX 4090) for scenes with millions of Gaussians [S5].
*   **Strengths**: Differentiable, real-time rendering, supports dynamic scenes (4D-GS) [S2, unofficial].
*   **Limitations**: High memory usage for large scenes, sensitive to hyperparameters like $n_{\text{expo}}$ and $\alpha_{\text{min}}$ [S5].

## Background
`GaussianRasterizationSettings` originates from the 3D Gaussian Splatting paper [S1, unofficial] and was extended for 4D Gaussian Splatting to handle dynamic scenes [S2, unofficial]. It is a core component of the differentiable rasterization pipeline used in training and rendering Gaussian splatting models.

## Sources
[S1] ldyken53/VEG, https://github.com/ldyken53/VEG, fetched 2026-10-26 (unofficial)
[S2] LogosIntelligenceLab/4d-gaussian-splatting, https://github.com/LogosIntelligenceLab/4d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S3] SorenZhang574/Laplacian-GS, https://github.com/SorenZhang574/Laplacian-GS, fetched 2026-10-27 (unofficial)
[S4] 3D-HGS: 3D Half-Gaussian Splatting*, https://doi.org/10.1109/cvpr52734.2025.01027, fetched 2026-10-30
[S5] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
